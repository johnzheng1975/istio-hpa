# istio-hpa

### Prepare
![Istio HPA](https://raw.githubusercontent.com/stefanprodan/istio-hpa/master/diagrams/istio-hpa-overview.png)

#### Some Reference
- https://github.com/prometheus-community/helm-charts/tree/main/charts
- https://github.com/kubernetes-sigs/prometheus-adapter/blob/master/docs/walkthrough.md

#### Installation
```
# Quick Install
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install prometheus  prometheus-community/prometheus
helm install prometheus-adapter prometheus-community/prometheus-adapter  -set prometheus.url=http://prometheus-server.default.svc.cluster.local    --set prometheus.port=80
helm ls -A

(trycluster02):  johnzhengaz@HP-CND3051QF51:~/tmp/Feb5/kubeadm-workshop/demos/monitoring$ helm ls -A
NAME                    NAMESPACE       REVISION        UPDATED              STATUS   CHART                      APP VERSION
                                                                                                       
prometheus              default         1               2025-02-05 17:51:22  deployed prometheus-27.3.0          3.1.0
prometheus-adapter      default         1               2025-02-05 17:58:01  deployed prometheus-adapter-4.11.0  0.12.0 

# Found the issue, and improve
$ k get pods -n default           prometheus-adapter-7b69fc9f66-72m47 -oyaml
  containers:
  - args:
    - --prometheus-url=http://prometheus.default.svc:9090

helm upgrade prometheus-adapter prometheus-community/prometheus-adapter	 --set prometheus.url=http://prometheus-server.default.svc.cluster.local    --set prometheus.port=80

helm install prometheus prometheus-community/prometheus \
  --namespace default \
  --version 25.30.0 \ 
  --set server.persistentVolume.size=8Gi \
  --set alertmanager.persistentVolume.size=2Gi \
  --wait --timeout 10m0s \
  --values prometheus-cm.yaml
  
# Dependency install, otherwise pv not be created.
https://github.com/kubernetes-sigs/aws-ebs-csi-driver
https://github.com/kubernetes-sigs/aws-ebs-csi-driver/blob/master/docs/example-iam-policy.json
  helm repo add aws-ebs-csi-driver https://kubernetes-sigs.github.io/aws-ebs-csi-driver
  helm repo update
  helm upgrade --install aws-ebs-csi-driver \
    --namespace kube-system \
    aws-ebs-csi-driver/aws-ebs-csi-driver
```


### Important configure for improve.
- Replace current adapter configmap with below, and restart pods for adapter.
```
# kubectl  get cm prometheus-adapter -oyaml
apiVersion: v1
data:
  config.yaml: |
    rules:
    - metricsQuery: 'sum(rate(<<.Series>>{<<.LabelMatchers>>}[2m])) by (<<.GroupBy>>)'
      name:
        as: "${1}_per_second"
        matches: "^(.*)_total"
      resources:
        overrides:
          namespace:
            resource: namespace
          pod:
            resource: pod
      seriesQuery: istio_requests_total{pod!="", namespace!=""}
kind: ConfigMap
metadata:
  name: prometheus-adapter
  namespace: default
```

- This will be create later
```
# kubectl  get hpa -n test podinfo -oyaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: podinfo
  namespace: test
spec:
  maxReplicas: 20
  metrics:
  - pods:
      metric:
        name: istio_requests  #should be istio_requests_per_second, otherwise, hpa will show unknown
      target:
        averageValue: 500m
        type: Value
    type: Pods
  minReplicas: 1
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: podinfo
```
 

### Installing the demo app
 
First create a `test` namespace with Istio sidecar injection enabled:

```bash
kubectl apply -f ./namespaces/
```

Create the podinfo deployment and ClusterIP service in the `test` namespace:

```bash
kubectl apply -f ./podinfo/deployment.yaml,./podinfo/service.yaml
```

In order to trigger the auto scaling, you'll need a tool to generate traffic.
Deploy the load test service in the `test` namespace:

```bash
kubectl apply -f ./loadtester/
```

Verify the install by calling the podinfo API.
Exec into the load tester pod and use `hey` to generate load for a couple of seconds:

```bash
export loadtester=$(kubectl -n test get pod -l "app=loadtester" -o jsonpath='{.items[0].metadata.name}')
kubectl -n test exec -it ${loadtester} -- sh

~ $ hey -z 5s -c 10 -q 2 http://podinfo.test:9898

Summary:
  Total:	5.0138 secs
  Requests/sec:	19.9451

Status code distribution:
  [200]	100 responses

  $ exit
```

The podinfo [ClusterIP service](https://github.com/stefanprodan/istio-hpa/blob/master/podinfo/service.yaml)
exposes port 9898 under the `http` name. When using the http prefix, the Envoy sidecar will
switch to L7 routing and the telemetry service will collect HTTP metrics.

### Querying the Istio metrics

The Istio telemetry service collects metrics from the mesh and stores them in Prometheus. One such metric is
`istio_requests_total`, with it you can determine the rate of requests per second a workload receives.

This is how you can query Prometheus for the req/sec rate received by podinfo in the last two minute:

```sql
   sum(rate(istio_requests_total{namespace="test",pod=~".*"}[2m])) by (namespace, pod)
```


### Configuring the HPA with Istio metrics

Using the req/sec query you can define a HPA that will scale the podinfo workload based on the number of requests
per second that each instance receives:

```yaml
# kubectl  get hpa -n test podinfo -oyaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: podinfo
  namespace: test
spec:
  maxReplicas: 20
  metrics:
  - pods:
      metric:
        name: istio_requests
      target:
        averageValue: 500m
        type: Value
    type: Pods
  minReplicas: 1
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: podinfo
```

The above configuration will instruct the Horizontal Pod Autoscaler to scale up the deployment when the average traffic
load goes over 10 req/sec per replica.

Create the HPA with:

```bash
kubectl apply -f ./podinfo/hpa-padapter.yaml
```

Start a load test and verify that the adapter computes the metric:

```
kubectl -n kube-system logs deployment/kube-metrics-adapter -f

Collected 1 new metric(s)
Collected new custom metric 'istio-requests-total' (44m) for Pod test/podinfo
```

List the custom metrics resources:

```bash
kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta1" | jq .
```

The Kubernetes API should return a resource list containing the Istio metric:

```json
{
  "kind": "APIResourceList",
  "apiVersion": "v1",
  "groupVersion": "custom.metrics.k8s.io/v1beta1",
  "resources": [
    #{
    #  "name": "pods/istio-requests-total",
    #  "singularName": "",
    #  "namespaced": true,
    #  "kind": "MetricValueList",
    #  "verbs": [
    #    "get"
    #  ]
    #}
	
    {
      "name": "pods/istio_requests",
      "singularName": "",
      "namespaced": true,
      "kind": "MetricValueList",
      "verbs": [
        "get"
      ]
    },
  ]
}
```

After a couple of seconds the HPA will fetch the metric from the adapter:

```bash
# kubectl  get hpa -n test
NAME      REFERENCE            TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
podinfo   Deployment/podinfo   0/500m    1         20        1          14h
```

### Autoscaling based on HTTP traffic

To test the HPA you can use the load tester to trigger a scale up event.

Exec into the tester pod and use `hey` to generate load for a 5 minutes:

```bash
kubectl -n test exec -it ${loadtester} -- sh

~ $  hey -z 500s -c 1 -q 2 http://podinfo.test:9898
```
Press ctrl+c then exit to get out of load test terminal if you wanna stop prematurely.

Prepare Prometheus:
# kubectl  port-forward svc/prometheus-server -n default 9090:80

```
# Prometheus query: sum(rate(istio_requests_total{namespace="test",pod=~".*"}[2m])) by (namespace, pod)
http://localhost:9090/graph?g0.expr=sum(rate(istio_requests_total%7Bnamespace%3D%22test%22%2Cpod%3D~%22.*%22%7D%5B2m%5D))%20by%20(namespace%2C%20pod)&g0.tab=1&g0.display_mode=lines&g0.show_exemplars=0&g0.range_input=1h
{namespace="test", pod="loadtester-6b7d9d45c8-ktkft"} 2
{namespace="test", pod="podinfo-74b6cdf4d-px56x"}     2

```

##### After minutes the HPA will start to scale up 


```bash
# kubectl  get hpa -n test
NAME      REFERENCE            TARGETS     MINPODS   MAXPODS   REPLICAS   AGE
podinfo   Deployment/podinfo   890m/500m   1         20        1 -> 2     14h
```

When the load test finishes, the number of requests per second will drop to zero and the HPA will
start to scale down the workload.
Note that the HPA has a back off mechanism that prevents rapid scale up/down events,
the number of replicas will go back to one after a couple of minutes.

By default the metrics sync happens once every 30 seconds and scaling up/down can only happen if there was
no rescaling within the last 3-5 minutes. In this way, the HPA prevents rapid execution of conflicting decisions
and gives time for the Cluster Autoscaler to kick in.



##### After muinutes, the replicas go upper to 4.  (2 / 0.5 = 4)
- In k8s command:
```
# kubectl  get hpa -n test
NAME      REFERENCE            TARGETS     MINPODS   MAXPODS   REPLICAS   AGE
podinfo   Deployment/podinfo   484m/500m   1         20        4          14h
```

- In Prometheus.

Query with: sum(rate(istio_requests_total{namespace="test",pod=~".*"}[2m])) by (namespace, pod)

Result:
```
{namespace="test", pod="loadtester-6b7d9d45c8-ktkft"}  2
{namespace="test", pod="podinfo-74b6cdf4d-px56x"}      0.5166666666666666
{namespace="test", pod="podinfo-74b6cdf4d-5gkl5"}      0.5
{namespace="test", pod="podinfo-74b6cdf4d-s6625"}      0.5333333333333333
{namespace="test", pod="podinfo-74b6cdf4d-r6zsf"}      0.4333333333333334
```

##### After minutes, 500s is over, the requests go down.

Query with: sum(rate(istio_requests_total{namespace="test",pod=~".*"}[2m])) by (namespace, pod)

Result:
```
{namespace="test", pod="loadtester-6b7d9d45c8-ktkft"}  0
{namespace="test", pod="podinfo-74b6cdf4d-px56x"}      0
{namespace="test", pod="podinfo-74b6cdf4d-5gkl5"}      0
{namespace="test", pod="podinfo-74b6cdf4d-s6625"}      0
{namespace="test", pod="podinfo-74b6cdf4d-r6zsf"}      0
```

```
# kubectl  get hpa -n test
NAME      REFERENCE            TARGETS     MINPODS   MAXPODS   REPLICAS   AGE
podinfo   Deployment/podinfo   159m/500m   1         20        4          14h
```

You can find some deploy to show metrics in hpa, and scale down. This is fine and as design.
 

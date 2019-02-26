# istio-hpa

Autoscaling is an approach to automatically scale up or down workloads based on the resource usage. 
Autoscaling in Kubernetes has two dimensions: the Cluster Autoscaler that deals with node scaling operations 
and the Horizontal Pod Autoscaler that automatically scales the number of pods in a deployment. 

By default the Horizontal Pod Autoscaler (HPA) can scale pods based on observed CPU utilization and memory usage.
Starting with Kubernetes 1.7, an aggregation layer was introduced that allows 3rd party applications to extend the 
Kubernetes API by registering themselves as API add-ons. 
Such an add-on can implement the Custom Metrics API and enable HPA access to arbitrary metrics.

![Istio HPA](https://raw.githubusercontent.com/stefanprodan/istio-hpa/master/diagrams/istio-hpa-overview.png)

What follows is a step-by-step guide on configuring HPA v2 with metrics provided by Istio telemetry service.
When installing Istio make sure that the telemetry service and Prometheus are enabled.

### Installing the custom metrics adapter 

In order to scale based on Istio metrics you need to have two components. 
One component that collects metrics from Istio telemetry service and stores them, 
that's the Prometheus time series database.
And a second component that extends the Kubernetes custom metrics API with the metrics supplied by the collect, 
the Zalando's [kube-metrics-adapter](https://github.com/zalando-incubator/kube-metrics-adapter).

The Zalando adapter is a great alternative to the k8s-prometheus-adapter. Instead of exporting all Prometheus metrics,
kube-metrics-adapter lets you specify a custom promql query. The query is stored as an annotation on the HPA object.
Zalando adapter will scan the HPA objects, execute the promql queries and store the result in-memory.

Clone the [istio-hpa](https://github.com/stefanprodan/istio-hpa) repository:

```bash
git clone https://github.com/stefanprodan/istio-hpa
cd istio-hpa
```

Deploy the metrics adapter in the `kube-system` namespace:

```bash
kubectl apply -f ./kube-metrics-adapter/
```

When the adapter starts, it will generate a self-signed cert and will register itself 
under the `custom.metrics.k8s.io` group.

The adapter is configured to fetch metrics from the Prometheus instance that's running in the `istio-system` namespace.

Verify the install by checking the adapter logs:

```bash
kubectl -n kube-system logs deployment/kube-metrics-adapter
```

### Installing the demo app

You will use a small Golang-based web app to test the Horizontal Pod Autoscaler (HPA).

First create a `test` namespace with Istio sidecar injection enabled:

```bash
kubectl apply -f ./namespaces/
```

Create the [podinfo](https://github.com/stefanprodan/k8s-podinfo) 
deployment and ClusterIP service in the `test` namespace:

```bash
kubectl create -f ./podinfo/deployment.yaml,./podinfo/service.yaml
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
```

Istio records the HTTP traffic metrics 

### Querying the Istio metrics

The Istio telemetry service collects metrics from the Envoy sidecars and stores them in Prometheus. One such metric is 
`istio_requests_total`, with this metric you can determine the rate of requests per second a workload receives.

This is how you can query Prometheus for the average req/sec rate in the last minute processed by podinfo:

```sql
sum(
    rate(
        istio_requests_total{
          destination_workload="podinfo",
          destination_workload_namespace="test",
          reporter="destination"
        }[1m]
    )
)
```

The HPA needs to know the req/sec that each pod receives. You can use the container memory usage metric 
that kubelet exposes to count the number of pods and calculate the Istio requests total per pod:

```sql
sum(
    rate(
        istio_requests_total{
          destination_workload="podinfo",
          destination_workload_namespace="test",
          reporter="destination"
        }[1m]
    )
) /
count(
    count(
        container_memory_usage_bytes{
          namespace="test",
          pod_name=~"podinfo.*"
        }
    ) by (pod_name)
)
```

### Configuring the HPA with Istio metrics

Using the req/sec query you can define a HPA that will scale the podinfo workload based on the number of requests 
per second that each instance receives:

```yaml
apiVersion: autoscaling/v2beta1
kind: HorizontalPodAutoscaler
metadata:
  name: podinfo
  namespace: test
  annotations:
    metric-config.object.istio-requests-total.prometheus/per-replica: "true"
    metric-config.object.istio-requests-total.prometheus/query: |
      sum(
        rate(
          istio_requests_total{
            destination_workload="podinfo",
            destination_workload_namespace="test"
          }[1m]
        )
      ) /
      count(
        count(
          container_memory_usage_bytes{
            namespace="test",
            pod_name=~"podinfo.*"
          }
        ) by (pod_name)
      )
spec:
  maxReplicas: 10
  minReplicas: 1
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: podinfo
  metrics:
    - type: Object
      object:
        metricName: istio-requests-total
        target:
          apiVersion: v1
          kind: Pod
          name: podinfo
        targetValue: 10
```

Create the HPA with:

```bash
kubectl apply -f ./podinfo/hpa.yaml
``` 

Verify that the adapter computes the metric:

```
kubectl -n kube-system logs deployment/kube-metrics-adapter -f

Collected 1 new metric(s)
Collected new custom metric 'istio-requests-total' (44m) for Pod test/podinfo
```

After a couple of seconds the HPA will fetch the metric from the adapter:

```bash
kubectl -n test get hpa/podinfo

NAME      REFERENCE            TARGETS   MINPODS   MAXPODS   REPLICAS
podinfo   Deployment/podinfo   44m/10    1         10        1
```

### Autoscaling based on HTTP traffic

To test the HPA you can use the load tester to trigger a scale up event.

Exec into the tester pod and use `hey` to generate load for a couple of minutes:

```bash
kubectl -n test exec -it ${loadtester} -- sh

~ $ hey -z 5m -c 10 -q 2 http://podinfo.test:9898
```

After a minute the HPA will start to scale up the workload until the req/sec per pod drops under the target value:

```bash
watch kubectl -n test get hpa/podinfo

NAME      REFERENCE            TARGETS     MINPODS   MAXPODS   REPLICAS
podinfo   Deployment/podinfo   25272m/10   1         10        3
```

When the load test finishes, the number of requests per second will drop to zero and the HPA will 
start to scale down the workload.
Note that the HPA has a back off mechanism that prevents rapid scale up/down events, 
the number of replicas will go back to one after a couple of minutes. 

By default the metrics sync happens once every 30 seconds and scaling up/down can only happen if there was 
no rescaling within the last 3-5 minutes. In this way, the HPA prevents rapid execution of conflicting decisions 
and gives time for the Cluster Autoscaler to kick in.

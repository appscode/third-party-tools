# Configuring Prometheus Server to Monitor Kubernetes Resources

Prometheus has native support for monitoring Kubernetes resources. This tutorial will show you how to configure and deploy a Prometheus server in Kubernetes to collect metrics from various Kubernetes resources.

## Before You Begin

At first, you need to have a Kubernetes cluster, and the kubectl command-line tool must be configured to communicate with your cluster. If you do not already have a cluster, you can create one by using [Minikube](https://github.com/kubernetes/minikube).

To keep Prometheus resources isolated, we will use a separate namespace to deploy Prometheus server.

```console
$ kubectl create ns demo
namespace/demo created
```

## Configure Prometheus

Prometheus is configured through a configuration file `prometheus.yaml`. This configuration file describe how Prometheus server should collect metrics from different resources.

In this tutorial, we are going to configure Prometheus to collect metrics from [Pod](https://kubernetes.io/docs/concepts/workloads/pods/pod-overview/), [Service Endpoints](https://kubernetes.io/docs/concepts/services-networking/service/), [Kubernetes API Server](https://kubernetes.io/docs/concepts/overview/components/#kube-apiserver) and [Nodes](https://kubernetes.io/docs/concepts/architecture/nodes/).

A typical configuration file should look like as below,

```yaml
global:
  # specifies configuration such as scrape_interval, evaluation_interval etc
  # that are valid for all configuration context

scrape_configs:
  # specifies the configuration about where and how to collect the metrcis.

rule_files:
  # Rule files specifies a list of globs. Rules and alerts are read from
  # all matching files.

alerting:
  # Alerting specifies settings related to the Alertmanager.

remote_write:
  # Settings related to the remote write feature.

remote_read:
  # Settings related to the remote read feature.
```

For this tutorial purpose, we will only configure `global` and `scrape_config` parts. To know about other configuration parts, please check the Prometheus official configuration guide from [here](https://prometheus.io/docs/prometheus/latest/configuration/configuration).

### `global` configuration

`global` configuration part specifies configuration thats are valid to all other configuration context. If you specify same configuration in local context, it will overwrite the global one. We are going to use following `global` configuration.

```yaml
global:
  scrape_interval: 30s
  scrape_timeout: 10s
```

Here, `scrape_interval: 30s` indicates that Prometheus server should scrape metrics with 30 second interval. `scrape_timeout: 10s` indicates how long until a scrape request times out.

### `scrape_config` configuration

`scrape_config` section specifies the targets of metric collection and how to collect it. It is actually an array of configuration called `job`. Each `job` specify the configuration to collect metrics from a specific resource or specific type of resources. Here, we are going to configure four different jobs `kubernetes-pod`, `kubernetes-service-endpoints`, `kubernetes-apiservers` and `kubernetes-nodes` to collect metrics from Pod, Service Endpoints, Kubernetes API Server and Nodes respectively.

#### `kubernetes-pod`

Here, we are going to configure Prometheus to collect metrics from Kubernetes Pods that have following three annotation,

```yaml
prometheus.io/scrape: true
prometheus.io/path: <metric path>
prometheus.io/port: <port>
```

Here, `prometheus.io/scrape: true` annotation indicate that Prometheus should scrape metrics from this pod. `prometheus.io/port: <port>` and `prometheus.io/path: <metric path>` specifies the port and path where the pod is serving the metrics.

Below is the yaml for a sample pod that exports Prometheus metrics at `/metrics` path of `9091` port.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-monitoring-demo
  namespace: demo
  labels:
    app: prometheus-demo
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "9091"
    prometheus.io/path: "/metrics"
spec:
  containers:
  - name: pushgateway
    image: prom/pushgateway
```

Now, if we want to scrape the metrics from the above pod, we should configure a job under `scrape_config` as below,

```yaml
#------------- configuration to collect pods metrics -------------------
- job_name: 'kubernetes-pods'
  honor_labels: true
  kubernetes_sd_configs:
  - role: pod
  relabel_configs:
  # select only those pods that has "prometheus.io/scrape: true" annotation
  - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
    action: keep
    regex: true
    # set metrics_path (default is /metrics) to the metrics path specified in "prometheus.io/path: <metric path>" annotation.
  - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
    action: replace
    target_label: __metrics_path__
    regex: (.+)
    # set the scrapping port to the port specified in "prometheus.io/port: <port>" annotation and set address accordingly.
  - source_labels: [__address__ __meta_kubernetes_pod_annotation_prometheus_io_port]
    action: replace
    regex: ([^:]+)(?::\d+)?;(\d+)
    replacement: $1:$2
    target_label: __address__
  - action: labelmap
    regex: __meta_kubernetes_pod_label_(.+)
  - source_labels: [__meta_kubernetes_namespace]
    action: replace
    target_label: kubernetes_namespace
  - source_labels: [__meta_kubernetes_pod_name]
    action: replace
    target_label: kubernetes_pod_name
```

Prometheus itself add some labels on collected metrics. If the label is already present in the metric, it cause conflict. In this case, Prometheus rename existing label and add `exported_` prefix to it. Then it add its own label with original name. Here, `honor_labels: true` tells Prometheus to respect existing label in case of any conflict. So, Prometheus will not add it's own label then.

`kubernetes_sd_configs:` tells Prometheus that we want to collect metrics form Kubernetes resource and the resource is pod in this case.

Here, `relabel_config` is used to dynamically configure the target. Prometheus select all pods as possible targets. Here, we are keeping only those pods that has `prometheus.io/scrape: "true"` annotation and dynamically configuring metrics path and port for each pod.

#### `kubernetes-service-endpoints`

#### `kubernetes-apiservers`

#### `kubernetes-nodes`

## Deploy Prometheus Server

**create sample workload:**
```console
$ kubectl apply -f ./monitoring/prometheus/builtin/sample-workloads.yaml
pod/pod-monitoring-demo created
pod/service-endpoint-monitoring-demo created
service/pushgateway-service created
```

**create configMap:**
```console
$ kubectl apply -f ./monitoring/prometheus/builtin/configmap.yaml
configmap/prometheus-config created
```

**create rbac:**
```console
$ kubectl apply -f ./monitoring/prometheus/builtin/rbac.yaml
clusterrole.rbac.authorization.k8s.io/prometheus created
serviceaccount/prometheus created
clusterrolebinding.rbac.authorization.k8s.io/prometheus created
```

**create deployment:**
```console
$ kubectl apply -f ./monitoring/prometheus/builtin/deployment.yaml
deployment.apps/prometheus created
````

**prometheus pod:**
```console
$ kubectl get pod -n demo -l=app=prometheus
NAME                          READY   STATUS    RESTARTS   AGE
prometheus-8568c86d86-vpzx5   1/1     Running   0          102s
```

**portforward:**

```console
$ kubectl port-forward -n demo prometheus-8568c86d86-vpzx5  9090
Forwarding from 127.0.0.1:9090 -> 9090
Forwarding from [::1]:9090 -> 9090
```

## Cleanup

To cleanup the Kubernetes resources created by this tutorial, run:

```console
# delete prometheus resources
$ kubectl delete all -n demo -l=app=prometheus-demo
pod "pod-monitoring-demo" deleted
pod "service-endpoint-monitoring-demo" deleted
service "pushgateway-service" deleted
deployment.apps "prometheus" deleted

# delete rbac stuff
$ kubectl delete clusterrole -l=app=prometheus-demo
clusterrole.rbac.authorization.k8s.io "prometheus" deleted

$ kubectl delete clusterrolebinding -l=app=prometheus-demo
clusterrolebinding.rbac.authorization.k8s.io "prometheus" deleted

$ kubectl delete serviceaccount -n demo -l=app=prometheus-demo
serviceaccount "prometheus" deleted


# delete namespace
$ kubectl delete ns demo
namespace "demo" deleted
```
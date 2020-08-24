# Configure Prometheus Server to Monitor Kubernetes Resources

Prometheus has native support for monitoring Kubernetes resources. This tutorial will show you how to configure and deploy a Prometheus server in Kubernetes to collect metrics from various Kubernetes resources.

## Before You Begin

At first, you need to have a Kubernetes cluster, and the kubectl command-line tool must be configured to communicate with your cluster. If you do not already have a cluster, you can create one by using [Minikube](https://github.com/kubernetes/minikube).

To keep Prometheus resources isolated, we will use a separate namespace `monitoring` to deploy Prometheus server. We will deploy sample workload on another separate namespace called `demo`.

```console
$ kubectl create ns monitoring
namespace/monitoring created

$ kubectl create ns demo
namespace/demo created
```

## Configure Prometheus

Prometheus is configured through a configuration file `prometheus.yaml`. This configuration file describe how Prometheus server should collect metrics from different resources.

In this tutorial, we are going to configure Prometheus to collect metrics from [Pod](https://kubernetes.io/docs/concepts/workloads/pods/pod-overview/), [Service Endpoints](https://kubernetes.io/docs/concepts/services-networking/service/), [Kubernetes API Server](https://kubernetes.io/docs/concepts/overview/components/#kube-apiserver) and [Nodes](https://kubernetes.io/docs/concepts/architecture/nodes/).

A typical configuration file should look like this,

```yaml
global:
  # specifies configuration such as scrape_interval, evaluation_interval etc
  # that are valid for all configuration context

scrape_configs:
  # specifies the configuration about where and how to collect the metrics.

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

For this tutorial, we will only configure `global` and `scrape_config` parts. To know about other configuration parts, please check the Prometheus official configuration guide from [here](https://prometheus.io/docs/prometheus/latest/configuration/configuration).

### `global` configuration

`global` configuration part specifies configuration that are valid to all other configuration context. If you specify same configuration in local context, it will overwrite the global one. We are going to use following `global` configuration.

```yaml
global:
  scrape_interval: 30s
  scrape_timeout: 10s
```

Here, `scrape_interval: 30s` indicates that Prometheus server should scrape metrics with 30 seconds interval. `scrape_timeout: 10s` indicates how long until a scrape request times out.

### `scrape_config` configuration

`scrape_config` section specifies the targets of metric collection and how to collect it. It is actually an array of configuration called `job`. Each `job` specify the configuration to collect metrics from a specific resource or specific type of resources. Here, we are going to configure four different jobs `kubernetes-pod`, `kubernetes-service-endpoints`, `kubernetes-apiservers` and `kubernetes-nodes` to collect metrics from Pod, Service Endpoints, Kubernetes API Server and Nodes respectively.

#### `kubernetes-pod`

Here, we are going to configure Prometheus to collect metrics from Kubernetes Pods that have following three annotation,

```yaml
prometheus.io/scrape: true
prometheus.io/path: <metric path>
prometheus.io/port: <port>
```

Here, `prometheus.io/scrape: true` annotation indicate that Prometheus should scrape metrics from this pod. `prometheus.io/port: <port>` and `prometheus.io/path: <metric path>` specifies the port and path where the pod is serving metrics.

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

Now, if we want to scrape metrics from above pod, we should configure a job under `scrape_config` as below,

```yaml
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

Prometheus itself add some labels on collected metrics. If the label is already present in the metric, it cause conflict. In this case, Prometheus rename existing label and add `exported_` prefix to it. Then it add its own label with original name. Here, `honor_labels: true` tells Prometheus to respect existing label in case of any conflict. So, Prometheus will not add its own label then.

`kubernetes_sd_configs` tells Prometheus that we want to collect metrics form Kubernetes resource and the resource is `pod` in this case.

Here, `relabel_config` is used to dynamically configure the target. Prometheus select all pods as possible targets. Here, we are keeping only those pods that has `prometheus.io/scrape: "true"` annotation and dynamically configuring metrics path, port etc. for each pod.

#### `kubernetes-service-endpoints`

Now, we are going to configure Prometheus to collect metrics from the endpoints of a Service. In this case, we will apply respective annotations in the Service instead of pod that we have done in earlier section.

We are going to collect metrics from below Pod,

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: service-endpoint-monitoring-demo
  namespace: demo
  labels:
    app: prometheus-demo
    pod: prom-pushgateway
spec:
  containers:
  - name: pushgateway
    image: prom/pushgateway
```

We are going to use below Service to collect metrics from that Pod,

```yaml
kind: Service
apiVersion: v1
metadata:
  name:  pushgateway-service
  namespace: demo
  labels:
    app: prometheus-demo
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "9091"
    prometheus.io/path: "/metrics"
spec:
  selector:
    pod: prom-pushgateway
  ports:
  - name:  metrics
    port:  9091
    targetPort:  9091
```

Look at the annotations of this service. This time, we have applied annotations with metrics information in the Service instead of the Pod.

Now, we have to configure a job under `scrape_config` as below to collect metrics using this Service.

```yaml
- job_name: 'kubernetes-service-endpoints'
  honor_labels: true
  kubernetes_sd_configs:
  - role: endpoints
  relabel_configs:
  # select only those endpoints whose service has "prometheus.io/scrape: true" annotation
  - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scrape]
    action: keep
    regex: true
  # set the metrics_path to the path specified in "prometheus.io/path: <metric path>" annotation.
  - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_path]
    action: replace
    target_label: __metrics_path__
    regex: (.+)
  # set the scrapping port to the port specified in "prometheus.io/port: <port>" annotation and set address accordingly.
  - source_labels: [__address__, __meta_kubernetes_service_annotation_prometheus_io_port]
    action: replace
    target_label: __address__
    regex: ([^:]+)(?::\d+)?;(\d+)
    replacement: $1:$2
  - action: labelmap
    regex: __meta_kubernetes_service_label_(.+)
  - source_labels: [__meta_kubernetes_namespace]
    action: replace
    target_label: kubernetes_namespace
  - source_labels: [__meta_kubernetes_service_name]
    action: replace
    target_label: kubernetes_name
```

Here, `role: endpoints` under `kubernetes_sd_configs` field tells Prometheus that the targeted resources are endpoints of a service.

#### `kubernetes-apiservers`

Kubernetes API Server exports metrics in a TLS secure endpoint. So, Prometheus server has to provide certificate to collect these metrics.

We have to configure a job under `scrape_config` as below to collect metrics from the API Server.

```yaml
- job_name: 'kubernetes-apiservers'
  honor_labels: true
  kubernetes_sd_configs:
  - role: endpoints
  # kubernetes apiserver serve metrics on a TLS secure endpoints. so, we have to use "https" scheme
  scheme: https
  # we have to provide certificate to establish tls secure connection
  tls_config:
    ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
  #  bearer_token_file is required for authorizating prometheus server to kubernetes apiserver
  bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token

  relabel_configs:
  - source_labels: [__meta_kubernetes_namespace, __meta_kubernetes_service_name, __meta_kubernetes_endpoint_port_name]
    action: keep
    regex: default;kubernetes;https
```

Look at the `tls_config` field. We are specifying root certificate path in `ca_file` field. Kubernetes automatically mount a secret with respective certificate and [service account token](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#service-account-tokens) in `/var/run/secrets/kubernetes.io/serviceaccount/` directory of Prometheus pod.

We need to authorize Prometheus server to the Kubernetes API Server in order to collect the metrics. So, we are providing service account token through `bearer_token_file` field.

> You can also collect metrics of a [Kubernetes Extension API Server](https://kubernetes.io/docs/tasks/access-kubernetes-api/setup-extension-api-server/) with similar configuration. However, you have to mount a secret with certificate of the Extension API Server to your Prometheus Deployment and you have to points that certificate with `ca_file` field. You will also require to add following in `relabel_config`.

```yaml
- target_label: __address__
  replacement: <extension apiserver address/service>:443
```

#### `kubernetes-nodes`

We can use Kubernets API Server to collect node metrics. The scrapping will be proxied through the API server. This enables Prometheus to collect node metrics without directly connecting to the node. This is particularly helpful when you are running Prometheus outside of the cluster or the nodes are not directly accessible to Prometheus server.

Below yaml show a job under `scrape_config` to collect node metrics.

```yaml
- job_name: 'kubernetes-nodes'
  honor_labels: true
  scheme: https
  tls_config:
    ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
  bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token

  kubernetes_sd_configs:
  - role: node
  relabel_configs:
  - action: labelmap
    regex: __meta_kubernetes_node_label_(.+)
  - target_label: __address__
    replacement: kubernetes.default.svc:443
  - source_labels: [__meta_kubernetes_node_name]
    regex: (.+)
    target_label: __metrics_path__
    replacement: /api/v1/nodes/${1}/proxy/metrics
```

Here, `replacement: /api/v1/nodes/${1}/proxy/metrics` line is responsible for proxy to the respective nodes.

### Rounding up the Configuration

Finally, our final Prometheus configuration file (`prometheus.yaml`) to collect metrics from these four sources should look like this,

```yaml
global:
  scrape_interval: 30s
  scrape_timeout: 10s
scrape_configs:
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
  - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
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

#-------------- configuration to collect metrics from service endpoints -----------------------
- job_name: 'kubernetes-service-endpoints'
  honor_labels: true
  kubernetes_sd_configs:
  - role: endpoints
  relabel_configs:
  # select only those endpoints whose service has "prometheus.io/scrape: true" annotation
  - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scrape]
    action: keep
    regex: true
  # set the metrics_path to the path specified in "prometheus.io/path: <metric path>" annotation.
  - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_path]
    action: replace
    target_label: __metrics_path__
    regex: (.+)
  # set the scrapping port to the port specified in "prometheus.io/port: <port>" annotation and set address accordingly.
  - source_labels: [__address__, __meta_kubernetes_service_annotation_prometheus_io_port]
    action: replace
    target_label: __address__
    regex: ([^:]+)(?::\d+)?;(\d+)
    replacement: $1:$2
  - action: labelmap
    regex: __meta_kubernetes_service_label_(.+)
  - source_labels: [__meta_kubernetes_namespace]
    action: replace
    target_label: kubernetes_namespace
  - source_labels: [__meta_kubernetes_service_name]
    action: replace
    target_label: kubernetes_name

#---------------- configuration to collect metrics from kubernetes apiserver -------------------------
- job_name: 'kubernetes-apiservers'
  honor_labels: true
  kubernetes_sd_configs:
  - role: endpoints
  # kubernetes apiserver serve metrics on a TLS secure endpoints. so, we have to use "https" scheme
  scheme: https
  # we have to provide certificate to establish tls secure connection
  tls_config:
    ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
  #  bearer_token_file is required for authorizating prometheus server to kubernetes apiserver
  bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token

  relabel_configs:
  - source_labels: [__meta_kubernetes_namespace, __meta_kubernetes_service_name, __meta_kubernetes_endpoint_port_name]
    action: keep
    regex: default;kubernetes;https

#--------------- configuration to collect metrics from nodes -----------------------
- job_name: 'kubernetes-nodes'
  honor_labels: true
  scheme: https
  tls_config:
    ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
  bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token

  kubernetes_sd_configs:
  - role: node
  relabel_configs:
  - action: labelmap
    regex: __meta_kubernetes_node_label_(.+)
  - target_label: __address__
    replacement: kubernetes.default.svc:443
  - source_labels: [__meta_kubernetes_node_name]
    regex: (.+)
    target_label: __metrics_path__
    replacement: /api/v1/nodes/${1}/proxy/metrics
```

Now, we can use this configuration file to deploy our Prometheus server.

## Deploy Prometheus Server

As we have configured `prometheus.yaml` to collect metrics from the targets. We are ready to deploy Prometheus Deployment.

**Create sample workload:**

At first, let's create the sample pods and service we have shown earlier so that we can verify our configured scrapping job for `kubernetes-pod` and `kubernetes-service-endpoints` are working.

```console
$ kubectl apply -f https://raw.githubusercontent.com/appscode/third-party-tools/master/monitoring/prometheus/builtin/artifacts/sample-workloads.yaml
pod/pod-monitoring-demo created
pod/service-endpoint-monitoring-demo created
service/pushgateway-service created
```

>YAML for sample workloads can be found [here](/monitoring/prometheus/builtin/artifacts/sample-workloads.yaml).

**Create ConfigMap:**

Now, we have to create a ConfigMap with the configuration (`prometheus.yaml`) file. We will mount this into Prometheus Deployment.

```console
$ kubectl apply -f https://raw.githubusercontent.com/appscode/third-party-tools/master/monitoring/prometheus/builtin/artifacts/configmap.yaml
configmap/prometheus-config created
```

>YAML for the ConfigMap can be found [here](/monitoring/prometheus/builtin/artifacts/configmap.yaml).

**Create RBAC resources:**

If you are using a RBAC enabled cluster, you have to give necessary permissions to Prometheus server. Let's create the necessary RBAC resources,

```console
$ kubectl apply -f https://raw.githubusercontent.com/appscode/third-party-tools/master/monitoring/prometheus/builtin/artifacts/rbac.yaml
clusterrole.rbac.authorization.k8s.io/prometheus created
serviceaccount/prometheus created
clusterrolebinding.rbac.authorization.k8s.io/prometheus created
```

>YAML for RBAC resources can be found [here](/monitoring/prometheus/builtin/artifacts/rbac.yaml).

**Create Deployment:**

Finally, let's deploy the Prometheus server.

```console
$ kubectl apply -f https://raw.githubusercontent.com/appscode/third-party-tools/master/monitoring/prometheus/builtin/artifacts/deployment.yaml
deployment.apps/prometheus created
````

Below the YAML for Prometheus Deployment that we have deployed above,

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus
  namespace: monitoring
  labels:
    app: prometheus-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: prometheus
  template:
    metadata:
      labels:
        app: prometheus
    spec:
      serviceAccountName: prometheus
      containers:
      - name: prometheus
        image: prom/prometheus:v2.20.1
        args:
        - "--config.file=/etc/prometheus/prometheus.yml"
        - "--storage.tsdb.path=/prometheus/"
        ports:
        - containerPort: 9090
        volumeMounts:
        - name: prometheus-config
          mountPath: /etc/prometheus/
        - name: prometheus-storage
          mountPath: /prometheus/
      volumes:
      - name: prometheus-config
        configMap:
          defaultMode: 420
          name: prometheus-config
      - name: prometheus-storage
        emptyDir: {}
```

>Use a persistent volume instead of `emptyDir` for `prometheus-storage` volume if you don't want to lose collected metrics on Prometheus pod restart.

**Verify Metrics:**

Prometheus server is running on port `9090`. We will use [port forwarding](https://kubernetes.io/docs/tasks/access-application-cluster/port-forward-access-application-cluster/) to access Prometheus dashboard.

At first, let's check if the Prometheus pod is in `Running` state.

```console
$ kubectl get pod -n monitoring -l=app=prometheus
NAME                          READY   STATUS    RESTARTS   AGE
prometheus-8568c86d86-vpzx5   1/1     Running   0          102s
```

Now, run following command on a separate terminal to forward `9090` port of `prometheus-8568c86d86-vpzx5` pod,

```console
$ kubectl port-forward -n monitoring prometheus-8568c86d86-vpzx5  9090
Forwarding from 127.0.0.1:9090 -> 9090
Forwarding from [::1]:9090 -> 9090
```

Now, we can access the dashboard at `localhost:9090`. Open [http://localhost:9090](http://localhost:9090) in your browser. You should see the configured jobs as target and they are in `UP` state which means Prometheus is able collect metrics from them.

<p align="center">
  <img alt="Prometheus Target" src="/monitoring/prometheus/builtin/images/prom-targets.png" style="padding:10px">
</p>

## Cleanup

To cleanup the Kubernetes resources created by this tutorial, run:

```console
# delete prometheus resources
$ kubectl delete all -n demo -l=app=prometheus-demo
pod "pod-monitoring-demo" deleted
pod "service-endpoint-monitoring-demo" deleted
service "pushgateway-service" deleted

$ kubectl delete all -n monitoring -l=app=prometheus-demo
deployment.apps "prometheus" deleted

# delete rbac stuff
$ kubectl delete clusterrole -l=app=prometheus-demo
clusterrole.rbac.authorization.k8s.io "prometheus" deleted

$ kubectl delete clusterrolebinding -l=app=prometheus-demo
clusterrolebinding.rbac.authorization.k8s.io "prometheus" deleted

$ kubectl delete serviceaccount -n monitoring -l=app=prometheus-demo
serviceaccount "prometheus" deleted

# delete namespace
$ kubectl delete ns monitoring
namespace "monitoring" deleted

$ kubectl delete ns demo
namespace "demo" deleted
```

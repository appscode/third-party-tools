# Deploy CoreOS Prometheus Operator

CoreOS [prometheus-operator](https://github.com/coreos/prometheus-operator) provides simple and Kubernetes native ways to deploy and configure the Prometheus server. This tutorial will show you how to deploy CoreOS prometheus-operator. You can also follow the official docs to deploy Prometheus operator from [here](https://github.com/coreos/prometheus-operator/blob/master/Documentation/user-guides/getting-started.md).

## Deploy Prometheus Operator

To follow this getting started you will need a Kubernetes cluster you have access to. This [example](https://github.com/prometheus-operator/prometheus-operator/blob/master/bundle.yaml) describes a Prometheus Operator Deployment, and its required ClusterRole, ClusterRoleBinding, Service Account, and Custom Resource Definitions.

Now we are going to deploy the above example manifest for the prometheus operator for release `release-0.41`. Let's deploy the above manifest using the following command,

```console
$ kubectl apply -f https://raw.githubusercontent.com/prometheus-operator/prometheus-operator/release-0.41/bundle.yaml
customresourcedefinition.apiextensions.k8s.io/alertmanagers.monitoring.coreos.com created
customresourcedefinition.apiextensions.k8s.io/podmonitors.monitoring.coreos.com created
customresourcedefinition.apiextensions.k8s.io/probes.monitoring.coreos.com created
customresourcedefinition.apiextensions.k8s.io/prometheuses.monitoring.coreos.com created
customresourcedefinition.apiextensions.k8s.io/prometheusrules.monitoring.coreos.com created
customresourcedefinition.apiextensions.k8s.io/servicemonitors.monitoring.coreos.com created
customresourcedefinition.apiextensions.k8s.io/thanosrulers.monitoring.coreos.com created
clusterrolebinding.rbac.authorization.k8s.io/prometheus-operator created
clusterrole.rbac.authorization.k8s.io/prometheus-operator created
deployment.apps/prometheus-operator created
serviceaccount/prometheus-operator created
service/prometheus-operator created
```

You can see above that all the resources of the prometheus operator and required stuff are created in the `default` namespace. we assumed that our cluster is RBAC enabled cluster.

Wait for Prometheus operator pod to be ready,

```console
$ kubectl get pod -n default | grep "prometheus-operator"
prometheus-operator-7589597769-gp46z   1/1     Running   0          6m13s
```

## Deploy Prometheus Server

To deploy the Prometheus server, we have to create [Prometheus](https://github.com/coreos/prometheus-operator/blob/master/Documentation/design.md#prometheus) cr. Prometheus cr defines a desired Prometheus server setup. It specifies which [ServiceMonitor](https://github.com/coreos/prometheus-operator/blob/master/Documentation/design.md#servicemonitor)'s should be covered by this Prometheus instance. ServiceMonitor cr defines a set of services that should be monitored dynamically.

Prometheus operator watches for `Prometheus` cr. Once a `Prometheus` cr is created, prometheus operator generates respective configuration (`prometheus.yaml` file) and creates a StatefulSet to run the desired Prometheus server.

#### Create RBAC

We assumed that our cluster is RBAC enabled.  Below is the YAML of RBAC resources for Prometheus cr that we are going to create,

<details>
<summary>RBAC resources for Prometheus cr</summary>

```yaml
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  name: prometheus
rules:
- apiGroups: [""]
  resources:
  - nodes
  - nodes/metrics
  - services
  - endpoints
  - pods
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources:
  - configmaps
  verbs: ["get"]
- nonResourceURLs: ["/metrics"]
  verbs: ["get"]
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: prometheus
  namespace: default
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: prometheus
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: prometheus
subjects:
- kind: ServiceAccount
  name: prometheus
  namespace: default
```
</details>
<br>

Let's create the following RBAC resources for Prometheus cr.

```console
$ kubectl apply -f https://raw.githubusercontent.com/appscode/third-party-tools/master/monitoring/prometheus/coreos-operator/artifacts/prometheus-rbac.yaml
clusterrole.rbac.authorization.k8s.io/prometheus created
serviceaccount/prometheus created
clusterrolebinding.rbac.authorization.k8s.io/prometheus created
```

#### Create Prometheus CR

Below is the YAML of `Prometheus` cr that we are going to create for this tutorial,

```yaml
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  name: prometheus
  labels:
    prometheus: prometheus
spec:
  replicas: 1
  serviceAccountName: prometheus
  serviceMonitorSelector:
    matchLabels:
      k8s-app: prometheus
  serviceMonitorNamespaceSelector:
    matchLabels:
      prometheus: prometheus
  resources:
    requests:
      memory: 400Mi
```

This Prometheus cr will select all `ServiceMonitor` that meet up the below conditions:

- `ServiceMonitor` will have the `k8s-app: prometheus` label.
- `ServiceMonitor` will be created in that namespaces which have `prometheus: prometheus` label.

Let's create the `Prometheus` cr we have shown above,

```console
$ kubectl apply -f https://raw.githubusercontent.com/appscode/third-party-tools/master/monitoring/prometheus/coreos-operator/artifacts/prometheus.yaml
prometheus.monitoring.coreos.com/prometheus created
```

Now, wait for a few seconds. Prometheus operator will create a StatefulSet. Let's check StatefulSet has been created,

```console
$ kubectl get statefulset -n default -l prometheus=prometheus
NAME                    READY   AGE
prometheus-prometheus   1/1     3m5s
```

Check StatefulSet's pod is running,

```console
$ kubectl get pod -n default -l prometheus=prometheus
NAME                      READY   STATUS    RESTARTS   AGE
prometheus-prometheus-0   3/3     Running   1          3m40s
```

Prometheus server is running on port `9090`. Now, we are ready to access Prometheus dashboard. We can use `NodePort` type service to access Prometheus server. In this tutorial, we will use [port forwarding](https://kubernetes.io/docs/tasks/access-application-cluster/port-forward-access-application-cluster/) to access Prometheus dashboard. Run the following command on a separate terminal,

```console
$ kubectl port-forward -n default prometheus-prometheus-0 9090
Forwarding from 127.0.0.1:9090 -> 9090
Forwarding from [::1]:9090 -> 9090
```

Now, you can access the Prometheus dashboard at `localhost:9090`.

## Cleanup

To clean up the Kubernetes resources created by this tutorial, run:

```console
# cleanup Prometheus resources
kubectl delete -f https://raw.githubusercontent.com/appscode/third-party-tools/master/monitoring/prometheus/coreos-operator/artifacts/prometheus.yaml
kubectl delete -f https://raw.githubusercontent.com/appscode/third-party-tools/master/monitoring/prometheus/coreos-operator/artifacts/prometheus-rbac.yaml

# cleanup Prometheus operator resources
kubectl delete -f https://raw.githubusercontent.com/prometheus-operator/prometheus-operator/release-0.41/bundle.yaml
```

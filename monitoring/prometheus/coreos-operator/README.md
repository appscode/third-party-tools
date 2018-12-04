# Deploy CoreOS Prometheus Operator

CoreOS [prometheus-operator](https://github.com/coreos/prometheus-operator) provides simple and kubernetes native way to deploy and configure Prometheus server. This tutorial will show you how to deploy CoreOS prometheus-operator. You can also follow the official docs to deploy Prometheus operator from [here](https://github.com/coreos/prometheus-operator/blob/master/Documentation/user-guides/getting-started.md).

To keep Prometheus resources isolated, we will use a separate namespace to deploy Prometheus operator and respective resources.

```console
$ kubectl create ns demo
namespace/demo created
```

## Deploy Prometheus Operator

#### Create RBAC

If you are using an RBAC enabled cluster, you have to give necessary permissions to Prometheus operator. Let's create necessary RBAC stuff.

```console
$ kubectl apply -f ./monitoring/prometheus/coreos-operator/artifacts/operator-rbac.yaml
clusterrole.rbac.authorization.k8s.io/prometheus-operator created
serviceaccount/prometheus-operator created
clusterrolebinding.rbac.authorization.k8s.io/prometheus-operator created
```

Here, we have created following RBAC resources,

**ClusterRole:**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: prometheus-operator
rules:
- apiGroups:
  - apiextensions.k8s.io
  resources:
  - customresourcedefinitions
  verbs: ["*"]
- apiGroups:
  - monitoring.coreos.com
  resources:
  - alertmanagers
  - prometheuses
  - prometheuses/finalizers
  - alertmanagers/finalizers
  - servicemonitors
  - prometheusrules
  verbs: ["*"]
- apiGroups:
  - apps
  resources:
  - statefulsets
  verbs: ["*"]
- apiGroups:
  - ""
  resources:
  - configmaps
  - secrets
  verbs: ["*"]
- apiGroups:
  - ""
  resources:
  - pods
  verbs: ["list","delete"]
- apiGroups:
  - ""
  resources:
  - services
  - endpoints
  verbs: ["get","create","update"]
- apiGroups:
  - ""
  resources:
  - nodes
  verbs: ["list","watch"]
- apiGroups:
  - ""
  resources:
  - namespaces
  verbs: ["get","list","watch"]

```

**ServiceAccount:**

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: prometheus-operator
  namespace: demo
```

**ClusterRoleBinding:**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: prometheus-operator
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: prometheus-operator
subjects:
- kind: ServiceAccount
  name: prometheus-operator
  namespace: demo
```

#### Create Deployment

Now, we can deploy Prometheus operator. Create operator Deployment using following command,

```console
$ kubectl apply -f ./monitoring/prometheus/coreos-operator/artifacts/operator.yaml
deployment.apps/prometheus-operator created
```

Below the definition of deployment we have created above,

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    k8s-app: prometheus-operator
  name: prometheus-operator
  namespace: demo
spec:
  replicas: 1
  selector:
    matchLabels:
      k8s-app: prometheus-operator
  template:
    metadata:
      labels:
        k8s-app: prometheus-operator
    spec:
      containers:
      - args:
        - --kubelet-service=kube-system/kubelet
        - --config-reloader-image=quay.io/coreos/configmap-reload:v0.0.1
        image: quay.io/coreos/prometheus-operator:v0.25.0
        name: prometheus-operator
        ports:
        - containerPort: 8080
          name: http
        resources:
          limits:
            cpu: 200m
            memory: 100Mi
          requests:
            cpu: 100m
            memory: 50Mi
      securityContext:
        runAsNonRoot: true
        runAsUser: 65534
      serviceAccountName: prometheus-operator
```

Wait for Prometheus operator pod to be ready,

```console
$ kubectl get pods -n demo -l k8s-app=prometheus-operator
NAME                                   READY   STATUS    RESTARTS   AGE
prometheus-operator-589fcd78c4-8fhks   1/1     Running   0          5m30s
```

## Deploy Prometheus Server

In order to deploy Prometheus server, we have to create [Prometheus](https://github.com/coreos/prometheus-operator/blob/master/Documentation/design.md#prometheus) crd. Prometheus crd defines a desired Prometheus server setup. It specifes which [ServiceMonitor](https://github.com/coreos/prometheus-operator/blob/master/Documentation/design.md#servicemonitor)'s should be covered by this Prometheus instance. ServiceMonitor crd defines a set of services that should be monitored dynamically.

Prometheus operator watches for `Prometheus` crd. Once a `Prometheus` crd is created, Prometheus operator generates respective configuration (`prometheus.yaml` file) and creates a StatefulSet to run desired Prometheus server.

#### Create RBAC

If you are using an RBAC enabled cluster, create following RBAC resources for Prometheus crd.

```console
$ kubectl apply -f ./monitoring/prometheus/coreos-operator/artifacts/prometheus-rbac.yaml
clusterrole.rbac.authorization.k8s.io/prometheus created
serviceaccount/prometheus created
clusterrolebinding.rbac.authorization.k8s.io/prometheus created
```

Here, we have created following RBAC resources,

**ClusterRole:**

```yaml
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  name: prometheus
rules:
- apiGroups: [""]
  resources:
  - nodes
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
```

**ServiceAccount:**

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: prometheus
  namespace: demo
```

**ClusterRoleBinding:**

```yaml
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
  namespace: demo
```

#### Create Prometheus CRD

Now, create Prometheus crd. Below is the YAML of `Prometheus` crd that we are going to create for this tutorial,

```yaml
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  name: prometheus
  namespace: demo
  labels:
    prometheus: prometheus
spec:
  replicas: 1
  serviceAccountName: prometheus
  serviceMonitorSelector:
    matchLabels:
      k8s-app: prometheus # change this according to your setup
  resources:
    requests:
      memory: 400Mi
```

This Prometheus crd will select all ServiceMonitor in `demo` namespace which has  `k8s-app: prometheus` label.

Let's create the `Prometheus` crd we have shown above,

```console
$ kubectl apply -f ./monitoring/prometheus/coreos-operator/artifacts/prometheus.yaml
prometheus.monitoring.coreos.com/prometheus created
```

Now, wait for few seconds. Prometheus operator will create a StatefulSet. Let's check StatefulSet has been created,

```console
$ kubectl get statefulset -n demo
NAME                    DESIRED   CURRENT   AGE
prometheus-prometheus   1         1         87s
```

Check StatefulSet's pod is running,

```console
$ kubectl get pod prometheus-prometheus-0 -n demo
NAME                      READY   STATUS    RESTARTS   AGE
prometheus-prometheus-0   2/2     Running   0          6m
```

Prometheus server is running on port `9090`. Now, we are ready to access Prometheus dashboard. We can use `NodePort` type service to access Prometheus server. In this tutorial, we will use [port forwarding](https://kubernetes.io/docs/tasks/access-application-cluster/port-forward-access-application-cluster/) to access Prometheus dashboard. Run following command on a separate terminal,

```console
$ kubectl port-forward -n demo prometheus-prometheus-0 9090
Forwarding from 127.0.0.1:9090 -> 9090
Forwarding from [::1]:9090 -> 9090
```

Now, you can access Prometheus dashboard at `localhost:9090`.

## Cleanup

To cleanup the Kubernetes resources created by this tutorial, run:

```console
# cleanup prometheus resources
kubectl delete -n demo prometheus prometheus
kubectl delete -n demo clusterrolebinding prometheus
kubectl delete -n demo clusterrole prometheus
kubectl delete -n demo serviceaccount prometheus
kubectl delete -n demo service prometheus-operated

# cleanup prometheus operator resources
kubectl delete -n demo deployment prometheus-operator
kubectl delete -n dmeo serviceaccount prometheus-operator
kubectl delete clusterrolebinding prometheus-operator
kubectl delete clusterrole prometheus-operator

# delete namespace
kubectl delete ns demo
```

# Use Grafana Dashboard

[Grafana](https://grafana.com) provides an elegant graphical user interface to visualize data. You can create beautiful dashboard easily with a meaningful representation of your Prometheus metrics in Grafana.

To keep Grafana resources isolated, we will use a separate namespace `demo`.

```console
$ kubectl create ns demo
namespace/demo created
```

## Deploy Grafana

Below the YAML for deploying grafana using a Deployment.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grafana
  namespace: demo
  labels:
    app: grafana
spec:
  replicas: 1
  selector:
    matchLabels:
      app: grafana
  template:
    metadata:
      labels:
        app: grafana
    spec:
      containers:
      - name: grafana
        image: grafana/grafana:5.3.1
```

Let's create the deployment we have shown above,

```console
$ kubectl apply -f ./monitoring/grafana/grafana.yaml
deployment.apps/grafana created
```

Wait for grafana pod to goes in running state,

```console
$ kubectl get pod -n demo -l=app=grafana
NAME                       READY   STATUS    RESTARTS   AGE
grafana-7f594dc9c6-xwkf2   1/1     Running   0          3m22s
```

Grafana is running on port `3000`. We will forward this port to access grafana UI. Run following command on a separate terminal,

```console
$ kubectl port-forward -n demo grafana-7f594dc9c6-xwkf2 3000
Forwarding from 127.0.0.1:3000 -> 3000
Forwarding from [::1]:3000 -> 3000
```

Now, we can access grafana UI in `localhost:3000`. Use `username: admin` and `password:admin` to login to the UI.

## Cleanup

To cleanup the Kubernetes resources created by this tutorial, run:

```console
kubectl delete -n demo deployment grafana
kubectl delete ns demo
```

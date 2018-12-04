# Deploy TLS Secured Minio Server in Kubernetes

[Minio](https://minio.io/) is an open source object storage server compatible with [Amazon S3](https://aws.amazon.com/s3/) cloud storage service. You can deploy Minio server in docker container, locally, Kubernetes cluster, Microsoft Azure, GCP etc.

This tutorial will show you how to deploy a TLS secured Minio server in Kubernetes. This will also show you how to access this TLS secured Minio server both from inside and outside of the Kubernetes cluster.

>You will find official guides for using Minio server at [here](https://docs.minio.io/).

## Before You Begin

At first, you need to have a Kubernetes cluster, and the `kubectl` command-line tool must be configured to communicate with your cluster. If you do not already have a cluster, you can create one by using [Minikube](https://github.com/kubernetes/minikube).

To keep things isolated, we will use a separate namespace called `demo` throughout this tutorial.

```console
$ kubectl create ns demo
namespace/demo created
```

## Generate self-signed  Certificate

TLS is crucial to secure your production services over the web. Usually, a certificate issued by a trusted third party known as [Certificate Authority](https://en.wikipedia.org/wiki/Certificate_authority) is used for TLS secured application. However, we can also use a self-signed certificate. In this tutorial, we will use self-signed certificate to secure a Minio server.

We will use a tool called [onessl](https://github.com/kubepack/onessl) developed by [AppsCode](https://appscode.com/) to generate self signed certificate. `onessl` makes generating self-signed certificate very easy and matter of two or three commands. If you already don't have `onessl` installed, please install it first from [here](https://github.com/kubepack/onessl/releases).

**Generate Root CA :**

At first, let's generate root certificate,

```console
$ onessl create ca-cert
```

This will create two files `ca.crt` and `ca.key` in your working directory. This root certificate will be used to create server certificates.

**Generate Server Certificate :**

Now, we will generate server certificate using the root certificate. Now, we have to provide the `domain` or `ip address` for which this certificate will be valid.

We want to access Minio server both from inside and outside the cluster. In order to access Minio from inside the cluster, we will use a service named `minio` in `demo` namespace. So, our domain will be `minio.demo.svc`. To access Minio from outside of cluster through `NodePort`, we will require Cluster's IP address. As I am using minikube, it is `192.168.99.100`. We will create a certificate that is valid for both `minio.demo.svc` domain and `198.168.99.100` ip address.

Let's create server certificates,

```console
$ onessl create server-cert --domains minio.demo.svc --ips 192.168.99.100
```

This will generate two files `server.crt` and `server.key`.

>Generated certificate will have key size of 2048 bytes and valid for 1 years.

**Prepare Certificates for Minio Server :**

Minio server will start TLS secure service if it find `public.crt` and `private.key` files in `/root/.minio/certs/` directory of the container. The `public.crt` file is concatenation of `server.crt` and `ca.crt` where `private.key` file is only the `server.key` file.

Let's generate `public.crt` and `private.key` file,

```console
$ cat {server.crt,ca.crt} > public.crt
$ cat server.key > private.key
```

Be careful about the order of `server.crt`  and `ca.crt`. The order will be `server's certificate > intermediate certificates > CA's root certificate`. The intermediate certificates are required if the server certificate is created using a certificate which is not the root certificate but signed by the root certificate. [onessl](https://github.com/kubepack/onessl) use root certificate by default to generate server certificate if no certificate path is specified by `--cert-dir` flag. Hence, the intermediate certificates are not required here.

We will create a Kubernetes secret with this `public.crt` and `private.key` files and mount the secret to `/root/.minio/certs/` directory of minio container.

> Minio server will not trust a self-signed certificate by default. We can mark the self-signed certificate as a trusted certificate by adding `public.crt` file in `/root/.minio/certs/CAs` directory.

## Deploy Minio Server

Now, we are ready to deploy TLS secured Minio server. At first, we will create a Secret with credentials and certificates. Then, we will create a PVC for Minio to store data. Finally, we will deploy Minio server using a Deployment.

**Create Secret :**

Now, let's create a secret `minio-server-secret` with  credentials `MINIO_ACCESS_KEY`, `MINIO_SECRET_KEY` and certificates `public.crt`, `private.key` files,

```console
$ echo -n '<your-minio-access-key>' > MINIO_ACCESS_KEY
$ echo -n '<your-minio-secret-key>' > MINIO_SECRET_KEY

$ kubectl create secret generic -n demo minio-server-secret \
    --from-file=./MINIO_ACCESS_KEY \
    --from-file=./MINIO_SECRET_KEY \
    --from-file=./public.crt \
    --from-file=./private.key
secret/minio-server-secret created
```

Now, verify that the credentials and certificate data are present in the secret,

```console
$ kubectl get secret -n demo minio-server-secret -o yaml
```

```yaml
apiVersion: v1
data:
  MINIO_ACCESS_KEY: bXktYWNjZXNzLWtleQ==
  MINIO_SECRET_KEY: bXktc2VjcmV0LWtleQ==
  private.key: <base64 encoded private.key data>
  public.crt: <base64 encoded public.key data>
kind: Secret
metadata:
  creationTimestamp: 2018-11-30T05:17:54Z
  name: minio-server-secret
  namespace: demo
  resourceVersion: "7057"
  selfLink: /api/v1/namespaces/demo/secrets/minio-server-secret
  uid: 4a4c0365-f45f-11e8-ae3b-0800279630e8
type: Opaque
```

**Create Persistent Volume Claim :**

Minio server needs a Persistent Volume to store data. Let's create a [PersistentVolumeClaim](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#persistentvolumeclaims) to request Persistent Volume from the cluster.

```console
$ kubectl apply -f ./storage/minio/artifacts/pvc.yaml
persistentvolumeclaim/minio-pvc created
```

YAML for PersistentVolumeClaim,

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: minio-pvc
  namespace: demo
spec:
  storageClassName: standard
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
```

Verify that the cluster has provisioned the claimed volume

```console
$ kubectl get pvc -n demo minio-pvc
NAME        STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
minio-pvc   Bound    pvc-3842dfb9-f460-11e8-ae3b-0800279630e8   5Gi        RWO            standard       53s
```

**Create Deployment :**

Now, let's create deployment for Minio server,

```console
$ kubectl apply -f ./storage/minio/artifacts/deployment.yaml
deployment.apps/minio-deployment created
```

Below the YAML for `minio-deployment` that we have created above,

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: minio-deployment
  namespace: demo
  labels:
    app: minio
spec:
  selector:
    matchLabels:
      app: minio
  template:
    metadata:
      labels:
        app: minio
    spec:
      containers:
      - name: minio
        image: minio/minio
        args:
        - server
        - --address
        - ":443"
        - /storage
        env:
        # credentials to access minio server. use from secret "minio-server-secret"
        - name: MINIO_ACCESS_KEY
          valueFrom:
            secretKeyRef:
              name: minio-server-secret
              key: MINIO_ACCESS_KEY
        - name: MINIO_SECRET_KEY
          valueFrom:
            secretKeyRef:
              name: minio-server-secret
              key: MINIO_SECRET_KEY
        ports:
        - name: https
          containerPort: 443
        volumeMounts:
        - name: storage # mount the "storage" volume into the pod
          mountPath: "/storage"
        - name: minio-certs # mount the certificates in "/root/.minio/certs" directory
          mountPath: "/root/.minio/certs"
      volumes:
      - name: storage # use "minio-pvc" to store data
        persistentVolumeClaim:
          claimName: minio-pvc
      - name: minio-certs # use secret "minio-server-secret" as volume to mount the certificates
        secret:
          secretName: minio-server-secret
          items:
          - key: public.crt
            path: public.crt
          - key: private.key
            path: private.key
          - key: public.crt
            path: CAs/public.crt # mark self signed certificate as trusted
```

**Minio Web UI :**

Minio server is running on port `:443`. We will use [port forwarding](https://kubernetes.io/docs/tasks/access-application-cluster/port-forward-access-application-cluster/) to access Minio Web UI.

At first, let's check if the Minio pod is in `Running` state.

```console
$ kubectl get pod -n demo -l=app=minio
NAME                                READY   STATUS    RESTARTS   AGE
minio-deployment-7d4c847d9d-8trcr   1/1     Running   0          13m
```

Now, run following command on a separate terminal to forward `:443` port of `minio-deployment-7d4c847d9d-8trcr` pod,

```console
$ kubectl port-forward -n demo minio-deployment-7d4c847d9d-8trcr :443
Forwarding from 127.0.0.1:37817 -> 443
Forwarding from [::1]:37817 -> 443
```

Our host port `31817` has been forward to `:443` port of the pod. Now, we can access the dashboard at `https://localhost:31817`. Open the url in your to access Minio Web UI.

As we are using self-signed certificate, the browser will not trust it. If you are using Google Chrome browser, you will be greeted with following message,

<p align="center">
  <img alt="Warning: Your connection is not private" src="/storage/minio/images/minio-1.png" style="padding:10px">
</p>

Click on `ADVANCED` marked by a red rectangle in the above image. Then click on `Proceed to localhost(unsafe)` as marked in below image.

<p align="center">
  <img alt="Proceed to localhost(unsafe)" src="/storage/minio/images/minio-2.png" style="padding:10px">
</p>

Then, you will be taken to Minio Login UI. Log in with your `MINIO_ACCESS_KEY` and `MINIO_SECRET_KEY`. If you succeed, you will see below UI,

<p align="center">
  <img alt="Mino Web UI" src="/storage/minio/images/minio-3.png" style="padding:10px">
</p>

## Accessing TLS Secured Minio Server

This section will show you how to access the TLS secured Minio server we have deployed above from both inside and outside of the Kubernetes cluster. We will use a tool called [osm](https://github.com/appscode/osm) developed by [AppsCode](https://appscode.com/) that gives simple and easy way to interact with various cloud storage services.

### Accessing from Outside of Cluster

Althugh we have already accessed the Minio Web UI from a browser that runs outside of the cluster, it did not used TLS secured connection. In this section, we will show how an application can access the Minio server with TLS secured connection.

Here, we will use [osm](https://github.com/appscode/osm) command line binary to interact with the Minio server. If you haven't installed `osm` already, please install it first.

**Create a NodePort type Service :**

We need a `NodePort` type service so that we can access the Minio server from outside of the cluster. Let's create a `NodePort` type Service first,

```console
$ kubectl apply -f ./storage/minio/artifacts/nodeport-svc.yaml
service/minio-nodeport-svc created
```

Here is YAML for the Service we have created above,

```yaml
apiVersion: v1
kind: Service
metadata:
  name: minio-nodeport-svc
  namespace: demo
spec:
  type: NodePort
  ports:
  - name: https
    port: 443
    targetPort: https
    protocol: TCP
  selector:
    app: minio # must match with the label used in minio deployment
```

We need to know the `NodePort` allocated for this service.

```console
$ kubectl get service -n demo minio-nodeport-svc
NAME                 TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)         AGE
minio-nodeport-svc   NodePort   10.108.252.121   <none>        443:32733/TCP   6m53s
```

Notice the `PORT(S)` field. Here, `32733` is the allocated `NodePort` for this service. Now, we can connect to Minio server using `https://<cluster ip>:32733` url. As I am using minikube for this tutorial, my cluster ip is `192.168.99.100`. I have already used this IP address while generating self-signed certificate to make it valid for this IP.

**Connect with Minio Server :**

Now, let's use `osm` to create bucket and upload some files to the Minio server.

At first, create `osm` configuration for the Minio server. `osm` configuration holds the connection information of the cloud bucket. So, every time you will run an operation, you don't have to provide the them again.

For `s3` compatible Minio server, we have to provide following connection information while creating the `osm` configuration.

- `--provider` tells `osm` that it is s3 or s3 compatible cloud storage.
- `--s3.access_key_id` is used to provide your `MINIO_ACCESS_KEY`.
- `--s3.secret_key` is used to provide your `MINIO_SECRET_KEY`.
- `--s3.endpoint` is used to specify the endpoint where your Minio server is running.
- `--s3.cacert_file` is used to provide the root certificate for TLS secured endpoint.

Let's create a `osm` configuration named `minio` for our Minio server.

```console
$ osm config set-context minio --provider=s3 --s3.access_key_id=my-access-key --s3.secret_key=my-secret-key --s3.endpoint=https://192.168.99.100:32733 --s3.cacert_file=./ca.crt
```

Check that `osm` has set this newly created configuration as it's current context,

```console
$ osm config current-context
minio
```

Let's create a bucket named `external-bucket` in our Minio server,

```console
# Here, mc = make container
$ osm mc external-bucket
Successfully created container external-bucket
```

Check the bucket has been created successfully by,

```console
# Here, lc = list container
$ osm lc
external-bucket
Found 1 container in
```

You can also check in the Minio Web UI to see if the bucket has been created.

<p align="center">
  <img alt="Mino UI: external-bucket" src="/storage/minio/images/external-bucket.png" style="padding:10px">
</p>

Let's upload a file in `external-bucket`

```console
$ osm push -c external-bucket ./deployment.yaml deployment.yaml
Successfully pushed item deployment.yaml
```

List all files of `external-bucket`,

```console
$ osm ls external-bucket
deployment.yaml
Found 1 item in container external-bucket
```

You can also browse the Web UI to see if the files are present in `external-bucket`

<p align="center">
  <img alt="Mino Web UI: files in external-bucket" src="/storage/minio/images/external-bucket-file.png" style="padding:10px">
</p>

**Try Without Certificates :**

Now, let's try to connect with the Minio server without certificates. Let's create another osm configuration that doesn't provide certificate. This time we will not provide `--s3.cacert_file=./ca.crt` flag while creating `osm` configuration.

```console
$ osm config set-context minio-not-ca --provider=s3 --s3.access_key_id=my-access-key --s3.secret_key=my-secret-key --s3.endpoint=192.168.99.100:32733
# check if current context is `minio-not-ca`
$ osm config current-context
minio-not-ca
```

Now, let's try to list files from `external-bucket`,

```console
$ osm ls external-bucket
Container, getting the bucket location: RequestError: send request failed
caused by: Get https://192.168.99.100:32733/external-bucket?location=: x509: certificate signed by unknown authority
Container, getting the bucket location: RequestError: send request failed
caused by: Get https://192.168.99.100:32733/external-bucket?location=: x509: certificate signed by unknown authority
```

So, we can see our Minio server is rejecting the request if we don't provide the root certificate.

### Accessing from Inside Cluster

Now, we will show how to connect with the Minio server from inside the Kubernetes cluster. This time, we will use a `ClusterIP` type service to access the Minio server from a pod running inside the cluster.

**Create ClusterIP type Service :**

We have used `minio.demo.svc` domain while generating the self-signed certificates. So, our certificate is valid for a Service named `minio` in `demo` namespace. Let's create the service first,

```console
$ kubectl apply -f ./storage/minio/artifacts/minio-svc.yaml
service/minio created
```

Below is the YAML for the service we have created above,

```yaml
apiVersion: v1
kind: Service
metadata:
  name: minio
  namespace: demo
spec:
  ports:
  - name: https
    port: 443
    targetPort: https
    protocol: TCP
  selector:
    app: minio # must match with the label used in minio deployment
```

**Create Secret :**

We will create a Secret with credentials to access the Minio server and the root certificate so that our client pod can access the Minio server over TLS.

Let's create the client secret,

```console
$ echo -n '<your-minio-access-key>' > AWS_ACCESS_KEY_ID
$ echo -n '<your-minio-secret-key>' > AWS_SECRET_ACCESS_KEY

$ kubectl create secret generic -n demo minio-client-secret \
    --from-file=./AWS_ACCESS_KEY_ID \
    --from-file=./AWS_SECRET_ACCESS_KEY \
    --from-file=./ca.crt
secret/minio-client-secret created
```

**Create Pod :**

Now, let's create a simple pod that will create a bucket named `internal-bucket` in the Minio server. This time, we will use [appscodeci/osm](https://hub.docker.com/r/appscodeci/osm/) docker image that is created from same `osm` binary we have used earlier.

Below the YAML for simple `osm-pod` that will just create a bucket named `internal-bucket` in Minio server then will go to `Complted` state.

```yaml
kind: Pod
apiVersion: v1
metadata:
  name: osm-pod
  namespace: demo
spec:
  restartPolicy: Never
  containers:
  - name: osm
    image: appscodeci/osm
    env:
    - name: PROVIDER
      value: s3
    - name: AWS_ENDPOINT
      value: https://minio.demo.svc
    - name: AWS_ACCESS_KEY_ID
      valueFrom:
        secretKeyRef:
          name: minio-client-secret
          key: AWS_ACCESS_KEY_ID
    - name: AWS_SECRET_ACCESS_KEY
      valueFrom:
        secretKeyRef:
          name: minio-client-secret
          key: AWS_SECRET_ACCESS_KEY
    - name: CA_CERT_FILE
      value: /etc/minio/certs/ca.crt # root ca has been mounted here
    args:
    - "mc internal-bucket" # create a bucket named "internal-bucket"
    volumeMounts: # mount root ca in /etc/minio/certs directory
    - name: credentials
      mountPath: /etc/minio/certs
  volumes:
  - name: credentials
    secret:
      secretName:  minio-client-secret
      items:
      - key: ca.crt
        path: ca.crt
```

Let's create the above pod,

```console
$ kubectl apply -f ./osm-pod.yaml
pod/osm-pod created
```

Now, wait for the pod to go in `Running` state. Once, it is in `Running` state, it will create a bucket in the Minio server.

You can check the pod's log to see if the bucket was created successfully.

```console
$ kubectl logs -n demo osm-pod -f
Configuring osm context for s3 storage
osm config set-context s3 --provider=s3 --s3.access_key_id=my-access-key --s3.secret_key=my-secret-key --s3.endpoint=https://minio.demo.svc --s3.cacert_file=/etc/minio/certs/ca.crt
Successfully configured

Running main command.....
osm mc internal-bucket
Successfully created container internal-bucket
```

You can also check Minio Web UI to ensure that the bucket is showing there.

<p align="center">
  <img alt="Mino Web UI: internal-bucket" src="/storage/minio/images/internal-bucket.png" style="padding:10px">
</p>

## Cleanup

To cleanup the Kubernetes resources created by this tutorial run following commands,

```console
kubectl delete -n demo secret/minio-server-secret
kubectl delete -n demo deployment/minio-deployment
kubectl delete -n demo persistentvolumeclaim/minio-pvc
kubectl delete -n demo service/minio-nodeport-svc
kubectl delete -n demo service/minio
kubectl delete -n demo secret/minio-client-secret
kubectl delete -n demo pod/osm-pod

kubectl delete ns demo
```

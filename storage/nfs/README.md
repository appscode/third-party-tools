# Deploying NFS Server in Kubernetes

NFS (Network File System) volumes can be mounted(https://kubernetes.io/docs/concepts/storage/volumes/#nfs) as a `PersistentVolume` in Kubernetes pods. You can also pre-populate an `nfs` volume. An `nfs` volume can be shared between pods. It is particularly helpful when you need some files to be writable between multiple pods.

This tutorial will show you how to deploy an NFS server in Kubernetes. This tutorial also shows you how to use `nfs` volume in a pod.

## Before You Begin

At first, you need to have a Kubernetes cluster, and the `kubectl` command-line tool must be configured to communicate with your cluster. If you do not already have a cluster, you can create one by using [Minikube](https://github.com/kubernetes/minikube).

To keep NFS resources isolated, we will use a separate namespace called `storage` throughout this tutorial. We will also use another namespace called `demo` to deploy sample workloads.

```console
$ kubectl create ns storage
namespace/storage created

$ kubectl create ns demo
namespace/demo created
```

**Add Kubernetes cluster's DNS in host's `resolved.conf` :**

There is an [issue](https://github.com/kubernetes/minikube/issues/2218) that prevents accessing NFS server through service DNS. However, accessing through IP address works fine. If you face this issue, you have to add IP address of `kube-dns` Service into your host's `/etc/systemd/resolved.conf` and restart `systemd-networkd`, `systemd-resolved`.

We are using Minikube for this tutorial. Below steps show how we can do this in Minikube.

1. Get IP address of `kube-dns` Service.
    ```console
    $ kubectl get service -n kube-system kube-dns
    NAME       TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)         AGE
    kube-dns   ClusterIP   10.96.0.10   <none>        53/UDP,53/TCP   173m
    ```
    Look at the `CLUSTER-IP` field. Here, `10.96.0.10` is the IP address of `kube-dns` service.

2. Add the IP address into `/etc/systemd/resolved.conf` file of Minikube and restart networking stuff.
   ```console
   # Login to minikube
   $ minikube ssh
   # Run commands as root
   $ su
   # Add IP address in `/etc/systemd/resolved.conf` file
   $ echo "DNS=10.96.0.10" >> /etc/systemd/resolved.conf
   # Restart netwokring stuff
   $ systemctl daemon-reload
   $ systemctl restart systemd-networkd
   $ systemctl restart systemd-resolved
   ````

Now, we will be able to access NFS server using DNS of a Service i.e. `{service name}.{namespace}.svc.cluster.local`.

## Deploy NFS Server

We will deploy NFS server using a [Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/). We will configure our NFS server to store data in [hostPath](https://kubernetes.io/docs/concepts/storage/volumes/#hostpath). You can also use any cloud volume such as [awsElasticBlockStore](https://kubernetes.io/docs/concepts/storage/volumes/#awselasticblockstore), [azureDisk](https://kubernetes.io/docs/concepts/storage/volumes/#azuredisk), [gcePersistentDisk](https://kubernetes.io/docs/concepts/storage/volumes/#gcepersistentdisk) etc as a persistent store for NFS server.

Then, we will create a [Service](https://kubernetes.io/docs/concepts/services-networking/service/) for this NFS server so that pods can consume `nfs` volume using this Service.

**Create Deployment :**

Below the YAML for the Deployment we are using to deploy NFS server.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nfs-server
  namespace: storage
spec:
  selector:
    matchLabels:
      app: nfs-server
  template:
    metadata:
      labels:
        app: nfs-server
    spec:
      containers:
      - name: nfs-server
        image: k8s.gcr.io/volume-nfs:0.8
        ports:
        - name: nfs
          containerPort: 2049
        - name: mountd
          containerPort: 20048
        - name: rpcbind
          containerPort: 111
        securityContext:
          privileged: true
        volumeMounts:
        - name: storage
          mountPath: /exports
      volumes:
      - name: storage
        hostPath:
          path: /data/nfs # store all data in "/data/nfs" directory of the node where it is running
          type: DirectoryOrCreate # if the directory does not exist then create it
```

Here, we have mounted `/data/nfs` directory of the host as `storage` volume in `/exports` directory of NFS pod. All the data stored in this NFS server will be located at `/data/nfs` directory of the host node where it is running.

Let's create the deployment we have shown above,

```console
$ kubectl apply -f https://raw.githubusercontent.com/appscode/third-party-tools/master/storage/nfs/artifacts/nfs-server.yaml
deployment.apps/nfs-server created
```

**Create Service :**

As we have deployed the NFS server using a Deployment, the server IP address can change in case of pod restart. So, we need a stable DNS/IP address so that our apps can consume the volume using it. So, we are going to create a `Service` for NFS server pods.

Below is the YAML for the `Service` we are going to create for our NFS server.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nfs-service
  namespace: storage
spec:
  ports:
  - name: nfs
    port: 2049
  - name: mountd
    port: 20048
  - name: rpcbind
    port: 111
  selector:
    app: nfs-server # must match with the label of NFS pod
```

Let's create the service we have shown above,

```console
$ kubectl apply -f https://raw.githubusercontent.com/appscode/third-party-tools/master/storage/nfs/artifacts/nfs-service.yaml
service/nfs-service created
```

Now, we can access the NFS server using `nfs-service.storage.svc.cluster.local` domain name.

>If you want to access the NFS server from outside of the cluster, you have to create [NodePort](https://kubernetes.io/docs/concepts/services-networking/service/#nodeport) or [LoadBalancer](https://kubernetes.io/docs/concepts/services-networking/service/#loadbalancer) type Service.

## Use NFS Volume

This section will show you how to use `nfs` volume in a pod. Here, we will demonstrate how we can use the `nfs` volume directly in a pod or through a [Persistent Volume Claim (PVC)](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#persistentvolumeclaims). We will also show how `nfs` volume can be used to share files between multiple pods.

### Use NFS volume directly in Pod

Let's create a simple pod that directly mount a directory from the NFS server as volume.

Below is the YAML for a sample pod that mount `/exports/nfs-direct` directory of the NFS server as volume,

```yaml
kind: Pod
apiVersion: v1
metadata:
  name: nfs-direct
  namespace: demo
spec:
  containers:
  - name: busybox
    image: busybox
    command:
    - sleep
    - "3600"
    volumeMounts:
    - name: data
      mountPath: /demo/data
  volumes:
  - name: data
    nfs:
      server: "nfs-service.storage.svc.cluster.local"
      path: "/nfs-direct" # "nfs-direct" folder must exist inside "/exports" directory of NFS server
```

Here, we have mounted `/exports/nfs-direct` directory of NFS server into `/demo/data` directory. Now, if we write anything in `/demo/data` directory of this pod, it will be written on `/exports/nfs-direct` directory of the NFS server.

>Note that  `path: "/nfs-direct"` is relative to `/exports` directory of NFS server.

At first, let's create `nfs-direct` folder inside `/exports` directory of NFS server.

```console
$ kubectl exec -n storage nfs-server-f9c6cbc7f-n85lv mkdir /exports/nfs-direct
```

Verify that directory has been created successfully,

```console
$ kubectl exec -n storage nfs-server-f9c6cbc7f-n85lv ls /exports
index.html
nfs-direct
```

Now, let's create the pod we have shown above,

```console
$ kubectl apply -f https://raw.githubusercontent.com/appscode/third-party-tools/master/storage/nfs/artifacts/nfs-direct.yaml
pod/nfs-direct created
```

When the pod is ready, let's create a sample file inside `/demo/data` directory. This file will be written on `/exports/nfs-direct` directory of NFS server.

```console
 $ kubectl exec -n demo nfs-direct touch /demo/data/demo.txt
```

Verify that the file has been stored in the NFS server,

```console
$ kubectl exec -n storage nfs-server-f9c6cbc7f-n85lv ls /exports/nfs-direct
demo.txt
```

### Use NFS volume through PVC

You can also use the NFS volume through a `PersistentVolumeClaim`. You have to create a `PersistentVolume` that will hold the information about NFS server. Then, you have to create a `PersistentVolumeClaim` that will be bounded with the `PersistentVolume`. Finally, you can mount the PVC into a pod.

**Create PersistentVolume :**

Below the YAML for `PersistentVolume` that provision volume from `/exports/pvc` directory of the NFS server.

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-pv
  labels:
    app: nfs-data
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteMany
  nfs:
    server: "nfs-service.storage.svc.cluster.local"
    path: "/pvc" # "pvc" folder must exist in "/exports" directory of NFS server
```

>Note that  `path: "/pvc"` is relative to `/exports` directory of NFS server.

At first, let's create `pvc` folder inside `/exports` directory of the NFS server.

```console
$ kubectl exec -n storage nfs-server-f9c6cbc7f-n85lv mkdir /exports/pvc
```

Verify that the directory has been created successfully,

```console
$ kubectl exec -n storage nfs-server-f9c6cbc7f-n85lv ls /exports
index.html
nfs-direct
pvc
```

Now, let's create the `PersistentVolume` we have shown above,

```console
$ kubectl apply -f https://raw.githubusercontent.com/appscode/third-party-tools/master/storage/nfs/artifacts/nfs-pv.yaml
persistentvolume/nfs-pv created
```

**Create a PersistentVolumeClaim:**

Now, we have to create a `PersistentVolumeClaim`. This PVC will be bounded with the `PersistentVolume` we have created above.

Below is the YAML for the `PersistentVolumeClaim` that we are going to create,

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nfs-pvc
  namespace: demo
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: ""
  resources:
    requests:
      storage: 1Gi
  selector:
    matchLabels:
      app: nfs-data
```

Let's create the PVC we have shown above,

```console
$ kubectl apply -f https://raw.githubusercontent.com/appscode/third-party-tools/master/storage/nfs/artifacts/nfs-pvc.yaml
persistentvolumeclaim/nfs-pvc created
```

Verify that the `PersistentVolumeClaim` has been bounded with the `PersistentVolume`.

```console
$ kubectl get pvc -n demo nfs-pvc
NAME      STATUS   VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
nfs-pvc   Bound    nfs-pv   1Gi        RWX                           10s
```

**Create Pod :**

Finally, we can deploy the pod that will use the NFS volume through a PVC.

Below is the YAML for sample pod that we are going to create for this purpose,

```yaml
kind: Pod
apiVersion: v1
metadata:
  name: nfs-pod-pvc
  namespace: demo
spec:
  containers:
  - name: busybox
    image: busybox
    command:
    - sleep
    - "3600"
    volumeMounts:
    - name: data
      mountPath: /demo/data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: nfs-pvc
```

Here, we have mounted PVC `nfs-pvc` into `/demo/data` directory. Now, if we write anything in `/demo/data` directory of this pod, it will be written on `/exports/pvc` directory of the NFS server.

Let's create the pod we have shown above,

```console
$ kubectl apply -f https://raw.githubusercontent.com/appscode/third-party-tools/master/storage/nfs/artifacts/nfs-pod-pvc.yaml
pod/nfs-pod-pvc created
```

When the pod is ready, let's create a sample file inside `/demo/data` directory. This file will be written on `/exports/pvc` directory of NFS server.

```console
$ kubectl exec -n demo nfs-pod-pvc touch /demo/data/demo.txt
```

Verify that the file has been stored in the NFS server,

```console
$ kubectl exec -n storage nfs-server-f9c6cbc7f-n85lv ls /exports/pvc
demo.txt
```

### Share file between multiple pods using NFS volume

Sometimes we need to share some common files (i.e. configuration file) between multiple pods. We can easily achieve it using a shared NFS volume. This section will show you how to we can share a common directory between multiple pods.

**Create a Shared Directory :**

At first, let's create the directory that we want to share in `/exports` directory of NFS server.

```console
$ kubectl exec -n storage nfs-server-f9c6cbc7f-n85lv mkdir /exports/shared
```

Verify that the directory has been created successfully,

```console
$ kubectl exec -n storage nfs-server-f9c6cbc7f-n85lv ls /exports
index.html
nfs-direct
pvc
shared
```

Now, we will mount this `shared` directory as `nfs` volume into the pods who need to share files with others.

**Create Pods :**

Below YAML show a pod that mount `shared` directory of the NFS server as volume.

```yaml
kind: Pod
apiVersion: v1
metadata:
  name: shared-pod-1
  namespace: demo
spec:
  containers:
  - name: busybox
    image: busybox
    command:
    - sleep
    - "3600"
    volumeMounts:
    - name: data
      mountPath: /demo/data
  volumes:
  - name: data
    nfs:
      server: "nfs-service.storage.svc.cluster.local"
      path: "/shared" # "shared" folder must exist inside "/exports" directory of NFS server
```

Here, we have mounted `/exports/shared` directory of NFS server into `/demo/data` directory. Now, if we write anything in `/demo/data` directory of this pod, it will be written on `/exports/shared` directory of the NFS server.

>Note that  `path: "/shared"` is relative to `/exports` directory of NFS server.

Now, let's create two pod with this configuration.

```console
# create shared-pod-1
$ kubectl apply -f https://raw.githubusercontent.com/appscode/third-party-tools/master/storage/nfs/artifacts/shared-pod-1.yaml
pod/shared-pod-1 created

# create shared-pod-2
$ kubectl apply -f https://raw.githubusercontent.com/appscode/third-party-tools/master/storage/nfs/artifacts/shared-pod-2.yaml
pod/shared-pod-2 created
```

Now, if we write any file in `/demo/data` directory of any pod, it will be instantly visible from other pod.

**Verify File Sharing :**

Let's verify that file change in `/demo/data` directory one pod is reflected in another pod.

```console
# create a file "file-1.txt" in "shared-pod-1"
$ kubectl exec -n demo shared-pod-1 touch /demo/data/file-1.txt

# check if "file-1.txt" is available in "shared-pod-2"
$ kubectl exec -n demo shared-pod-2 ls /demo/data
file-1.txt

# create a file "file-2.txt" in "shared-pod-2"
$ kubectl exec -n demo shared-pod-2 touch /demo/data/file-2.txt

# check if "file-2.txt" is available in "shared-pod-1"
$ kubectl exec -n demo shared-pod-1 ls /demo/data
file-1.txt
file-2.txt
```

So, we can see from above that any change in `/demo/data` directory of one pod is instantly reflected on the other pod.

## Cleanup

To clean-up the Kubernetes resources created by this tutorial run following commands,

```console
kubectl delete -n demo pod/shared-pod-1
kubectl delete -n demo pod/shared-pod-2

kubectl delete -n demo pod/nfs-pod-pvc
kubectl delete -n demo pvc/nfs-pvc
kubectl delete -n demo pv/nfs-pv

kubectl delete -n demo pod/nfs-direct

kubectl delete -n storage svc/nfs-service
kubectl delete -n storage deployment/nfs-server

kubectl delete ns storage
kubectl delete ns demo
```

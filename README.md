# gke-storage-provisioning
Demo showing how to provision storage manually and dynamically in GKE.

## Manual provisioning

Make sure no disks has been previously provisioned (except those that belong to the worker nodes of the Kubernetes cluster, of course).

```cli
$ gcloud compute disks list
NAME                                                             LOCATION        LOCATION_SCOPE  SIZE_GB  TYPE         STATUS
gke-my-first-cluster-1-default-pool-892e2f28-mr45                europe-west2-c  zone            32       pd-standard  READY
gke-my-first-cluster-1-default-pool-892e2f28-sp58                europe-west2-c  zone            32       pd-standard  READY
gke-my-first-cluster-1-default-pool-892e2f28-z4cl                europe-west2-c  zone            32       pd-standard  READY
```

```cli
$ kubectl get pv, pvc
No resources found in default namespace.
```

Provision a new 10 GB SSD disk in Google Cloud manually:

```cli
$ gcloud compute disks create --type=pd-ssd --size=10GB --zone=europe-west2-c manual-disk-1
$ gcloud compute disks list
```

Now, create the PV:

pv_manual.yaml
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: manually-created-pv
spec:
  accessModes:
  - ReadWriteOnce
  capacity:
    storage: 10Gi
  persistentVolumeReclaimPolicy: Retain
  gcePersistentDisk:
    pdName: manual-disk-1
```

```cli
$ kubectl apply -f pv_manual.yaml
```

Create a PVC that requests 10Gi of storage. Notice that the `storageClassName` is an empty string `""`.

pvc_manual.yaml
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mypvc
spec:
  storageClassName: ""
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

Verify both the PV and PVC have been created and they are bounded.

```cli
$ kubectl get pv,pvc
NAME                                   CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM           STORAGECLASS   REASON   AGE
persistentvolume/manually-created-pv   10Gi       RWO            Retain           Bound    default/mypvc                           7m37s

NAME                          STATUS   VOLUME                CAPACITY   ACCESS MODES   STORAGECLASS   AGE
persistentvolumeclaim/mypvc   Bound    manually-created-pv   10Gi       RWO                           30s
```

Run a pod that mounts the PV through the PVC:

```cli
$ kubectl run sleepypod --image=gcr.io/google_containers/busybox --restart=Never --dry-run -o yaml -- sleep 6000 > sleepypod.yaml
```

and mount the volume:

```yaml
  labels:
    run: sleepypod
  name: sleepypod
spec:
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: mypvc
  containers:
  - args:
    - sleep
    - "6000"
    image: gcr.io/google_containers/busybox
    name: sleepypod
    resources: {}
    volumeMounts:
      - mountPath: /data
        name: data
        readOnly: false
  dnsPolicy: ClusterFirst
  restartPolicy: Never
status: {}
```

```cli
$ kubectl get pods
NAME        READY   STATUS    RESTARTS   AGE
sleepypod   1/1     Running   0          15s
```

Delete the resources created:

```cli
$ kubectl delete -f sleepypod.yaml
pod "sleepypod" deleted
```

```cli
$ kubectl delete -f pvc_manual.yaml
persistentvolumeclaim "mypvc" deleted
```

```cli
$ kubectl delete -f pv_manual.yaml
persistentvolume "manually-created-pv" deleted
```

```cli
$ kubectl get pv,pvc
No resources found in default namespace.
```

```cli
gcloud compute disks delete manual-disk-1 --zone=europe-west2-c
The following disks will be deleted:
 - [manual-disk-1] in [europe-west2-c]
Do you want to continue (Y/n)?  y
Deleted [https://www.googleapis.com/compute/v1/projects/<your-project-id>/zones/europe-west2-c/disks/manual-disk-1].
```

## Dynamic provisioning

Create a storage class named `fast` that uses the `pd-ssd` type that will allow us to provision persistent SSDs in Google Cloud.

storage_class.yaml
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-ssd
```
List the storage classes.

```cli
$ kubectl get sc
NAME                 PROVISIONER            AGE
fast                 kubernetes.io/gce-pd   3s
standard (default)   kubernetes.io/gce-pd   107m
```

Create a storage request of `10Gi` that uses the `fast` storage class name.

pvc_dynamic.yaml
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mypvc
spec:
  storageClassName: fast
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

```cli
$ kubectl apply -f pvc_dynamic.yaml
persistentvolumeclaim/mypvc created
```

List the PV and PVC. Notice the PVC is bounded to the PV.

```cli
$ kubectl get pv,pvc
NAME                                                        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM           STORAGECLASS   REASON   AGE
persistentvolume/pvc-234df599-cdd6-11ea-8889-42010a9a0032   10Gi       RWO            Delete           Bound    default/mypvc   fast                    26s
NAME                          STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
persistentvolumeclaim/mypvc   Bound    pvc-234df599-cdd6-11ea-8889-42010a9a0032   10Gi       RWO            fast           29s
```

```cli
$ gcloud compute disks list
NAME                                                             LOCATION        LOCATION_SCOPE  SIZE_GB  TYPE         STATUS
gke-my-first-cluster-1-default-pool-892e2f28-mr45                europe-west2-c  zone            32       pd-standard  READY
gke-my-first-cluster-1-default-pool-892e2f28-sp58                europe-west2-c  zone            32       pd-standard  READY
gke-my-first-cluster-1-default-pool-892e2f28-z4cl                europe-west2-c  zone            32       pd-standard  READY
gke-my-first-cluster-1-pvc-234df599-cdd6-11ea-8889-42010a9a0032  europe-west2-c  zone            10       pd-ssd       READY
```

Now use the same Pod that was defined in the previous example:

```
$ kubectl apply -f sleepypod.yaml
```

```
$ kubectl delete -f sleepypod.yaml
```

**Notice that when deleting the pvc, the pv will also be deleted!**

```cli
$ kubectl delete -f pvc_dynamic.yaml
persistentvolumeclaim "mypvc" deleted
```

```cli
$ kubectl get pv,pvc
No resources found in default namespace.
```

Notice that the SSD disk `gke-my-first-cluster-1-pvc-234df599-cdd6-11ea-8889-42010a9a0032` has been removed and thus is not in the disks list anymore:

```
$ gcloud compute disks list
NAME                                                             LOCATION        LOCATION_SCOPE  SIZE_GB  TYPE         STATUS
gke-my-first-cluster-1-default-pool-892e2f28-mr45                europe-west2-c  zone            32       pd-standard  READY
gke-my-first-cluster-1-default-pool-892e2f28-sp58                europe-west2-c  zone            32       pd-standard  READY
gke-my-first-cluster-1-default-pool-892e2f28-z4cl                europe-west2-c  zone            32       pd-standard  READY
```

###  Storage
 
 There are several concepts:

 **Volumes** tied to the lifecycle of a Pod. Data is lost when the Pod is deleted(not restarted!). Suitable for ephemeral data.

 **PersistentVolumes(PV)**: A piece of storage in the cluster whose lifecycle is independent of nay Pod. Provisioned by an Administrator.  

 **PersistentVolumeClaims(PVC)**: A request for storage by a user. Acts as a claim on a PV resource. 


 #### **The Binding Process**

 1. A user creates a PVC requesting a certain size and access mode
 2. The Control Plane looks for an available PV that satisfied the claim.
 3. If a suitable PV is found, the PVC is **bound** to PV in a one-to-one mapping
 4. A Pod can tan mount the storage by referencing the PVC by name.   

 #### Access modes

 * **ReadWriteOnce (RWO)**: Read-write by a **single node**. Most common. But data corruption is possible
 * **ReadOnlyMany(ROX)**: Read-only by **many nodes** 
 * **ReadWriteMany(RWX)**: Read-write by many nodes. The underline system should support this (e.g., NFS)
 * **ReadWriteOncePod(RWOP)**: Read-write by a **single Pod**. Most secure. Check RWO mode 


 If your storage driver or older Kubernetes version does not support ReadWriteOncePod, you must use cluster-level scheduling tricks to ensure only one Pod handles the storage:
 * **Deployment Strategy**: Set your *deployment spec.strategy.type* to *Recreate*. Avoid *RollingUpdate*. *RollingUpdate* spins up a second Pod before killing the first one, meaning both Pods briefly attach to the RWO volume simultaneously on the same node, leading to a race condition.
 * **StatefulSets**: Use *StatefulSets* with *volumeClaimTemplates*. This natively gives every single Pod its own unique, isolated PV instead of forcing them to share one PVC.
 * **Pod Anti-Affinity**: Use strict *podAntiAffinity* rules in your manifests to prevent instances of the same application from ever being scheduled on the same host node

 #### Reclaim Polices 

 What to do with PV after its bound PVC is deleted:

 * **Retain**: The PV remains. The data and underlying storage is not deleted. Requires manual cleanup.
 * **Delete**: The PV and the associated external storage assets are deleted automatically. Default for dynamically provisioned PV. 


**Example**

1. Create a PV:

```yaml
#pv.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: task-pv-volume
spec:
  capacity:
    storage: 500Mi
  volumeMode: Filesystem  # Clearly states it uses a file system
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: manual
  hostPath:
    path: "/mnt/data"   # Just for experiments! Uses local node`s dir. 
```
>**Important notes regarding hostPath. When using a hostPath volume, Kubernetes mounts the directory from the exact physical node where the Pod is actively running. It does not automatically sync files across different servers in your cluster. It can happen that Pod is running on a worker node, but you are checking the /mnt/data folder on your master node.**

Apply PV:

```bash
deploy@lab12-kub-master-1:~/training$ kubectl apply -f pv.yaml
persistentvolume/task-pv-volume created
```

Check existing PV in the cluster:

```bash
deploy@lab12-kub-master-1:~/training$ kubectl get pv -o wide
NAME             CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE   VOLUMEMODE
task-pv-volume   500Mi      RWO            Retain           Available           manual         <unset>                          92s   Filesystem
```

2. Create a PVC:

```yaml
pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: task-pv-claim
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi  
```

Apply PVC:
```bash
deploy@lab12-kub-master-1:~/training$ kubectl apply -f pvc.yaml
persistentvolumeclaim/task-pv-claim created
```

Check the PV,PVC in the cluster:

```bash
deploy@lab12-kub-master-1:~/training$ kubectl get pv,pvc
NAME                              CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                   STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE
persistentvolume/task-pv-volume   500Mi      RWO            Retain           Bound    default/task-pv-claim   manual         <unset>                          9m3s

NAME                                  STATUS   VOLUME           CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
persistentvolumeclaim/task-pv-claim   Bound    task-pv-volume   500Mi      RWO            manual         <unset>                 51s
```

After PVC is bound to PV we can use the storage in a Pod:

```yaml
pod-storage.yaml
apiVersion: v1
kind: Pod
metadata:
  name: storage-pod
spec:
  containers:
    - name: nginx
      image: nginx
      volumeMounts: 
      - mountPath: "/usr/share/nginx/html"
        name: storage
  volumes:
    - name: storage
      persistentVolumeClaim:
        claimName: task-pv-claim            
```


>**PVs and PVCs are cluster-wide control plane objects, so you always apply their YAML files from your master node using kubectl. However, you can use nodeAffinity inside the PV manifest to tell Kubernetes exactly which worker node owns that physical /mnt/data folder
**

```yaml
hostPath:
    path: "/mnt/data"   # This path must exist ON THE WORKER NODE
  nodeAffinity:         # <-- PIN TO WORKER NODE
    required:
      nodeSelectorTerms:
        - matchExpressions:
            - key: kubernetes.io/hostname
              operator: In
              values:
                - lab12-kub-worker-1 # <-- Replace with your worker node name
```

The Pod did not complain because, from its perspective, the mount worked perfectly.
Here is exactly what happened behind the scenes:
* **The Scheduler picked the worker node**: Kubernetes looked at your Pod and chose to run it on your worker node.
* **The Kubelet looked for the folder**: The Kubelet service running on that worker node checked its local disk for /mnt/data.
* **Automatic Folder Creation**: By default, if a hostPath target directory does not exist on the hosting node, Kubernetes automatically creates an empty root-owned directory at that path during initialization.
* **Successful Mount**: The Kubelet successfully mapped that newly created, blank local folder into /usr/share/nginx/html inside your container.

Because everything was technically valid on the worker node's local disk, the Pod reported a completely healthy Running status. It had no way of knowing you were looking at a completely different physical disk partition over on the master node!


### StorageClasses and Dynamic Provisioning

An Administrator describes a "class" of storage, specifying a provisioner (e.g., ebs.csi.aws.com) and parameters.

When user creates a PVC referencing a storageClassName, the provisioner automatically creates a matching PV. 

For testing purpose we can install local-path-storage provisioner:

https://github.com/rancher/local-path-provisioner

```bash
deploy@lab12-kub-master-1:~/training$ kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/v0.0.36/deploy/local-path-storage.yaml
namespace/local-path-storage created
serviceaccount/local-path-provisioner-service-account created
role.rbac.authorization.k8s.io/local-path-provisioner-role created
clusterrole.rbac.authorization.k8s.io/local-path-provisioner-role created
rolebinding.rbac.authorization.k8s.io/local-path-provisioner-bind created
clusterrolebinding.rbac.authorization.k8s.io/local-path-provisioner-bind created
deployment.apps/local-path-provisioner created
storageclass.storage.k8s.io/local-path created
configmap/local-path-config created
```

```bash
deploy@lab12-kub-master-1:~/training$ kubectl get storageclass
NAME         PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
local-path   rancher.io/local-path   Delete          WaitForFirstConsumer   false                  53s
```

Create a PVC which uses storageClass:

```yaml
#storage-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: storage-claim
spec:
  storageClassName: local-class # Use the StorageClass just created above 
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi 
```

After applying our PVC is in "pending" state waiting a node which requests it.

```bash
deploy@lab12-kub-master-1:~/training$ kubectl get pvc
NAME            STATUS    VOLUME           CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
storage-claim   Pending                                              local-class    <unset>                 14s
task-pv-claim   Bound     task-pv-volume   500Mi      RWO            manual         <unset>                 106m
```

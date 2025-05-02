<!-- CKA Self-Study Mod 4 -->


# Implement storage classes and dynamic volume provisioning

StorageClass objects allow users to dynamically provision PVs through persistent volume claims. Each storageClass object represents a storage backend that can create a new PV. A StorageClass object is usually deployed alongside a Container Storage Interface (CSI) plugin, and provides the interface and agent necessary for a user to successfully request storage for their workload.

The example below shows a storage class for Longhorn:

<pre class="wp-block-code"><code>
allowVolumeExpansion: true
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
  name: longhorn
parameters:
  dataLocality: disabled
  fromBackup: ""
  fsType: ext4
  numberOfReplicas: "3"
  staleReplicaTimeout: "30"
provisioner: driver.longhorn.io
reclaimPolicy: Delete
volumeBindingMode: Immediate
</code></pre>

The StorageClass object describes a <code>provisioner</code>, which acts as the point of communication between the Kubernetes cluster and the storage backend.

A StorageClass is consumed by declaring the name of the desired storage class in a persistent volume claim's <code>storageClassName</code> key. If a matching StorageClass exists, the backing provisioner will contact the storage backend to provision volume per the parameters set in the PVC. 

</code></pre>
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: longhorn-test
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: longhorn
</code></pre>

Once the backing storage is created, the provisioner also creates a matching PV. The name of the storage class that created the PV will be recorded as the PV's <code>storageClassName</code>.

</code></pre>
apiVersion: v1
kind: PersistentVolume
metadata:
  annotations:
    longhorn.io/volume-scheduling-error: ""
    pv.kubernetes.io/provisioned-by: driver.longhorn.io
  creationTimestamp: "2023-01-09T23:31:14Z"
  finalizers:
  - kubernetes.io/pv-protection
  name: pvc-6f945eec-bc1b-4a20-89f1-af3f1b96cbe1
  resourceVersion: "3486339"
  uid: e9141b23-b299-4b83-82f8-180f5e98ddfd
spec:
  accessModes:
  - ReadWriteOnce
  capacity:
    storage: 10Gi
  claimRef:
    apiVersion: v1
    kind: PersistentVolumeClaim
    name: longhorn-test
    namespace: default
    resourceVersion: "3486296"
    uid: 6f945eec-bc1b-4a20-89f1-af3f1b96cbe1
  csi:
    driver: driver.longhorn.io
    fsType: ext4
    volumeAttributes:
      dataLocality: disabled
      fromBackup: ""
      fsType: ext4
      numberOfReplicas: "3"
      staleReplicaTimeout: "30"
      storage.kubernetes.io/csiProvisionerIdentity: 1672853863767-8081-driver.longhorn.io
    volumeHandle: pvc-6f945eec-bc1b-4a20-89f1-af3f1b96cbe1
  persistentVolumeReclaimPolicy: Delete
  storageClassName: longhorn
  volumeMode: Filesystem
status:
  phase: Bound
</code></pre>

The <code>storageClassName</code> of a Persistent Volume is treated like another identifier. Users creating the PVs themselves in static provisioning workflows can set any value in the <code>storageClassName</code> field of a PV, resulting in a custom storage class. PVCs can use the custom storage class set by the user to help narrow the selection of PVs that the PVC can claim.

[Learn more about Storage Classes in Kubernetes here](https://kubernetes.io/docs/concepts/storage/storage-classes/).


# Configure volume types, access modes and reclaim policies


## Volume Types

Volumes are the primary way to configure storage for apps running under Kubernetes. Certain volume types, PersistentVolumes, PersistentVolumeClaims, ConfigMaps and Secrets, etc. are valid entries under the <code>volumes</code> array. The Kubernetes documentation describes the <a href="https://kubernetes.io/docs/concepts/storage/volumes/#volume-types" target="_blank" rel="noreferrer noopener">types of volumes</a> that are valid. Each volume entry is given a name. Volumes are declared at the pod level, then mounted at the container level, as shown below:

<pre class="wp-block-code"><code>
apiVersion: v1
kind: Pod
metadata:
  name: cka-volumes
spec:
  restartPolicy: OnFailure
  containers:
    - name: cka-volume
      image: alpine
      command:
      - top
      volumeMounts:
      - name: applogs
        mountPath: /logs
  volumes:
    - name: applogs
      hostPath:
        path: /tmp/app/logs
</code></pre>

Most volume lifespans are tied to the pods they are configured for, and usually expire when the pod is removed. To decouple storage from pod lifecycles, Kubernetes has persistentVolumes objects. PersistentVolumes are resources within the cluster that provide storage that persists outside of pod lifespans.

The primary way to use PersistentVolumes (PVs) is through PersistentVolumeClaims.


## PV / PVC Access Modes, Volume Modes, and Reclaim Policies

Persistent Volumes use the <code>accesModes</code> array to ensure that the resulting volume mounts in a way supported by the resource provider’s filesystem. There are three access modes supported by Kubernetes:

<li><code>ReadWriteOnce</code> (<code>RWO</code>) – A single node may mount volume with read-write permissions</li>
<li><code>ReadOnlyMany</code> (<code>ROX</code>) – Many nodes may mount the volume with read-only permissions</li>
<li><code>ReadWriteMany</code> (<code>RWX</code>) – Many nodes may mount the volume with read-write permissions</li>

<p>Kubernetes also supports two <code>volumeModes</code> for persistent volumes: <code>Filesystem</code> (default) and <code>Block</code>. With <code>Filesystem</code>, the volume in mounted into a directory whereas the <code>Block</code> type presents the volume as a block device without a filesystem.</p>

<p>Below is an example of a persistent volume that allows <code>ReadWriteOnce</code> and <code>ReadOnlyMany</code> access modes and a <code>Retain</code> reclaim policy:</p>

<pre class="wp-block-code"><code>
apiVersion: v1
kind: PersistentVolume
metadata:
  name: app-pv
spec:
  storageClassName: manual
  capacity:
    storage: 50Gi
  accessModes:
    - ReadWriteOnce
    - ReadOnlyMany
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: "/app/logs"
</code></pre>

<a href="https://kubernetes.io/docs/concepts/storage/">Learn more about how Kubernetes handles storage</a> as well as persistent volume attributes:

<li><a href="https://kubernetes.io/docs/concepts/storage/persistent-volumes/#access-modes" target="_blank" rel="noreferrer noopener">Access modes</a></li>
<li><a href="https://kubernetes.io/docs/concepts/storage/persistent-volumes/#reclaim-policy" target="_blank" rel="noreferrer noopener">Reclaim policies</a></li>


# Manage persistent volumes and persistent volume claims


## Persistent Volume (PV)

A persistent volume is a storage object provisioned from the cluster’s infrastructure that is managed by the Kubernetes cluster. Persistent volumes allow storage to remain beyond an individual pod’s lifespan. Persistent volumes describe details of a storage implementation for the cluster, including:

<li>Access modes for the volume</li>
<li>The total capacity of the volume</li>
<li>What happens to the data after the volume is unbound</li>
<li>The type of storage</li>
<li>An optional, storage class identifier declaring which CSI created the PV or a custom value set by the user</li>

The following example shows a statically provisioned persistent volume. This volume is bound to the host’s filesystem at /tmp/pvc and claims 50 gigabytes of storage.

<pre class="wp-block-code"><code>
apiVersion: v1
kind: PersistentVolume
metadata:
  name: fivegigpv
spec:
  storageClassName: manual
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /tmp/pv
</code></pre>

Persistent volumes exist as resources in the cluster that any pod can claim using a persistent volume claim. 

[Learn more about persistent volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/).


## Persistent Volume Claim (PVC)

A persistent volume claim is a request for storage and is an abstraction of persistent volumes. Persistent volume claims bind to persistent volumes on a number of factors like label selectors, storage class name, storage capacity, and access mode. Persistent volume claims will bind to existing persistent volumes in the cluster that fulfill their requirements or dynamically create persistent volumes using an existing storage class.

Below is an example of a persistent volume claim that binds to the persistent volume example shown above:

<pre class="wp-block-code"><code>
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-pvc
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 50Gi
</code></pre>

The persistent volume claim must find a persistent volume with up to 50 gigabytes of storage and the ReadWriteOnce access mode in its manifest. It will only be matched with any persistent volumes within the manual storage class (as declared by the PV's own storageClassName key). The kube-controller-manager is responsible for taking the parameters of the PVC and finding a PV that it can bind to.

[Learn more about persistent volume claims](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#persistentvolumeclaims).


## Using Volumes with Pods

The <code>volumes</code> array under a pod manifest and <code>volumeMounts</code> array in a container manifest configure how applications running under Kubernetes use persistent storage.

Entries under the <code>volumes</code> array in a pod manifest declare what storage plugins or objects a pod has available for its containers to use as storage. Containers under pods use the <code>volumeMounts</code> array to mount any volumes made available by the pod. The container references the volume by the name configured on the pod level.

The example below shows a pod configured to mount the PersistentVolumeClaim example above for its containers to use:

<pre class="wp-block-code"><code>
apiVersion: v1
kind: Pod
metadata:
  name: cka-volumes
spec:
  restartPolicy: OnFailure
  containers:
    - name: cka-volumes
      image: alpine
      command:
      - top
      volumeMounts:
      - name: app-logs
        mountPath: /logs
  volumes:
    - name: app-logs
      persistentVolumeClaim:
        claimName: app-log-pvc
</code></pre>

This example uses a persistentVolumeClaim to store application log data. Learn more about volumes with your applications on Kubernetes using:
<li>[Basic volume plugins](https://kubernetes.io/docs/tasks/configure-pod-container/configure-volume-storage/)</li>
<li>[Persistent volumes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-persistent-volume-storage/).</li>


# Practice Drill

Create a pod that runs <code>centos/httpd</code> and stores its log directory at <code>/var/log/httpd</code> under <code>/tmp/httpd/</code> on the host.
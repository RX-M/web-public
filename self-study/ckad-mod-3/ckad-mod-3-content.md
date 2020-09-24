<!-- CKAD Self-Study Mod 3 -->


# Deployments and Rolling Updates

A deployment is a controller that ensures an application’s pods run according to a desired state. Deployments create and control replicaSets, which create and remove pods according to the deployment’s desired state. Kubelets report the current state to the Kubernetes API server. The API server compares the current state to the desired state (stored in etcd). If the current and desired states differ, the Kubernetes API server tells the kubelet(s) to make deployment changes to match the desired state.

The deployment spec declares the desired state of pod configurations under the pod template. The following example is a deployment of 3 nginx pods using the nginx version 1.16 image:

<pre class="wp-block-code"><code>
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    run: nginx
  name: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      run: nginx
  template:
    metadata:
      labels:
        run: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.16
</code></pre>

Updates to the deployment’s pod template trigger a gradual update. When a deployment’s pod template is updated, a new replicaSet is created that then creates new pods based on the updated pod spec. When the new pods are created, the previous version’s replicaSet is scaled to zero to remove the old pods. This strategy is known as a rolling update.

The following example creates a deployment of nginx pods with 3 replicas. The <code>--record</code> option annotates and saves the <code>kubectl </code> command for future reference. The deployment’s rollout status and history are verified with <code>kubectl rollout</code> .

<pre class="wp-block-code"><code>
$ kubectl run nginx --image=nginx:1.16 --replicas=3 --record

kubectl run --generator=deployment/apps.v1 is DEPRECATED and will be removed in a future version. Use kubectl run --generator=run-pod/v1 or kubectl create instead.
deployment.apps/nginx created

$ kubectl rollout status deploy nginx

deployment "nginx" successfully rolled out

$ kubectl rollout history deploy nginx

deployment.apps/nginx
REVISION    CHANGE-CAUSE
1                    kubectl run nginx --image=nginx:1.16 --replicas=3 --record=true

$
</code></pre>
Because the <code>--record</code> option was used to create the deployment, the annotation is listed under the <code>CHANGE-CAUSE</code> column. If <code>--record</code> was not used to annotate then <code>none</code> would appear under <code>CHANGE-CAUSE</code> for revision 1.

Next, update the deployment to use the nginx version 1.17 image. This update will trigger a rolling update. A new replicaSet will be created and the pods under old replicaSets will be terminated (scaled to 0). After updating the deployment, check the rollout status immediately to capture the rolling update.

<pre class="wp-block-code"><code>
$ kubectl set image deploy nginx nginx=nginx:1.17 --record

deployment.apps/nginx image updated

$ kubectl rollout status deploy nginx

Waiting for deployment "nginx" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "nginx" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "nginx" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "nginx" rollout to finish: 1 old replicas are pending termination...
Waiting for deployment "nginx" rollout to finish: 1 old replicas are pending termination...
deployment "nginx" successfully rolled out

$
</code></pre>

[Learn more about deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/) and [updating deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#updating-a-deployment).


# Deployments and Rollbacks

Kubernetes allows users to undo deployment updates. Deployments can be rolled back to a previous version with <code>kubectl rollout undo deploy <deployment_name></code> or you can specify a specific revision.

Using the previous example, let’s look at the revisions available.

<pre class="wp-block-code"><code>
$ kubectl rollout history deploy nginx

deployment.apps/nginx
REVISION  CHANGE-CAUSE
1         kubectl run nginx --image=nginx:1.16 --replicas=3 --record=true
2         kubectl set image deploy nginx nginx=nginx:1.17 --record=true

$
</code></pre>

The deployment’s update is now under revision 2. Again, if <code>--record</code> was not used to annotate then <code>none</code> would be listed under the <code>CHANGE-CAUSE</code> column.

Next we undo the rollout to a specific revision, watch the status, and check the rollout history.

<pre class="wp-block-code"><code>
$ kubectl rollout undo deploy nginx --to-revision=1

deployment.apps/nginx rolled back

$ kubectl rollout status deploy nginx

Waiting for deployment "nginx" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "nginx" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "nginx" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "nginx" rollout to finish: 1 old replicas are pending termination...
Waiting for deployment "nginx" rollout to finish: 1 old replicas are pending termination...
deployment "nginx" successfully rolled out

$ kubectl rollout history deploy nginx

deployment.apps/nginx
REVISION  CHANGE-CAUSE
2         kubectl set image deploy nginx nginx=nginx:1.17 --record=true
3         kubectl run nginx --image=nginx:1.16 --replicas=3 --record=true

$
</code></pre>

The deployment is back to using the nginx 1.16 image.

[Learn more about rolling back deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#rolling-back-to-a-previous-revision).


# Jobs and CronJobs

Jobs complete tasks from start to finish. A job is complete when the pod finishes the task and the pod exits successfully on completion.

There are three types of jobs:
<li>Non-parallel jobs - a job that runs one pod</li>
<li>Parallel jobs with a fixed completion - jobs run multiple pods in parallel and defines the number of completions when the job is finished</li>
<li>Parallel jobs without a fixed completion - jobs run multiple pods in parallel and when one pod is successful then the job is complete and all other pods terminate; this is also called a work queue</li>

The following manifest describes a parallel job with a fixed number of completions. The job outputs the date to the container’s standard out. The job will run 5 pods in parallel and stop after 20 successful completions.

<pre class="wp-block-code"><code>
apiVersion: batch/v1
kind: Job
metadata:
  name: date-job
spec:
  parallelism: 5
  completions: 20
  template:
    metadata:
      name: date-job
    spec:
      containers:
      - name: busybox
        image: busybox
        command:                        
        - /bin/sh
        - -c
        - date
      restartPolicy: OnFailure
</code></pre>

At the end of this job, there would be 20 completed pods. Obtaining the container log for any of the 20 pods outputs the date the container ran.

CronJobs are jobs with a schedule and are used to automate tasks. The following CronJob manifest creates a CronJob that runs every minute and outputs the date to the container’s standard out.

<pre class="wp-block-code"><code>
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: cron-job
spec:
  schedule: "*/1 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: busybox
            image: busybox
            args:
            - /bin/sh
            - -c
            - date
          restartPolicy: OnFailure
</code></pre>

Learn more about:
<li>[Jobs](https://kubernetes.io/docs/concepts/workloads/controllers/jobs-run-to-completion/)</li>
<li>[CronJobs](https://kubernetes.io/docs/tasks/job/automated-tasks-with-cron-jobs/).</li>


# Labels, Selectors, Annotations

Labels are key/value pairs attached to Kubernetes objects such as pods, persistent volumes, and cluster nodes. Labels help manage and organize Kubernetes objects into logical groups and can also be used to qualify Kubernetes objects for resources to execute on. For example, a network policy targets pods within the same namespace using labels on pods.

Some commands use selectors to identify and select Kubernetes objects by their labels. Selectors are used with the <code>-l</code> or <code>--selector</code> flag that filters on labels.

There are two selector types:
<li>Equality/Inequality-based</li>
  <ul>
    <li><code>=</code> or<code>==</code> for equality</li>
    <li><code>!=</code> for inequality</li>
  </ul>
<li>Set-based</li>
  <ul>
    <li><code>in</code> for labels that have keys with values in this set</li>
    <li><code>notin</code> for labels that have keys not in this set</li>
    <li><code>key_name</code> for labels with the key name</li>
  </ul>

Take a look at using labels and selectors. Run the following deployments and jobs to launch pods with an environment and a release label. The pods can have an environment label of <code>prod</code>, <code>dev</code>, or <code>qa</code> and a release label with <code>stable</code> or <code>edg</code>. Then use selectors to filter for pods using labels.

N.B. When creating deployments, the first <code>-l</code> option labels the deployment and the second <code>-l</code> option labels pods.

<pre class="wp-block-code"><code>
$ kubectl run nginx-deploy --image=nginx:1.9 --replicas=5 -l environment=prod -l environment=prod,release=stable

kubectl run --generator=deployment/apps.v1 is DEPRECATED and will be removed in a future version. Use kubectl run --generator=run-pod/v1 or kubectl create instead.
deployment.apps/nginx-deploy created

$ kubectl run nginx-pod --generator=run-pod/v1 --image=nginx:latest -l environment=dev,release=edge

pod/nginx-pod created

$ kubectl run nginx-qa --image=nginx:latest --replicas=3 -l environment=qa -l environment=qa,release=edge

kubectl run --generator=deployment/apps.v1 is DEPRECATED and will be removed in a future version. Use kubectl run --generator=run-pod/v1 or kubectl create instead.
deployment.apps/nginx-qa created

$
</code></pre>

Now we have 9 pods running.

<pre class="wp-block-code"><code>
$ kubectl get pods

NAME                                            READY   STATUS    RESTARTS   AGE
nginx-deploy-86f8b8c8d4-8j78k    1/1           Running    0                   21s
nginx-deploy-86f8b8c8d4-cbsbz   1/1           Running    0                   21s
nginx-deploy-86f8b8c8d4-jq2cb    1/1           Running    0                   21s
nginx-deploy-86f8b8c8d4-l8ck8    1/1           Running    0                   21s
nginx-deploy-86f8b8c8d4-smfmf   1/1           Running    0                   21s
nginx-pod                                       1/1           Running    0                   14s
nginx-qa-55d6b56d5c-fssmn         1/1           Running    0                    8s
nginx-qa-55d6b56d5c-mmsp9       1/1           Running    0                    8s
nginx-qa-55d6b56d5c-xlks7          1/1            Running    0                    8s

$
</code></pre>


Let’s use selectors to filter labels to identify the appropriate pods that we’re looking for.

Let’s get pods that are not running in production

<pre class="wp-block-code"><code>
$ kubectl get pod -l environment!=prod --show-labels

NAME                                       READY   STATUS    RESTARTS   AGE   LABELS
nginx-pod                                  1/1          Running     0                   67s     environment=dev,release=edge
nginx-qa-55d6b56d5c-fssmn    1/1          Running     0                   61s   environment=qa,pod-template-hash=55d6b56d5c,release=edge
nginx-qa-55d6b56d5c-mmsp9  1/1          Running     0                   61s   environment=qa,pod-template-hash=55d6b56d5c,release=edge
nginx-qa-55d6b56d5c-xlks7      1/1          Running     0                   61s   environment=qa,pod-template-hash=55d6b56d5c,release=edge

$
</code></pre>

We can also retrieve non production pods with set-based requirements:

<pre class="wp-block-code"><code>
$ kubectl get pods -l "environment notin (prod)" --show-labels

NAME                                       READY   STATUS    RESTARTS   AGE   LABELS
nginx-pod                                  1/1          Running     0                   84s   environment=dev,release=edge
nginx-qa-55d6b56d5c-fssmn    1/1          Running     0                   78s   environment=qa,pod-template-hash=55d6b56d5c,release=edge
nginx-qa-55d6b56d5c-mmsp9  1/1          Running     0                   78s   environment=qa,pod-template-hash=55d6b56d5c,release=edge
nginx-qa-55d6b56d5c-xlks7     1/1          Running     0                   78s   environment=qa,pod-template-hash=55d6b56d5c,release=edge

$
</code></pre>

Using the comma separator acts like a logical and (<code>&&</code>) operator. The following example lists pods in the dev environment and with an edge release:

<pre class="wp-block-code"><code>
$ kubectl get pods -l environment=dev,release=edge --show-labels

NAME        READY   STATUS    RESTARTS   AGE    LABELS
nginx-pod   1/1          Running     0                   108s    environment=dev,release=edge

$
</code></pre>

Annotations are similar to labels in that they are metadata key/value pairs. Annotations differ from labels in that they are not used for object selection but are typically used by external applications. Annotations are retrievable by API clients, tools, and libraries. Annotations are created in the manifest.
The following example is a pod manifest with annotations for build and image information.

<pre class="wp-block-code"><code>
apiVersion: v1
kind: Pod
metadata:
  name: hostinfo
  annotations:
    build: one
    builder: rxmllc
    imageregistery: “https://hub.docker.com/r/rxmllc/hostinfo”
spec:
  containers:
  - name: hostinfo
    image: rxmllc/hostinfo
  restartPolicy: Never
</code></pre>

Learn more about:
<li>[Labels and Selectors](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/)</li>
<li>[Annotations](https://kubernetes.io/docs/concepts/overview/working-with-objects/annotations/).</li>


# Persistent Volume Claims

A Kubernetes persistent volume exists outside the lifecycle of any pod that mounts it. Persistent volumes are storage objects managed by the Kubernetes cluster and provisioned from the cluster’s infrastructure (like the host’s filesystem).

Persistent volumes describe details of a storage implementation for the cluster, including:
Access modes for the volume
The total capacity of the volume
What happens to the data after the volume is unclaimed
The type of storage
An optional, custom storage class identifier

Persistent volume claims are an abstraction of persistent volumes. A persistent volume claim is a request for storage. Persistent volume claims bind to existing persistent volumes on a number of factors like label selectors, storage class name, storage capacity, and access mode. Persistent volume claims can dynamically create persistent volumes using an existing storage class. Pods bind to persistent volume claims by name in the pod’s manifest.

Let’s see how pods bind to a persistent volume claim and how a persistent volume claim binds to a persistent volume.

The manifest below is for a persistent volume with the following characteristics:
Label of k8scluster: master
Storage class name is local
Storage capacity is 200Mi
One node can mount the volume as read-write (access mode = ReadWriteOnce)
The persistent volume is released when a bounded persistent volume claim is deleted but not available until the persistent volume is deleted (persistentVolumeReclaimPolicy = Retain)
Mounted to a host path of /home/ubuntu/persistentvolume.

<pre class="wp-block-code"><code>
apiVersion: v1
kind: PersistentVolume
metadata:
  name: local-volume
  labels:
    k8scluster: master
spec:
  storageClassName: local
  capacity:
    storage: 200Mi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /home/ubuntu/persistentvolume
</code></pre>

In the example above, a persistent volume claim can use one or more of the following to bind to the persistent volume:  label, storage class name, storage capacity, and access mode.

The following example describes a persistent volume claim that binds to the ‘local-volume’ persistent volume by using a selector to select the label <code>k8scluster:master</code>, storage class name of local, and matching storage capacity and access mode.

<pre class="wp-block-code"><code>
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: local-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 200Mi
  storageClassName: local
  selector:
    matchLabels:
      k8scluster: master
</code></pre>

After creating the persistent volume and persistent volume claim with <code>kubectl apply -f yaml_file.yaml</code> we can verify the binding but describing the persistent volume and persistent volume claim.

<pre class="wp-block-code"><code>
$ kubectl describe pv local-volume | grep -A1 Status

Status:          Bound
Claim:           default/local-pvc

$ kubectl describe pvc local-pvc | grep -A1 Status

Status:        Bound
Volume:        local-volume

$
</code></pre>

Now let’s create a pod that binds to the persistent volume claim. The following pod manifest binds to the persistent volume claim by name and mounts the volume to the container’s <code>/usr/share/nginx/html</code> directory.

<pre class="wp-block-code"><code>
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
    - name: nginx
      image: nginx:latest
      volumeMounts:
        - name: data
          mountPath: /usr/share/nginx/html
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: local-pvc
</code></pre>

After creating this pod. Verify the binding by describing the persistent volume claim and grep for “Mounted By” then describe the pod and grep for “Volumes”

<pre class="wp-block-code"><code>
$ kubectl describe pvc local-pvc | grep “Mounted By”

Mounted By:    nginx

$ kubectl describe pod nginx | grep -A3 Volumes

Volumes:
  data:
    Type:       PersistentVolumeClaim (a reference to a PersistentVolumeClaim in the same namespace)
    ClaimName:  local-pvc

$
</code></pre>


[Learn more about persistent volumes and persistent volume claims](https://kubernetes.io/docs/concepts/storage/persistent-volumes/).


# Practice Drill

<li>Create a deployment that creates 2 replicas of pods using the <code>nginx:1.9</code> image.</li>
<li>Update the deployment to use the latest <code>nginx</code> image.</li>
<li>Undo the image update and rollback the deployment to use the <code>nginx:1.9</code> image.</li>

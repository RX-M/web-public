<!-- CKAD Self-Study Mod 1 -->

<h1>Application Design and Build</h1>

<h2>Define, Build, and Modify Container Images</h2>

All workloads on Kubernetes run as containers created from container images. With containers, the deployment of applications becomes fast, reliable, and repeatable. It is up to the developer to know how to define, build, and modify container images meant to run on a Kubernetes cluster.

In addition to the ability to easily run applications within containers, most container runtimes like Docker or Podman include build tools that allow developers to create images. Container images are immutable files containing metadata and (usually) a filesystem that encapsulates all of the code, libraries, and other dependencies needed to run an application.

To create container images, one must start by defining a series of instructions in a text file commonly referred to as a Dockerfile. Dockerfiles allow developers to:
<ul>
<li>Define a "base" image that provides the appropriate environment to build or run an application</li>
<li>Run arbitrary commands during the build process to prepare the image's included filesystem to run an application</li>
<li>Define environment variables to be consumed by the containerized application</li>
<li>Inject additional metadata into the image to inform users of used ports or authorship</li>
<li>Define binaries that will be run when a container is first created</li>
</ul>

A simple Dockerfile typically looks like this:

<pre class="wp-block-code"><code>FROM debian:12
RUN apt update & apt install -y bash
CMD while true; do which vim; sleep 2; done
</code></pre>

After defining the instructions to create a container image, the image must be built. Tools like Docker and Podman (through buildah) usually have an accompanying "build" instruction that parses a Dockerfile, executes its steps in order, and produces a tagged image.

<pre class="wp-block-code"><code>$ docker build -t debian-vim:latest .

[+] Building 5.3s (6/6) FINISHED                                                                                                                                        docker:default
 => [internal] load build definition from Dockerfile                                                                                                                              0.0s
 => => transferring dockerfile: 134B                                                                                                                                              0.0s
 => [internal] load metadata for docker.io/library/debian:12                                                                                                                      0.3s
 => [internal] load .dockerignore                                                                                                                                                 0.0s
 => => transferring context: 2B                                                                                                                                                   0.0s
 => CACHED [1/2] FROM docker.io/library/debian:12@sha256:1dc55ed6871771d4df68d393ed08d1ed9361c577cfeb903cd684a182e8a3e3ae                                                         0.0s
 => [2/2] RUN apt update && apt install -y bash                                                                                                                                   4.8s
 => exporting to image                                                                                                                                                            0.1s
 => => exporting layers                                                                                                                                                           0.1s
 => => writing image sha256:32c960212cc9851347859089fd26b4e2aea7991e0ccc2c7f6f1d605fe4004ed8                                                                                      0.0s 
 => => naming to docker.io/library/debian-top:latest
</code></pre>

Each "build" invocation results in a single tagged image that can the be:
<ul>
<li>Pushed to a repository referred to in the tag like DockerHub or a locally hosted solution with a "push" command
<li>Have its contents to a tarball using a command like "save"
<li>Create containers to run applications with a "create" or "run" command
</ul>

The commands available to you for building, pushing, or even saving images will vary depending on your choice of container runtime tool. The "help" outputs and man pages of such tools typically display such commands:

<pre class="wp-block-code"><code>$ docker image build -h

Flag shorthand -h has been deprecated, please use --help

Usage:  docker image build [OPTIONS] PATH | URL | -

Build an image from a Dockerfile

Options:
      --add-host list           Add a custom host-to-IP mapping (host:ip)
      --build-arg list          Set build-time variables
      --cache-from strings      Images to consider as cache sources

...
</code></pre>

Learn more about how <strong><a href="https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/">Kubernetes CLI commands map to a tool like Docker here.</a></strong>


<h2>Understand Jobs and CronJobs</h2>

Jobs complete tasks from start to finish. A job is complete when the pod finishes the task and the pod exits successfully on completion.

There are three types of jobs:
<ul>
<li>Non-parallel jobs - a job that runs one pod</li>
<li>Parallel jobs with a fixed completion - jobs run multiple pods in parallel and defines the number of completions when the job is finished</li>
<li>Parallel jobs without a fixed completion - jobs run multiple pods in parallel and when one pod is successful then the job is complete and all other pods terminate; this is also called a work queue</li>
</ul>

The following manifest describes a parallel job with a fixed number of completions. The job outputs the date to the container’s standard out. The job will run 5 pods in parallel and stop after 20 successful completions.

<pre class="wp-block-code"><code>apiVersion: batch/v1
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
        image: docker.io/busybox:latest
        command:
        - /bin/sh
        - -c
        - date
      restartPolicy: OnFailure
</code></pre>

At the end of this job, there would be 20 completed pods. Obtaining the container log for any of the 20 pods outputs the date the container ran.

CronJobs are jobs with a schedule and are used to automate tasks. The following CronJob manifest creates a CronJob that runs every minute and outputs the date to the container’s standard out.

<pre class="wp-block-code"><code>
apiVersion: batch/v1
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
            image: docker.io/busybox:latest
            args:
            - /bin/sh
            - -c
            - date
          restartPolicy: OnFailure
</code></pre>

Learn more about <strong><a href="https://kubernetes.io/docs/concepts/workloads/controllers/job/">Jobs</a></strong> and <strong><a href="https://kubernetes.io/docs/tasks/job/automated-tasks-with-cron-jobs/">CronJobs</a></strong>.


<h2>Understand Multi-Container Pod Design Patterns</h2>

A pod may run one or more containers. Multi-container pods are tightly coupled in that the containers are co-located, co-scheduled and the containers share the same <code>network</code>, <code>uts</code>, and <code>ipc</code> namespaces. There are three patterns of multi-container pods:

<ul>
<li>Sidecar - sidecar containers extend and enhance the "main" container in the pod. The diagram below shows a web server container that saves its logs to a shared filesystem. The log saving sidecar container sends the webserver’s logs to a log aggregator.</li>
</ul>

<ul>
 <li>Ambassador - ambassador containers proxy a pod’s local connection to the outside world. The diagram shows a three-node Redis cluster (1, 2, 3). The ambassador container is a proxy that sends the appropriate reads and writes from the main application container to the Redis cluster. The main application container is configured to connect to a local Redis server since the two containers share the same uts namespace.</li>
</ul>

<ul>
 <li>Adapter - adapter containers standardize and normalize output for remote monitoring systems that require standard data formats. The diagram below shows a monitoring adapter container running an agent that reads the main application’s data, processes it, then exports the normalized data to monitoring systems elsewhere in the network.</li>
</ul>


A multi-container pod is created by specifying one or more additional container entries in a pod manifest. Shown below is an example of a multi-container pod with an <code>nginx</code> main container and an <code>fluent-bit</code> container sidecar in yaml. The nginx container writes its logs to a file at <code>/tmp/nginx/</code>, which is shared between all containers in the pod. The Fluent-Bit container reads the file from the shared directory and outputs it to its own standard output.

<pre class="wp-block-code"><code>apiVersion: v1
kind: Pod
metadata:
  name: sidecar
spec:
  containers:
  - name: nginx
    image: docker.io/nginx:latest
    volumeMounts:
    - name: shared-vol
      mountPath: /tmp/nginx/    
    command:
    - /bin/sh
    - -c
    - nginx -g 'daemon off;' > /tmp/nginx/nginx.log
  - name: adapter
    image: docker.io/fluent/fluent-bit
    command:
    - /fluent-bit/bin/fluent-bit
    - -i
    - tail
    - -p
    - path=/nginx/nginx.log
    - -o
    - stdout
    volumeMounts:
    - name: shared-vol
      mountPath: /nginx
  volumes:
  -  name: shared-vol
     emptyDir: {}
  restartPolicy: OnFailure
</code></pre>

Learn more about <strong><a href="https://kubernetes.io/blog/2015/06/the-distributed-system-toolkit-patterns/">multi-container pod patterns</a></strong>.


<h2>Utilize Persistent and Ephemeral Volumes</h2>

A Kubernetes persistent volume exists outside the lifecycle of any pod that mounts it. Persistent volumes are storage objects managed by the Kubernetes cluster and provisioned from the cluster’s infrastructure (like the host’s filesystem).

Persistent volumes describe details of a storage implementation for the cluster, including:
<ul>
<li>Access modes for the volume</li>
<li>The total capacity of the volume</li>
<li>What happens to the data after the volume is unclaimed</li>
<li>The type of storage</li>
<li>An optional, custom storage class identifier</li>
</ul>

Persistent volume claims are an abstraction of persistent volumes. A persistent volume claim is a request for storage. Persistent volume claims bind to existing persistent volumes on a number of factors like label selectors, storage class name, storage capacity, and access mode. Persistent volume claims can dynamically create persistent volumes using an existing storage class. Pods bind to persistent volume claims by name in the pod’s manifest.

Let’s see how pods bind to a persistent volume claim and how a persistent volume claim binds to a persistent volume.

The manifest below is for a persistent volume with the following characteristics:
<ul>
<li>Label of k8scluster: control</li>
<li>Storage class name is local</li>
<li>Storage capacity is 200Mi</li>
<li>One node can mount the volume as read-write (access mode = ReadWriteOnce)</li>
<li>The persistent volume is released when a bounded persistent volume claim is deleted but not available until the persistent volume is deleted (persistentVolumeReclaimPolicy = Retain)</li>
<li>Mounted to a host path of /home/ubuntu/persistentvolume.</li>
</ul>

<pre class="wp-block-code"><code>apiVersion: v1
kind: PersistentVolume
metadata:
  name: local-volume
  labels:
    k8scluster: control
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

The following example describes a persistent volume claim that binds to the ‘local-volume’ persistent volume by using a selector to select the label <code>k8scluster: control</code>, storage class name of local, and matching storage capacity and access mode.

<pre class="wp-block-code"><code>apiVersion: v1
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
      k8scluster: control
</code></pre>

After creating the persistent volume and persistent volume claim with <code>kubectl apply -f yaml_file.yaml</code> we can verify the binding but describing the persistent volume and persistent volume claim.

<pre class="wp-block-code"><code>$ kubectl describe pv local-volume | grep -A1 Status

Status:          Bound
Claim:           default/local-pvc

$ kubectl describe pvc local-pvc | grep -A1 Status

Status:        Bound
Volume:        local-volume

$
</code></pre>

Now let’s create a pod that binds to the persistent volume claim. The following pod manifest binds to the persistent volume claim by name and mounts the volume to the container’s <code>/usr/share/nginx/html</code> directory.

<pre class="wp-block-code"><code>apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
    - name: nginx
      image: docker.io/nginx:latest
      volumeMounts:
        - name: data
          mountPath: /usr/share/nginx/html
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: local-pvc
</code></pre>

After creating this pod. Verify the binding by describing the persistent volume claim and grep for "Mounted By" then describe the pod and grep for "Volumes"

<pre class="wp-block-code"><code>$ kubectl describe pvc local-pvc | grep "Mounted By"

Mounted By:    nginx

$ kubectl describe pod nginx | grep -A3 Volumes

Volumes:
  data:
    Type:       PersistentVolumeClaim (a reference to a PersistentVolumeClaim in the same namespace)
    ClaimName:  local-pvc

$
</code></pre>

Learn more about <strong><a href="https://kubernetes.io/docs/concepts/storage/persistent-volumes/">persistent volumes and persistent volume claims</a></strong>.


<h2>Practice Drills</h2>

<!-- Drill 1 -->

Build an image using the following Dockerfile tagged <code>self-study/webserver:v1</code>:</li>
<pre class="wp-block-code"><code>FROM docker.io/centos/httpd
RUN /bin/sh -c "echo welcome" > /usr/share/httpd/noindex/index.html
</code></pre>

<!-- Drill 2 -->

Define a pod named <code>self-study-pod-1</code> which has one container named <code>primary</code> running the <code>self-study/webserver:v1</code> image you just built. The <code>primary</code> container should have an ephemeral volume named <code>share</code> mounted at <code>/var/log/httpd</code>. This pod should also have an adapter container named <code>logger</code> running the <code>fluent/fluent-bit:1.9.2</code> image that mounts volume the <code>share</code> volume at <code>/httpd</code> and runs the command <code>/fluent-bit/bin/fluent-bit -i tail -p path=/httpd/access_log -o stdout</code>.

If you built the image in Docker and run containerd as your container runtime, run the following commands to transfer the image:

<pre class="wp-block-code"><code>sudo docker save docker.io/self-study/webserver:v1 -o chlg-img.tar
sudo ctr -n k8s.io image import chlg-img.tar
</code></pre>

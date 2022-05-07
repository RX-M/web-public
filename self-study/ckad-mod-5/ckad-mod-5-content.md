<!-- CKAD Self-Study Mod 5 -->

<h1>Application Observability and Maintenance</h1>


<h2>Understand API Deprecations</h2>

As an ongoing project, new features will move through various stages of maturity as Kubernetes continues to grow. Every feature is served by a new API that is registered to the Kubernetes API server. There are 3 stages of maturity for these APIs:
<ul>
<li>Alpha - These APIs serve features that are highly experimental and not recommended for any production use. Kubernetes includes these Alpha APIs but they must be enabled by setting the appropriate feature gate on the API server. Alpha APIs can either move to beta status or get removed depending on how well the feature is received and used.</li>
<li>Beta - These APIs serve features that are still in highly active development but considered ready for production use. Beta APIs and features are enabled by default. Depending on their development cycles, Beta APIs can move up through multiple versions between Kubernetes releases or regress back to Alpha for further development.</li>
<li>Stable - These APIs are highly mature and do not see too many changes between versions. Stable APIs are enabled by default. At this point, stable APIs can only be retired if their feature is no longer necessary or if a new version of the API supplants its use.</li>
</ul>

You can view all of the APIs registered with your Kubernetes API server using the <code>kubectl api-versions</code> command:

<pre class="wp-block-code"><code>$ kubectl api-versions

admissionregistration.k8s.io/v1
apiextensions.k8s.io/v1
apiregistration.k8s.io/v1
apps/v1
authentication.k8s.io/v1
authorization.k8s.io/v1
autoscaling/v1
autoscaling/v2
autoscaling/v2beta1
autoscaling/v2beta2

...

$
</code></pre>

API deprecations are a major event in the Kubernetes development cycle and come with many announcements. After a deprecation announcement, the affected features and/or APIs that get deprecated will:
<ul>
<li>Begin generating warnings in the CLI when invoked or submitted to the Kubernetes API server</li>
<li>Have their functionality disabled while continuing to generate warnings</li>
<li>Eventually get removed from the codebase all together, usually within 2-3 releases</li>
</ul>

<pre class="wp-block-code"><code>$ kubectl run -o yaml --dry-run=client pod-with-reqs-limits --requests cpu=64m,memory=256Mi --limits cpu=256m,memory=512Mi --image nginx

Flag --requests has been deprecated, has no effect and will be removed in 1.24.
Flag --limits has been deprecated, has no effect and will be removed in 1.24.
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: pod-with-reqs-limits
  name: pod-with-reqs-limits
spec:
  containers:
  - image: nginx
    name: pod-with-reqs-limits
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}

$
</code></pre>

For existing manifests, the <code>apiVersion</code> line must be changed to a non-deprecated API version. This can be done with <code>kubectl-convert</code>, an optional kubectl plugin that examines older manifests and updates them to reflect the latest API versions.

The <code>kubectl-convert</code> plugin must be installed separate, with instructions found <strong><a href="https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/#install-kubectl-convert-plugin">here</a></strong>.

<pre class="wp-block-code"><code>$ curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl-convert"

$ sudo install -o root -g root -m 0755 kubectl-convert /usr/local/bin/kubectl-convert

$
</code></pre>

Given the following file, which describes a deployment from v1.16 (when deployments were under the old <code>extensions</code> API):

<pre class="wp-block-code"><code>$ cat old-deploy.yaml

apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: old-deploy
spec:
  replicas: 3
  selector:
    matchLabels:
      app: old-deploy
  template:
    metadata:
      labels:
        app: old-deploy
    spec:
      containers:
      - image: nginx
        name: nginx

$
</code></pre>

<code>kubectl-convert</code> can read the file and automatically make the necessary changes to update it to the latest api (<code>apps/v1</code> in 1.23):

<pre class="wp-block-code"><code>$ kubectl-convert -f old-deploy.yaml --output-version apps/v1

apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: old-deploy
  name: old-deploy
spec:
  progressDeadlineSeconds: 2147483647
  replicas: 3
  revisionHistoryLimit: 2147483647
  selector:
    matchLabels:
      app: old-deploy
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
    type: RollingUpdate
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: old-deploy
    spec:
      containers:
      - image: nginx
        imagePullPolicy: Always
        name: nginx
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
status: {}

$
</code></pre>

There are usually about 2-3 minor versions between a deprecation notice and the removal of a feature, so users typically have at least 6 months.

Learn more about and keep track of the <strong><a href="https://kubernetes.io/docs/reference/using-api/deprecation-guide/">API Deprecation process</a></strong> and how to <strong><a href="https://kubernetes.io/docs/reference/using-api/deprecation-guide/#migrate-to-non-deprecated-apis">use kubectl-convert</a></strong>.


<h2>Liveness Probes and Readiness Probes</h2>

A liveness probe is a health check that tells a kubelet when to restart a container. Liveness probes help catch locks where an application seems to be running but can not proceed. Implementing a liveness probe in a deployment is a start to making a self-healing application.

An application may not immediately be ready to accept traffic when its container first starts; a readiness probe informs Kubernetes when it is okay to start sending traffic to a container after it boots. For example, a container of a java application might take minutes to load and may not be ready to accept traffic until it’s fully running. In this scenario, the readiness probe mitigates long loading times.

There are three options for liveness and readiness probes:
<ul>
<li>exec - takes a command, an exit code of 0 is success</li>
<li>httpGet - performs an http get, a 200-399 status is good</li>
<li>tcpSocket - a successful connection to a specified port is success</li>
</ul>

Liveness and readiness probes are similarly configured for each container in a pod. For example the following pod manifest has an <code>nginx</code>. container with a liveness probe that runs an http get to the root path to port 80. If the <code>nginx</code> web server replies with a 200 - 399 code then the pod is alive. The liveness probe waits 10 seconds before the first check and periodically checks every 20 seconds.

<pre class="wp-block-code"><code>
apiVersion: v1
kind: Pod
metadata:
  name: ready-pod
  labels:
    app: ready-pod
spec:
  containers:
  - name: nginx
    image: nginx:latest
    livenessProbe:
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 10
      periodSeconds: 20
</code></pre>

Learn more about <strong><a href="https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/">Liveness and Readiness Probes</a></strong>.


<h2>Container Logging</h2>

Kubernetes retrieves container logs from a container’s standard output and standard error streams. The applications running in the container must output their logs to a place read by the container logging engine (commonly STDOUT).

You can retrieve container logs with the <code>kubectl logs pod_name</code> command. If the pod is a multi-container pod then the container must be specified with the <code>-c</code> option with the container name <code>kubectl logs pod_name -c container_name</code>

In this example, we create a pod called <code>logging-pod</code> that outputs the system date and time every second.

<pre class="wp-block-code"><code>$ kubectl run logging-pod --image=busybox:latest --command -- /bin/sh -c 'while true; do date; sleep 1; done'

pod/logging-pod created

$
</code></pre>

Then the container logs are retrieved with <code>kubectl logs</code>:

<pre class="wp-block-code"><code>$ kubectl logs logging-pod

Wed Apr 20 16:31:45 UTC 2022
Wed Apr 20 16:31:46 UTC 2022

$
</code></pre>

Here are other useful container logging options:
<ul>
<li><code>--previous</code> - retrieves container logs from the previous instantiation of a container, this is helpful for containers that are crash looping</li>
<li><code>-f</code> - streams container logs</li>
<li><code>--since</code> - prints logs since a specific time period e.g. --since 15m</li>
<li><code>--timestamps</code> - includes timestamps</li>
</ul>

Learn more about <strong><a href="https://kubernetes.io/docs/concepts/cluster-administration/logging/">container logging</a></strong>.


<h2>Monitoring Applications</h2>

For the CKAD exam, the scope of monitoring applications is only within the scope of Kubernetes (including the Kubernetes metrics server). Key metrics to monitor are resources such as CPU and memory along with deployments and their running pods. Labels filter for a specific application or domain during monitoring.

The Kubernetes metrics server is not installed with the Kubernetes installation using kubeadm. Install the Kubernetes metrics-server and learn more about the <strong><a href="https://github.com/kubernetes-sigs/metrics-server">metrics-server</a></strong>.

With the metrics server installed you can view the resources used reported by the kubelets on each pod with <code>kubectl top pods</code>:

<pre class="wp-block-code"><code>$ kubectl top pods -n kube-system

NAME                                       CPU(cores)   MEMORY(bytes)
coredns-64897985d-5gn7r                    1m           11Mi
coredns-64897985d-vbl5w                    1m           12Mi
etcd-ip-172-31-57-184                      15m          43Mi
kube-apiserver-ip-172-31-57-184            52m          276Mi
kube-controller-manager-ip-172-31-57-184   10m          45Mi
kube-proxy-rw4nw                           2m           10Mi
kube-scheduler-ip-172-31-57-184            3m           17Mi
metrics-server-6f7946fdd7-dq6hd            4m           13Mi
weave-net-tbhqd                            1m           45Mi
</code></pre>

Learn more about <strong><a href="https://kubernetes.io/docs/tasks/debug-application-cluster/resource-usage-monitoring/">the tools for monitoring resources on Kubernetes</a></strong>, <strong><a href="https://kubernetes.io/blog/2017/05/kubernetes-monitoring-guide/">monitoring</a></strong>, and the <strong><a href="https://kubernetes.io/docs/tasks/debug-application-cluster/resource-metrics-pipeline/#metrics-server">metrics server</a></strong>.


<h2>Debugging</h2>

Debugging running applications in Kubernetes starts by retrieving simple status information about the pods.
Here are a few places to start looking:
<ul>
<li>Pod status - is the pod running, pending, or crash looping?</li>
<li>Pod restart count - does the pod have many recent restarts?</li>
<li>Pod resources - is the pod requesting more resources available in its namespace or on its node?</li>
</ul>

Pod details are retrieved by introspecting the pod with <code>kubectl describe pod pod_name</code>.

Take a look at how common pod issues are debugged using the pod’s description.

Here are key segments in a description of a pending pod:

<pre class="wp-block-code"><code>$ kubectl describe pod busybox

Name:         busybox
Namespace:    default
...
Status:       Pending
IP:           10.32.0.5
IPs:
  IP:  10.32.0.5
Containers:
  myapp-container:
    Container ID:
    Image:         busyBOX
  ...
Conditions:
  Type              Status
  Initialized       True
  Ready             False
  ContainersReady   False
  PodScheduled      True
...
Events:
  Type     Reason         Age              From               Message
  ----     ------         ----             ----               -------
  Normal   Scheduled      10s              default-scheduler  Successfully assigned default/busybox to ubuntu
  Warning  InspectFailed  9s (x2 over 9s)  kubelet, ubuntu     Failed to apply default image tag "busyBOX": couldn't parse image reference "busyBOX": invalid reference format: repository name must be lowercase
  Warning  Failed         9s (x2 over 9s)  kubelet, ubuntu     Error: InvalidImageName

$
</code></pre>

The <code>Events</code> section of the description points out the image name is invalid. Correcting the image name will resolve this issue.

Here’s another pending pod:

<pre class="wp-block-code"><code>$ kubectl get pods

NAME    READY   STATUS    RESTARTS   AGE
nginx   0/1     Pending   0          2m15s

$
</code></pre>

This nginx pod has been pending for over 2 minutes.

Let’s look at the pod’s <code>Events</code>.

<pre class="wp-block-code"><code>$ kubectl describe pod nginx | grep -A4 Events

Events:
  Type     Reason            Age                  From               Message
  ----     ------            ----                 ----               -------
  Warning  FailedScheduling  76s (x4 over 3m54s)  default-scheduler  0/1 nodes are available: 1 Insufficient cpu.

$
</code></pre>

Not enough resources are available in the cluster. We can see if the pod is making a hard resource request by looking at the <code>Containers</code> section of the pod description.

<pre class="wp-block-code"><code>$ kubectl describe pod nginx | grep -A10 Containers

Containers:
  nginx:
    Image:      nginx
    Port:       <none>
    Host Port:  <none>
    Limits:
      cpu:     2
      memory:  2Gi
    Requests:
      cpu:        1500m
      memory:     1536Mi

$
</code></pre>

The pod is requesting 1500m of cpu. There are no nodes in the cluster with 1500m free of cpu so the pod stays pending. There are a few ways to resolve this issue: reduce the container’s cpu request, free up cpu on a cluster node, or add a new worker node to the cluster.

Learn more about <strong><a href="https://kubernetes.io/docs/tasks/debug-application-cluster/">application debugging</a></strong>.


<h2>Practice Drill</h2>

The pod defined in the <code>problem.yaml</code> below creates a pod meant to run a perpetual tail command in the <code>busybox</code> image. The pod, which runs an init container with the <code>alpine:latest</code> does not run as expected. Apply the manifest to your cluster, identify the problem and repair it so that the pod runs as expected.

Apply the <code>problem.yaml</code> manifest to your cluster using the following command:

<pre class="wp-block-code"><code>kubectl apply -f https://raw.githubusercontent.com/RX-M/bust-a-kube/master/workload-1/problem.yaml</code></pre>

The pod must be in the running state with all containers ready for this problem to be considered resolved.

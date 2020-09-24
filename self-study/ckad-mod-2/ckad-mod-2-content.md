<!-- CKAD Self-Study Mod 2 -->


<h2 class="fl-heading">Multi-Container Pods</h2>
<h1>Multi-Container Pods</h1>
# Multi-Container Pods
## Multi-Container Pods
### Multi-Container Pods
#### Multi-Container Pods
##### Multi-Container Pods

A pod may run one or more containers. Multi-container pods are tightly coupled in that the containers are co-located, co-scheduled and the containers share the same <code>network</code>, <code>uts</code>, and <code>ipc</code> namespaces. There are three patterns of multi-container pods:

<li>Sidecar - sidecar containers extend and enhance the “main” container in the pod. The diagram below shows a web server container that saves its logs to a shared filesystem. The log saving sidecar container sends the webserver’s logs to a log aggregator.</li>

<img src="https://rx-m.com/wp-content/uploads/2020/09/sidecar-containers.png" alt=”sidecar” width=”500" height="200">
![Sidecar](https://rx-m.com/wp-content/uploads/2020/09/sidecar-containers.png "Sidecar")

<li>Ambassador - ambassador containers proxy a pod’s local connection to the outside world. The diagram shows a three-node Redis cluster (1, 2, 3). The ambassador container is a proxy that sends the appropriate reads and writes from the main application container to the Redis cluster. The main application container is configured to connect to a local Redis server since the two containers share the same uts namespace.</li>

<img src="https://rx-m.com/wp-content/uploads/2020/09/ambassador-containers.png" alt=”ambassador” width=”500" height="200">

<li>Adapter - adapter containers standardize and normalize output for remote monitoring systems that require standard data formats. The diagram below shows a monitoring adapter container running an agent that reads the main application’s data, processes it, then exports the normalized data to monitoring systems elsewhere in the network.</li>

<img src="https://rx-m.com/wp-content/uploads/2020/09/adapter-containers.png" alt=”adapter” width=”500" height="200">

A multi-container pod is created by specifying one or more additional container entries in a pod manifest. Shown below is an example of a multi-container pod with an <code>nginx</code> main container and an <code>fluent-bit</code> container sidecar in yaml. The nginx container writes its logs to a file at <code>/tmp/nginx/</code>, which is shared between all containers in the pod. The Fluent-Bit container reads the file from the shared directory and outputs it to its own standard output.

<img src="https://rx-m.com/wp-content/uploads/2020/09/sidecar-containers.png" alt=”sidecar” width=”500" height="200">
<img src="https://rx-m.com/wp-content/uploads/2020/09/sidecar-containers.png" alt=”sidecar”>
<img src="https://rx-m.com/wp-content/uploads/2020/09/sidecar-containers.png" alt=”sidecar” width=”500">
<img src="https://rx-m.com/wp-content/uploads/2020/09/sidecar-containers.png" alt=”sidecar” height="200">

<pre class="wp-block-code"><code>
apiVersion: v1
kind: Pod
metadata:
  name: sidecar
spec:
  containers:
  - name: nginx
    image: nginx:latest
    volumeMounts:
    - name: shared-vol
      mountPath: /tmp/nginx/    
    command:
    - /bin/sh
    - -c
    - nginx -g 'daemon off;' > /tmp/nginx/nginx.log
  - name: adapter
    image: fluent/fluent-bit
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

[Learn more about multi-container pod patterns](https://kubernetes.io/blog/2015/06/the-distributed-system-toolkit-patterns/).


# Liveness Probes and Readiness Probes

A liveness probe is a health check that tells a kubelet when to restart a container. Liveness probes help catch locks where an application seems to be running but can not proceed. Implementing a liveness probe in a deployment is a start to making a self-healing application.

An application may not immediately be ready to accept traffic when its container first starts; a readiness probe informs Kubernetes when it is okay to start sending traffic to a container after it boots. For example, a container of a java application might take minutes to load and may not be ready to accept traffic until it’s fully running. In this scenario, the readiness probe mitigates long loading times.

There are three options for liveness and readiness probes:
<li>exec - takes a command, an exit code of 0 is success</li>
<li>httpGet - performs an http get, a 200-399 status is good</li>
<li>tcpSocket - a successful connection to a specified port is success</li>

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

[Learn more about Liveness and Readiness Probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/).


# Container Logging

Kubernetes retrieves container logs from a container’s standard output and standard error streams. The applications running in the container must output their logs to a place read by the container logging engine (commonly STDOUT).

You can retrieve container logs with the <code>kubectl logs pod_name</code> command. If the pod is a multi-container pod then the container must be specified with the <code>-c</code> option with the container name <code>kubectl logs pod_name -c container_name</code>

In this example, we create a pod called <code>logging-pod</code> that outputs the system date and time every second.

<pre class="wp-block-code"><code>
$ kubectl run logging-pod --generator=run-pod/v1 --image=busybox:latest --command -- /bin/sh -c 'while true; do date; sleep 1; done'

pod/logging-pod created

$
</code></pre>

Then the container logs are retrieved with <code>kubectl logs</code>:

<pre class="wp-block-code"><code>
$ kubectl logs logging-pod

Mon Feb 24 23:25:16 UTC 2020
Mon Feb 24 23:25:17 UTC 2020
Mon Feb 24 23:25:18 UTC 2020
Mon Feb 24 23:25:19 UTC 2020
Mon Feb 24 23:25:20 UTC 2020

$
</code></pre>

Here are other useful container logging options:
<li><code>--previous</code> - retrieves container logs from the previous instantiation of a container, this is helpful for containers that are crash looping</li>
<li><code>-f</code> - streams container logs</li>
<li><code>--since</code> - prints logs since a specific time period e.g. --since 15m</li>
<li><code>--timestamps</code> - includes timestamps</li>

[Learn more about container logging](https://kubernetes.io/docs/concepts/cluster-administration/logging/).


# Monitoring Applications

For the CKAD exam, the scope of monitoring applications is only within the scope of Kubernetes (including the Kubernetes metrics server). Key metrics to monitor are resources such as CPU and memory along with deployments and their running pods. Labels filter for a specific application or domain during monitoring.

The Kubernetes metrics server is not installed with the Kubernetes installation using kubeadm. Install the Kubernetes metrics-server and [learn more about the metrics-server](https://github.com/kubernetes-sigs/metrics-server).

With the metrics server installed you can view the resources used reported by the kubelets on each pod with <code>kubectl top pods</code>:

<pre class="wp-block-code"><code>
$ kubectl top pods -n kube-system

NAME                                              CPU(cores)   MEMORY(bytes)   
coredns-6955765f44-dwv9q            2m                8Mi             
coredns-6955765f44-kjlc7               2m                8Mi             
etcd-ubuntu                                     11m               71Mi            
kube-apiserver-ubuntu                    27m               225Mi           
kube-controller-manager-ubuntu     8m                 36Mi            
kube-proxy-zhhs8                            1m                 14Mi            
kube-scheduler-ubuntu                    2m                 15Mi            
metrics-server-64cfb5b5d8-cxq7d   1m                 10Mi            
weave-net-fv9mb                             1m                 45Mi  
</code></pre>

Learn more about:
<li>[monitoring](https://kubernetes.io/blog/2017/05/kubernetes-monitoring-guide/)</li>
<li>[metrics server](https://kubernetes.io/docs/tasks/debug-application-cluster/resource-metrics-pipeline/#metrics-server).</li>


# Debugging

Debugging running applications in Kubernetes starts by retrieving simple status information about the pods.
Here are a few places to start looking:
Pod status - is the pod running, pending, or crash looping?
Pod restart count - does the pod have many recent restarts?
Pod resources - is the pod requesting more resources available in its namespace or on its node?

Pod details are retrieved by introspecting the pod with <code>kubectl describe pod pod_name</code>.

Take a look at how common pod issues are debugged using the pod’s description.

Here are key segments in a description of a pending pod:

<pre class="wp-block-code"><code>
$ kubectl describe pod busybox

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

<pre class="wp-block-code"><code>
$ kubectl get pods

NAME    READY   STATUS    RESTARTS   AGE
nginx   0/1     Pending   0          2m15s

$
</code></pre>

This nginx pod has been pending for over 2 minutes.
Let’s look at the pod’s <code>Events</code>.

<pre class="wp-block-code"><code>
$ kubectl describe pod nginx | grep -A4 Events

Events:
  Type     Reason            Age                  From               Message
  ----     ------            ----                 ----               -------
  Warning  FailedScheduling  76s (x4 over 3m54s)  default-scheduler  0/1 nodes are available: 1 Insufficient cpu.

$
</code></pre>

Not enough resources are available in the cluster. We can see if the pod is making a hard resource request by looking at the <code>Containers</code> section of the pod description.

<pre class="wp-block-code"><code>
$ kubectl describe pod nginx | grep -A10 Containers

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

[Learn more about application debugging](https://kubernetes.io/docs/tasks/debug-application-cluster/).


# Practice Drill

Create a multi-container pod named <code>ckad-sidecar</code> that has the following:
- Main container is named <code>nginx</code> and runs the <code>nginx</code> image
- Sidecar container is named <code>busybox</code> and runs the <code>busybox</code> image
- From the busybox container execute the command from shell <code>sleep 15 && wget -qO - http://ckad-sidecar | awk NR==4 && tail -f /dev/null</code>
- The pod restarts only on failure

Then obtain the container logs from the busybox container.

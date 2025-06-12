<!-- CKAD Self-Study Mod 5 -->

# Application Observability and Maintenance


# Understand API deprecations

As an ongoing project, new features will move through various stages of maturity as Kubernetes continues to grow. Every feature is served by a new API that is registered to the Kubernetes API server. There are 3 stages of maturity for these APIs:
<ul>
<li>Alpha - These APIs serve features that are highly experimental and not recommended for any production use. Kubernetes includes these Alpha APIs but they must be enabled by setting the appropriate feature gate on the API server. Alpha APIs can either move to beta status or get removed depending on how well the feature is received and used.</li>
<li>Beta - These APIs serve features that are still in highly active development but considered ready for production use. Beta APIs and features are enabled by default. Depending on their development cycles, Beta APIs can move up through multiple versions between Kubernetes releases or regress back to Alpha for further development.</li>
<li>Stable - These APIs are highly mature and do not see too many changes between versions. Stable APIs are enabled by default. At this point, stable APIs can only be retired if their feature is no longer necessary or if a new version of the API supplants its use.</li>
</ul>

You can view all of the APIs registered with your Kubernetes API server using the <code>kubectl api-versions</code> command:

<pre class="wp-block-code"><code>
$ kubectl api-versions

admissionregistration.k8s.io/v1
apiextensions.k8s.io/v1
apiregistration.k8s.io/v1
apps/v1
authentication.k8s.io/v1
authorization.k8s.io/v1

...
</code></pre>

API deprecations are a major event in the Kubernetes development cycle and come with many announcements. After a deprecation announcement, the affected features and/or APIs that get deprecated will:
<ul>
<li>Begin generating warnings in the CLI when invoked or submitted to the Kubernetes API server</li>
<li>Have their functionality disabled while continuing to generate warnings</li>
<li>Eventually get removed from the codebase all together, usually within 2-3 releases</li>
</ul>

<a href="https://kubernetes.io/docs/reference/using-api/deprecation-guide/">The API Deprecation page on the Kubernetes docs</a> shows a list of upcoming deprecations.

You will usually see warnings in the client regarding deprecations: here is one from prior to 1.24 regarding the previous ability to set resource requests in kubectl:

> N.B: Depending on your version, this command may not work anymore but it still illustrates a deprecation notice well.

<pre class="wp-block-code"><code>
$ kubectl run -o yaml --dry-run=client pod-with-reqs-limits --requests cpu=64m,memory=256Mi --limits cpu=256m,memory=512Mi --image nginx

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
</code></pre>

For existing manifests, the <code>apiVersion</code> line must be changed to a non-deprecated API version. This can be done with <code>kubectl-convert</code>, an optional kubectl plugin that examines older manifests and updates them to reflect the latest API versions.

The <code>kubectl-convert</code> plugin must be installed separate, with instructions found <a href="https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/#install-kubectl-convert-plugin">here</a>.

<pre class="wp-block-code"><code>
$ curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl-convert"

$ sudo install -o root -g root -m 0755 kubectl-convert /usr/local/bin/kubectl-convert
</code></pre>

Given the following file, which describes a deployment from v1.16 (when deployments were under the old <code>extensions</code> API):

<pre class="wp-block-code"><code>
$ cat old-deploy.yaml

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
</code></pre>

<code>kubectl-convert</code> can read the file and automatically make the necessary changes to update it to the latest api (<code>apps/v1</code> in 1.23):

<pre class="wp-block-code"><code>
$ kubectl-convert -f old-deploy.yaml --output-version apps/v1

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
</code></pre>

There are usually about 2-3 minor versions between a deprecation notice and the removal of a feature, so users typically have at least 6 months.

Learn more about and keep track of the <a href="https://kubernetes.io/docs/reference/using-api/deprecation-guide/" target="_blank" rel="noreferrer noopener">API Deprecation process</a> and how to <a href="https://kubernetes.io/docs/reference/using-api/deprecation-guide/#migrate-to-non-deprecated-apis" target="_blank" rel="noreferrer noopener">use kubectl-convert</a>.


# Implement probes and health checks

A liveness probe is a health check that tells a kubelet when to restart a container. Liveness probes help catch locks where an application seems to be running but can not proceed. Implementing a liveness probe in a deployment is a start to making a self-healing application.

An application may not immediately be ready to accept traffic from clients or peers when its container first starts; a readiness probe informs Kubernetes when it is okay to start sending traffic to a container after it boots. For example, a container of a java application might take minutes to load and may not be ready to accept traffic until it’s fully running.

In that same scenario, the startup probe mitigates long loading times, preventing liveness and/or readiness probes from failing too early.

There are several options for liveness, readiness, and startup probes:

<ul>
<li>exec - takes a command, an exit code of 0 is success</li>
<li>grpc – takes a port, uses the <a href="https://github.com/grpc/grpc/blob/master/doc/health-checking.md" target="_blank" rel="noreferrer noopener">API Deprecation process</a></li>
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

Learn more about <a href="https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/" target="_blank" rel="noreferrer noopener">Liveness and Readiness Probes</a>.


# Use built-in CLI tools to monitor Kubernetes applications

For the CKAD exam, the scope of monitoring applications is only within the scope of Kubernetes (including the Kubernetes metrics server). Key metrics to monitor are resources such as CPU and memory along with deployments and their running pods. Labels filter for a specific application or domain during monitoring.

The Kubernetes metrics server will be installed on your exam cluster(s) but is not installed with the Kubernetes installation using kubeadm. Install the Kubernetes metrics-server on your practice cluster and learn more about the <a href="https://github.com/kubernetes-sigs/metrics-server" target="_blank" rel="noreferrer noopener">metrics-server</a>.

With the metrics server installed you can view the resources used reported by the kubelets on each pod with <code>kubectl top pods</code>:

<pre class="wp-block-code"><code>$ kubectl top pods -n kube-system

NAME                               CPU(cores)   MEMORY(bytes)   
cilium-drdlf                       12m          315Mi           
cilium-operator-65496b9554-tzh4g   3m           98Mi            
coredns-7db6d8ff4d-2sgbz           1m           13Mi            
coredns-7db6d8ff4d-p2mh8           1m           57Mi            
etcd-labsys                        15m          97Mi            
kube-apiserver-labsys              58m          289Mi           
kube-controller-manager-labsys     12m          63Mi            
kube-proxy-zw9lj                   2m           62Mi            
kube-scheduler-labsys              2m           65Mi            
metrics-server-86776f5749-pj25p    4m           13Mi
</code></pre>

Learn more about <a href="https://kubernetes.io/docs/tasks/debug-application-cluster/resource-usage-monitoring/" target="_blank" rel="noreferrer noopener">the tools for monitoring resources on Kubernetes</a>, <a href="https://kubernetes.io/blog/2017/05/kubernetes-monitoring-guide/" target="_blank" rel="noreferrer noopener">monitoring</a>, and the <a href="https://kubernetes.io/docs/tasks/debug-application-cluster/resource-metrics-pipeline/#metrics-server" target="_blank" rel="noreferrer noopener">metrics server</a>.


# Utilize container logs

Kubernetes retrieves container logs from a container’s standard output and standard error streams. The applications running in the container must output their logs to a place read by the container logging engine (commonly STDOUT).

You can retrieve container logs with the <code>kubectl logs pod_name</code> command. If the pod is a multi-container pod then the container must be specified with the <code>-c</code> option with the container name <code>kubectl logs pod_name -c container_name</code>

In this example, we create a pod called <code>logging-pod</code> that outputs the system date and time every second.

<pre class="wp-block-code"><code>
$ kubectl run logging-pod --image=busybox:latest --command -- /bin/sh -c 'while true; do date; sleep 1; done'

pod/logging-pod created
</code></pre>

Then the container logs are retrieved with <code>kubectl logs</code>:

<pre class="wp-block-code"><code>
$ kubectl logs logging-pod

Sat Jul 13 01:08:34 UTC 2024
Sat Jul 13 01:08:35 UTC 2024
</code></pre>

Here are other useful container logging options:
<ul>
<li><code>--previous</code> - retrieves container logs from the previous instantiation of a container, this is helpful for containers that are crash looping</li>
<li><code>-f</code> - streams container logs</li>
<li><code>--since</code> - prints logs since a specific time period e.g. --since 15m</li>
<li><code>--timestamps</code> - includes timestamps</li>
</ul>

Learn more about <a href="https://kubernetes.io/docs/concepts/cluster-administration/logging/" target="_blank" rel="noreferrer noopener">container logging</a>.


# Debugging in Kubernetes

Debugging running applications in Kubernetes starts by retrieving simple status information about the pods. Here are a few places to start looking:

<ul>
<li>Pod status - is the pod running, pending, or crash looping?</li>
<li>Pod restart count - does the pod have many recent restarts?</li>
<li>Pod resources - is the pod requesting more resources available in its namespace or on its node?</li>
</ul>

Pod details are retrieved by introspecting the pod with <code>kubectl describe pod pod_name</code>.

Take a look at how common pod issues are debugged using the pod’s description.

Here's a description of a pending pod:

<pre class="wp-block-code"><code>
$ kubectl describe pod in-cluster-client-pod

Name:             in-cluster-client-pod
Namespace:        default
Priority:         0
Service Account:  default
Node:             labsys/10.0.2.15
Start Time:       Sat, 13 Jul 2024 01:12:42 +0000
Labels:           run=in-cluster-client-pod
Annotations:      <none>
Status:           Pending
IP:               10.0.0.232
IPs:
  IP:  10.0.0.232
Containers:
  in-cluster-client-pod:
    Container ID:   
    Image:          busyBOX:latest
    Image ID:       
    Port:           <none>
    Host Port:      <none>
    State:          Waiting
      Reason:       InvalidImageName
    Ready:          False
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-h2d7z (ro)
Conditions:
  Type                        Status
  PodReadyToStartContainers   True 
  Initialized                 True 
  Ready                       False 
  ContainersReady             False 
  PodScheduled                True 
Volumes:
  kube-api-access-h2d7z:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   BestEffort
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type     Reason         Age               From               Message
  ----     ------         ----              ----               -------
  Normal   Scheduled      10s               default-scheduler  Successfully assigned default/in-cluster-client-pod to labsys
  Warning  InspectFailed  9s (x2 over 10s)  kubelet            Failed to apply default image tag "busyBOX:latest": couldn't parse image name "busyBOX:latest": invalid reference format: repository name (library/busyBOX) must be lowercase
  Warning  Failed         9s (x2 over 10s)  kubelet            Error: InvalidImageName
</code></pre>

The <code>Events</code> section of the description points out the image name is invalid. Correcting the image name will resolve this issue.

Here’s another pending pod:

<pre class="wp-block-code"><code>
$ kubectl get pod nginx

NAME    READY   STATUS    RESTARTS   AGE
nginx   0/1     Pending   0          2m
</code></pre>

This nginx pod has been pending for over 2 minutes. Let’s look at the pod’s <code>Events</code>.

<pre class="wp-block-code"><code>
$ kubectl describe pod nginx | grep -A4 Events

Events:
  Type     Reason            Age   From               Message
  ----     ------            ----  ----               -------
  Warning  FailedScheduling  20s   default-scheduler  0/1 nodes are available: persistentvolumeclaim "local-pvc" not found. preemption: 0/1 nodes are available: 1 Preemption is not helpful for scheduling.
</code></pre>

No pvc (persistent volume claim) found. The pod is referencing one that does not exist (or does exist, but is not in the `Bound` state) so the pod stays pending. 

Learn more about <a href="https://kubernetes.io/docs/tasks/debug-application-cluster/" target="_blank" rel="noreferrer noopener">application debugging</a> and check out RX-M's <a href="https://github.com/RX-M/bust-a-kube" target="_blank" rel="noreferrer noopener">bust-a-kube</a> repo for troubleshooting scenarios you can test yourself with!


# Practice Drill

The pod defined in the <code>problem.yaml</code> below creates a pod meant to run a perpetual tail command in the <code>busybox</code> image. The pod, which runs an init container with the <code>alpine:latest</code> does not run as expected. Apply the manifest to your cluster, identify the problem and repair it so that the pod runs as expected.

Apply the <code>problem.yaml</code> manifest to your cluster using the following command:

<pre class="wp-block-code"><code>kubectl apply -f https://raw.githubusercontent.com/RX-M/bust-a-kube/master/workload-1/problem.yaml</code></pre>

The pod must be in the running state with all containers ready for this problem to be considered resolved.

This practice drill comes from RX-M's own <a href="https://github.com/RX-M/bust-a-kube" target="_blank" rel="noreferrer noopener">bust-a-kube</a> repository. You can visit the repo to get additional practice to help you prepare.
<!-- CKA Self-Study Mod 5 -->

# Evaluate cluster and node logging

Most Kubernetes clusters that follow the same pattern of deployment as kubeadm (or are created by kubeadm itself) have the following logging setup in an "out of the box" state:

<ul>
<li>The control plane components (kube-apiserver, kube-scheduler, kube-controller-manager) run as pods, accessible through the container runtime and <code>kubectl logs</code></li>
<li>The kubelet runs as a system service, accessible through the host OS's init system and its log tools (<code>systemctl</code> and <code>journalctl</code> on Linux)</li>
</ul>

To evaluate whether your current cluster follows the control plane as pods pattern, look for pods in the <code>kube-system</code> namespace:

<pre class="wp-block-code"><code>
~$ kubectl get pods -n kube-system
NAME                                       READY   STATUS    RESTARTS     AGE
coredns-787d4945fb-jlgcp                   1/1     Running   1 (8h ago)   32h
coredns-787d4945fb-rd8d6                   1/1     Running   1 (8h ago)   32h
etcd-ip-172-31-18-190                      1/1     Running   1 (8h ago)   32h
kube-apiserver-ip-172-31-18-190            1/1     Running   1 (8h ago)   32h
kube-controller-manager-ip-172-31-18-190   1/1     Running   1 (8h ago)   32h
kube-proxy-nn7fp                           1/1     Running   1 (8h ago)   32h
kube-scheduler-ip-172-31-18-190            1/1     Running   1 (8h ago)   32h
weave-net-hgshk                            2/2     Running   3 (8h ago)   32h
</code></pre>

Depending on your cluster setup, there may be additional logging agents running as DaemonSets. These act as node-level logging agents that may read the container runtime logs or other system logs. This pattern is not universal, but depending on the type of logger it is very likely you would find them as daemonsets.

To view the Daemonsets in your cluster, search for the `daemonset` API resource type in all namespaces of your cluster:

<pre class="wp-block-code"><code>
~$ kubectl get daemonsets -A
NAMESPACE     NAME         DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
kube-system   kube-proxy   1         1         1       1            1           kubernetes.io/os=linux   32h
kube-system   weave-net    1         1         1       1            1           <none>                   32h
</code></pre>

Finally, your pods may implement an additional container specifically for logging purposes. To find these pods, simply look for pods that report more than one container in the <code>kubectl get pod</code> output:

<pre class="wp-block-code"><code>
~$ kubectl get pods

NAME               READY   STATUS    RESTARTS   AGE
webserver-logged   2/2     Running   0          33m
</code></pre>

You will have to look into each pod and see if they are configured as such. This information is available from the <code>kubectl describe pod</code> output's "Containers" section:

<pre class="wp-block-code"><code>
~$ kubectl describe pods webserver-logged

Name:             webserver-logged
Namespace:        default
Priority:         0
Service Account:  default
Node:             ip-172-31-18-190/172.31.18.190
Start Time:       Thu, 09 Feb 2023 01:20:37 +0000
Labels:           app=website
                  component=webserver
                  vendor=rx-m
Annotations:      <none>
Status:           Running
IP:               10.32.0.4
IPs:
  IP:  10.32.0.4
Containers:
  webserver:
    Container ID:  containerd://a5809913eb9cb2d3f74705b4e071d9b773ec5b7884d73ad1bc3bd10e7ed42295
    Image:         nginx:1.23.3
    Image ID:      docker.io/library/nginx@sha256:c54fb26749e49dc2df77c6155e8b5f0f78b781b7f0eadd96ecfabdcdfa5b1ec4
    Port:          <none>
    Host Port:     <none>
    Args:
      /bin/sh
      -c
      nginx -g "daemon off;" > /log/server.log 2>&1
    State:          Running
      Started:      Thu, 09 Feb 2023 01:20:37 +0000
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /log from log (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-w5wwv (ro)
  fluent-bit:
    Container ID:  containerd://93848fa95384c83e742842c15bcd7108f3727d76606d12ecd5a15cf1ec93b4ad
    Image:         fluent/fluent-bit:1.9.7
    Image ID:      docker.io/fluent/fluent-bit@sha256:14ffcdfbafba6145348effd470793bca0755ee7d2fed7e75c7689d0ae3e8d135
    Port:          <none>
    Host Port:     <none>
    Command:
      /fluent-bit/bin/fluent-bit
      -i
      tail
      -p
      path=/log/*.log
      -o
      stdout
    State:          Running
      Started:      Thu, 09 Feb 2023 01:20:37 +0000
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /log from log (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-w5wwv (ro)
</code></pre>

[Learn more about the logging architecture present in most Kubernetes deployments here](https://kubernetes.io/docs/concepts/cluster-administration/logging/)


# Understand how to monitor applications

Kubernetes is a distributed system by nature: its components and workloads potentially run over many machines. In order to efficiently monitor those programs, you need to implement some kind of central monitoring system. The most monitoring that can be performed on a kubeadm-based cluster out of the box is through the API, specifically:

<ul>
<li>Finding API resources running in the cluster using <code>kubectl get</code></li>
<li>Describing the resource availability of each node using <code>kubectl describe node</code></li>
</ul>

While tools like Prometheus and Grafana are popular additions to any Kubernetes cluster, there is a subproject of Kubernetes called the Metrics Server. The Metrics Server contacts each of the Kubelets (which provide the resource availability stats in the node describe outputs) and publishes them to the Kubernetes API. These metrics can then be used with things like HorizontalPodAutoscalers to automatically scale applications based on reported CPU or memory use.

A Kubernetes cluster with the metric server enabled enables the <code>kubectl top</code> command.

You can use the <code>kubectl top nodes</code> to view rudimentary memory and CPU use statistics on a per node basis:

<pre class="wp-block-code"><code>
~$ kubectl top nodes

NAME               CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
ip-172-31-18-190   134m         6%     1872Mi          49%
</code></pre>

You can also use the <code>kubectl top pods</code> to view the same memory and CPU use statistics on a per pod basis:

<pre class="wp-block-code"><code>
~$ kubectl top pods -A

NAMESPACE     NAME                                       CPU(cores)   MEMORY(bytes)
default       webserver-logged                           1m           5Mi
kube-system   coredns-787d4945fb-jlgcp                   2m           12Mi
kube-system   coredns-787d4945fb-rd8d6                   2m           12Mi
kube-system   etcd-ip-172-31-18-190                      17m          93Mi
kube-system   kube-apiserver-ip-172-31-18-190            32m          239Mi
kube-system   kube-controller-manager-ip-172-31-18-190   11m          41Mi
kube-system   kube-proxy-nn7fp                           1m           13Mi
kube-system   kube-scheduler-ip-172-31-18-190            3m           17Mi
kube-system   metrics-server-5bbd5b4cd4-fzhk9            3m           14Mi
kube-system   weave-net-hgshk                            1m           63Mi
</code></pre>

You can also use the <code>kubectl top pods</code> command with the <code>--containers</code> option to view the same memory and CPU use statistics on a per container basis:

<pre class="wp-block-code"><code>
~$ kubectl top pods -A --containers

NAMESPACE     POD                                        NAME                      CPU(cores)   MEMORY(bytes)
default       webserver-logged                           fluent-bit                1m           2Mi
default       webserver-logged                           webserver                 0m           2Mi
kube-system   coredns-787d4945fb-jlgcp                   coredns                   2m           12Mi
kube-system   coredns-787d4945fb-rd8d6                   coredns                   2m           12Mi
kube-system   etcd-ip-172-31-18-190                      etcd                      17m          93Mi
kube-system   kube-apiserver-ip-172-31-18-190            kube-apiserver            32m          239Mi
kube-system   kube-controller-manager-ip-172-31-18-190   kube-controller-manager   11m          41Mi
kube-system   kube-proxy-nn7fp                           kube-proxy                1m           13Mi
kube-system   kube-scheduler-ip-172-31-18-190            kube-scheduler            3m           17Mi
kube-system   metrics-server-5bbd5b4cd4-fzhk9            metrics-server            3m           14Mi
kube-system   weave-net-hgshk                            weave                     1m           46Mi
kube-system   weave-net-hgshk                            weave-npc                 1m           17Mi
</code></pre>

It is important to remember that the Kubernetes metrics server only provides more basic metrics reporting. For more sophisticated metrics output, there is the custom metrics API. The custom metrics API is populated with query results from one of many adapters tailored for specific metrics backends (like Prometheus). As the CKA stays within the bounds of Kubernetes itself, these are out of scope for the exam.

[Learn more about the Monitoring approach to Kubernetes here](https://kubernetes.io/docs/tasks/debug/debug-cluster/resource-usage-monitoring/). There is also [additional information on how monitoring solutions work with Kubernetes here](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/#autoscaling-on-multiple-metrics-and-custom-metrics).


# Manage container stdout & stderr logs

Kubernetes can display container logs on request using the <code>kubectl logs</code> command. In order for <code>kubectl logs</code> to work properly, two conditions must be met:

<ul>
<li>At least one of the containers in the pod are outputting logs to STDOUT</li>
<li>The container runtime is configured to store container logs locally</li>
</ul>

It is common for containerized applications to output all of their logs to STDOUT and most container runtimes default to capturing container outputs to a JSON file. A majority of the time, <code>kubectl logs</code> will work without any additional configuration.

If you have a different setup, such as the application outputting to a file, then an additional container for the pod can work to read the resulting file and output the contents to its own STDOUT stream.

The following pod spec launches a webserver that outputs to a log file, then uses an adapter container to read the generated file:

<pre class="wp-block-code"><code>
~$ cat testpod.yaml

apiVersion: v1
kind: Pod
metadata:
  name: webserver-logged
  labels:
    app: website
    component: webserver
    vendor: rx-m
spec:
  volumes:
  - name: log
    emptyDir: {}
  containers:
  - name: webserver
    image: nginx:1.23.3
    args:
    - /bin/sh
    - -c
    - nginx -g "daemon off;" > /log/server.log 2>&1
    volumeMounts:
    - name: log
      mountPath: /log
  - name: fluent-bit
    image: fluent/fluent-bit:1.9.7
    command:
    - /fluent-bit/bin/fluent-bit
    - -i
    - tail
    - -p
    - path=/log/*.log
    - -o
    - stdout
    volumeMounts:
    - name: log
      mountPath: /log
</code></pre>

Applying the YAML, the resulting pod shows two containers:

<pre class="wp-block-code"><code>
~$ kubectl apply -f testpod.yaml

pod/webserver-logged created

~$ kubectl get pods -o wide

NAME               READY   STATUS    RESTARTS   AGE   IP          NODE               NOMINATED NODE   READINESS GATES
webserver-logged   2/2     Running   0          3s    10.32.0.4   ip-172-31-18-190   <none>           <none>

~$ curl 10.32.0.4:80

... Welcome to nginx!...

</code></pre>

Checking the logs, you find that even though the webserver is not returning any output when queried for logs directly, the adapter container running the log agent shows the activity received.

<pre class="wp-block-code"><code>
~$ kubectl logs webserver-logged -c webserver

~$ kubectl logs webserver-logged -c fluent-bit

Fluent Bit v1.9.7
* Copyright (C) 2015-2022 The Fluent Bit Authors
* Fluent Bit is a CNCF sub-project under the umbrella of Fluentd
* https://fluentbit.io

[2023/02/09 01:20:37] [ info] [fluent bit] version=1.9.7, commit=265783ebe9, pid=1
[2023/02/09 01:20:37] [ info] [storage] version=1.2.0, type=memory-only, sync=normal, checksum=disabled, max_chunks_up=128
[2023/02/09 01:20:37] [ info] [cmetrics] version=0.3.5
[2023/02/09 01:20:37] [ info] [sp] stream processor started
[2023/02/09 01:20:37] [ info] [input:tail:tail.0] inotify_fs_add(): inode=1036058 watch_fd=1 name=/log/server.log
[2023/02/09 01:20:37] [ info] [output:stdout:stdout.0] worker #0 started
[0] tail.0: [1675905647.814285446, {"log"=>"10.32.0.1 - - [09/Feb/2023:01:20:47 +0000] "GET / HTTP/1.1" 200 615 "-" "curl/7.81.0" "-""}]

~$
</code></pre>

[Learn more about how additional containers can help provide extra logging capabitilies without the need to rebuild application containers](https://kubernetes.io/blog/2015/06/the-distributed-system-toolkit-patterns/#example-3-adapter-containers).


# Troubleshoot Application Failure

Applications running under Kubernetes run in containers within pods. There are several ways to troubleshoot application failures:

<ul>
<li>Reading the application container’s logs using <code>kubectl logs</code></li>
<li>Viewing events associated with the pod using <code>kubectl events</code></li>
<li>Interacting with the application container directly using <code>kubectl exec</code></li>
</ul>

<pre class="wp-block-code"><code>
$ kubectl logs debugapp

2020-02-28 16:05:48+00:00 [Note] [Entrypoint]: Entrypoint script for MySQL Server 1:10.4.12+maria~bionic started.
2020-02-28 16:05:49+00:00 [Note] [Entrypoint]: Switching to dedicated user 'mysql'
2020-02-28 16:05:49+00:00 [Note] [Entrypoint]: Entrypoint script for MySQL Server 1:10.4.12+maria~bionic started.
2020-02-28 16:05:49+00:00 [ERROR] [Entrypoint]: Database is uninitialized and password option is not specified
	You need to specify one of MYSQL_ROOT_PASSWORD, MYSQL_ALLOW_EMPTY_PASSWORD and MYSQL_RANDOM_ROOT_PASSWORD

$
</code></pre>

[Learn more about troubleshooting and debugging your applications running on Kubernetes](https://kubernetes.io/docs/tasks/debug-application-cluster/debug-application/).



# Troubleshoot cluster component failure

## Control Plane Failures

Control plane components like the kube-apiserver, kube-scheduler, etcd, and kube-controllermanager all run in static pods in a kubeadm-bootstrapped cluster. Control plane components manifests are found in the master node’s kubelet static path directory, the default is: <code>/etc/kubernetes/manifests</code>.

Since the control plane components in a cluster run as pods, <code>kubectl logs</code> and <code>kubectl events</code> provide insight on any activity and possible issues these control plane components face. In clusters where control plane components run as system-level daemons, using <code>systemctl</code> and <code>journalctl</code> will provide the same approach.

<pre class="wp-block-code"><code>
$ kubectl logs -n kube-system kube-scheduler-$(hostname)

I0225 19:54:25.957861       1 serving.go:312] Generated self-signed cert in-memory
W0225 19:54:26.142234       1 configmap_cafile_content.go:102] unable to load initial CA bundle for: "client-ca::kube-system::extension-apiserver-authentication::client-ca-file" due to: configmap "extension-apiserver-authentication" not found
W0225 19:54:26.142423       1 configmap_cafile_content.go:102] unable to load initial CA bundle for: "client-ca::kube-system::extension-apiserver-authentication::requestheader-client-ca-file" due to: configmap "extension-apiserver-authentication" not found
W0225 19:54:26.157329       1 authorization.go:47] Authorization is disabled
W0225 19:54:26.157413       1 authentication.go:92] Authentication is disabled
I0225 19:54:26.157435       1 deprecated_insecure_serving.go:51] Serving healthz insecurely on [::]:10251
I0225 19:54:26.159618       1 configmap_cafile_content.go:205] Starting client-ca::kube-system::extension-apiserver-authentication::requestheader-client-ca-file
I0225 19:54:26.159658       1 shared_informer.go:197] Waiting for caches to sync for client-ca::kube-system::extension-apiserver-authentication::requestheader-client-ca-file
I0225 19:54:26.159628       1 secure_serving.go:178] Serving securely on 127.0.0.1:10259

$
</code></pre>

[Learn more about debugging your control plane components](https://kubernetes.io/docs/tasks/debug-application-cluster/debug-cluster/).


## Worker Node Failures

A node that runs pods in a Kubernetes cluster is represented by a kubelet. Each kubelet requires a container runtime (like Docker) to run containers. Since the kubelet and container runtime run as system agents, you must use host-level tools like <code>systemctl</code> or <code>journalctl</code> to view their logs.

<pre class="wp-block-code"><code>
$ sudo systemctl status kubelet

● kubelet.service - kubelet: The Kubernetes Node Agent
   Loaded: loaded (/lib/systemd/system/kubelet.service; enabled; vendor preset: enabled)
  Drop-In: /etc/systemd/system/kubelet.service.d
           └─10-kubeadm.conf
   Active: activating (auto-restart) (Result: exit-code) since Fri 2020-02-28 08:08:03 PST; 8s ago
     Docs: https://kubernetes.io/docs/home/
  Process: 2175 ExecStart=/usr/bin/kubelet $KUBELET_KUBECONFIG_ARGS $KUBELET_CONFIG_ARGS $KUBELET_KUBE
 Main PID: 2175 (code=exited, status=255)

Feb 28 08:08:03 labsys systemd[1]: kubelet.service: Main process exited, code=exited, status=255/n/a
Feb 28 08:08:03 labsys systemd[1]: kubelet.service: Unit entered failed state.
Feb 28 08:08:03 labsys systemd[1]: kubelet.service: Failed with result 'exit-code'.

$ journalctl -u kubelet | tail -5

Feb 28 08:09:15 labsys kubelet[2771]: I0228 08:09:15.417176    2771 server.go:641] --cgroups-per-qos enabled, but --cgroup-root was not specified.  defaulting to /
Feb 28 08:09:15 labsys kubelet[2771]: F0228 08:09:15.417372    2771 server.go:273] failed to run Kubelet: running with swap on is not supported, please disable swap! or set --fail-swap-on flag to false. /proc/swaps contained: [Filename                                Type                Size        Used        Priority /dev/dm-1                               partition        4194300        0        -1]
Feb 28 08:09:15 labsys systemd[1]: kubelet.service: Main process exited, code=exited, status=255/n/a
Feb 28 08:09:15 labsys systemd[1]: kubelet.service: Unit entered failed state.
Feb 28 08:09:15 labsys systemd[1]: kubelet.service: Failed with result 'exit-code'.

$
</code></pre>

Kubelets (and thus worker nodes) require functional container runtimes to operate pods. In default installations, they also need to have system swap disabled.

[Learn more about debugging your worker nodes and kubelets](https://kubernetes.io/docs/tasks/debug-application-cluster/debug-cluster/#worker-nodes).


# Troubleshoot Networking

Every pod running in a Kubernetes cluster must have its own IP address. Pods receive their unique IPs from the cluster Container Network Interface (CNI) plugin. In order to properly use a CNI plugin, each Kubelet must be configured to expect CNI plugins in their systemd service.

Without a functional CNI, the kubelet is unable to configure the container runtime’s network and does not report a “ready” status to the cluster:

<pre class="wp-block-code"><code>
$ sudo systemctl status kubelet

● kubelet.service - kubelet: The Kubernetes Node Agent
   Loaded: loaded (/lib/systemd/system/kubelet.service; enabled; vendor preset: enabled)
  Drop-In: /etc/systemd/system/kubelet.service.d
           └─10-kubeadm.conf
   Active: active (running) since Fri 2020-02-28 08:50:27 PST; 5min ago
     Docs: https://kubernetes.io/docs/home/
 Main PID: 19302 (kubelet)
    Tasks: 16 (limit: 512)
   Memory: 45.2M
      CPU: 3.761s
   CGroup: /system.slice/kubelet.service
           └─19302 /usr/bin/kubelet --bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf --config=/var/lib/kubelet/config.yaml --cgroup-driver=cgroupfs

Feb 28 08:55:58 labsys kubelet[19302]: E0228 08:55:58.668145   19302 kubelet.go:2183] Container runtime network not ready: NetworkReady=false reason:NetworkPluginNotReady message:docker: network plugin i
Feb 28 08:56:02 labsys kubelet[19302]: W0228 08:56:02.452412   19302 cni.go:237] Unable to update cni config: no networks found in /etc/cni/net.d

$
</code></pre>

Other symptoms of a non-functional or misconfigured CNI plugin include:

Pods receiving a Docker local IP
New pods staying in the ContainerCreating phase even if the image is pulled

Other networking failures may involve factors within the cluster’s underlying infrastructure, such as firewall rules or full network partitions preventing communication.

[Learn more about debugging networking at various levels of the cluster](https://kubernetes.io/docs/tasks/debug-application-cluster/debug-cluster/#a-general-overview-of-cluster-failure-modes) and [checking if the kube-proxy is working](https://kubernetes.io/docs/tasks/debug-application-cluster/debug-service/#is-the-kube-proxy-working).


# Practice Drill

Run the following command. This command will start a pod that will not run:

<code>kubectl run --restart Never --image redis:0.2 redispod</code>

Retrieve the events associated with this pod (listed in any order) and write them to a file: <code>/tmp/troubleshooting-answer.txt</code>

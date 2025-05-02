<!-- CKA Self-Study Mod 5 -->


# Troubleshoot clusters and nodes


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
</code></pre>

[Learn more about debugging your control plane components](https://kubernetes.io/docs/tasks/debug-application-cluster/debug-cluster/).


## Node Failures

A node that runs pods in a Kubernetes cluster is represented by a kubelet. Each kubelet requires a container runtime (like containerd) to run containers. Since the kubelet and container runtime run as system agents, you must use host-level tools like <code>systemctl</code> or <code>journalctl</code> to view their logs.

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

Logs on worker nodes are typically found in the following paths:

<li><code>/var/log/kubelet.log</code> - logs from the kubelet</li>
<li><code>/var/log/kube-proxy.log</code> - logs from <code>kube-proxy</code></li>
<li><code>/var/log/containerd.log</code> - logs from the conainerd container runtime</li>
<li><code>/var/log/syslog</code> - general messages for the node</li>
<li><code>/var/log/kern.log</code> - kernel logs</li>


### Debugging a Node using kubectl

The <code>kubectl debug</code> command can deploy a pod to a node that you want to troubleshoot so that you do not have to use SSH to access it. The <code>debug</code> command will open an interactive shell in a pod running on the target node and map the host's filesystem under <code>/host</code> in the pod so that you have access to the log paths listed just above.

<pre class="wp-block-code"><code>
$ kubectl debug node/worker1 -it --image=rxmllc/tools --profile=sysadmin

Creating debugging pod node-debugger-worker1-8pzlk with container debugger on node worker1.
If you don't see a command prompt, try pressing enter.

/ # chroot /host

root@worker1:/#
</code></pre>

Learn more about <a href="https://kubernetes.io/docs/tasks/debug/debug-cluster/kubectl-node-debug/" target="_blank" rel="noreferrer noopener">debugging Kubernetes nodes</a> with the <code>debug</code> command.


### Using crictl for Debugging on Nodes

The <code>crictl</code> CLI can be installed on your nodes from the cri-tools releases page and configured to talk to containerd with a config file. The commands below illustrate how to do this:

<pre class="wp-block-code"><code>
$ crictl_ver=$(curl -s https://api.github.com/repos/kubernetes-sigs/cri-tools/releases/latest | grep tag_name | cut -d '"' -f 4 | cut -b 2-)

$ wget "https://github.com/kubernetes-sigs/cri-tools/releases/download/v${crictl_ver}/crictl-v${crictl_ver}-linux-amd64.tar.gz"

$ sudo tar zxvf "crictl-v${crictl_ver}-linux-amd64.tar.gz" -C /usr/local/bin

$ rm -f "crictl-v${crictl_ver}-linux-amd64.tar.gz"

$ echo "runtime-endpoint: unix:///run/containerd/containerd.sock" | sudo tee /etc/crictl.yaml
</code></pre>

crictl has several commands you can use to interact with pods, containers, and images on a node:

<li><code>crictl pods</code> - lists pods</li>
<li><code>crictl ps</code> - lists containers</li>
<li><code>crictl images</code> - lists images</li>
<li><code>crictl exec</code> - execute a command in a running container</li>
<li><code>crictl logs</code> - get a container's logs</li>


# Troubleshoot cluster components

The Kubernetes is composed of several components that exhibit differing symptoms when misbehaving.


### kube-apiserver

The most obvious symptom of API Server loss is the inability to run any <code>kubectl</code> command. However, existing application pods and services should continue to work normally (unless they depend on the Kubernetes API).

To check its status, you may need to use a node-level tool like <code>ps</code> or query the node's container runtime (using <code>crictl</code> for example) for the status of the <code>kube-apiserver-&lt;hostname></code> pod. In clusters initialized with kubeadm, this is often caused by a malformed or missing <code>kube-apiserver.yaml</code> in the control plane node's kubelet static pod path.


### kube-scheduler

Pod will not be scheduled and will stay in a <code>pending</code> state. This includes newly created pods and pods that controllers (Deployments, StatefulSets, etc.) are replacing during a rolling update.

The <code>kubectl get events</code> command will not show any events because Kubernetes can be fully functional without a scheduler (assuming users schedule pods manually using <code>nodeName</code> in their pod specs). Examine the pod's logs to determine the cause or look for a malformed or missing <code>kube-scheduler.yaml</code> in the control plane node's kubelet static pod path.


### kube-controller-manager

Reconciliation will stop functioning. For application pods this means if a pod is lost from a set of replicas, it will not be replaced. For nodes, a failing node will remain in the <code>Ready</code> state despite being offline or down because the node controller is not updating node statuses.

The <code>kubectl get events</code> command will not show any events because Kubernetes can be fully functional without the Controller Manager. Examine the pod's logs to determine the cause or look for a malformed or missing <code>kube-controller-manager.yaml</code> in the control plane node's kubelet static pod path.


### kube-proxy

Traffic may not be delivered to application pods on a node where kube-proxy has failed. This is because iptables or ipvs tables is not being updated. As the entries continue to age, the problem will worsen as service backend targets become invalid. In a relatively static cluster a failed kube-proxy may not be obvious but in a highly volatile cluster the routing mesh on the affected node will be outdated quickly.

Because kube-proxy runs as a DaemonSet-based set of pods, it should record a mismatch of desired vs current pods if one is missing, not ready, crashing, etc. Examine the related pod's logs and confirm with <code>iptables</code> or <code>ipvsadm</code> commands. Learn more about <a href="https://kubernetes.io/docs/tasks/debug-application-cluster/debug-service/#is-the-kube-proxy-working" target="_blank" rel="noreferrer noopener">checking if the kube-proxy is working</a>.


### kubelet

Symptoms may include:

<li>Pending pods - pods may be scheduled here before the node controller detects a failure and sets the node to the <code>NotReady</code> state</li>
<li>Pods stuck in the <code>Terminating</code> state - pods cannot be fully terminated unless the kubelet confirms the deletion</li>

The <code>kubectl get events</code> command will show the event when the node controller marked the node as unhealthy. Access system-level logs and events for cause.


## Accessing Component Logs

Most Kubernetes clusters that follow the same pattern of deployment as kubeadm (or are created by kubeadm itself) have the following logging setup in an "out of the box" state:

<li>Cluster components (kube-apiserver, kube-scheduler, kube-controller-manager, kube-proxy, operators, and CNI and CSI plugins) run as pods, accessible through the container runtime and <code>kubectl logs</code></li>
<li>The kubelet runs as a system service, accessible through the host OS's init system and its log tools (<code>systemctl</code> and <code>journalctl</code> on Linux)</li>

To evaluate whether your current cluster follows the control plane as pods pattern, look for pods in the <code>kube-system</code> namespace:

<pre class="wp-block-code"><code>
~$ kubectl get pods -n kube-system

NAME                                      READY   STATUS    RESTARTS      AGE
cilium-ngkpt                              1/1     Running   1 (11m ago)   17m
cilium-operator-65496b9554-vn6z7          1/1     Running   1 (11m ago)   17m
coredns-7db6d8ff4d-gqnxv                  1/1     Running   0             17m
coredns-7db6d8ff4d-pr78k                  1/1     Running   0             17m
etcd-ip-172-31-21-39                      1/1     Running   4 (11m ago)   17m
kube-apiserver-ip-172-31-21-39            1/1     Running   1 (11m ago)   17m
kube-controller-manager-ip-172-31-21-39   1/1     Running   1 (11m ago)   17m
kube-proxy-2mjrc                          1/1     Running   1 (11m ago)   17m
kube-scheduler-ip-172-31-21-39            1/1     Running   1 (11m ago)   17m
</code></pre>

Depending on your cluster setup, there may be additional logging agents running as DaemonSets. These act as node-level logging agents that may read the container runtime logs or other system logs. This pattern is not universal, but depending on the type of logger it is very likely you would find them as daemonsets.

To view the Daemonsets in your cluster, search for the <code>daemonset</code> API resource type in all namespaces of your cluster:

<pre class="wp-block-code"><code>
~$ kubectl get daemonsets -A

NAMESPACE     NAME         DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
kube-system   cilium       1         1         1       1            1           kubernetes.io/os=linux   17m
kube-system   kube-proxy   1         1         1       1            1           kubernetes.io/os=linux   18m
</code></pre>

Finally, your pods may implement an additional container specifically for logging purposes. To find these pods, simply look for pods that report more than one container in the <code>kubectl get pod</code> output:

<pre class="wp-block-code"><code>
~$ kubectl get pods

NAME               READY   STATUS    RESTARTS   AGE
webserver-logged   1/1     Running   0          7s
</code></pre>

You will have to look into each pod and see if they are configured as such. This information is available from the <code>kubectl describe pod</code> output's "Containers" section:

<pre class="wp-block-code"><code>
~$ kubectl describe pods webserver-logged

Name:             webserver-logged
Namespace:        default
Priority:         0
Service Account:  default
Node:             ip-172-31-21-39/172.31.21.39
Start Time:       Fri, 05 Jul 2024 18:43:17 +0000
Labels:           run=webserver-logged
Annotations:      <none>
Status:           Running
IP:               10.0.0.55
IPs:
  IP:  10.0.0.55
Containers:
  webserver-logged:
    Container ID:   containerd://3fec76f28ff636a2a2a5724ca01b764d82eb0b03f5362e7f6185720bd464436c
    Image:          docker.io/nginx:1.23.3
    Image ID:       docker.io/library/nginx@sha256:f4e3b6489888647ce1834b601c6c06b9f8c03dee6e097e13ed3e28c01ea3ac8c
    Port:           <none>
    Host Port:      <none>
    State:          Running
      Started:      Fri, 05 Jul 2024 18:43:22 +0000
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-fs6sh (ro)

...
</code></pre>

Learn more about <a href="https://kubernetes.io/docs/tasks/debug/debug-cluster/" target="_blank" rel="noreferrer noopener">troubleshooting clusters</a> in the Kubernetes docs.


# Monitor cluster and application resource usage

Kubernetes is a distributed system by nature: its components and workloads potentially run over many machines. In order to efficiently monitor those programs, you need to implement some kind of central monitoring system. The most monitoring that can be performed on a kubeadm-based cluster out of the box is through the API, specifically:

<ul>
<li>Finding API resources running in the cluster using <code>kubectl get</code></li>
<li>Describing the resource availability of each node using <code>kubectl describe node</code></li>
</ul>

While tools like Prometheus and Grafana are popular additions to any Kubernetes cluster, there is a subproject of Kubernetes called the Metrics Server that can make use of metrics being collected and emitted by kubelets. The Metrics Server contacts each of the kubelets (which provide the resource availability stats in the node describe outputs) and publishes them to the Kubernetes resource metrics API. A Kubernetes cluster with the Metrics Server enabled enables the <code>kubectl top</code> command.

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

NAMESPACE     NAME                                      CPU(cores)   MEMORY(bytes)   
default       webserver-logged                          0m           2Mi             
kube-system   cilium-ngkpt                              14m          298Mi           
kube-system   cilium-operator-65496b9554-vn6z7          4m           99Mi            
kube-system   coredns-7db6d8ff4d-gqnxv                  2m           37Mi            
kube-system   coredns-7db6d8ff4d-pr78k                  2m           33Mi            
kube-system   etcd-ip-172-31-21-39                      24m          77Mi            
kube-system   kube-apiserver-ip-172-31-21-39            51m          308Mi           
kube-system   kube-controller-manager-ip-172-31-21-39   16m          120Mi           
kube-system   kube-proxy-2mjrc                          1m           62Mi            
kube-system   kube-scheduler-ip-172-31-21-39            4m           64Mi            
kube-system   metrics-server-86776f5749-p8m8d           3m           54Mi
</code></pre>

You can also use the <code>kubectl top pods</code> command with the <code>--containers</code> option to view the same memory and CPU use statistics on a per container basis:

<pre class="wp-block-code"><code>
~$ kubectl top pods -A --containers

NAMESPACE     POD                                       NAME                      CPU(cores)   MEMORY(bytes)   
default       webserver-logged                          webserver-logged          0m           2Mi             
kube-system   cilium-ngkpt                              cilium-agent              14m          298Mi           
kube-system   cilium-operator-65496b9554-vn6z7          cilium-operator           4m           99Mi            
kube-system   coredns-7db6d8ff4d-gqnxv                  coredns                   2m           37Mi            
kube-system   coredns-7db6d8ff4d-pr78k                  coredns                   2m           33Mi            
kube-system   etcd-ip-172-31-21-39                      etcd                      22m          78Mi            
kube-system   kube-apiserver-ip-172-31-21-39            kube-apiserver            61m          312Mi           
kube-system   kube-controller-manager-ip-172-31-21-39   kube-controller-manager   16m          120Mi           
kube-system   kube-proxy-2mjrc                          kube-proxy                1m           62Mi            
kube-system   kube-scheduler-ip-172-31-21-39            kube-scheduler            4m           64Mi            
kube-system   metrics-server-86776f5749-p8m8d           metrics-server            5m           53Mi
</code></pre>

It is important to remember that the Kubernetes metrics server only provides more basic metrics reporting. For more sophisticated metrics output, there is the custom metrics API. The custom metrics API is populated with query results from one of many adapters tailored for specific metrics backends (like Prometheus). As the CKA stays within the bounds of Kubernetes itself, these are out of scope for the exam.

<a href="https://kubernetes.io/docs/tasks/debug/debug-cluster/resource-usage-monitoring/" target="_blank" rel="noreferrer noopener">Learn more about the Monitoring approach to Kubernetes here</a>. 

In addition to the metrics server, you also have the ability to query some of the Kubernetes components themselves for metrics. These metrics are served at the <code>/metrics</code> endpoint for the following cluster components:

<li>kube-controller-manager</li>
<li>kube-proxy</li>
<li>kube-apiserver</li>
<li>kube-scheduler</li>
<li>kubelet</li>

To access metrics exposed by cluster components you can use a service account granted the <code>get</code> permission on the <code>/metrics</code> <code>nonResourceURL</code> and the <code>nodes/metrics</code> resource via a cluster role. Below is an example of using a service account named "metrics-sa" to query the kube-apiserver for metrics:

<pre class="wp-block-code"><code>
$ TOKEN=$(kubectl create token metrics-sa)

$ curl -sk https://localhost:6443/metrics --header "Authorization: Bearer $TOKEN" | sed -e '/^#/d' | awk 'BEGIN { FS = "{" }{print $1}' | sort | uniq | head -20

aggregator_openapi_v2_regeneration_count
aggregator_openapi_v2_regeneration_duration
aggregator_unavailable_apiservice
apiserver_admission_controller_admission_duration_seconds_bucket
apiserver_admission_controller_admission_duration_seconds_count
apiserver_admission_controller_admission_duration_seconds_sum
apiserver_admission_step_admission_duration_seconds_bucket
apiserver_admission_step_admission_duration_seconds_count
apiserver_admission_step_admission_duration_seconds_sum
apiserver_admission_step_admission_duration_seconds_summary
apiserver_admission_step_admission_duration_seconds_summary_count
apiserver_admission_step_admission_duration_seconds_summary_sum
apiserver_audit_event_total 0
apiserver_audit_requests_rejected_total 0
apiserver_cache_list_fetched_objects_total
apiserver_cache_list_returned_objects_total
apiserver_cache_list_total
apiserver_client_certificate_expiration_seconds_bucket
apiserver_client_certificate_expiration_seconds_count 43434
apiserver_client_certificate_expiration_seconds_sum 1.369355609923495e+12
</code></pre>

Learn more about <a href="https://kubernetes.io/docs/concepts/cluster-administration/system-metrics/" target="_blank" rel="noreferrer noopener">metrics available from Kubernetes components</a>.

There are also additional tools you can deploy to <a href="https://kubernetes.io/docs/tasks/debug/debug-cluster/monitor-node-health/" target="_blank" rel="noreferrer noopener">monitor node health</a>


# Manage and evaluate container output streams

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
    image: docker.io/nginx:1.23.3
    args:
    - /bin/sh
    - -c
    - nginx -g "daemon off;" > /log/server.log 2>&1
    volumeMounts:
    - name: log
      mountPath: /log
  - name: fluent-bit
    image: docker.io/fluent/fluent-bit:1.9.7
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

NAME               READY   STATUS    RESTARTS   AGE   IP           NODE              NOMINATED NODE   READINESS GATES
webserver-logged   2/2     Running   0          6s    10.0.0.215   ip-172-31-21-39   <none>           <none>

~$ curl 10.0.0.215:80

... Welcome to nginx!...

</code></pre>

Checking the logs, you find that even though the webserver is not returning any output when queried for logs directly, the adapter container running the log agent shows the activity received.

<pre class="wp-block-code"><code>
$ kubectl logs webserver-logged -c webserver

$ kubectl logs webserver-logged -c fluent-bit

Fluent Bit v1.9.7
* Copyright (C) 2015-2022 The Fluent Bit Authors
* Fluent Bit is a CNCF sub-project under the umbrella of Fluentd
* https://fluentbit.io

[2024/07/05 18:53:54] [ info] [fluent bit] version=1.9.7, commit=265783ebe9, pid=1
[2024/07/05 18:53:54] [ info] [storage] version=1.2.0, type=memory-only, sync=normal, checksum=disabled, max_chunks_up=128
[2024/07/05 18:53:54] [ info] [cmetrics] version=0.3.5
[2024/07/05 18:53:54] [ info] [sp] stream processor started
[2024/07/05 18:53:54] [ info] [input:tail:tail.0] inotify_fs_add(): inode=2159449 watch_fd=1 name=/log/server.log
[2024/07/05 18:53:54] [ info] [output:stdout:stdout.0] worker #0 started
[0] tail.0: [1720205650.616114331, {"log"=>"10.0.0.246 - - [05/Jul/2024:18:54:10 +0000] "GET / HTTP/1.1" 200 615 "-" "curl/8.5.0" "-""}]
</code></pre>

[Learn more about how additional containers can help provide extra logging capabitilies without the need to rebuild application containers](https://kubernetes.io/blog/2015/06/the-distributed-system-toolkit-patterns/#example-3-adapter-containers).


# Troubleshoot services and networking


## Services

Services should be available by their DNS name within the cluster. Tools like <code>nslookup</code> and <code>dig</code> should be able to resolves services by their name in the same namespace or by the pattern: <code>service-name.namespace</code> in any namespace in the cluster. If DNS resolution fails, examine the <code>/etc/resolv.conf</code> file in the client pod to make sure the settings are correct. The nameserver should be a clusterIP (<code>10.96.0.10</code> in kubeadm clusters with the default configuration) and the search domains should include the pod's namespace and the cluster's domain.

To eliminate DNS from your debugging efforts, use the clusterIP of the service instead of its DNS name. Sending requests to the clusterIP of the service should work from any pod <em>and</em> any node in the cluster since clusterIPs are written to each node's IPTables or IPVS tables.

Listing endpoints with <code>kubectl get ep &lt;service-name&gt;</code> can reveal if the endpoints list is empty. If it is, you can replicate the service's selector by using <code>kubectl get po -l &lt;selector-key&gt;=&lt;selector-value&gt;</code>. Using this technique you may discover if any/all the target pods are not ready due to probe failures, nodes where pods run being down or not reporting ready, etc.

Kube-proxy needs to be actively updating the IPTables or IPVS tables of each node. There are several steps to <a href="https://kubernetes.io/docs/tasks/debug-application-cluster/debug-service/#is-the-kube-proxy-working" target="_blank" rel="noreferrer noopener">check if the kube-proxy is working</a>.

Lastly, ensure the network policies are not blocking ingress/egress traffic to and from services and clients within the cluster.

Learn more about <a href="https://kubernetes.io/docs/tasks/debug/debug-application/debug-service/" target="_blank" rel="noreferrer noopener">troubleshooting services</a>.


## Networking

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
</code></pre>

Note the message in the output above: <code>Container runtime network not ready: NetworkReady=false reason:NetworkPluginNotReady</code>.

Other symptoms of a non-functional or misconfigured CNI plugin include: new pods staying in the ContainerCreating phase even if the image is pulled.

Other networking failures may involve factors within the cluster’s underlying infrastructure, such as firewall rules or full network partitions preventing communication.

[Learn more about debugging networking at various levels of the cluster](https://kubernetes.io/docs/tasks/debug-application-cluster/debug-cluster/#a-general-overview-of-cluster-failure-modes) and [checking if the kube-proxy is working](https://kubernetes.io/docs/tasks/debug-application-cluster/debug-service/#is-the-kube-proxy-working).


# Practice Drill

Run the following command. This command will start a pod that will not run:

<code>kubectl run --restart Never --image redis:0.2 redispod</code>

Retrieve the events associated with this pod (listed in any order) and write them to a file: <code>/tmp/troubleshooting-answer.txt</code>


# CONTENT FROM BEFORE FEB 2025 REMAINS BELOW IN CASE THE EXAM INCLUDES IT IN THE FUTURE


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
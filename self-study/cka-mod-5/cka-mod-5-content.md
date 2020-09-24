<!-- CKA Self-Study Mod 5 -->


# Troubleshoot Application Failure

Applications running under Kubernetes run in containers within pods. There are several ways to troubleshoot application failures:

Reading the application container’s logs using <code>kubectl logs</code>
Viewing events associated with the pod using <code>kubectl events</code>
Interacting with the application container directly using <code>kubectl exec</code>

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


# Troubleshoot Control Plane Failure

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


# Troubleshoot Worker Node Failure

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

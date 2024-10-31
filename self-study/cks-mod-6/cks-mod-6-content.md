# CKS Self-Study Mod 6


## Perform behavior analytics of syscall process and file activities at the host and container level to detect malicious activities

Containerized applications still need to make system calls, or syscalls, at some point during their lifetimes. Tools that can track those syscalls, like Falco, sysdig, and even the more basic strace and ftrace, enable security administrators to detect, log, and analyze syscalls.

Falco can be configured to alert on certain conditions by analyzing syscalls made to the host Kernel. For example, the rules below instruct Falco to log an event every time a container is run with one of the following images using the <code>--privileged</code> flag:

<pre class="wp-block-code"><code>
# These container images are allowed to run with --privileged
- list: falco_privileged_images
  items: [
    docker.io/calico/node,
    calico/node,
    docker.io/cloudnativelabs/kube-router,
    docker.io/docker/ucp-agent,
    docker.io/falcosecurity/falco,
    docker.io/mesosphere/mesos-slave,
    docker.io/rook/toolbox,
    docker.io/sysdig/falco,
    docker.io/sysdig/sysdig,
    falcosecurity/falco,
    gcr.io/google_containers/kube-proxy,
    gcr.io/google-containers/startup-script,
    gcr.io/projectcalico-org/node,
    gke.gcr.io/kube-proxy,
    gke.gcr.io/gke-metadata-server,
    gke.gcr.io/netd-amd64,
    gcr.io/google-containers/prometheus-to-sd,
    k8s.gcr.io/ip-masq-agent-amd64,
    k8s.gcr.io/kube-proxy,
    k8s.gcr.io/prometheus-to-sd,
    quay.io/calico/node,
    sysdig/falco,
    sysdig/sysdig,
    sematext_images
    ]

- macro: falco_privileged_containers
  condition: (openshift_image or
              user_trusted_containers or
              container.image.repository in (trusted_images) or
              container.image.repository in (falco_privileged_images) or
              container.image.repository startswith istio/proxy_ or
              container.image.repository startswith quay.io/sysdig/)
</code></pre>

When implemented, a distinct log event is created each time such a container is created:

<pre class="wp-block-code"><code>
01:00:03.426003000: Notice Privileged container started (user=<NA> user_loginuid=0 command=container:fae86fdf29be k8s_weave-npc_weave-net-ckcmb_kube-system_0164606c-6df7-42e9-a93b-a3f4f4b4ae86_0 (id=fae86fdf29be) image=weaveworks/weave-npc:2.8.1)
01:00:03.430800000: Notice Privileged container started (user=<NA> user_loginuid=0 command=container:1f5bedd5c8f4 k8s_weave_weave-net-ckcmb_kube-system_0164606c-6df7-42e9-a93b-a3f4f4b4ae86_1 (id=1f5bedd5c8f4) image=weaveworks/weave-kube:2.8.1)
01:00:03.453054000: Notice Privileged container started (user=<NA> user_loginuid=0 command=container:fca27e217de3 testme (id=fca27e217de3) image=nginx:latest)

</code></pre>

All of this monitoring occurs outside of Kubernetes on the host, meaning all activity on this particular host can be instrumented to provide some kind of event logging.

On the exam, you have several options for tracking syscall activity, including:

- <code>sysdig</code> - A tool from the same creators of Falco
- <code>strace</code> - A common syscall tracking tool available in most Linux distributions
- <code>ftrace</code> - Another tool used to track specific functions as they make calls to the kernel

Do note that this approach is different from abstracting or Kernel syscalls through the use of secure container runtimes, like gVisor or NABLA containers. Those methods actively prevent certain syscalls from executing based on the configuration of those tools.

To learn more about Falco and how you can use it to monitor the behavior of all application syscall activity, check [here](https://falco.org/docs/examples/#write-to-directory-holding-system-binaries).


## Detect threats within the physical infrastructure, apps, networks, data, users, and workloads

This topic requires an awareness of ways to identify threats based on various symptoms in the infrastructure that can only be retrieved from infrastructure component logs, like <code>journalctl</code> or even the <code>systemctl status</code> outputs for various services.

- Unusual access patterns from previously well-known users or clients (i.e., when a stateless application's username attempts to connect to a database)
- The presence of containers known to spread common threats (like the XMRig cryptominers) in the local container cache

Setting up strong audit policies and tooling that checks incoming and outgoing access to and from the machine is key to detecting these threats. It is also important to note that many of these threats also target common misconfigurations or even the default settings, which include:

- Running system agents with overly privileged users
- Using default ports that are open and exposed to the network
- Running containers from public or untrusted sources

There are four major phases according to the [CNCF Cloud Native Security whitepaper](https://github.com/cncf/tag-security/blob/main/security-whitepaper/v2/cloud-native-security-whitepaper.md) where attacks can occur:

- Develop - Threats can manifest by introducing compromised artifacts into the application code due to old or insecure code inclusions
- Distribute - When assembling applications for distributions into binaries or containers, any third-party inclusions can result in a compromised package
- Deploy - Compromised automated distribution pipelines can spread threats across a large infrastructure
- Runtime - Running applications and containers have the ability to interact with the network and potentially have all the tools and credentials needed to compromise a network.

Deploying logging and auditing infrastructure and securing all parts of the above phases is essential to improving your cloud-native security stance. To learn more about how to perform threat modeling, look through the [CNCF TAG Security project assessment repository on GitHub](https://github.com/cncf/tag-security/tree/main/assessments/projects) to see recommendations made to various projects regarding security improvements.


## Detect all phases of attack regardless of where it occurs and how it spreads

Many threats manifest in the following ways:

- Additional resource overhead under "normal" conditions
- Degraded user experience
- Presence of unknown or peculiar resources within the system

Once those symptoms manifest, it is typically in the later post-breach stages of an attack. At this point, the following elements of your infrastructure may have already been compromised:

- Private container image registries
- Version Control System repositories
- The DNS provider or its components in your infrastructure

Understanding where these attacks can come from is a key part of the knowledge required for this topic. One good resource is the MITRE attack matrices, which describe the common areas and components that present a potential attack surface that can be exploited by an attacker. MITRE itself publishes the [Container ATT&CK Framework](https://attack.mitre.org/matrices/enterprise/containers/), which describes ways of breaching highly containerized environments and what is at risk. Derivatives of the MITRE ATT&CK framework exist too - Microsoft publishes the [Kubernetes Attack Matrix](https://www.microsoft.com/security/blog/2021/03/23/secure-containerized-environments-with-updated-threat-matrix-for-kubernetes/) which provides Kubernetes-specific attack vectors that you need to watch.

Reviewing case studies is the best way to prepare yourself for questions that fall under this topic's coverage. Optive has [a good breakdown of a Kubernetes cluster's attack surface](https://www.optiv.com/insights/source-zero/blog/kubernetes-attack-surface) on their website that exposes many of the factors you should be aware of when it comes to securing Kubernetes and preparing for the CKS.


## Perform deep analytical investigation and identification of bad actors within an environment

Like in the previous two topics, bad actors tend to leave the following clues that hint at their presence within a system:

- Artifacts and resources that do not follow established naming schemes
- Unusual references to artifacts in specification or configuration files
- Failed logins or other rejected requests from a certain user

User behavior is best tracked by some kind of auditing mechanism, which produces a log of a request and, most critically, includes who made the request. Many Linux systems keep track of such behavior in an audit log. For example, on an Ubuntu 18.04 system, the <code>/var/log/auth.log</code> and its derivatives provide some information on connections being made to the machine:

<pre class="wp-block-code"><code>
~$ cat /var/log/auth.log

Oct 10 13:04:12 oct-b dbus-daemon[587]: [system] Rejected send message, 1 matched rules; type="error", sender=":1.40" (uid=1000 pid=1237 comm="/usr/bin/pulseaudio --start --log-target=syslog " label="unconfined") interface="(unset)" member="(unset)" error name="org.bluez.MediaEndpoint1.Error.NotImplemented" requested_reply="0" destination=":1.218" (uid=0 pid=76962 comm="/usr/lib/bluetooth/bluetoothd " label="unconfined")
Oct 10 13:04:12 oct-b dbus-daemon[587]: [system] Rejected send message, 1 matched rules; type="error", sender=":1.40" (uid=1000 pid=1237 comm="/usr/bin/pulseaudio --start --log-target=syslog " label="unconfined") interface="(unset)" member="(unset)" error name="org.bluez.MediaEndpoint1.Error.NotImplemented" requested_reply="0" destination=":1.218" (uid=0 pid=76962 comm="/usr/lib/bluetooth/bluetoothd " label="unconfined")
Oct 10 13:04:12 oct-b dbus-daemon[587]: message repeated 2 times: [ [system] Rejected send message, 1 matched rules; type="error", sender=":1.40" (uid=1000 pid=1237 comm="/usr/bin/pulseaudio --start --log-target=syslog " label="unconfined") interface="(unset)" member="(unset)" error name="org.bluez.MediaEndpoint1.Error.NotImplemented" requested_reply="0" destination=":1.218" (uid=0 pid=76962 comm="/usr/lib/bluetooth/bluetoothd " label="unconfined")]
Oct 10 13:17:01 oct-b CRON[68258]: pam_unix(cron:session): session opened for user root by (uid=0)
Oct 10 13:17:01 oct-b CRON[68258]: pam_unix(cron:session): session closed for user root

</code></pre>

Here, you can see that a user, <code>root</code>, is included with a <code>uid</code> value. Other distributions of Linux and other OSes have similar mechanisms, like the Windows Event Log.

You will read about it later in this module, but if you want to see more information about auditing in Kubernetes, see the [documentation](https://kubernetes.io/docs/tasks/debug-application-cluster/audit/).


## Ensure immutability of containers at runtime

This topic requires an understanding of how to ensure containers that are running in your Kubernetes cluster cannot impose lasting changes to their filesystems.

Container images are already immutable: any changes you make to a container created from a container image are not persisted in the image. A container must commit any changes made to its filesystem to persist those changes to new containers made from that image. That process in itself creates a new container image.

Once a container is created, a writable layer is placed over an existing container image. Any changes made to the container filesystem will persist over the lifetime of that container. This presents an attack surface that bad actors can use to potentially exploit the container, especially in Kubernetes.

In Kubernetes, you can set a container's filesystem to read-only by setting the <code>readOnlyRootFilesystem</code> key to <code>true</code> under a container's security context:

<pre class="wp-block-code"><code>
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: rofspod
  name: rofspod
spec:
  containers:
  - command:
    - top
    image: busybox
    name: rofspod
    securityContext:
      readOnlyRootFilesystem: true
</code></pre>

When this pod is created, any write requests sent to the container are rejected:

<pre class="wp-block-code"><code>
~$ kubectl exec rofspod -- touch /tmp/hello

touch: /tmp/hello: Read-only file system
command terminated with exit code 1
</code></pre>

The <code>readOnlyRootFilesystem</code> only protects the filesystem packaged with the container. Volumes used by containers in a pod must be separately set to a read-only state.

The following pod spec exposes a host directory to containers in the pod and mounts it to the container that has a <code>readOnlyRootFilesystem</code> <code>securityContext</code> key set to <code>true</code>:

<pre class="wp-block-code"><code>
~$ cat roho-fspod.yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: rohofspod
  name: rohofspod
spec:
  containers:
  - command:
    - top
    image: busybox
    name: rohofspod
    securityContext:
      readOnlyRootFilesystem: true
    volumeMounts:
    - name: hosttmp
      mountPath: /tmp/hostpath
  volumes:
  - name: hosttmp
    hostPath:
      path: /tmp/pod
</code></pre>

The container is allowed to write to the host directory:

<pre class="wp-block-code"><code>
~$ kubectl exec rohofspod -- touch /tmp/hostpath/hello

~$ ls -l /tmp/pod

total 0
-rw-r--r-- 1 root root 0 Oct 10 13:31 hello
</code></pre>

The volume mount on the container level must be set to <code>readOnly</code> to ensure that the volume itself is mounted in a <code>readOnly</code> mode and rendered immutable:

<pre class="wp-block-code"><code>
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: rohofspod
  name: rohofspod
spec:
  containers:
  - command:
    - top
    image: busybox
    name: rohofspod
    securityContext:
      readOnlyRootFilesystem: true
    volumeMounts:
    - name: hosttmp
      mountPath: /tmp/hostpath
      readOnly: true
  volumes:
  - name: hosttmp
    hostPath:
      path: /tmp/pod
</code></pre>

Pods that use secrets or configmaps to populate files in the filesystem already project files in a read-only mode by default, so no additional configuration is necessary for that. You can optionally set those configmaps and secrets to be immutable as well, ensuring that pods that use those resources need to be deleted and recreated if an update is required. This also ensures that the configmaps and secrets themselves cannot be tampered with after being deployed.

To learn more about the settings you can use to ensure your containers are immutable while they run under Kubernetes, check out [the Pod SecurityContext page on the Kubernetes documentation](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/).


## Use Audit Logs to monitor access

The Kubernetes API Server has the ability to generate sophisticated access events to help users audit their cluster usage. When configured properly, the cluster audits the activities generated by users, by applications that use the Kubernetes API, and by the control plane itself. The main purpose of auditing is to help detect abnormal or otherwise unusual activity that could indicate a breach, whether it is in progress or happened recently.

Auditing in Kubernetes is configured using audit policies. Each policy can be as broad as logging all access events at the metadata level:

<pre class="wp-block-code"><code>
# Log all requests at the Metadata level.
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
  - level: Metadata
</code></pre>

To something as specific as the requests and responses associated with the creation of a pod in the <code>kube-system</code> namespace:

<pre class="wp-block-code"><code>
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
  - level: RequestResponse
    verbs: ["create"]
    resources:
    - group: ""
      resources: ["pods"]
    namespaces: ["kube-system"]
</code></pre>

Once you have an audit policy created, enabling auditing is relatively simple, though since it requires an API Server configuration, there will be some downtime necessary to enable it.

Audit logging is enabled by setting the following options on your kube-apiserver binary:

<pre class="wp-block-code"><code>
--audit-policy-file=/etc/kubernetes/audit/your-audit-policy.yaml
--audit-log-path=/etc/kubernetes/audit/api-audit.log
--audit-log-mode=blocking
</code></pre>

In this case, the API Server is being told to use your audit policy file, write all audit log events to a specific file path, and block other API Server requests while the auditing process occurs.

Once in place, requests get logged in JSON format:

<pre class="wp-block-code"><code>
{"kind":"EventList","apiVersion":"audit.k8s.io/v1","metadata":{},"items":[{"level":"RequestResponse","auditID":"d2a97a82-f39e-41d0-8b31-dd74f42ac7fe","stage":"RequestReceived","requestURI":"/api/v1/namespaces/kube-system/pods?fieldManager=kubectl-run","verb":"create","user":{"username":"kubernetes-admin","groups":["system:masters","system:authenticated"]},"sourceIPs":["192.168.229.157"],"userAgent":"kubectl/v1.20.1 (linux/amd64) kubernetes/c4d7527","objectRef":{"resource":"pods","namespace":"kube-system","apiVersion":"v1"},"requestReceivedTimestamp":"2021-02-10T00:28:46.451356Z","stageTimestamp":"2021-02-10T00:28:46.451356Z"}]}

{"kind":"EventList","apiVersion":"audit.k8s.io/v1","metadata":{},"items":[{"level":"RequestResponse","auditID":"d2a97a82-f39e-41d0-8b31-dd74f42ac7fe","stage":"ResponseComplete","requestURI":"/api/v1/namespaces/kube-system/pods?fieldManager=kubectl-run","verb":"create","user":{"username":"kubernetes-admin","groups":["system:masters","system:authenticated"]},"sourceIPs":["192.168.229.157"],"userAgent":"kubectl/v1.20.1 (linux/amd64) kubernetes/c4d7527","objectRef":{"resource":"pods","namespace":"kube-system","name":"webhook-audit","apiVersion":"v1"},"responseStatus":{"metadata":{},"code":201},"requestObject":{"kind":"Pod","apiVersion":"v1","metadata":{"name":"webhook-audit","creationTimestamp":null,"labels":{"run":"webhook-audit"}},"spec":{"containers":[{"name":"webhook-audit","image":"httpd","resources":{},"terminationMessagePath":"/dev/termination-log","terminationMessagePolicy":"File","imagePullPolicy":"Always"}],"restartPolicy":"Always","terminationGracePeriodSeconds":30,"dnsPolicy":"ClusterFirst","securityContext":{},"schedulerName":"default-scheduler","enableServiceLinks":true},"status":{}},"responseObject":{"kind":"Pod","apiVersion":"v1","metadata":{"name":"webhook-audit","namespace":"kube-system","uid":"8006453e-d338-4cb2-b16b-d619b28e2583","resourceVersion":"961180","creationTimestamp":"2021-02-10T00:28:46Z","labels":{"run":"webhook-audit"},"managedFields":[{"manager":"kubectl-run","operation":"Update","apiVersion":"v1","time":"2021-02-10T00:28:46Z","fieldsType":"FieldsV1","fieldsV1":{"f:metadata":{"f:labels":{".":{},"f:run":{}}},"f:spec":{"f:containers":{"k:{\"name\":\"webhook-audit\"}":{".":{},"f:image":{},"f:imagePullPolicy":{},"f:name":{},"f:resources":{},"f:terminationMessagePath":{},"f:terminationMessagePolicy":{}}},"f:dnsPolicy":{},"f:enableServiceLinks":{},"f:restartPolicy":{},"f:schedulerName":{},"f:securityContext":{},"f:terminationGracePeriodSeconds":{}}}}]},"spec":{"volumes":[{"name":"default-token-h2nlx","secret":{"secretName":"default-token-h2nlx","defaultMode":420}}],"containers":[{"name":"webhook-audit","image":"httpd","resources":{},"volumeMounts":[{"name":"default-token-h2nlx","readOnly":true,"mountPath":"/var/run/secrets/kubernetes.io/serviceaccount"}],"terminationMessagePath":"/dev/termination-log","terminationMessagePolicy":"File","imagePullPolicy":"Always"}],"restartPolicy":"Always","terminationGracePeriodSeconds":30,"dnsPolicy":"ClusterFirst","serviceAccountName":"default","serviceAccount":"default","securityContext":{},"schedulerName":"default-scheduler","tolerations":[{"key":"node.kubernetes.io/not-ready","operator":"Exists","effect":"NoExecute","tolerationSeconds":300},{"key":"node.kubernetes.io/unreachable","operator":"Exists","effect":"NoExecute","tolerationSeconds":300}],"priority":0,"enableServiceLinks":true,"preemptionPolicy":"PreemptLowerPriority"},"status":{"phase":"Pending","qosClass":"BestEffort"}},"requestReceivedTimestamp":"2021-02-10T00:28:46.451356Z","stageTimestamp":"2021-02-10T00:28:46.467111Z","annotations":{"authorization.k8s.io/decision":"allow","authorization.k8s.io/reason":""}}]}
</code></pre>

To learn more about how to enable auditing capabilities in Kubernetes, check out [the Kubernetes documentation page on auditing](https://kubernetes.io/docs/tasks/debug-application-cluster/audit/).


## Practice Drill

Create a deployment named <code>secure-webserver</code> that:

<ul>
<li>Deploys 3 pods</li>
<li>Each pod should have an empty directory volume named <code>bootstrap</code></li>
<li>Each pod should have two containers:</li>
  <ul>
  <li>One thats run the <code>rxmllc/trash-levels:1.1</code> image named webserver that mounts the <code>bootstrap</code> volume at <code>/bootstrap</code></li>
  <li>An init container that runs <code>alpine:3.13.6</code> which mounts the <code>bootstrap</code> volume at <code>/trash-levels/boot</code></li>
    <ul>
    <li>The init container should run the command: <code>/bin/sh -c "touch /trash-levels/boot/bootfile"</code></li>
    </ul>
  </ul>
<li>Container filesystems should be immutable where possible</li>
</ul>

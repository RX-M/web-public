<!-- CKAD Self-Study Mod 1 -->


# Basic Pods

Pods are the atomic unit of deployment in Kubernetes and are made up of one or more containers in different arrays in a PodSpec:

- Containers (required) - long running containers for applications, proxies, logging/monitoring sidecars, etc.
- Init containers (optional) - bootstrapping containers that run once to bootstrap long running app containers
- Ephemeral containers (optional) - an alpha feature in Kubernetes, these containers can be added at runtime for ad hoc troubleshooting

A basic pod would contain a single container and could be created with yaml or imperatively:

<pre class="wp-block-code"><code>
$ kubectl run ckad-basic-pod --generator=run-pod/v1 --image=nginx:latest
</code></pre>

[Learn more about pods](https://kubernetes.io/docs/concepts/workloads/pods/pod/).


# SecurityContext

This is a setting in a PodSpec that enhances security for one or all of the containers in a pod and have the following settings:
- Discretionary Access Control: define user ID (UID) and group ID (GID) settings for processes inside containers
- Security Enhanced Linux (SELinux): invoke predefined security labels
- Linux Capabilities: coarse-grained control of system calls to the Linux kernel in a whitelist or blacklist
  - Marking a pod with privileged = true grants all capabilities
- AppArmor: invoke predefined program profiles to restrict the capabilities of individual programs
- Seccomp: Fine-grained control over a process’s system calls through the use of json policies
- AllowPrivilegeEscalation: Controls whether a process can gain more privileges than its parent

SecurityContext settings can be set for the pod and/or each container in the pod, for example:

<pre class="wp-block-code"><code>
apiVersion: v1
kind: Pod
metadata:
  name: ckad-training-pod
spec:
  securityContext:              # pod securitycontext
    fsGroup: 2000
  containers:
  - name: ckad-training-container
    image: nginx
    securityContext:            # container securitycontext
      capabilities:
        add: ["NET_ADMIN"]
</code></pre>

[Learn more about SecurityContexts](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/).

# Resource Requests and Limits

Resource requests and limits are set on a per-container basis within a pod. By specifying a resource request we tell the Kubernetes scheduler the _minimum_ amount of each resource (CPU and memory) a container will need. By specifying limits, we set up cgroup constraints on the node where the process runs. An example of setting requests/limits looks like:

<pre class="wp-block-code"><code>
apiVersion: v1
kind: Pod
metadata:
  name: ckad-resource-pod
spec:
  containers:
  - name: ckad-resource-container
    image: my-app:v3.3
    resources:
      limits:
        cpu: "1"
        memory: “1Gi”
      requests:
        cpu: "0.5"
        memory: “500Mi”
</code></pre>

[Learn more about pod resource requests/limits](https://kubernetes.io/docs/tasks/configure-pod-container/assign-cpu-resource/).


# ConfigMaps

ConfigMaps are decoupled configuration artifacts keeping containerized applications portable.
The ConfigMap API resource provides mechanisms to inject containers with configuration data while
keeping containers agnostic of Kubernetes. A ConfigMap can be used to store fine-grained information like individual properties or coarse-grained information like entire config files or JSON blobs.

There are multiple ways to create a ConfigMap: from a directory upload, a file, or from literal values in command line as shown in the following example:

<pre class="wp-block-code"><code>
$ kubectl create configmap ckad-example-config --from-literal foo=bar
</code></pre>

[Learn more about ConfigMaps](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/).


# Secrets

Secrets hold sensitive information, such as passwords, OAuth tokens, and SSH keys. Putting this information in a secret is safer and more flexible than putting it verbatim in a pod definition or a Docker image!

There are three types of secrets, explained by the `--help` flag:

<pre class="wp-block-code"><code>
$ kubectl create secret --help

Create a secret using specified subcommand.
Available Commands:
  docker-registry Create a secret for use with a Docker registry
  generic     	Create a secret from a local file, directory or literal value
  tls         	Create a TLS secret
</code></pre>

Example of creating a secret imperatively:

<pre class="wp-block-code"><code>
$ kubectl create secret generic my-secret --from-literal=username=ckad-user --from-literal=password=Char1!3-K!10-Alpha-D31ta
</code></pre>

[Learn more about secrets](https://kubernetes.io/docs/concepts/configuration/secret/).


# Mounting ConfigMaps/Secrets as volumes or environment variables

ConfigMaps and Secrets are mounted by Pods as either volumes or environment variables to be used by container in a Pod.

ConfigMaps and Secrets can be used with a pod in two ways:
Files in a volume
Environment variables

Secrets can also be used by the kubelet when pulling images for a pod, called an imagePullSecret

The following Pod manifest mounts the ConfigMap ckad-example-config as a volume to the `/etc/myapp` directory in the container and uses a secret called “`ckad-training-docker-token`” as an imagePullSecret:

<pre class="wp-block-code"><code>
apiVersion: v1
kind: Pod
metadata:
  name: pod-config
spec:
  containers:
    - name: nginx
      image: nginx:latest
    imagePullSecrets:
    - name: ckad-training-docker-token
      volumeMounts:
      - name: config
        mountPath: /etc/myapp
  volumes:
    - name: config
      configMap:
        name: ckad-example-config
</code></pre>

Learn more about mounting:
- [ConfigMaps](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/)
- [Secrets](https://kubernetes.io/docs/concepts/configuration/secret/).


# ServiceAccounts


Service Accounts are users managed by the Kubernetes API that provide processes in a pod with an identity in the cluster. Service Accounts are bound to a set of credentials stored as secrets in the same namespace in the cluster. Every container in a pod within a namespace inherits credentials from their designated service account.

Service Accounts are entirely managed by the API, and are created by making API calls to the Kubernetes API server. `kubectl` automates the process of creating service accounts with the `create` subcommand. The example below shows an imperative command that creates a serviceAccount called `ckadexample` under the namespace called `ckadtraining`:

<pre class="wp-block-code"><code>
$ kubectl create namespace ckadtraining

$ kubectl create serviceaccount ckadexample --namespace ckadtraining
</code></pre>

A service account has no permissions within the cluster by default. The service account must be bound to a role that defines its permissions using a rolebinding. The following example creates a role that allows our new service account to view pods within the ckadtraining namespace and a rolebinding that grants those permissions to the ckadexample SA:

<pre class="wp-block-code"><code>
$ kubectl create role ckadsarole\
--namespace ckadtraining \
--verb=get,list,watch \
 --resource=pods

$ kubectl create rolebinding ckadsarolebinding \
--namespace ckadtraining \
--role=mysarole \
--serviceaccount=ckadtraining:ckadexample

$
</code></pre>

[Learn more about ServiceAccounts](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/).


# Practice Drill

Create a pod that runs the <code>nginx</code> image and uses a ServiceAccount called <code>my-sa</code>.

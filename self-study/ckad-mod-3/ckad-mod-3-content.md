<!-- CKAD Self-Study Mod 3 -->

<h1>Application Environment Configuration and Security</h1>


<h2>Discover and Use Resources that Extend Kubernetes</h2>

A Kubernetes cluster's functionality is extended by registering additional APIs to the API Server. Custom APIs usually bring their own set of custom resources which can be specified. If your cluster has been expanded to include custom resource definitions, there are two primary ways to identify them.

First is to see the list of APIs that have been registered, which you can see with <code>kubectl api-versions</code>:

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
batch/v1
batch/v1beta1
certificates.k8s.io/v1
coordination.k8s.io/v1
discovery.k8s.io/v1
discovery.k8s.io/v1beta1
events.k8s.io/v1
events.k8s.io/v1beta1
flowcontrol.apiserver.k8s.io/v1beta1
flowcontrol.apiserver.k8s.io/v1beta2
networking.k8s.io/v1
node.k8s.io/v1
node.k8s.io/v1beta1
policy/v1
policy/v1beta1
rbac.authorization.k8s.io/v1
scheduling.k8s.io/v1
storage.k8s.io/v1
storage.k8s.io/v1beta1
v1

$
</code></pre>

Any additional APIs you have installed as part of various cluster extensions, like operators,

The resources available to your cluster are viewable with <code>kubectl api-resources</code>, which shows the kinds of
resources you can create in a cluster:

<pre class="wp-block-code"><code>$ kubectl api-resources
NAME                              SHORTNAMES   APIVERSION                             NAMESPACED   KIND
bindings                                       v1                                     true         Binding
componentstatuses                 cs           v1                                     false        ComponentStatus
configmaps                        cm           v1                                     true         ConfigMap
endpoints                         ep           v1                                     true         Endpoints
events                            ev           v1                                     true         Event
limitranges                       limits       v1                                     true         LimitRange
namespaces                        ns           v1                                     false        Namespace
nodes                             no           v1                                     false        Node
persistentvolumeclaims            pvc          v1                                     true         PersistentVolumeClaim
persistentvolumes                 pv           v1                                     false        PersistentVolume
pods                              po           v1                                     true         Pod
podtemplates                                   v1                                     true         PodTemplate
replicationcontrollers            rc           v1                                     true         ReplicationController
resourcequotas                    quota        v1                                     true         ResourceQuota
secrets                                        v1                                     true         Secret
serviceaccounts                   sa           v1                                     true         ServiceAccount
services                          svc          v1                                     true         Service

...

$
</code></pre>

Learn more about <strong><a href="https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/">custom resource definitions</a></strong>.


<h2>Understanding Authentication, Authorization and Admission Control</h2>

Roles, ClusterRoles, RoleBinding and ClusterRoleBindings control user account permissions that control how they interact with resources deployed in the cluster. ClusterRoles and ClusterRoleBindings are non-namespaced resources. Roles and RoleBindings sets permissions and bind permissions in a specific namespace.

Kubernetes uses Role-based access control (RBAC) mechanisms to control the ability of users to perform a specific task on Kubernetes objects. Clusters bootstrapped with kubeadm have RBAC enabled by default.

Permissions to API resources are granted using Roles and ClusterRoles (the only difference being that clusterRoles apply to the entire cluster while regular roles apply to their namespace). Permissions are scoped to API resources and objects under the API resources. Verbs control what operations can be performed by each role.

Roles can be created imperatively using <code>kubectl create role</code>. You can specify the API resources and verbs associated with the permissions the role will grant:

<pre class="wp-block-code"><code>$ kubectl create role default-appmanager --resource pod,deploy,svc,ingresses --verb get,list,watch,create -o yaml

apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: default-appmanager
  namespace: default
rules:
- apiGroups:
  - ""
  resources:
  - pods
  - services
  verbs:
  - get
  - list
  - watch
  - create
  - delete
- apiGroups:
  - apps
  resources:
  - deployments
  verbs:
  - get
  - list
  - watch
  - create
  - delete

$
</code></pre>

Roles and clusterRoles are assigned to users and processes using roleBindings and clusterRoleBindings. Rolebindings associate a user, like a service account, with a role. Any permissions granted by a role are passed to the user through the rolebinding.

Rolebindings can also be created imperatively using <code>kubectl create rolebinding</code>. Rolebindings bind roles to users using the <code>--user</code> flag and serviceAccounts using the <code>--serviceaccount</code> flag. The following example binds the default-appmanager role to the default namespace’s default service account:

<pre class="wp-block-code"><code>$ kubectl create rolebinding default-appmanager-rb \
--serviceaccount default:default \
--role default-appmanager

rolebinding.rbac.authorization.k8s.io/default-appmanager-rb created

$
</code></pre>

Learn more about <strong><a href="https://kubernetes.io/docs/reference/access-authn-authz/rbac/">configuring role-based access control</a></strong>.


<h2>Resource Requests, Limits, and LimitRanges</h2>

Resource requests and limits are set on a per-container basis within a pod. By specifying a resource request we tell the Kubernetes scheduler the <em>minimum</em> amount of each resource (CPU and memory) a container will need. By specifying limits, we set up cgroup constraints on the node where the process runs. An example of setting requests/limits looks like:

<pre class="wp-block-code"><code>apiVersion: v1
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

Learn more about <strong><a href="https://kubernetes.io/docs/tasks/configure-pod-container/assign-cpu-resource/">pod resource requests/limits</a></strong>.


<h2>LimitRanges</h2>

Users who have control over their namespaces can also define a LimitRange, which is an API object that ensures pods maintain a minimum and maximum value for certain resources. This is enforced using a validating webhook that either rejects pods whose containers violate the set resource limits for their containers or inserts a limit into all containers of a pod that do not define any resource limits.

LimitRanges must be defined in YAML, and can ensure that either CPU or memory constraints are enforced within the namespace:

<pre class="wp-block-code"><code>apiVersion: v1
kind: LimitRange
metadata:
  name: cpu-limit
  namespace: limited
spec:
  limits:
  - max:
      cpu: "512m"
    min:
      cpu: "64m"
    type: Container
</code></pre>

Once defined, the limitrange can be found within the description.

<pre class="wp-block-code"><code>$ kubectl apply -f limitrange.yaml

limitrange/cpu-limit created

$ kubectl describe namespace limited

Name:         limited
Labels:       kubernetes.io/metadata.name=limited
Annotations:  <none>
Status:       Active

No resource quota.

Resource Limits
 Type       Resource  Min  Max   Default Request  Default Limit  Max Limit/Request Ratio
 ----       --------  ---  ---   ---------------  -------------  -----------------------
 Container  cpu       64m  512m  512m             512m           -

</code></pre>

Once a limitrange like the one described is in place, any pods you create will have that limit injected into their containers:

<pre class="wp-block-code"><code>$ kubectl run -n limited --image nginx -o yaml webserver

apiVersion: v1
kind: Pod
metadata:
  annotations:
    kubernetes.io/limit-ranger: 'LimitRanger plugin set: cpu request for container
      webserver; cpu limit for container webserver'
  creationTimestamp: "2022-04-19T23:35:55Z"
  labels:
    run: webserver
  name: webserver
  namespace: limited
  resourceVersion: "105825"
  uid: f9132086-5b4d-4c05-a7e3-991cf8227360
spec:
  containers:
  - image: nginx
    imagePullPolicy: Always
    name: webserver
    resources:
      limits:
        cpu: 512m
      requests:
        cpu: 512m

...

$
</code></pre>

Learn more about <strong><a href="https://kubernetes.io/docs/concepts/policy/limit-range/">LimitRanges</a></strong> and how they enforce resource constraints in your namespaces.


<h2>Namespace Quotas</h2>

In addition to limiting resources for containers in pods, users also have options to control the resources on the Kubernetes namespace level.

Namespace quotas are API objects that place limits on:

<ul>
<li>The number of certain resources, like pods or services, inside a namespace</li>
<li>The total utilization of certain machine resources, like cpu or memory, by containers within pods of the namespace</li>
</ul>

Quotas are enforced in two different ways:
<ul>
<li>Soft - where a warning is presented to the client if a request that violates the quota is made</li>
<li>Hard - where a request that violates the quota is rejected</li>
</ul>

Quotas can be placed by defining a specification for the quota inside a given namespace, which can be done using <code>kubectl create quota</code>:

<pre class="wp-block-code"><code>$ kubectl create quota --hard pods=3 pod-limit

resourcequota/pod-limit created

$ kubectl describe namespace default

Name:         default
Labels:       kubernetes.io/metadata.name=default
Annotations:  <none>
Status:       Active

Resource Quotas
  Name:     pod-limit
  Resource  Used  Hard
  --------  ---   ---
  pods      5     3

No LimitRange resource.

$
</code></pre>

Once create, quotas are visible in the describe output for a given namespace.

As this quota has hard enforcement, any requests that would violate a quota in a namespace is rejected, generating an error:

<pre class="wp-block-code"><code>$ kubectl get pods

No resources found in default namespace.

$ kubectl create deploy webserver --replicas=3 --image nginx

deployment.apps/webserver created

$ kubectl run webserver-new --image httpd

Error from server (Forbidden): pods "webserver-new" is forbidden: exceeded quota: pod-limit, requested: pods=1, used: pods=3, limited: pods=3

$
</code></pre>

Quotas are a great way of limiting the resource pools within namespaces.

Learn more about <strong><a href="https://kubernetes.io/docs/concepts/policy/resource-quotas/">resource quotas for namespaces</a></strong>.


<h2>ConfigMaps</h2>

ConfigMaps are decoupled configuration artifacts keeping containerized applications portable.
The ConfigMap API resource provides mechanisms to inject containers with configuration data while
keeping containers agnostic of Kubernetes. A ConfigMap can be used to store fine-grained information like individual properties or coarse-grained information like entire config files or JSON blobs.

There are multiple ways to create a ConfigMap: from a directory upload, a file, or from literal values in command line as shown in the following example:

<pre class="wp-block-code"><code>$ kubectl create configmap ckad-example-config --from-literal foo=bar -o yaml

apiVersion: v1
data:
  foo: bar
kind: ConfigMap
metadata:
  name: ckad-example-config
  namespace: default

$
</code></pre>

Learn more about <strong><a href="https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/">ConfigMaps</a></strong>.


<h2>Secrets</h2>

Secrets hold sensitive information, such as passwords, OAuth tokens, and SSH keys. Putting this information in a secret is safer and more flexible than putting it verbatim in a pod definition or a Docker image!

There are three types of secrets, explained by the <code>--help</code> flag:

<pre class="wp-block-code"><code>$ kubectl create secret --help

Create a secret using specified subcommand.
Available Commands:
  docker-registry Create a secret for use with a Docker registry
  generic     	Create a secret from a local file, directory or literal value
  tls         	Create a TLS secret
</code></pre>

Example of creating a secret imperatively:

<pre class="wp-block-code"><code>$ kubectl create secret generic my-secret --from-literal=username=ckad-user --from-literal=password="Char1!3-K!10-Alpha-D31ta" -o yaml

apiVersion: v1
data:
  password: Q2hhcjFLd2hpbGUgdHJ1ZSA7IGRvIHdoaWNoIHZpbSA7QWxwaGEtRDMxdGE=
  username: Y2thZC11c2Vy
kind: Secret
metadata:
  name: my-secret
  namespace: default
type: Opaque

$
</code></pre>

Learn more about <strong><a href="https://kubernetes.io/docs/concepts/configuration/secret/">secrets</a></strong>.


<h2>Mounting ConfigMaps/Secrets as Volumes or Environment Variables</h2>

ConfigMaps and Secrets are mounted by Pods as either volumes or environment variables to be used by container in a Pod.

ConfigMaps and Secrets can be used with a pod in two ways:
<ul>
 	<li>Files in a volume</li>
 	<li>Environment variables</li>
</ul>

Secrets can also be used by the kubelet when pulling images for a pod, called an imagePullSecret

The following Pod manifest mounts the ConfigMap ckad-example-config as a volume to the <code>/etc/myapp</code> directory in the container and uses a secret called ""<code>ckad-training-docker-token</code>" as an imagePullSecret:

<pre class="wp-block-code"><code>apiVersion: v1
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
<ul>
<li><strong><a href="https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/">ConfigMaps</a></strong></li>
<li><strong><a href="https://kubernetes.io/docs/concepts/configuration/secret/">Secrets</a></strong></li>
</ul>

<h2>Service Accounts</h2>

Service Accounts are users managed by the Kubernetes API that provide processes in a pod with an identity in the cluster. Service Accounts are bound to a set of credentials stored as secrets in the same namespace in the cluster. Every container in a pod within a namespace inherits credentials from their designated service account.

Service Accounts are entirely managed by the API, and are created by making API calls to the Kubernetes API server. <code>kubectl</code> automates the process of creating service accounts with the <code>create</code> subcommand. The example below shows an imperative command that creates a serviceAccount called <code>ckadexample</code> under the namespace called <code>ckadtraining</code>:

<pre class="wp-block-code"><code>$ kubectl create namespace ckadtraining

$ kubectl create serviceaccount ckadexample --namespace ckadtraining
</code></pre>

A service account has no permissions within the cluster by default. The service account must be bound to a role that defines its permissions using a rolebinding. The following example creates a role that allows our new service account to view pods within the ckadtraining namespace and a rolebinding that grants those permissions to the ckadexample SA:

<pre class="wp-block-code"><code>$ kubectl create role ckadsarole\
--namespace ckadtraining \
--verb=get,list,watch \
 --resource=pods

$ kubectl create rolebinding ckadsarolebinding \
--namespace ckadtraining \
--role=mysarole \
--serviceaccount=ckadtraining:ckadexample

$
</code></pre>

Learn more about <strong><a href="https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/">Service Accounts</a></strong>.


<h2>SecurityContext</h2>

This is a setting in a PodSpec that enhances security for one or all of the containers in a pod and have the following settings:
<ul>
<li>Discretionary Access Control - define user ID (UID) and group ID (GID) settings for processes inside containers</li>
<li>Security Enhanced Linux (SELinux) - invoke predefined security labels</li>
<li>Linux Capabilities - coarse-grained control of system calls to the Linux kernel in a whitelist or blacklist</li>
  <ul>
    <li>Marking a pod with privileged = true grants all capabilities</li>
  </ul>
<li>AppArmor - invoke predefined program profiles to restrict the capabilities of individual programs</li>
<li>Seccomp - Fine-grained control over a process’s system calls through the use of json policies</li>
<li>AllowPrivilegeEscalation - Controls whether a process can gain more privileges than its parent</li>
</ul>

SecurityContext settings can be set for the pod and/or each container in the pod, for example:

<pre class="wp-block-code"><code>apiVersion: v1
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

Learn more about <strong><a href="https://kubernetes.io/docs/tasks/configure-pod-container/security-context/">SecurityContexts</a></strong>.


<h2>Practice Drill</h2>

Create a pod that runs the <code>nginx</code> image and uses a ServiceAccount called <code>my-sa</code>.

# CKS Self-Study Mod 2


# Use Role Based Access Controls to minimize exposure

By default, clients sending requests to the API Server do not have any permissions. When granting permissions, it is best to adhere to the principle of least privilege and grant the minimum viable RBAC permissions to clients that need to communicate. Take time to understand exactly what your applications or users need to do when making requests to the API Server. For example, an Ingress Controller being added to the cluster must be able to interact with Ingress-related objects (<code>Ingress</code> and <code>IngressClass</code>). The role that grants the Ingress controller those permissions will look like:

<pre class="wp-block-code"><code>
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: nginx-ingress
rules:
-  apiGroups:
   - ""
   resources:
   - services
   - endpoints
   verbs:
   - get
   - list
   - watch
-  apiGroups:
   - ""
   resources:
   - secrets
   verbs:
   - get
   - list
   - watch
-  apiGroups:
   - ""
   resources:
   - configmaps
   verbs:
   - get
   - list
   - watch
   - update
   - create
-  apiGroups:
   - ""
   resources:
   - pods
   verbs:
   - list
-  apiGroups:
   - ""
   resources:
   - events
   verbs:
   - create
   - patch
-  apiGroups:
   - “networking.k8s.io”
   resources:
   - ingresses
   - ingressclasses
   verbs:
   - list
   - watch
   - get
-  apiGroups:
   - “networking.k8s.io”
   resources:
   - pods
   verbs:
   - list
-  apiGroups:
   - ""
   resources:
   - ingresses/status
   verbs:
   - update
</code></pre>

Looking at the permissions granted by this role, you will see the following trends:

<ul>
<li>Most verbs granted are used to retrieve information, i.e. <code>get</code>, <code>list</code>, <code>watch</code></li>
<li>No <code>delete</code> verbs</li>
<li>Verbs that allow editing of resources (<code>patch</code> and <code>update</code>) are used sparingly</li>
</ul>

[Minimizing permissions and other helpful tips you should consider when securing Kubernetes can be found here.](https://kubernetes.io/blog/2016/08/security-best-practices-kubernetes-deployment/)


# Exercise caution in using service accounts, e.g., disable defaults, minimize permissions on newly created ones

As stated before, clients that connect to the API Server do not have any permissions associated with them by default. Following the principle of least privilege also applies to this topic. Roles meant to allow ServiceAccounts to make requests to the API Server should grant as few permissions as possible.

ServiceAccounts are meant to enable applications running under pods in a cluster to communicate with the API Server. Common use cases for service accounts include:

<ul>
<li>Granting monitoring software running in the cluster the ability to look up other resources in the cluster</li>
<li>Perform mutating actions like updating specific resources</li>
</ul>

User credentials are best provided by an external system, like LDAP or another user directory service.

The following <code>role</code> allows a client to update its own deployment, configmap, and secret resources in the namespace. This role is created using the <code>patch</code> function:

<pre class="wp-block-code"><code>
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  creationTimestamp: null
  name: app-deploy-sa-role
rules:
- apiGroups:
  - ""
  resourceNames:
  - app-deployment
  resources:
  - configmaps
  - secrets
  verbs:
  - patch
- apiGroups:
  - apps
  resourceNames:
  - app-deployment
  resources:
  - deployments
  verbs:
  - patch
</code></pre>

By limiting the <code>patch</code> permission to just the deployment named <code>app-deployment</code>, the service account bound to this role can only affect the resources it is expected to interact with (the deployments, secrets, and configmaps related to the <code>app-deployment</code> application).

If your application does not need to communicate with the API server, consider opting out of the mounting of the service account credential from the pod by either:

<ul>
<li>Setting <code>spec.automountServiceAccountToken: false</code> in the service pod spec to opt out of mounting the credential from the pod</li>
<li>Setting <code>automountServiceAccountToken: false</code> in the Service Account spec itself to opt out of mounting its token in all pods that use this SA</li>
</ul>

This prevents the credential from being injected into the pod's containers, ensuring that it is not exposed should the container in the pod be compromised somehow.

If you would like to learn more about RBAC and Service accounts, navigate to this page on the docs [here](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#service-account-permissions).


# Restrict access to Kubernetes API

Access to the Kubernetes API is controlled using the role-based access control (RBAC) system in Kubernetes. Role-based access control is set up using two major components: Roles and Role Bindings.

Roles define permissions for a selected resource. Permissions are assigned as specific verbs, like <code>get</code>, <code>list</code>, <code>create</code>, <code>update</code>, or <code>delete</code>.

Each of the verbs defined in a role is associated with registered resources in the Kubernetes API, which range from entire API Groups (v1, apps, networking.k8s.io) to specific resource types (pods, deployments, ingresses) to specific instances of resources (pods named <code>unidentified-avian</code>). The following example illustrates a role that allows a user to get, list, and delete pods:

<pre class="wp-block-code"><code>
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: example-role
rules:
- apiGroups:
  - ""
  resources:
  - pods
  verbs:
  - get
  - list
  - delete
</code></pre>

Permissions are granted to subjects (which include users, clients, or any other entity that needs to communicate with the API Server) using role bindings. Role bindings declare a role and associate specific subjects to that role. The most common subject types include: Kubernetes Service Accounts and user or group names stored inside TLS certificates.

The following role binding object binds the <code>example-role</code> to a service account named <code>pod-operator</code> in the default namespace:

<pre class="wp-block-code"><code>
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: example-rolebinding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: example-role
subjects:
- kind: ServiceAccount
  name: pod-operator
  namespace: default
</code></pre>

With this role in place, the <code>pod-operator</code> in the default namespace can retrieve lists/details about pods and delete pods in the <code>default</code> namespace. These permissions are verifiable using the <code>kubectl auth can-i</code> function:

<pre class="wp-block-code"><code>
$ kubectl auth can-i delete pods --as system:serviceaccount:default:pod-operator

yes
</code></pre>

[You can learn more about using role-based access control to restrict access to the Kubernetes API here.](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)


# Upgrade Kubernetes to avoid vulnerabilities

Kubernetes is an ongoing project with a current target of 3 releases per year. Each update brings a variety of major and minor changes to the functionality of Kubernetes. Major developments in the Kubernetes ecosystem in those versions include:

<ul>
<li>A major reduction of API resources managed by various API groups (e.g., the reduction of resources in the </code>extensions</code> API group in 1.17)</li>
<li>The removal of certain APIs as features mature</li>
<li>Many long-term <code>beta</code> features, like Ingresses, moving to general availability</li>
<li>Various CVE-related fixes</li>
</ul>

Ensuring your Kubernetes cluster is up to date ensures that you are protected from the most recently disclosed security vulnerabilities. Between minor Kubernetes versions (1.31, 1.32) are several "patch" versions which address those vulnerabilities (in addition to fixing miscellaneous bugs and features).

The procedure for upgrading Kubernetes clusters differs with the installation method. Users who used <code>kubeadm</code> to initialize their clusters can use <code>kubeadm upgrade</code> to plan and execute version upgrades automatically.

To do a kubeadm upgrade, first, you must ensure you have a version of <code>kubeadm</code> that corresponds to your desired minor version of Kubernetes. Do so by retrieving that binary from the Kubernetes website:

<pre class="wp-block-code"><code>
$ nano /etc/apt/sources.list.d/kubernetes.list ; cat $_

# Edit the URL, changing v1.31 to v1.32
deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.32/deb/ /

$ sudo apt update

...

$ sudo apt install kubeadm=1.32.4-1.1

Reading package lists... Done
Building dependency tree
Reading state information... Done
The following packages will be upgraded:
  kubeadm
1 upgraded, 0 newly installed, 0 to remove and 46 not upgraded.
Need to get 0 B/8582 kB of archives.
After this operation, 12.3 kB of additional disk space will be used.
(Reading database ... 120961 files and directories currently installed.)
Preparing to unpack .../kubeadm_1.32.4-1.1_amd64.deb ...
Unpacking kubeadm (1.32.4-1.1) over (1.31.2-1.1) ...
Setting up kubeadm (1.32.4-1.1) ...
</code></pre>

Once kubeadm is upgraded, use kubeadm upgrade plan to see your options. kubeadm upgrade will recommend going to the latest patch version of the current minor version (i.e., to 1.31.5 from 1.31.2) or the latest patch version of the next minor version (to 1.32.4 from 1.31.2).

<pre class="wp-block-code"><code>
$ sudo kubeadm upgrade plan

...

Components that must be upgraded manually after you have upgraded the control plane with 'kubeadm upgrade apply':
COMPONENT   NODE      CURRENT   TARGET
kubelet     labsys    v1.31.5   v1.32.4

Upgrade to the latest stable version:

COMPONENT                 NODE      CURRENT    TARGET
kube-apiserver            labsys    v1.31.5    v1.32.4
kube-controller-manager   labsys    v1.31.5    v1.32.4
kube-scheduler            labsys    v1.31.5    v1.32.4
kube-proxy                          v1.31.5    v1.32.4
CoreDNS                             v1.11.3    v1.11.3
etcd                      labsys    3.5.15-0   3.5.16-0

You can now apply the upgrade by executing the following command:

        kubeadm upgrade apply v1.32.4

...
</code></pre>

From there, <code>kubeadm</code> provides instructions on how to execute or <code>apply</code> the upgrade to your cluster. There are a few things you must consider:

<ul>
<li>You can only move one minor version at a time (i.e., 1.30 clusters can only upgrade to 1.31 and cannot go directly to 1.32)</li>
<li>The <code>kubeadm upgrade</code> process does not update the kubelets or anything not managed by <code>kubeadm</code>, so you must execute those upgrades separately.</li>
</ul>

For more information on how to upgrade your Kubernetes clusters and stay up to date with the latest feature and fixes, go [here](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/).


# Practice Drill

<ul>
<li>Create a new namespace in your cluster named <code>secure-ns</code></li>
<li>In the <code>secure-ns</code> namespace, create a service account named <code>app-api-ops</code></li>
<li>Create the RBAC resources (naming them <code>app-api-rbac</code>) needed to grant the <code>app-api-ops</code> service account under the <code>secure-ns</code> namespace to do the following:</li>
  <ul>
  <li><code>get</code> and <code>list</code> all pods in the <code>secure-ns</code> namespace only</li>
  <li><code>create</code>, <code>update</code>, <code>get</code>, and <code>list</code> deployments in the <code>seucred-ns</code> namespace only</li>
  <li><code>get</code> and <code>list</code> all configmaps and secrets in the <code>secure-ns</code> namespace only</li>
  </ul>
</ul>
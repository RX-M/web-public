<!-- CKA Self-Study Mod 1 -->


# Manage role based access control (RBAC)

Roles, ClusterRoles, RoleBinding and ClusterRoleBindings control user account permissions that control how they interact with resources deployed in the cluster. ClusterRoles and ClusterRoleBindings are non-namespaced resources. Roles and RoleBindings sets permissions and bind permissions in a specific namespace.

Kubernetes uses Role-based access control (RBAC) mechanisms to control the ability of users to perform a specific task on Kubernetes objects. Clusters bootstrapped with kubeadm have RBAC enabled by default.

Permissions to API resources are granted using Roles and ClusterRoles (the only difference being that clusterRoles apply to the entire cluster while regular roles apply to their namespace). Permissions are scoped to API resources and objects under the API resources. Verbs control what operations can be performed by each role.

Roles can be created imperatively using <code>kubectl create role</code>. You can specify the API resources and verbs associated with the permissions the role will grant:

<pre class="wp-block-code"><code>
$ kubectl create role default-appmanager --resource pod,deploy,svc,ingresses --verb get,list,watch,create -o yaml

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
</code></pre>

Roles and clusterRoles are assigned to users and processes using roleBindings and clusterRoleBindings. Rolebindings associate a user, like a service account, with a role. Any permissions granted by a role are passed to the user through the rolebinding.

Rolebindings can also be created imperatively using <code>kubectl create rolebinding</code>. Rolebindings bind roles to users using the <code>--user</code> flag and serviceAccounts using the <code>--serviceaccount</code> flag. The following example binds the default-appmanager role to the default namespace’s default service account:

<pre class="wp-block-code"><code>
$ kubectl create rolebinding default-appmanager-rb \
--serviceaccount default:default \
--role default-appmanager

rolebinding.rbac.authorization.k8s.io/default-appmanager-rb created
</code></pre>

[Learn more about configuring role-based access control](https://kubernetes.io/docs/reference/access-authn-authz/rbac/).


# Use Kubeadm to install a basic cluster

<code>kubeadm</code> is the reference installer for Kubernetes that sets up a minimally viable Kubernetes cluster using some best practices. It simplifies the initialization of control plane nodes, the addition (or removal) of nodes to a Kubernetes cluster, and also handles control plane and Kubelet configuration updates.

Kubeadm has a variety of commands and subcommands that will allow you to:

<ul>
  <li>Create a control plane <code>kubeadm init</code></li>
  <li>Add a node <code>kubeadm join</code></li>
  <li>Regenerate certificates <code>kubeadm certificates renew</code></li>
  <li>Upgrade clusters <code>kubeadm upgrade</code></li>
</ul>

A typical kubeadm setup consists of the following characteristics (which you are present in many Kubernetes distributions):

<ul>
  <li>Control plane components (like the API Server or scheduler) running as pods</li>
  <li>Certificate-based communication between the API server and its clients</code></li>
  <li>kube-proxy to set up services</li>
  <li>CoreDNS to provide in-cluster DNS</li>
</ul>

In order to successfully use Kubeadm, the node must have a kubelet and container runtime installed on the machine:

<pre class="wp-block-code"><code>
$ sudo apt-get update && sudo apt-get install -y kubelet kubeadm kubectl
</code></pre>

Once installed, <code>kubeadm init</code> will initialize a control plane for your cluster.

<pre class="wp-block-code"><code>
$ sudo kubeadm init --cri-socket=unix:///var/run/containerd/containerd.sock

[init] Using Kubernetes version: v1.26.0
[preflight] Running pre-flight checks
[preflight] Pulling images required for setting up a Kubernetes cluster

...

Your Kubernetes control-plane has initialized successfully!
</code></pre>

Depending on your container runtime of choice, you will need to supply the appropriate value for the <code>--cri-socket</code> option.

[Learn more about setting up a Kubernetes cluster with Kubeadm here](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/)


# Manage a highly-available Kubernetes cluster

Kubernetes is a microservice-based system: all requests go to an API server microservice which is assisted by a variety of other components. To achieve high availability with a Kubernetes cluster, you typically add additional control plane nodes. Each control plane node hosts additional instances of the API Server, Scheduler, and Controller Manager. If etcd is deployed on the control plane node, then it will also add another member to the etcd cluster.

In a multi-node control plane, multiple API Servers run in tandem in a highly available pattern. Each API Server communicates with the same etcd cluster, so any client requests are processed using the same data available to all other API servers. All communications to and from the API Servers go through a common endpoint (like a load balancer which is set up outside of Kubernetes) which point to all instances of the API Server.

The other control plane components like the scheduler and controller manager operate on a failover pattern. Each instance of those microservices elect a single active leader that processes all the necessary functions. The remaining instances stand by, performing no function unless they take over as the leader (which happens if the elected leader go offline for any reason).

High availability on the control plane is just one part of the HA equation for Kubernetes. If your workloads need high availability, you may need to add additional nodes and configure your workloads to deploy additional copies that coordinate with each other somehow. In any case, the addition of nodes, control plane or otherwise, contributes to the high availability in a Kubernetes cluster.

<code>kubeadm</code> makes it easy to add, or join, nodes to your Kubernetes cluster.

First, you must acquire the command necessary to join a node. This command must include the API server's address, a special "join token", and the SHA hash of the target cluster's certificate authority (CA) certificate.

This is easily acquired by running <code>kubeadm token create --print-join-command</code> on one of your control plane nodes:

<pre class="wp-block-code"><code>
$ kubeadm token create --print-join-command

kubeadm join 192.168.100.100:6443 --token 3ua85a.rl5riytxhvc7fs1e --discovery-token-ca-cert-hash sha256:3d239f1c87cac3549334a91ed24580bea67e96cf78a4a83b20371af1c973922f 
</code></pre>

Run this command on any additional nodes that meet the prerequisites mentioned earlier in this module:

<pre class="wp-block-code"><code>
workerNode:~$ sudo kubeadm join 192.168.100.100:6443 --token 3ua85a.rl5riytxhvc7fs1e --discovery-token-ca-cert-hash sha256:3d239f1c87cac3549334a91ed24580bea67e96cf78a4a83b20371af1c973922f 

[preflight] Running pre-flight checks
[preflight] Reading configuration from the cluster...
[preflight] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Starting the kubelet
[kubelet-start] Waiting for the kubelet to perform the TLS Bootstrap...

This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.

Run 'kubectl get nodes' on the control-plane to see this node join the cluster.
</code></pre>

<code>kubeadm join</code> has a variety of flags that allow you to influence what type of node you add to your cluster - be sure to familiarize yourself with those flags.

[Learn more about expanding your Kubernetes cluster to improve its availability here](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#join-nodes)


# Provision underlying infrastructure to deploy a Kubernetes cluster

Kubernetes has several pre-requisites:

<ul>
  <li>For Control Plane nodes, Linux</li>
  <li>A container runtime</li>
  <li>Kernel modules that enable networking features like IPv4/IPv6 forwarding or bridge network awareness for IPTables</li>
  <li>The Node Agent, Kubelet</li>
</ul>

The operations you need to perform will vary based on your operation system of choice. The following demonstration blocks show the necessary commands for Ubuntu-based systems that use <code>apt</code>.

For the container runtime (assuming your sources are already configured):

<pre class="wp-block-code"><code>
$ sudo apt-get update && sudo apt-get install -y containerd.io
</code></pre>

System tools like <code>systemctl</code> can then be used to validate the state of whether those prerequisites are met:

<pre class="wp-block-code"><code>
$ sudo systemctl status containerd

● containerd.service - containerd container runtime
     Loaded: loaded (/lib/systemd/system/containerd.service; enabled; vendor preset: enabled)
     Active: active (running) since Mon 2023-01-09 16:28:02 UTC; 1h 29min ago
       Docs: https://containerd.io
   Main PID: 3905928 (containerd)
      Tasks: 15
     Memory: 1.3G
     CGroup: /system.slice/containerd.service
             └─3905928 /usr/bin/containerd
</code></pre>

Take note that on the exam, things like the container runtime and package sources will already be installed and configured. You will need to be aware of which packages to download as well as any other setup steps for Kernel modules, which are covered in the Kubernetes documentation.

Learn more about [the container runtime and other prerequisites here](https://kubernetes.io/docs/setup/production-environment/container-runtimes/).


# Perform a version upgrade on a Kubernetes cluster using Kubeadm

A Kubernetes upgrade entails updates to the components that serve Kubernetes functionality and the APIs server by Kubernetes. In order to upgrade a Kubernetes cluster, the following components are usually updated:

<ul>
  <li>Kubernetes API Server</li>
  <li>Scheduler</code></li>
  <li>Controller Manager</li>
  <li>etcd</li>
  <li>Kubelets</li>
</ul>

Kubeadm's design allows it to upgrade the control plane components easily, since all it needs to do is change the container versions of each control plane components. It will also perform other tasks, like regenerating certificates or renewing the kubelet configurations. These upgrade operations are driven by the <code>kubeadm upgrade</code> family of commands and subcommands.

Before upgrading Kubernetes, you must choose which version you want to upgrade to. Major API updates occur with minor version releases (e.g. 1.25 to 1.26) while fixes or security patches occur in patch releases (1.26.0 to 1.26.1). Kubeadm only allows movement between one minor version at a time: so a Kubeadm-initialized cluster at version 1.24 can only go to 1.25; if 1.26 is desired, then separate upgrade operation must occur to go to 1.25 to 1.26. The kubeadm binary's version represents the highest version that can be upgraded to (kubeadm 1.25.0 cannot upgrade a cluster to Kubernetes 1.26.0).

Cluster upgrades involve updating the version of the Kubernetes control plane components and kubelets. In general, the API server determines the version of the Kubernetes cluster and should be the newest component at any given time. The kubelet may be up two minor versions older than the API server. The other control plane components may be up to one minor version older than the API server. The kubectl client may be one version newer or older than the API server.

The details of the version support policy are detailed on the [version skew policy page](https://kubernetes.io/docs/setup/release/version-skew-policy/) in the Kubernetes documentation.

To upgrade the control-plane node we must do the following:
<li>Retrieve updated Kubernetes binaries</li>
<li>Install the newer version of kubeadm</li>
<li>Use <code>kubeadm upgrade plan</code> to check and fetch the new control plane component versions</li>
<li>Apply the upgrade</li>
<li>Upgrade the kubelet and kubectl installations on the target machine</li>

You may also use <code>kubectl drain</code> to remove any reschedulable workloads from the node you are upgrading. Just be sure to use <code>kubectl uncordon</code> to allow those workloads to come back if necessary.

The following is an example of upgrading a Kubernetes control plane node from Kubernetes v1.25.0 to v1.26.0 on Ubuntu 20.04:

Update the apt repository:

<pre class="wp-block-code"><code>
$ sudo apt update

…

$
</code></pre>

Install the newer kubeadm version e.g. v1.26.0:

<pre class="wp-block-code"><code>
$ sudo apt install kubeadm=1.26.0-00

…

$
</code></pre>

Run <code>kubeadm upgrade plan</code> with <code>sudo</code> to check and fetch updated control plane components:

<pre class="wp-block-code"><code>
$ sudo kubeadm upgrade plan
…
Components that must be upgraded manually after you have upgraded the control plane with 'kubeadm upgrade apply':
COMPONENT   CURRENT       AVAILABLE
kubelet                1 x v1.25.0     v1.26.0

Upgrade to the latest stable version:

COMPONENT                   CURRENT   AVAILABLE
kube-apiserver                   v1.25.0   v1.26.0
kube-controller-manager   v1.25.0   v1.26.0
kube-scheduler                  v1.25.0   v1.26.0
kube-proxy                         v1.25.0   v1.26.0
CoreDNS                           1.6.7        1.7.0
etcd                                    3.5.3-0   3.5.5-1

You can now apply the upgrade by executing the following command:

        kubeadm upgrade apply v1.26.0

Note: Before you can perform this upgrade, you have to update kubeadm to v1.26.0.

_____________________________________________________________________


The table below shows the current state of component configs as understood by this version of kubeadm.
Configs that have a "yes" mark in the "MANUAL UPGRADE REQUIRED" column require manual config upgrade or
resetting to kubeadm defaults before a successful upgrade can be performed. The version to manually
upgrade to is denoted in the "PREFERRED VERSION" column.

API GROUP                 CURRENT VERSION   PREFERRED VERSION   MANUAL UPGRADE REQUIRED
kubeproxy.config.k8s.io   v1alpha1          v1alpha1            no
kubelet.config.k8s.io     v1beta1           v1beta1             no
_____________________________________________________________________

$
</code></pre>

The upgrade is executed using <code>kubeadm upgrade apply</code>:

<pre class="wp-block-code"><code>
$ sudo kubeadm upgrade apply v1.26.0
…
[upgrade/successful] SUCCESS! Your cluster was upgraded to "v1.26.0". Enjoy!

[upgrade/kubelet] Now that your control plane is upgraded, please proceed with upgrading your kubelets if you haven't already done so.

</code></pre>

Kubeadm does not affect the Kubelet nor Kubectl, so those must be updated by acquiring the new binaries separately.

Install the corresponding versions of the kubelet and kubectl:

<pre class="wp-block-code"><code>
$ sudo apt install kubelet=1.26.0-00 kubectl=1.26.0-00
…
Setting up kubelet (1.26.0-00) ...
Setting up kubectl (1.26.0-00) ...
</code></pre>

Repeat this across the cluster until all nodes are at the desired versions.

[Learn more about upgrading your cluster with kubeadm](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/).


# Implement etcd backup and restore

etcd holds the current, running state of the cluster. The API Server(s) in a cluster write data to and retrieve data from etcd.

All data is recorded to a write ahead log (WAL) file in the form of the transactions performed to write data to the etcd cluster's members. Each member of the etcd cluster maintains their own WAL file (full of transactions requested by the "leader" of the etcd cluster). These data are periodically compacted into a series of snapshots to save space. Snapshotting occurs automatically as part of etcd's normal operation, though a client like <code>etcdctl</code> can create a snapshot on demand using its <code>snapshot save</code> command:

<pre class="wp-block-code"><code>
$ ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 \
--cacert=/etc/etcd/ca.crt --cert=/etc/etcd/server.crt --key=/etc/etcd/server.key \
snapshot save /var/lib/etcd/backup.db
</code></pre>

This command connects to an etcd cluster and saves its contents to a snapshot. Note the certificate related options: The etcd deployed by kubeadm uses certificates to authenticate its peers (other etcd instances) and any clients (like the API Server and etcdctl).

Snapshots produced by etcd or etcdctl can then be restored with all of the original data that the snapshot was created with. Depending on the "freshness" of the snapshot, the recovery time of the etcd cluster (and any Kubernetes clusters that use it) can vary. The <code>etcdctl snapshot restore</code> command will take the contents of a snapshot and place them into the target cluster's directory.

[Learn more about performing backup and restore operations for etcd here.](https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/)




# Practice Drill

Create a role and a role-binding that gives a user named <code>networker</code> permissions to get and list the ingresses and network policies.
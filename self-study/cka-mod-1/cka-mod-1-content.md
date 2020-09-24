<!-- CKA Self-Study Mod 1 -->

# Configure Authentication and Authorization

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

$
</code></pre>

Roles and clusterRoles are assigned to users and processes using roleBindings and clusterRoleBindings. Rolebindings associate a user, like a service account, with a role. Any permissions granted by a role are passed to the user through the rolebinding.

Rolebindings can also be created imperatively using <code>kubectl create rolebinding</code>. Rolebindings bind roles to users using the <code>--user</code> flag and serviceAccounts using the <code>--serviceaccount</code> flag. The following example binds the default-appmanager role to the default namespace’s default service account:

<pre class="wp-block-code"><code>
$ kubectl create rolebinding default-appmanager-rb \
--serviceaccount default:default \
--role default-appmanager

rolebinding.rbac.authorization.k8s.io/default-appmanager-rb created

$
</code></pre>

Learn more about configuring role-based access control [here](https://kubernetes.io/docs/reference/access-authn-authz/rbac/).


# Upgrading Kubernetes with Kubeadm

Cluster upgrades involve updating the version of the Kubernetes control plane components and the kubelets that run on every node in the cluster. In general, the API server determines the version of the Kubernetes cluster. The kubelet may be up two minor versions older than the API server. The other control plane components may be up to one minor version older than the API server. The kubectl client may be one version newer or older than the API server.

The details of the version support policy are detailed on the [version skew policy page](https://kubernetes.io/docs/setup/release/version-skew-policy/) in the Kubernetes documentation.

To upgrade the control-plane node we must do the following:
- retrieve updated kubernetes binaries
- install the newer version of kubadm
- drain the control plane node
- use <code>kubeadm upgrade plan</code> to check and fetch the new control plane component versions
- apply the upgrade
- upgrade the kubelet and kubectl
- uncordon the control plan node

The following is an example of upgrading a Kubernetes control plane node from Kubernetes v1.18.0 to v.19.0 on Ubuntu 18.04:

Update the apt repository:

<pre class="wp-block-code"><code>
$ sudo apt update
…
$
</code></pre>

Install the newer kubeadm version e.g. v1.19.0:

<pre class="wp-block-code"><code>
$ sudo apt install kubeadm=1.19.0-00
…
$
</code></pre>

Drain the control plan node:

<pre class="wp-block-code"><code>
$ kubectl drain <control-plane-node> --ignore-daemonsets
…
$
</code></pre>

Run <code>kubeadm upgrade plan</code> with <code>sudo</code> to check and fetch updated control plane components:

<pre class="wp-block-code"><code>
$ sudo kubeadm upgrade plan
…
Components that must be upgraded manually after you have upgraded the control plane with 'kubeadm upgrade apply':
COMPONENT   CURRENT       AVAILABLE
kubelet                1 x v1.18.0     v1.19.1

Upgrade to the latest stable version:

COMPONENT                   CURRENT   AVAILABLE
kube-apiserver                   v1.18.0   v1.19.1
kube-controller-manager   v1.18.0   v1.19.1
kube-scheduler                  v1.18.0   v1.19.1
kube-proxy                         v1.18.0   v1.19.1
CoreDNS                           1.6.7        1.7.0
etcd                                    3.4.3-0   3.4.9-1

You can now apply the upgrade by executing the following command:

        kubeadm upgrade apply v1.19.1

Note: Before you can perform this upgrade, you have to update kubeadm to v1.19.1.

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

Notice that the kubelet must be upgraded manually after upgrading the control plane.
We see that v1.19.1 is available but let’s upgrade to v1.19.0:
<pre class="wp-block-code"><code>
$ sudo kubeadm upgrade apply v1.19.0
…
[upgrade/successful] SUCCESS! Your cluster was upgraded to "v1.19.0". Enjoy!

[upgrade/kubelet] Now that your control plane is upgraded, please proceed with upgrading your kubelets if you haven't already done so.

$
</code></pre>

Install the corresponding versions of the kubelet and kubectl:

<pre class="wp-block-code"><code>
$ sudo apt install kubelet=1.19.0-00 kubectl=1.19.0-00
…
Setting up kubelet (1.19.0-00) ...
Setting up kubectl (1.19.0-00) ...

$
</code></pre>

Uncordon the control plan node:
<pre class="wp-block-code"><code>
$ kubectl uncordon <control-plane-node>
…
$
</code></pre>

Learn more about upgrading your cluster with kubeadm [here](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/).


# Implement Backup and Restore Methodologies

The state of a Kubernetes cluster is contained in the etcd instance(s) backing the cluster. Backing up a Kubernetes cluster is a matter of backing up the etcd instance(s).

One way to perform a backup is by using the <code>etcdctl</code> command:

<pre class="wp-block-code"><code>
$ ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 \
--cacert=/etc/etcd/ca.crt --cert=/etc/etcd/server.crt --key=/etc/etcd/server.key \
snapshot save /var/lib/etcd/backup.db
</code></pre>

This command connects to an etcd cluster and saves its contents to a database file. This database file is then used to restore the entire cluster on a new set of nodes.

[Kubernetes etcd instance backup and restore procedures](https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/#backing-up-an-etcd-cluster).


# Practice Drill

Create a role and a role-binding that gives a user named <code>networker</code> permissions to get and list the ingresses and network policies.

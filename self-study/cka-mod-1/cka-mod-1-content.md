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


# Prepare underlying infrastructure for installing a Kubernetes cluster

The exam requires that you know how to prepare a Linux-based node for Kubernetes using standard Linux tools and CLIs. Kubernetes has several prerequisites:

<ul>
  <li>For Control Plane nodes, Linux (worker nodes can use Windows for their operating system)</li>
  <li>A container runtime</li>
  <li>Kernel modules that enable networking features like IPv4/IPv6 forwarding or bridge network awareness for IPTables</li>
  <li>The Node Agent, Kubelet</li>
</ul>

The operations you need to perform will vary based on your Linux distribution. The exam is based on Ubuntu so the following demonstration blocks show the necessary commands for Ubuntu-based systems.

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

By default, containerd uses <code>cgroupfs</code> to manage the Linux <code>cgroups</code> for container isolation. Modify containerd to use the systemd cgroup management driver required by K8s:

<pre class="wp-block-code"><code>
$ sudo sed -i -e 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
</code></pre>

We need to run another <code>sed</code> command to ensure containerd also uses the correct pause container image. Pause containers are used by Kubernetes to help enable the “pod sandbox” – the shared isolation used to enable more complex container patterns when running containers within Kubernetes.

<pre class="wp-block-code"><code>
$ sudo sed -i -e 's/pause:3.8/pause:3.10/' /etc/containerd/config.toml
</code></pre>

> N.B. The versions of the pause container will change over time so you may need to change the <code>sed</code> command to match the pause version in the file and the pause version being used with the current release of K8s.

On the exam, things like the container runtime and package sources will already be installed and configured. You will need to be aware of which packages to download as well as any other setup steps for Kernel modules, which are covered in the Kubernetes documentation.

Learn more about [the container runtime and other prerequisites here](https://kubernetes.io/docs/setup/production-environment/container-runtimes/).


# Create and manage Kubernetes clusters using kubeadm

<code>kubeadm</code> is the reference installer for Kubernetes that sets up a minimally viable Kubernetes cluster using some best practices. It simplifies the initialization of control plane nodes, the addition (or removal) of nodes to a Kubernetes cluster, and also handles control plane and Kubelet configuration updates.

Kubeadm has a variety of commands and subcommands that will allow you to:

<ul>
  <li>Create a control plane: <code>kubeadm init</code></li>
  <li>Add a node: <code>kubeadm join</code></li>
  <li>Regenerate certificates: <code>kubeadm certificates renew</code></li>
  <li>Upgrade clusters: <code>kubeadm upgrade</code></li>
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

[init] Using Kubernetes version: v1.30.2
[preflight] Running pre-flight checks
[preflight] Pulling images required for setting up a Kubernetes cluster

...

Your Kubernetes control-plane has initialized successfully!
</code></pre>

Depending on your container runtime of choice, you will need to supply the appropriate value for the <code>--cri-socket</code> option.

[Learn more about setting up a Kubernetes cluster with Kubeadm here](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/)


# Manage the lifecycle of Kubernetes clusters

Managing the lifecycle of Kubernetes clusters includes upgrading the cluster components. A Kubernetes upgrade entails updates to the components that serve Kubernetes functionality and the APIs server by Kubernetes. In order to upgrade a Kubernetes cluster, the following components are usually updated:

<ul>
  <li>Kubernetes API Server</li>
  <li>Scheduler</code></li>
  <li>Controller Manager</li>
  <li>etcd</li>
  <li>Kubelets</li>
</ul>

Kubeadm's design allows it to upgrade the control plane components easily, since all it needs to do is change the container versions of each control plane components. It will also perform other tasks, like regenerating certificates or renewing the kubelet configurations. These upgrade operations are driven by the <code>kubeadm upgrade</code> family of commands and subcommands.

Before upgrading Kubernetes, you must choose which version you want to upgrade to. Major API updates occur with minor version releases (e.g. 1.31 to 1.32) while fixes or security patches occur in patch releases (1.32.0 to 1.32.1). Kubeadm only allows movement between one minor version at a time: so a Kubeadm-initialized cluster at version 1.30 can only go to 1.31; if 1.32 is desired, then separate upgrade operation must occur to go to 1.31 to 1.32. The kubeadm binary’s version represents the highest version that can be upgraded to (a kubeadm 1.31 binary cannot upgrade a cluster to Kubernetes 1.32).

Cluster upgrades involve updating the version of the Kubernetes control plane components and kubelets. In general, the API server determines the version of the Kubernetes cluster and should be the newest component at any given time. The kubelet may be up two minor versions older than the API server. The other control plane components may be up to one minor version older than the API server. The kubectl client may be one version newer or older than the API server.

The details of the version support policy are detailed on the [version skew policy page](https://kubernetes.io/docs/setup/release/version-skew-policy/) in the Kubernetes documentation.

To upgrade the control-plane node we must do the following:
<li>Retrieve updated Kubernetes binaries</li>
<li>Install the newer version of kubeadm</li>
<li>Use <code>kubeadm upgrade plan</code> to check and fetch the new control plane component versions</li>
<li>Apply the upgrade</li>
<li>Upgrade the kubelet and kubectl installations on the target machine</li>

You may also use <code>kubectl drain</code> to remove any reschedulable workloads from the node you are upgrading. Just be sure to use <code>kubectl uncordon</code> to allow those workloads to come back if necessary.

The following is an example of upgrading a Kubernetes control plane node from Kubernetes v1.31 to v1.32 on Ubuntu 24.04:

Change the repository to the latest minor version and update the indexes:

<pre class="wp-block-code"><code>
$ nano /etc/apt/sources.list.d/kubernetes.list ; cat $_

# Edit the URL, changing v1.31 to v1.32
deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.32/deb/ /

$ sudo apt update

...
</code></pre>

Install the newer kubeadm version:

<pre class="wp-block-code"><code>
$ apt-cache madison kubeadm

   kubeadm | 1.30.1-1.1 | https://pkgs.k8s.io/core:/stable:/v1.30/deb  Packages
   kubeadm | 1.30.0-1.1 | https://pkgs.k8s.io/core:/stable:/v1.30/deb  Packages

$ sudo apt install kubeadm=1.32.1-1.1

...
</code></pre>

Run <code>kubeadm upgrade plan</code> with <code>sudo</code> to check and fetch updated control plane components:

<pre class="wp-block-code"><code>
$ sudo kubeadm upgrade plan

...

Components that must be upgraded manually after you have upgraded the control plane with 'kubeadm upgrade apply':
COMPONENT   NODE      CURRENT   TARGET
kubelet     labsys    v1.31.5   v1.32.1

Upgrade to the latest stable version:

COMPONENT                 NODE      CURRENT    TARGET
kube-apiserver            labsys    v1.31.5    v1.32.1
kube-controller-manager   labsys    v1.31.5    v1.32.1
kube-scheduler            labsys    v1.31.5    v1.32.1
kube-proxy                          v1.31.5    v1.32.1
CoreDNS                             v1.11.3    v1.11.3
etcd                      labsys    3.5.15-0   3.5.16-0

You can now apply the upgrade by executing the following command:

        kubeadm upgrade apply v1.32.1

...
</code></pre>

The upgrade is executed using <code>kubeadm upgrade apply</code>:

<pre class="wp-block-code"><code>
$ sudo kubeadm upgrade apply v1.32.1

...

[upgrade/successful] SUCCESS! Your cluster was upgraded to "v1.32.1". Enjoy!

[upgrade/kubelet] Now that your control plane is upgraded, please proceed with upgrading your kubelets if you haven't already done so.
</code></pre>

Kubeadm does not affect the Kubelet nor Kubectl, so those must be updated by acquiring the new binaries separately.

Install the corresponding versions of the kubelet and kubectl:

<pre class="wp-block-code"><code>
$ sudo apt install kubelet=1.32.1-1.1 kubectl=1.32.1-1.1

...

Setting up kubelet (1.32.1-1.1) ...
Setting up kubectl (1.32.1-1.1) ...
</code></pre>

Repeat this across the cluster until all nodes are at the desired versions.

[Learn more about upgrading your cluster with kubeadm](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/).


# Implement and configure a highly-available control plane

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


# Use Helm and Kustomize to install cluster components

## Helm

A popular method of deploying applications or cluster components on Kubernetes is through the use of templating tools like Helm. These tools allow users to specify all of the components of an application meant to run on Kubernetes in a series of templated YAML files. Tools like Helm take these templated YAML files, populate them with user-defined values, and deploy all of those resources together.

Helm takes the concept of an application package and applies it to Kubernetes. The application’s resource templates are grouped into packages known as charts. Charts include all of the YAML templates that represent Kubernetes resources deployed by Helm and supporting components like metadata, information on dependencies, custom resource definitions, and tests to be run.

Deploying an application with helm is straightforward. First, identify a chart that you want to install. Common chart repositories include [artifacthub.io](https://artifacthub.io/), which in provides as a central space to find charts.

Once you have identified a chart, you need to add that chart as a source to your Helm instance:

<pre class="wp-block-code"><code>
$ helm repo add bitnami https://charts.bitnami.com/bitnami

"bitnami" has been added to your repositories

$ helm repo update

Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "bitnami" chart repository
Update Complete. ⎈Happy Helming!⎈
</code></pre>

After adding the chart repository to <code>helm</code>, your helm command can now search and install charts from the selected repository.

If you want to install, say, an NGINX web server using helm from the recently added repository, you use helm's <code>install</code> command to generate a release of that chart. The release is the grouping of Kubernetes resources generated from the template in the chart. The resources are populated with values provided by the user (using the <code>--set</code> flag or providing a yaml file with values) or the default values of provided by the chart during the installation:


<pre class="wp-block-code"><code>
$ kubectl get all --show-labels

NAME                                    READY   STATUS    RESTARTS   AGE   LABELS
pod/self-study-nginx-7ccd4b56d9-q29bb   1/1     Running   0          73s   app.kubernetes.io/instance=self-study-nginx,app.kubernetes.io/managed-by=Helm,app.kubernetes.io/name=nginx,helm.sh/chart=nginx-10.1.1,pod-template-hash=7ccd4b56d9

NAME                       TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE   LABELS
service/kubernetes         ClusterIP      10.96.0.1              443/TCP        19h   component=apiserver,provider=kubernetes
service/self-study-nginx   LoadBalancer   10.99.49.207        80:30959/TCP   73s   app.kubernetes.io/instance=self-study-nginx,app.kubernetes.io/managed-by=Helm,app.kubernetes.io/name=nginx,helm.sh/chart=nginx-10.1.1

NAME                               READY   UP-TO-DATE   AVAILABLE   AGE   LABELS
deployment.apps/self-study-nginx   1/1     1            1           73s   app.kubernetes.io/instance=self-study-nginx,app.kubernetes.io/managed-by=Helm,app.kubernetes.io/name=nginx,helm.sh/chart=nginx-10.1.1

NAME                                          DESIRED   CURRENT   READY   AGE   LABELS
replicaset.apps/self-study-nginx-7ccd4b56d9   1         1         1       73s   app.kubernetes.io/instance=self-study-nginx,app.kubernetes.io/managed-by=Helm,app.kubernetes.io/name=nginx,helm.sh/chart=nginx-10.1.1,pod-template-hash=7ccd4b56d9
</code></pre>

Once a release is generated, you can change the parameters by changing values (again using the <code>--set</code> or providing an updated values yaml file) and using Helm's <code>upgrade</code> subcommand:

<pre class="wp-block-code"><code>
$ helm upgrade self-study-nginx --set service.type=NodePort bitnami/nginx

Release "self-study-nginx" has been upgraded. Happy Helming!
NAME: self-study-nginx
LAST DEPLOYED: Tue Apr 19 19:24:14 2022
NAMESPACE: default
STATUS: deployed
REVISION: 2
TEST SUITE: None
NOTES:
CHART NAME: nginx
CHART VERSION: 10.1.1
APP VERSION: 1.21.6

** Please be patient while the chart is being deployed **

NGINX can be accessed through the following DNS name from within your cluster:

    self-study-nginx.default.svc.cluster.local (port 80)

To access NGINX from outside the cluster, follow the steps below:

1. Get the NGINX URL by running these commands:

    export NODE_PORT=$(kubectl get --namespace default -o jsonpath="{.spec.ports[0].nodePort}" services self-study-nginx)
    export NODE_IP=$(kubectl get nodes --namespace default -o jsonpath="{.items[0].status.addresses[0].address}")
    echo "http://${NODE_IP}:${NODE_PORT}"
</code></pre>

The releases themselves are managed directly by Helm, and can be uninstalled using Helm's <code>uninstall</code> command:

<pre class="wp-block-code"><code>
$ helm uninstall self-study-nginx

release "self-study-nginx" uninstalled

$ kubectl get all

NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
service/kubernetes   ClusterIP   10.96.0.1            443/TCP   19h
</code></pre>

Learn more about the [Helm package manager and its use](https://helm.sh/docs/).


## Kustomize

Kustomize is a standalone tool to customize Kubernetes objects through a kustomization file. Since 1.14, kubectl also supports the management of Kubernetes objects using a kustomization file. To view resources found in a directory containing a kustomization file, run the following command: <code>kubectl kustomize <kustomization_directory></code>

To apply those resources, run kubectl apply with <code>--kustomize</code> or <code>-k</code> flag: <code>kubectl apply -k <kustomization_directory></code>

Kustomize features include:

Generating resources from other resources

Setting cross-cutting fields for groups of resources

Composing and customizing collections of resources

Imagine we want to update our project to include standard project labels and other features. We can create a <code>kustomization.yaml</code> file something like this:

<pre class="wp-block-code"><code>
namespace: my-namespace
namePrefix: dev-
nameSuffix: "-001"
labels:
  - pairs:
      app: bingo
    includeSelectors: true
commonAnnotations:
  oncallPager: 800-555-1212
resources:
- deployment.yaml
</code></pre>

An example deployment that would be modified by this kustomization:

<pre class="wp-block-code"><code>
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  labels:
    app: my-app
spec:
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: app
        image: my-app
</code></pre>

This will add prefixes and suffixes to all of our project files. Build the project:

<pre class="wp-block-code"><code>
$ kubectl kustomize ./

apiVersion: apps/v1
kind: Deployment
metadata:
  annotations:
    oncallPager: 800-555-1212
  labels:
    app: bingo
  name: dev-my-app-001
  namespace: my-namespace
spec:
  selector:
    matchLabels:
      app: bingo
  template:
    metadata:
      annotations:
        oncallPager: 800-555-1212
      labels:
        app: bingo
    spec:
      containers:
      - image: my-app
        name: app
</code></pre>

Kustomize provides similar features to helm but in a lighter weight solution. Both have their advantages and shortcomings.


# Understand extension interfaces (CNI, CSI, CRI, etc.)

Kubelet uses 3 key interfaces to enable flexible container runtime, network, and storage features:

- Container Runtime Interface (CRI) – enables the kubelet to use any CRI compliant container runtime to run containers
- Container Network Interface (CNI) – enables the kubelet to use any CNI compliant container networking solution to attach containers to networks
- Container Storage Interface (CSI) – enables the kubelet to use any CSI compliant container storage implementation connect containers to storage volumes

These interfaces are implemented as either:

- an executable (CNI)
- or daemon (CRI, CSI)


## CRI

CRI enables pluggable container runtimes in Kubernetes. Kubelet previously had integral support for Docker and Rocket (rkt). CRI Consists of:

- Specifications and requirements
- Defines a gRPC API comprised of two services:
  - ImageService – pull, inspect, and remove images
  - RuntimeService – manage containers (create / start / exec / attach / etc.)

Kubelet calls the container runtime (or a shim) over a Unix socket. Depending on your container runtime of choice, you will need to supply the appropriate value for the <code>--cri-socket</code> option: <code>sudo kubeadm init --cri-socket=unix:///var/run/containerd/containerd.sock</code>


## CNI

Kubernetes kubelets create Pods, which requires the kubelet to configure pod networking. The CNI plugin is selected by passing kubelet the <code>--network-plugin=cni</code> command-line option. Kubelet reads a file from <code>--cni-conf-dir</code> (default <code>/etc/cni/net.d</code>) and uses the CNI configuration from that file to set up each pod’s network. The CNI configuration file must match the CNI specification. Any required CNI plugins referenced by the configuration must be present in <code>--cni-bin-dir</code> (default <code>/opt/cni/bin</code>). It is the CNI plugin’s responsibility to, among other things, assign IP addresses to pods in the cluster. Different CNI plugins will achieve this in different ways.


## CSI

CSI is a standard for exposing block and file storage systems to containerized workloads on container orchestration systems like Kubernetes. Allows third parties to write and deploy plugins exposing new storage systems in Kubernetes without ever having to touch the core Kubernetes code.

How it works:

- Administrators deploy an in-cluster agent, known as a driver, which acts as provisioner for a Kubernetes storage class
- Users define a PVC that uses the driver’s storage class
- When the PVC is created, the driver will use an external provisioner that calls the backend to create the a volume for the requested PV that binds the PVC


# Understand CRDs, install and configure operators

Custom resources are software extensions of the Kubernetes API and represent a customization of a given Kubernetes cluster. Custom resources make Kubernetes modular; a cluster or platform administrator or organization can pick and choose which additional features to add or remove to/from clusters.

Custom Resource Definitions (CRDs) are the K8s resources that declare custom APIs. They define a name for the API, endpoint, and a schema for type enforcement. On their own, CRDs simply allow users to declare new custom resources to the K8s API for storage in the etcd database. A custom controller is required to keep the current state of Kubernetes objects in sync with your declared/desired state. The combination of CRD(s) and custom controller(s) is known as the “operator pattern”.

The most common way to deploy an “operator” is to add one or more CRDs and their associated custom controller to a cluster. The controller itself runs outside of the control plane, like any other containerized application: as a pod (or set of pods) controlled by a standard K8s controller like a Deployment.

As an example, we will use an operator from Bitnami called Sealed Secrets. The Sealed Secrets project offers an officially-supported Helm chart we can use that will install the CRDs and the custom controller. The example below assumes you have previously installed helm.

<pre class="wp-block-code"><code>
$ helm repo add sealed-secrets https://bitnami-labs.github.io/sealed-secrets

$ helm install sealed-secrets -n kube-system \
  --set-string fullnameOverride=sealed-secrets-controller \
  sealed-secrets/sealed-secrets
</code></pre>

Now list your CRDs:

<pre class="wp-block-code"><code>
$ kubectl get crd | grep bitnami

sealedsecrets.bitnami.com   2025-02-06T01:05:36Z
</code></pre>

List your API resources:

<pre class="wp-block-code"><code>
$ kubectl api-resources --api-group=bitnami.com

NAME            SHORTNAMES   APIVERSION             NAMESPACED   KIND
sealedsecrets                bitnami.com/v1alpha1   true         SealedSecret
</code></pre>

Check that the custom controller was successfully deployed:

<pre class="wp-block-code"><code>
$ kubectl -n kube-system get all -l app.kubernetes.io/name=sealed-secrets

NAME                                             READY   STATUS    RESTARTS   AGE
pod/sealed-secrets-controller-7b76d5bb5b-czhv6   1/1     Running   0          100s

NAME                                        TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
service/sealed-secrets-controller           ClusterIP   10.99.98.163    <none>        8080/TCP   100s
service/sealed-secrets-controller-metrics   ClusterIP   10.107.63.112   <none>        8081/TCP   100s

NAME                                        READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/sealed-secrets-controller   1/1     1            1           100s

NAME                                                   DESIRED   CURRENT   READY   AGE
replicaset.apps/sealed-secrets-controller-7b76d5bb5b   1         1         1       100s
</code></pre>

Understanding how to deploy and find custom APIs, custom resources, and custom controllers should be something you are familiar with for the exam.


# Practice Drill

Create a role and a role-binding that gives a user named <code>networker</code> permissions to get and list the ingresses and network policies in the default namespace.


# CONTENT FROM BEFORE FEB 2025 REMAINS BELOW IN CASE THE EXAM INCLUDES IT IN THE FUTURE


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

[Learn more about performing backup and restore operations for etcd here.](https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/).
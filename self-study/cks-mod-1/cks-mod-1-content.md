# CKS Self-Study Mod 1


### Using Network Security Policies to restrict cluster level access

Network Policies are crucial to controlling pod-to-pod access in a cluster. Network policies enable cluster administrators
to enforce:

- Pod to Pod communication within and between namespaces in the cluster
  - Can by done by port or by label
- Pod to other destinations, like certain CIDR blocks (0.0.0.0, 172.168.0.0/16, 10.255.255.255/32)

When a network policy is put into place in a namespace, by default all incoming traffic to pods within that namespace (AKA ingress) is blocked while all outgoing traffic (called Egress) remains unblocked. Additional rules to block egress or allow ingress to certain pods must be present:

The following Network Policy selects all pods in the `meta` namespace and explicitly prevents all Ingress traffic into those pods:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: meta
spec:
  podSelector: {}
  policyTypes:
  - Ingress
```

If you add `- Egress` to the `policyTypes` section of the network policy, the network policy will prevent all outgoing traffic from all affected pods in the `meta` namespace.

To allow incoming traffic to pods in the affected namespace, you must add the `ingress` key into the network policy's spec. Under that key, you can define one or more rules that define what pods, namespaces, or IP Ranges (CIDRs).

The following Network Policy spec allows Ingress traffic from all pods within the `10.0.0.0/8` CIDR block (covering IP addresses ranging from 10.0.0.0 to 10.255.255.255) that are labeled with the key-value pair `network=approved`:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: test-network-policy
  namespace: default
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  ingress:
  - from:
    - ipBlock:
        cidr: 10.0.0.0/8
      podSelector:
        matchLabels:
          network: approved
```

Network policies are an essential way to control network access between pods in your cluster. [You can learn more about Kubernetes network policies and how to use them here](https://kubernetes.io/docs/concepts/services-networking/network-policies/).


### Use CIS benchmark to review the security configuration of Kubernetes components

The Center for Internet Security (CIS) publishes documents known as CIS Benchmarks which cover many common security configurations for popular systems, including Kubernetes. The CIS benchmark for Kubernetes provides guidance on how to establish the best possible security posture for your Kubernetes control plane components.  These include:

- The API Server
- Controller Manager
- Scheduler
- etcd
- kube-proxy

Each of the items on the CIS Benchmark document presents a configuration option, an associated vulnerability, how to audit your system for the vulnerability, and, if possible, a remediation for the vulnerability. One such entry for basic authentication on the API Server is presented like this:

>**1.2.2 Ensure that the --basic-auth-file argument is not set (Automated)**
>
>**Profile	Applicability:**
>
>•		Level	1	- Master	Node
>
>**Description:**
>
>Do	not	use	basic	authentication.
>
>**Rationale:**
>
>Basic	authentication	uses	plaintext	credentials	for	authentication.	Currently,	the	basic	authentication	credentials	last	indefinitely,	and	the	password	cannot	be	changed	without	restarting	the	API	server.	The	basic	authentication	is	currently	supported	for	convenience. Hence,	basic	authentication	should	not	be	used.
>
>**Audit:**
>
> Run	the	following	command	on	the	master	node:
> 
> `ps -ef | grep kube-apiserver`
>
> Verify	that	the	--basic-auth-file argument	does	not	exist.
>
>**Remediation:**
>
> Follow	the	documentation	and	configure	alternate	mechanisms	for	authentication.	Then,	edit	the	API	server	pod	specification	file	/etc/kubernetes/manifests/kubeapiserver.yaml on	the	master	node	and	remove	the	--basic-auth-file=<filename> parameter.
>
>**Impact:**
>
>You	will	have	to	configure	and	use	alternate	authentication	mechanisms	such	as	tokens	and	certificates.	Username	and	password	for	basic	authentication	could	no	longer	be	used.
>
>**Default	Value:**
>
>By	default,	basic	authentication	is	not	set.

The CIs Benchmark guideline above is noted as `automated`, meaning most deployment tools (like Kubeadm) will perform this step for users upon creation. The other category of guidelines are those marked as `manual` which means users much take manual action in order to meet the specified guideline.

The curriculum emphasizes the CIS benchmarks because every Certified Kubernetes Security Specialist should be aware of the CIS Benchmark and more importantly how to implement its guidelines into their clusters in production. You do not need to memorize every single guideline (as there are over 100) but you need to know how to configure your cluster to meet or exceed those guidelines.

To check and/or implement the CIS Benchmark, for example, you need to understand how your API Server is running in production (whether it is a standalone Linux service or as a static pod in your Kubernetes cluster) and apply the suggested changes to the API Server's runtime options.

Take the CIS guideline listed above, which wants us to ensure that the API Server does not use basic authentication. It is an automated guideline, meaning the the tool you used to initialize your cluster may have already done this for you. It is worth checking, so check your API Server's static pod manifest and see if the `--basic-auth-file` argument is present:

First, you need to know where your API Server's configuration is stored. In `kubeadm` initialized clusters, you will find these in `/etc/kubernetes/manifests/`. Once you locate the file, use a tool like `grep` to find the relevant text:

```
$ sudo cat /etc/kubernetes/manifests/kube-apiserver.yaml | grep basic-auth-file

$ 
```

Your query returns no result, so that means your API Server is complying with this particular CIS Benchmark guideline!

Make sure to check back often since the CIS Benchmark may add or change guidelines as new security issues are identified. [The CIS Benchmark for Kubernetes is available here for free with sign up.](https://www.cisecurity.org/benchmark/kubernetes/) 


### Setting up Ingress objects with security control

The Kubernetes Ingress subsystem (not to be confused with network policy ingress) provides external access to workloads running in your cluster through the use of a software load balancer managed by Kubernetes. The Ingress subsystem consists of three parts:

- A software load balancer that usually runs within the Kubernetes cluster which acts as the public endpoint for the exposed workloads.
- Ingress Controllers which are pieces of software that inform and configure the software load balancer to route incoming requests from the outside to services running inside the 
- Ingress Rules that are read by the Ingress Controllers which define the actual routing between intended service destination and the software load balancer

Ingress Controllers are typically bundled with software load balancers (webservers like NGINX or proxies like Envoy) and a set of custom resource definition that offer expanded routing and security capabilities over the standard Ingress rules provided by Kubernetes. Popular security features that many (if not all) Ingress controllers provide include rate limiting and TLS termination. 

One way to secure Ingress is rate limiting, which controls the amount of traffic that is accepted by the load balancer. Rate limiting is implemented differently depending on which Ingress Controller and load balancer you use, but in all cases you will typically have control over the maximum amount of connections to the Ingress controller's managed load balancer or how long those connections should be kept alive. Implementing rate limiting in your Ingress rules ensures that your cluster's worker nodes have some degree of protection against attacks like large scale distributed denial of service (DDOS) attacks.

The following Ingress rule tells the NGINX Ingress Controller to limit the maximum number of requests per second (rps) that it will accept:

```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: learning-service
  annotations:
    nginx.ingress.kubernetes.io/limit-rps: "5"
spec:
  tls:
  - hosts:
      - k8s.example.internal
    secretName: tls-termination
  rules:
  - host: k8s.example.internal
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: learning
            port:
              number: 80
```

Another way of securing ingress is by using TLS Termination, which configures the managed load balancer to handle incoming and outgoing TLS handshakes between cluster workloads and their clients. TLS termination allows The following Ingress resource definition implements TLS termination:

```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: learning-service
spec:
  tls:
  - hosts:
      - k8s.example.internal
    secretName: tls-termination
  rules:
  - host: k8s.example.internal
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: learning
            port:
              number: 80
```

Ingress rules backed by TLS termination must use a `tls` type of secret in the same namespace in order to successfully receive a client's request. Once TLS is terminated at the Ingress load balancer, all data is passed within the cluster as plaintext until the response from the service is returned to the client. TLS termination ensures that traffic to and from services that may not be able to handle TLS still benefit.


### Protecting Node Metadata

Public cloud services like AWS, GCE, Azure, or Linode provides ways of exposing node metadata (such as metrics, labels, or other information) to applications running on them. The way these metadata are presented depends on the public cloud provider, but they can include:

- Specific endpoints, like AWS' `169.254.169.254` link-local address
- Files on the VM's filesystem
- Third party services that run on your cloud instances

The most effective way to protect node metadata is to disable those capabilities when creating the instance on your cloud provider's website. You can also adjust your cloud provider's IAM permissions to prevent such access. In Kubernetes, you can use network policies to block outgoing pod communication (egress) to the node metadata endpoints above:

```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: restrict-node-metadata
spec:
  podSelector: {}
  egress:
  - to:
    - ipBlock:
        cidr: 0.0.0.0/0
        except:    
        - 169.254.169.254/32
  policyTypes:
  - Egress
```

This way, all pods in the namespace this network policy is deployed in are able to send outgoing requests everywhere but the link-local address that serves the instance's metadata. Now no pods running in that namespace on the cluster can touch the node metadata. Remember to check your cloud provider's particular implementation details for more information on how to control access to instance metadata. 

[As for creating the network policy, you can learn more about network policies here](https://kubernetes.io/docs/concepts/services-networking/network-policies/)




### Minimize use of, and access to, GUI elements

In 2018, attackers breached and exploited Tesla's Kubernetes infrastructure by exploiting an insecurely configured Kubernetes Dashboard GUI. After the breach, the attackers secured one of Tesla's AWS account keys and deployed cryptomining software in their infrastructure. There were a combination of factors that led to this breach:

- Sensitive information (specifically AWS access keys) were stored unencrypted in Kubernetes secrets
- The Kubernetes Dashboard did not have any authentication configured
- The Dashboard user had unrestricted permissions that allowed sensitive information like secrets to be shown in plaintext

Kubernetes secrets are a convenient but not the most secure way to distribute sensitive information in a Kubernetes cluster. Data stored in secrets are merely encoded in base64 which is easily unencoded. The Kubernetes API Server can be configured to encrypt secrets at rest, which is documented [here](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/).

Secret encryption is only the first step. The main factors were that the attackers were able to get unrestricted access to the Kubernetes cluster through the GUI. Even if the secrets are encrypted in etcd, any user that has permission to view those secrets and has access to the keys will still be able to see the contents. Thus, role-based access controls (RBAC) is the best way to limit Kubernetes GUI users. At the time of the breach, RBAC was not widely used as the default access control mechanism. 

The final factor of the attack was made possible due to a "Skip Login" option enabled by default in that version of the Kubernetes Dashboard, which is explained in [CVE-2018-18264](https://nvd.nist.gov/vuln/detail/CVE-2018-18264). By using the Skip Login option, the attackers accessed the Kubernetes Dashboard using the dashboard's service account. Combined with the excessive permissions from the lack of RBAC at the time enabled the attackers to see all of the secrets. The latest versions (those above 1.10.1) disable the "Skip Login" feature, so running the latest version of Dashboard addresses this weakness.

To address this specific attack, any service account allowed to use the GUI should omit any permission to access secrets. One such role can look like this:

```
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: myrole
rules:
- apiGroups:
  - ""
  resources:
  - services
  verbs:
  - get
  - list
  - create
  - update
  - delete
- apiGroups:
  - ""
  resources:
  - configmaps
  - pods
  verbs:
  - get
  - list
- apiGroups:
  - apps
  resources:
  - deployments
  verbs:
  - get
  - list
  - create
  - update
  - delete
```

This role grants a user enough permission to view non-sensitive resources in the cluster but still create workloads using the Kubernetes dashboard.

[This domain topic is very involved with Kubernetes Role-based access control, which you can learn more about here](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)


### Verifying platform binaries before deploying

Supply chain attacks can affect all pieces of software, including Kubernetes. Whenever you install Kubernetes whether through a package manager like `apt` or `yum` or the binaries from GitHub, you want to be sure that the binaries you use for your control plane or worker nodes are the genuine deal.

Every Kubernetes binary package for a given version is published with a matching sha512 hash.

After you download one of the binary packages, like the server binaries [here](https://kubernetes.io/docs/setup/release/notes/#server-binaries-4), use the `sha512sum` tool in conjunction with the published SHA512 hash on the release page to verify that the file you downloaded is the same one the Kubernetes team produced for the release.

An example of such a workflow is as follows:

- Download one of the release packages from the Kubernetes website or GitHub
- Start the `sha512sum` tool with the `--check` or `-c` options
- Enter the published sha512 hash from the release page followed by the name of the file you downloaded
- If the sha512 hash matches that calculated from the provided file, the `sha512sum` tool will report `OK` and you can be sure of the files authenticty.

```
$ wget -q https://dl.k8s.io/v1.20.0-beta.0/kubernetes-server-linux-amd64.tar.gz

$ sha512sum -c

30d982424ca64bf0923503ae8195b2e2a59497096b2d9e58dfd491cd6639633027acfa9750bc7bccf34e1dc116d29d2f87cbd7ae713db4210ce9ac16182f0576 kubernetes-server-linux-amd64.tar.gz

kubernetes-server-linux-amd64.tar.gz: OK

^C

$
```

[Download links to all of the Kubernetes release packages are available on the release notes page here](https://kubernetes.io/docs/setup/release/).


### Practice Drill

Test your knowledge with the following drill:

- Create a new namespace named `self-study`. 
- In that namespace, create a network policy that prevents all incoming and outgoing pod traffic. 
- Finally, create a network policy in the appropriate namespace that allows pods from namespaces labeled `approved=true` to communicate with all pods in the `self-study` namespace.
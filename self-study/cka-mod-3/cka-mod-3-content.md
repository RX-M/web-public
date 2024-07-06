<!-- CKA Self-Study Mod 3 -->


# Understand host networking configuration on the cluster nodes

A Kubernetes cluster consists of one or more computers that run the Kubernetes agent, kubelet. The Kubelet is responsible for running containers managed by a Kubernetes cluster. It does this by taking pod specifications from the Kubernetes API Server and contacting the appropriate agents, services, and other components to realize the state described by the pod spec. This includes creation of containers, fulfilling of storage requests, and also networking.

Ideally, the nodes in a Kubernetes cluster exist within the same network space (region, datacenter, VPC, subnet, etc.). However, as long as they can "see" each other's containers over the network then that is acceptable. 


# Understand connectivity between Pods

Once a CNI plugin is installed on a cluster, all containers in that cluster can establish connections to and accept connections from each other using their assigned Pod IP addresses. Communications are not restricted by default.

This can be controlled using Kubernetes Network Policy. Network Policy allows users to define whether pods can accept or deny the establishment of incoming/outgoing connection from other pods. Traffic can be controlled in the following ways:

<ul>
  <li>Between pods in the same namespaces</li>
  <li>Between pods in different namespaces</li>
  <li>Between pods and user-defined IP Blocks</li>
</ul>

An example use of network policy is to strictly enforce the communication path between one or more microservices of the same application. If component A in namespace alpha only establishes connections to component B in namespace bravo on port 9898, then a network policy can be created to restrict communications between those two pods on those parameters.

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


# Understand ClusterIP, NodePort, LoadBalancer service types and endpoints

A service is an abstraction of a logical set of pods and a policy that defines inbound and network access. A service uses a selector to target pods by the pods’ label. A service exposes a logical set of pods as a network service providing a single IP address, DNS name, or load balancing to access the pods.

The service type is defined in the manifest. The following are available service types:
ClusterIP - exposes the service on an internal IP in the Kubernetes cluster (default)
NodePort - exposes the service on the same port of each node in the Kubernetes cluster
LoadBalancer - creates an external load balancer with a cloud provider (e.g. GCE ForwardingRules, AWS Elastic Load Balancer, Azure Load Balancer) and assigns a public IP to the service
ExternalName - exposes the an external DNS address as an endpoint of a Kubernetes service (useful for allowing pods to communicate with that service using a Kubernetes-internal DNS name)

Services can be created imperatively for a running resource. At minimum the resource type, resource name, and the service’s exposed proxy port are required e.g. <code>kubectl expose <resource> <resource_name> --port=<port number></code>.

<pre class="wp-block-code"><code>
$ kubectl create deploy webserver --image nginx

deployment.apps/webserver created

$ kubectl expose deploy webserver --port 80

service/webserver exposed

$ kubectl get svc

NAME              TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
kubernetes        ClusterIP   10.96.0.1        <none>        443/TCP        33d
webserver         ClusterIP   10.103.175.171   <none>        80/TCP         4s

$
</code></pre>

Services select pods using labels, and for each pod creates an endpoint resource. The endpoint resource describes all active network targets (pods) that the service routes traffic to. Each endpoint object in a cluster places an additional iptables rule with a target pod’s IP. An alternative to endpoints are EndpointSlices. EndpointSlices are conceptually and functionally similar to endpoints, but are restricted to up to 100 endpoints to improve management at scale.

<pre class="wp-block-code"><code>
$ kubectl get endpoints webserver

NAME        ENDPOINTS      AGE
webserver   10.32.0.8:80   43s

$ kubectl get pods -o wide -l app=webserver

NAME                        READY   STATUS    RESTARTS   AGE   IP          NODE     NOMINATED NODE   READINESS GATES
webserver-d698d7bd6-ktxvn   1/1     Running   0          83s   10.32.0.8   ubuntu   <none>           <none>

$
</code></pre>

Ingresses are another resource that interact with services. Ingresses bind services to external endpoints that an Ingress controller on the cluster then exposes to the outside world. Ingresses reference services directly in their manifests, as shown here:

<pre class="wp-block-code"><code>
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: webserver-ingress
  annotations:
spec:
  rules:
  - http:
      paths:
      - path: /testpath
        backend:
          serviceName: webserver
          servicePort: 80
</code></pre>

Learn more about:
<li>[Services](https://kubernetes.io/docs/concepts/services-networking/service/),</li>
<li>[Endpoints](https://kubernetes.io/docs/concepts/services-networking/service/#services-without-selectors)</li>
<li>[EndpointSlices](https://kubernetes.io/docs/concepts/services-networking/endpoint-slices/)</li>
<li>[Ingresses](https://kubernetes.io/docs/concepts/services-networking/ingress/).</li>


# Know how to use Ingress controllers and Ingress resources

The Ingress resource manages external access to Kubernetes services via HTTP and HTTPS routes. An Ingress controller is required to satisfy an Ingress. The Ingress controller reads and implements the rules of the Ingress resource.

Use the following command to set up an Ingress Controller in your Kubernetes cluster:
<pre class="wp-block-code"><code>
$ kubectl apply -f https://raw.githubusercontent.com/RX-M/classfiles/master/bootcamp-drills/ingress-drill-setup.yaml

namespace/nginx-ingress created
serviceaccount/nginx-ingress created
clusterrole.rbac.authorization.k8s.io/nginx-ingress created
service/nginx-ingress created
clusterrolebinding.rbac.authorization.k8s.io/nginx-ingress created
secret/default-server-secret created
deployment.apps/nginx-ingress created

$
</code></pre>

Create the following deployment of Apache webserver that exposes the container port 80:
<pre class="wp-block-code"><code>
$ kubectl create deploy apache-webserver --image=httpd --port=80

deployment.apps/apache-webserver created

$
</code></pre>

Create a NodePort service to expose the <code>apache-webserver</code> deployment on the node port 30111 and maps port 80 on the ClusterIP to port 80 on the container:
<pre class="wp-block-code"><code>
$ kubectl create service nodeport apache-webserver --tcp=80:80 --node-port=30111

service/apache-webserver created

$
</code></pre>

Create the following Ingress resource for the <code>apache-webserver</code> service that controls traffic to the host domain www.example.com, exposes an http prefix path to /, routes all traffic sent to www.example.com:30111/ to the <code>apache-webserver</code> service on port 80:
<pre class="wp-block-code"><code>
$ nano apache-webserver-ingress.yaml && apache-webserver-ingress.yaml

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: apache-weberver-ingress
spec:
  ingressClassName: nginx
  rules:
  - host: www.example.com
    http:
      paths:
      - pathType: Prefix
        path: /
        backend:
          service:           
            name: apache-webserver
            port:
              number: 80

$ kubectl apply -f apache-webserver-ingress.yaml

ingress.networking.k8s.io/apached-webserver-ingress created

$
</code></pre>

Test the Ingress rules with <code>curl --resolve www.example.com:30111:<your-IP> http://www.example.com:30111/</code>:

<pre class="wp-block-code"><code>
$ curl --resolve www.example.com:30111:<Your IP> http://www.example.com:30111

<html><body><h1>It works!</h1></body></html>

$
</code></pre>

Learn more about:
- [Ingress resource](https://kubernetes.io/docs/concepts/services-networking/ingress/)
- [Ingress controllers](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/).


# Know how to configure and use CoreDNS

Kubernetes uses CoreDNS for DNS-based service discovery. CoreDNS is flexible and changes can be made in the ConfigMap for CoreDNS.

Every Service is assigned with a DNS name in the syntax: <code><service-name>.<namespace>.svc.cluster.local</code>.

Pods are assigned a DNS A record in the syntax of: <code><pod-hyphen-separated-ip>.<namespace>.pod.cluster.local</code>.

Let’s confirm the DNS entry of a service with a name server lookup with <code>nslookup</code> from within a pod.

Create a ClusterIP Service to test its DNS entry and retrieve it’s ClusterIP:
<pre class="wp-block-code"><code>
$ kubectl create service clusterip my-service --tcp=8080:8080

service/my-service created

$ kubectl get service

NAME                 TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
kubernetes         ClusterIP   10.96.0.1             <none>              443/TCP        10d
my-service         ClusterIP   10.109.165.220   <none>              8080/TCP       5m31s

$
</code></pre>

Run a pod with the <code>busybox</code> image and run a nslookup on the service’s IP:
<pre class="wp-block-code"><code>
$ kubectl run busybox --image=busybox -it -- /bin/sh

If you don’t see a command prompt, try pressing enter.
/ # nslookup 10.109.165.220

Server:		10.96.0.10
Address:	10.96.0.10:53

220.165.109.10.in-addr.arpa	name = my-service.default.svc.cluster.local

/ # exit

$
</code></pre>


Learn more about:
<li>[CoreDNS](https://kubernetes.io/docs/tasks/administer-cluster/coredns/)</li>
<li>[DNS for Services and Pods](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/).</li>


# Choose an appropriate container network interface plugin

The container network in a Kubernetes cluster is maintained by the container network interface (CNI) plugins installed by the cluster administrators. CNI plugins provide: 

<ul>
<li>The interface necessary to enable the Kubelet to request pod-level IP addresses for the cluster's containers</li>
<li>The virtual infrastructure meant to handle requests between containers in a cluster</li>
<li>Additional features like rate limiting, circuit breaking, or network policy to help control the network traffic in a cluster</li>
</ul>

A CNI plugin must be installed on a Kubernetes cluster before any workloads can run. If the CNI plugin and its components on a node go offline, then that node is considered not ready and tainted - restricting it from running workloads until the CNI plugin is operational.


# Practice Drill

Run the following command:

<code>kubectl run --image docker.io/nginx nginx-drill</code>

Create a NodePort service that allows you to send a curl request to the nginx-drill pod at port 80 through your machine’s IP address.

<!-- CKA Self-Study Mod 3 -->


# Understand connectivity between Pods

The Kubernetes project defines 4 distinct networking problems to solve:

- Highly-coupled container-to-container communications
- Pod-to-pod communications
- Pod-to-service communications
- External-to-internal communications

## Container-to-Container

Because all containers within a pod share the Linux kernel network namespace, they can communicate with each other just like processes on the same host did in the pre-container world. They can communicate over the loopback interface and can reach each other's ports via localhost. Containers share the same port space--which may cause conflicts unless care is taken to avoid using the same ports.

## Pod-to-Pod

Once a Container Network Interface (CNI) plugin is properly installed on a cluster, all pods in that cluster can establish connections to and accept connections from each other using their unique cluster-wide IP addresses. All pods can communicate with all other pods on the same node or on different nodes. Communications are not restricted by default and pods can communicate with each other directly without proxies or NAT. Similarly, host-level agents (like system daemons, the kubelet, etc.) can communicate with all pods on that node.

## Pod-to-Service

Services provide an abstraction that groups pods under a stable IP address or hostname. The pods making up the service can change over time but the service remains long-lived, beyond the lifecycle of any individual pod. Kubernetes manages EndpointSlice objects which provide information about the pods currently backing a service. The kube-proxy monitors services and EndpointSlice objects and programs IPTables or IPVS Tables to route service traffic to backends.

## External-to-Internal

The LoadBalancer service type generally implements this by setting up external load balancers (in public clouds) or informs MetalLB when using standard network equipment, which target all nodes in a cluster. When traffic arrives at a node, rules are written by the kube-proxy in IPTables or IPVS Tables so that the traffic is routed to a backend pod associated to the service.

Kubernetes Ingress, or more recently the Gateway API, enables making services accessible to external clients as well. Ingress may provide load balancing, SSL termination and name-based virtual hosting where LoadBalancer services do not. The Gateway API is an add-on containing API kinds that provide dynamic infrastructure provisioning and advanced traffic routing.

## Configuration

To facilitate these different communication patterns, Kubernetes clusters require non-overlapping IP address ranges for pods, services, and nodes. Addresses are are assigned by various components in the cluster:

- The Container Network Interface (CNI) plugin is responsible for assigning IP addresses to pods
- The kube-apiserver is configured to assign IP addresses to services using the --service-cluster-ip-range flag
- The kubelet or cloud-controller-manager assigns IP addresses to nodes


# Define and enforce Network Policies

Network Policy allows users to define whether pods can accept or deny the establishment of incoming/outgoing connection from other pods. Traffic can be controlled in the following ways:

<ul>
  <li>Between pods in the same namespaces</li>
  <li>Between pods in different namespaces</li>
  <li>Between pods and user-defined IP Blocks</li>
</ul>

An example use of network policy is to strictly enforce the communication path between one or more microservices of the same application. If component A in namespace alpha only establishes connections to component B in namespace bravo on port 9898, then a network policy can be created to restrict communications between those two pods on those parameters.

When a network policy is put into place in a namespace, by default all incoming traffic to pods within that namespace (AKA ingress) is blocked while all outgoing traffic (called Egress) remains unblocked. Additional rules to block egress or allow ingress to certain pods must be present:

The following Network Policy selects all pods in the <code>meta</code> namespace and explicitly prevents all Ingress traffic into those pods:

<pre class="wp-block-code"><code>
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: meta
spec:
  podSelector: {}
  policyTypes:
  - Ingress
</code></pre>

If you add <code>- Egress</code> to the <code>policyTypes</code> section of the network policy, the network policy will prevent all outgoing traffic from all affected pods in the <code>meta</code> namespace.

To allow incoming traffic to pods in the affected namespace, you must add the <code>ingress</code> key into the network policy's spec. Under that key, you can define one or more rules that define what pods, namespaces, or IP Ranges (CIDRs).

The following Network Policy spec allows ingress traffic to pods labeled <code>app=backend</code> from pods within the <code>meta</code> namespace that are labeled with the key-value pair <code>app=frontend</code>:

<pre class="wp-block-code"><code>
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: example-network-policy
  namespace: meta
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
</code></pre>

Network policies are an essential way to control network access between pods in your cluster. [You can learn more about Kubernetes network policies and how to use them here](https://kubernetes.io/docs/concepts/services-networking/network-policies/).


# Understand ClusterIP, NodePort, LoadBalancer service types and endpoints

A service is an abstraction of a logical set of pods and a policy that defines inbound and network access. A service uses a selector to target pods by the pods’ label. A service exposes a logical set of pods as a network service providing a single IP address, DNS name, or load balancing to access the pods.

The service type is defined in the manifest. The following are available service types:
ClusterIP - exposes the service on an internal IP in the Kubernetes cluster (default)
NodePort - exposes the service on the same port of each node in the Kubernetes cluster
LoadBalancer - creates an external load balancer with a cloud provider (e.g. GCE ForwardingRules, AWS Elastic Load Balancer, Azure Load Balancer) and assigns a public IP to the service
ExternalName - exposes the an external DNS address as an endpoint of a Kubernetes service (useful for allowing pods to communicate with that service using a Kubernetes-internal DNS name)

Services can be created imperatively for a running resource. At minimum the resource type, resource name, and the service’s exposed proxy port are required e.g. <code>kubectl expose <resource_name> --port=<port-number> --type=<type></code> where type is one of <code>ClusterIP</code> (default), <code>NodePort</code>, <code>LoadBalancer</code>, or <code>ExternalName</code>.

<pre class="wp-block-code"><code>
$ kubectl create deploy webserver --image nginx

deployment.apps/webserver created

$ kubectl expose deploy webserver --port 80

service/webserver exposed

$ kubectl get svc

NAME              TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
kubernetes        ClusterIP   10.96.0.1        <none>        443/TCP        33d
webserver         ClusterIP   10.103.175.171   <none>        80/TCP         4s
</code></pre>

Services select pods using labels, and for each pod creates an endpoint resource. The endpoint resource describes all active network targets (pods) that the service routes traffic to. Each endpoint object in a cluster places an additional iptables rule with a target pod’s IP. An alternative to endpoints are EndpointSlices. EndpointSlices are conceptually and functionally similar to endpoints, but are restricted to up to 100 endpoints to improve management at scale.

<pre class="wp-block-code"><code>
$ kubectl get endpoints webserver

NAME        ENDPOINTS      AGE
webserver   10.32.0.8:80   43s

$ kubectl get pods -o wide -l app=webserver

NAME                        READY   STATUS    RESTARTS   AGE   IP          NODE     NOMINATED NODE   READINESS GATES
webserver-d698d7bd6-ktxvn   1/1     Running   0          83s   10.32.0.8   ubuntu   <none>           <none>
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
          serviceName: webserver     # links the ingress to the service
          servicePort: 80
</code></pre>

Learn more about:
<li>[Services](https://kubernetes.io/docs/concepts/services-networking/service/),</li>
<li>[Endpoints](https://kubernetes.io/docs/concepts/services-networking/service/#services-without-selectors)</li>
<li>[EndpointSlices](https://kubernetes.io/docs/concepts/services-networking/endpoint-slices/)</li>


# Use the Gateway API to manage Ingress traffic


Traditionally, an Ingress is a collection of rules that allow traffic from outside the cluster to reach the services inside the cluster using HTTP/S. Ingresses gave services externally-reachable URLs and can load balance traffic, terminate SSL and more. Users requested ingress by creating Ingress resources. The Gateway API implements the same concept with a different set of Kubernetes resources:

- <code>GatewayClass</code>
- <code>Gateway</code>
- <code>Route</code> family of resources (e.g. <code>HTTPRoute</code> or <code>GRPCRoute</code>)

Gateway API presents a generic interface for providing an ingress-like feature set that is more granular and supports more route types compared to Ingress. It is up to various providers to create implementations of the Gateway API. Like all other extensions to a Kubernetes cluster, the Gateway API requires an agent in the cluster that handles the generic Gateway API resources. One such provider is the NGINX Gateway Fabric.

Unlike Kubernetes Ingress, which has an out-of-the-box API resource already in the cluster (and can thus be created even if there are no Ingress Controllers), the Gateway API uses custom resource definitions.

To start, install the generic Gateway API custom resources:

<pre class="wp-block-code"><code>
$ kubectl kustomize "https://github.com/nginx/nginx-gateway-fabric/config/crd/gateway-api/standard?ref=v1.5.1" | kubectl apply -f -

customresourcedefinition.apiextensions.k8s.io/gatewayclasses.gateway.networking.k8s.io created
customresourcedefinition.apiextensions.k8s.io/gateways.gateway.networking.k8s.io created
customresourcedefinition.apiextensions.k8s.io/grpcroutes.gateway.networking.k8s.io created
customresourcedefinition.apiextensions.k8s.io/httproutes.gateway.networking.k8s.io created
customresourcedefinition.apiextensions.k8s.io/referencegrants.gateway.networking.k8s.io created
</code></pre>

Next, there are a series of custom resources specific to the NGINX Gateway Fabric that need to be installed. Unlike the previous set of CRDs, they do not need to be rendered from a cluster-specific template.

<pre class="wp-block-code"><code>
$ kubectl apply -f https://raw.githubusercontent.com/nginx/nginx-gateway-fabric/v1.6.1/deploy/crds.yaml

customresourcedefinition.apiextensions.k8s.io/clientsettingspolicies.gateway.nginx.org created
customresourcedefinition.apiextensions.k8s.io/nginxgateways.gateway.nginx.org created
customresourcedefinition.apiextensions.k8s.io/nginxproxies.gateway.nginx.org created
customresourcedefinition.apiextensions.k8s.io/observabilitypolicies.gateway.nginx.org created
customresourcedefinition.apiextensions.k8s.io/snippetsfilters.gateway.nginx.org created
customresourcedefinition.apiextensions.k8s.io/upstreamsettingspolicies.gateway.nginx.org created
</code></pre>

With the CRDs in place, it's time to install the NGINX Gateway Fabric Controller resources:

<pre class="wp-block-code"><code>
$ kubectl apply -f https://raw.githubusercontent.com/nginx/nginx-gateway-fabric/v1.6.1/deploy/nodeport/deploy.yaml

namespace/nginx-gateway created
serviceaccount/nginx-gateway created
clusterrole.rbac.authorization.k8s.io/nginx-gateway created
clusterrolebinding.rbac.authorization.k8s.io/nginx-gateway created
configmap/nginx-includes-bootstrap created
service/nginx-gateway created
deployment.apps/nginx-gateway created
gatewayclass.gateway.networking.k8s.io/nginx created
nginxgateway.gateway.nginx.org/nginx-gateway-config created
</code></pre>

Define the Gateway which listens on port 80:

<pre class="wp-block-code"><code>
$ nano gw.yaml ; cat $_

apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: example-http
spec:
  gatewayClassName: nginx
  listeners:
  - name: http
    port: 80
    protocol: HTTP

$ kubectl apply -f gw.yaml

gateway.gateway.networking.k8s.io/example-http created
</code></pre>

The following rule (<code>web-httproute.yaml</code>) will send traffic hitting the gateway controller to a service called web-svc (note, we have not yet created the service).

<pre class="wp-block-code"><code>
$ nano web-httproute.yaml ; cat $_

apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: web
spec:
  parentRefs:
  - name: example-http   # References the Gateway by name
    namespace: default   # References the namespace of the Gateway (assumes the same ns if omitted)
  hostnames:
  - "www.example.com"    # The name of the host you want to the route valid on
  rules:
  - matches:
    - path:
        type: PathPrefix # States whether it is an exact match or a prefix
        value: /         # States the route address
    backendRefs:
    - name: web-svc      # References the service
      port: 80           # Reference's the service's port defined in its spec.ports list

$ kubectl apply -f web-httproute.yaml

httproute.gateway.networking.k8s.io/web created
</code></pre>

We are now ready to create the backend service. We will launch a demo app that echos its hostname and IP and expose via a service.

<pre class="wp-block-code"><code>
$ kubectl create deployment web-svc --image=docker.io/rxmllc/hostinfo:latest --replicas=3

deployment.apps/web-svc created

$ kubectl expose deploy/web-svc --port=80 --target-port=9898

service/web-svc exposed
</code></pre>

Next, we test the HTTP endpoints through the controller.

<pre class="wp-block-code"><code>
$ HTTP_NP=$(kubectl -n nginx-gateway get svc nginx-gateway -o jsonpath='{.spec.ports[0].nodePort}'); echo $HTTP_NP

30535

$ curl --resolve www.example.com:$HTTP_NP:$(hostname -i) http://www.example.com:$HTTP_NP/

web-svc-7cbb65f5c5-h7hh6 10.0.0.164

$ curl --resolve www.example.com:$HTTP_NP:$(hostname -i) http://www.example.com:$HTTP_NP/

web-svc-7cbb65f5c5-449fr 10.0.0.47
</code></pre>

It seems our controller is forwarding our traffic. Because the Gateway is submitting traffic to the service, it is subject to whatever routing strategy is in place for that service (in a default <code>kubeadm</code> cluster, this would be a random probability).

Learn more about the [Gateway API in the Kubernetes documentation](https://kubernetes.io/docs/concepts/services-networking/gateway/).


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
</code></pre>

Create the following deployment of Apache webserver that exposes the container port <code>80</code>:

<pre class="wp-block-code"><code>
$ kubectl create deploy apache-webserver --image=httpd --port=80

deployment.apps/apache-webserver created
</code></pre>

Create a NodePort service to expose the <code>apache-webserver</code> deployment on the node port <code>30111</code> and maps port <code>80</code> on the <code>ClusterIP</code> to port <code>80</code> on the pod:

<pre class="wp-block-code"><code>
$ kubectl create service nodeport apache-webserver --tcp=80:80 --node-port=30111

service/apache-webserver created
</code></pre>

Create the following Ingress resource for the <code>apache-webserver</code> service that controls traffic to the host domain <code>www.example.com</code>, exposes an http prefix path to <code>/</code>, routes all traffic sent to <code>www.example.com:30111/</code> to the <code>apache-webserver</code> service on port <code>80</code>:

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
</code></pre>

Test the Ingress rules with <code>curl --resolve www.example.com:30111:<your-IP> http://www.example.com:30111/</code>:

<pre class="wp-block-code"><code>
$ curl --resolve www.example.com:30111:<Your IP> http://www.example.com:30111

<html><body><h1>It works!</h1></body></html>
</code></pre>

Learn more about:
- [Ingress resource](https://kubernetes.io/docs/concepts/services-networking/ingress/)
- [Ingress controllers](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/).


# Understand and use CoreDNS

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

CoreDNS is configured via its Corefile which defines servers, which servers listen which ports and protocol, which zone each server is authoritative, which plugins are loaded. The Corefile is stored as a Kubernetes ConfigMap in the <code>kube-system</code> namespace:

<pre class="wp-block-code"><code>
$ kubectl get cm coredns -n kube-system -o yaml

apiVersion: v1
data:
  Corefile: |
    .:53 {
        errors
        health {
           lameduck 5s
        }
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {
           pods insecure
           fallthrough in-addr.arpa ip6.arpa
           ttl 30
        }
        prometheus :9153
        forward . /etc/resolv.conf {
           max_concurrent 1000
        }
        cache 30 {
           disable success cluster.local
           disable denial cluster.local
        }
        loop
        reload
        loadbalance
    }
kind: ConfigMap
metadata:
  creationTimestamp: "2024-06-26T01:18:34Z"
  name: coredns
  namespace: kube-system
  resourceVersion: "1570485"
  uid: 739ef761-c45d-4f32-a503-8e0b58b394fc
</code></pre>

Learn more about:
<li>[CoreDNS](https://kubernetes.io/docs/tasks/administer-cluster/coredns/)</li>
<li>[DNS for Services and Pods](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/).</li>


# Practice Drill

Run the following command:

<code>kubectl run --image docker.io/nginx nginx-drill</code>

Create a NodePort service that allows you to send a curl request to the nginx-drill pod at port 80 through your machine’s IP address.


# CONTENT FROM BEFORE FEB 2025 REMAINS BELOW IN CASE THE EXAM INCLUDES IT IN THE FUTURE


# Choose an appropriate container network interface plugin

The container network in a Kubernetes cluster is maintained by the container network interface (CNI) plugins installed by the cluster administrators. CNI plugins provide: 

<ul>
<li>The interface necessary to enable the Kubelet to request pod-level IP addresses for the cluster's containers</li>
<li>The virtual infrastructure meant to handle requests between containers in a cluster</li>
<li>Additional features like rate limiting, circuit breaking, or network policy to help control the network traffic in a cluster</li>
</ul>

A CNI plugin must be installed on a Kubernetes cluster before any workloads can run. If the CNI plugin and its components on a node go offline, then that node is considered not ready and tainted - restricting it from running workloads until the CNI plugin is operational.


# Understand host networking configuration on the cluster nodes

A Kubernetes cluster consists of one or more computers that run the Kubernetes agent, kubelet. The Kubelet is responsible for running containers managed by a Kubernetes cluster. It does this by taking pod specifications from the Kubernetes API Server and contacting the appropriate agents, services, and other components to realize the state described by the pod spec. This includes creation of containers, fulfilling of storage requests, and also networking.

Ideally, the nodes in a Kubernetes cluster exist within the same network space (region, datacenter, VPC, subnet, etc.). However, as long as they can "see" each other's containers over the network then that is acceptable. 
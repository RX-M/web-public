<!-- CKAD Self-Study Mod 4 -->

# Services & Networking


# Demonstrate basic understanding of NetworkPolicies

Network Policies are crucial to controlling pod-to-pod access in a cluster. Network policies enable cluster users<br>to enforce:

<ul class="wp-block-list">
<li>Pod to Pod communication within and between namespaces in the cluster (can by done by port or by label)</li>
<li>Pod to other destinations, like certain CIDR blocks (0.0.0.0, 172.168.0.0/16, 10.255.255.255/32)</li>
</ul>

When a network policy is put into place in a namespace, by default all incoming traffic to pods within that namespace (AKA ingress) is blocked while all outgoing traffic (called Egress) remains unblocked. Additional rules to block egress or allow ingress to certain pods must be present:

The following Network Policy selects all pods in the&nbsp;<code>meta</code>&nbsp;namespace and explicitly prevents all Ingress traffic into those pods:

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

If you add&nbsp;<code>- Egress</code>&nbsp;to the&nbsp;<code>policyTypes</code>&nbsp;section of the network policy, the network policy will prevent all outgoing traffic from all affected pods in the&nbsp;<code>meta</code>&nbsp;namespace.

To allow incoming traffic to pods in the affected namespace, you must add the&nbsp;<code>ingress</code>&nbsp;key into the network policy's spec. Under that key, you can define one or more rules that define what pods, namespaces, or IP Ranges (CIDRs).

The following Network Policy spec allows Ingress traffic from all pods within the&nbsp;<code>10.0.0.0/8</code>&nbsp;CIDR block (covering IP addresses ranging from 10.0.0.0 to 10.255.255.255) that are labeled with the key-value pair&nbsp;<code>network=approved</code>:

<pre class="wp-block-code"><code>
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
</code></pre>

Network policies are an essential way to control network access between pods in your cluster. You can learn more about <a href="https://kubernetes.io/docs/concepts/services-networking/network-policies/" target="_blank" rel="noreferrer noopener">Kubernetes network policies</a> and how to use them.


# Provide and troubleshoot access to applications via services

A service is an abstraction of a logical set of pods and a policy that defines inbound and network access. A service uses a selector to target pods by the pods’ label. A service exposes a logical set of pods as a network service providing a single IP address, DNS name, or load balancing to access the pods.

The service type is defined in the manifest. The following are available service types:
<ul>
<li>ClusterIP - exposes the service on an internal IP in the Kubernetes cluster (default)</li>
<li>NodePort - exposes the service on the same port of each node in the Kubernetes cluster</li>
<li>LoadBalancer - creates an external load balancer with a cloud provider (e.g. GCE ForwardingRules, AWS Elastic Load Balancer, Azure Load Balancer) and assigns a public IP to the service</li>
<li>ExternalName - exposes the service using an arbitrary name</li>
</ul>

Services can be created imperatively for a running resource. At minimum the resource type, resource name, and the service’s exposed proxy port are required e.g. <code>kubectl expose<resource> <resource_name> --port=<port number></code>.

<pre class="wp-block-code"><code>
$ kubectl create deploy webserver --image nginx

deployment.apps/webserver created

$ kubectl expose deploy webserver --port 80

service/webserver exposed

$ kubectl get svc

NAME              TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
kubernetes        ClusterIP   10.96.0.1        <none>        443/TCP        33d
webserver         ClusterIP   10.103.153.224   <none>        80/TCP         4s
</code></pre>

Services select pods using labels, and for each pod creates an endpoint resource. The endpoint resource describes all active network targets (pods) that the service routes traffic to. Each endpoint object in a cluster places an additional iptables rule with a target pod’s IP. An alternative to endpoints are EndpointSlices. EndpointSlices are conceptually and functionally similar to endpoints, but are restricted to up to 100 endpoints to improve management at scale.

<pre class="wp-block-code"><code>$ kubectl get endpoints webserver

NAME        ENDPOINTS      AGE
webserver   10.0.0.15:80   16s

$ kubectl get pods -o wide -l app=webserver

NAME                         READY   STATUS    RESTARTS   AGE   IP          NODE     NOMINATED NODE   READINESS GATES
webserver-7bc769cd4c-g5nc4   1/1     Running   0          28s   10.0.0.15   labsys   <none>           <none>
</code></pre>

Ingresses are another resource that interact with services. Ingresses bind services to external endpoints that an Ingress controller on the cluster then exposes to the outside world. Ingresses reference services directly in their manifests, as shown here:

<pre class="wp-block-code"><code>apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: webserver-ingress
  annotations:
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - path: /testpath
        pathType: Prefix
        backend:
          service:
            name: webserver
            port: 
              number: 80
</code></pre>

In the docs, you can learn more about <a href="https://kubernetes.io/docs/concepts/services-networking/service/" target="_blank" rel="noreferrer noopener">Services</a>, <a href="https://kubernetes.io/docs/concepts/services-networking/service/#services-without-selectors" target="_blank" rel="noreferrer noopener">Endpoints</a>, and <a href="https://kubernetes.io/docs/concepts/services-networking/endpoint-slices/" target="_blank" rel="noreferrer noopener">EndpointSlices</a>.


# Use Ingress rules to expose applications

The Ingress resource manages external access to Kubernetes services via HTTP and HTTPS routes. An Ingress controller is required to satisfy an Ingress. The Ingress controller reads and implements the rules of the Ingress resource.

Use the following command to set up an Ingress Controller in your Kubernetes cluster:

<pre class="wp-block-code"><code>
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.4.0/deploy/static/provider/baremetal/deploy.yaml

...
</code></pre>

Create the following deployment of Apache webserver that exposes the container port 80:

<pre class="wp-block-code"><code>
$ kubectl create deploy apache-webserver --image=httpd --port=80

deployment.apps/apache-webserver created
</code></pre>

Create a NodePort service to expose the <code>apache-webserver</code> deployment on the node port 30111 and maps port 80 on the ClusterIP to port 80 on the container:

<pre class="wp-block-code"><code>
$ kubectl create service nodeport apache-webserver --tcp=80:80 --node-port=30111

service/apache-webserver created
</code></pre>

Create the following Ingress resource for the <code>apache-webserver</code> service that controls traffic to the host domain www.example.com, exposes an http prefix path to /, routes all traffic sent to www.example.com:30111/ to the <code>apache-webserver</code> service on port 80:

<pre class="wp-block-code"><code>
$ nano apache-webserver-ingress.yaml ; cat $_

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

$ kubectl get svc -n ingress-nginx

NAME                                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
ingress-nginx-controller             NodePort    10.98.227.120   <none>        80:31614/TCP,443:31264/TCP   84s
ingress-nginx-controller-admission   ClusterIP   10.107.88.176   <none>        443/TCP                      84s
</code></pre>

Test the Ingress rules with <code>curl --resolve www.example.com:31614:$(hostname -i) http://www.example.com:31614/</code>:

<pre class="wp-block-code"><code>$ curl --resolve www.example.com:31614:$(hostname -i) http://www.example.com:31614

&lt;html&gt;&lt;body&gt;&lt;h1&gt;It works!&lt;/h1&gt;&lt;/body&gt;&lt;/html&gt;
</code></pre>

Learn more about <a href="https://kubernetes.io/docs/concepts/services-networking/ingress/" target="_blank" rel="noreferrer noopener">Ingress resources</a> and <a href="https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/" target="_blank" rel="noreferrer noopener">Ingress controllers</a>.


# Practice Drill

Run the following command:

<pre class="wp-block-code"><code>kubectl run --image nginx nginx-drill</code></pre>

Create a NodePort service that allows you to send a curl request to the nginx-drill pod at port 80 through your machine’s IP address.
<!-- CKAD Self-Study Mod 4 -->

# Services & Networking


## Services and Other Network Primitives

A service is an abstraction of a logical set of pods and a policy that defines inbound and network access. A service uses a selector to target pods by the pods’ label. A service exposes a logical set of pods as a network service providing a single IP address, DNS name, or load balancing to access the pods.

The service type is defined in the manifest. The following are available service types:
ClusterIP - exposes the service on an internal IP in the Kubernetes cluster (default)
NodePort - exposes the service on the same port of each node in the Kubernetes cluster
LoadBalancer - creates an external load balancer with a cloud provider (e.g. GCE ForwardingRules, AWS Elastic Load Balancer, Azure Load Balancer) and assigns a public IP to the service
ExternalName - exposes the service using an arbitrary name

Services can be created imperatively for a running resource. At minimum the resource type, resource name, and the service’s exposed proxy port are required e.g. <code>kubectl expose<resource> <resource_name> --port=<port number></code>.

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

On the docs, you can learn more about [Services](https://kubernetes.io/docs/concepts/services-networking/service/), [Endpoints](https://kubernetes.io/docs/concepts/services-networking/service/#services-without-selectors), and [EndpointSlices](https://kubernetes.io/docs/concepts/services-networking/endpoint-slices/).


## Ingress Controllers and Ingress Resources

The Ingress resource manages external access to Kubernetes services via HTTP and HTTPS routes. An Ingress controller is required to satisfy an Ingress. The Ingress controller reads and implements the rules of the Ingress resource.

Use the following command to set up an Ingress Controller in your Kubernetes cluster:
<pre class="wp-block-code"><code>
$ kubectl apply -f https://rx-m-k8s.s3-us-west-2.amazonaws.com/ingress-drill-setup.yaml

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
  ingressClass: nginx
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

Learn more about [Ingress resource](https://kubernetes.io/docs/concepts/services-networking/ingress/) and [Ingress controllers](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/).


### Using Network Policies

Network Policies are crucial to controlling pod-to-pod access in a cluster. Network policies enable cluster users
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


## Practice Drill

Run the following command:

<code>kubectl run --image nginx nginx-drill</code>

Create a NodePort service that allows you to send a curl request to the nginx-drill pod at port 80 through your machine’s IP address.

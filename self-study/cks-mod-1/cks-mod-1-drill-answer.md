<!-- CKS Self-Study Mod 1 -->

<li>Create a new namespace named <code>self-study</code>.</li>

<pre class="wp-block-code"><code>
$ kubectl create namespace self-study

namespace/self-study created

$
</code></pre>

<li>In that namespace, create a network policy that prevents all incoming and outgoing pod traffic.</li>

Create the following network policy:

<pre class="wp-block-code"><code>
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: self-study-netpol
  namespace: self-study
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
</code></pre>

You must define <code>Egress</code> as one of the policy types because, by default, a network policy will allow all outbound traffic.

<li>Finally, create a network policy in the appropriate namespace that allows pods from namespaces labeled <code>approved</code> to communicate with all pods in the <code>self-study</code> namespace.</li>

<pre class="wp-block-code"><code>
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: self-study-netpol
  namespace: self-study
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          approved: "true"
</code></pre>

RX-M can provide more help with preparing for the CKS exam in one of our CKS bootcamps; we offer open enrollments and private engagements for teams or organizations.

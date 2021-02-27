<!-- CKS Self-Study Mod 1 -->

- Create a new namespace named `self-study`. 

```
$ kubectl create namespace self-study

namespace/self-study created

$ 
```

- In that namespace, create a network policy that prevents all incoming and outgoing pod traffic.

Create the following network policy:

```
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
```

You must define `Egress` as one of the policy types because by default a network policy will allow all outbound traffic.

- Finally, create a network policy in the appropriate namespace that allows pods from namespaces labeled `approved` to communicate with all pods in the `self-study` namespace.

```
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
          approved: true
```

RX-M can provide more help with preparing for the CKS exam in one of our CKS bootcamps; we offer open enrollments and private engagements for teams or organizations.

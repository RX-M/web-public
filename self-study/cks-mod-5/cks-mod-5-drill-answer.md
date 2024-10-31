<!-- CKS Self-Study Mod 5 -->

Modify the following Pod Specification to conform with the [Kubernetes Pod Security Standard](https://kubernetes.io/docs/concepts/security/pod-security-standards/) Restrict category in these ways:

<li>Running all containers as Non-root<li>
<li>Seccomp profile to the runtime default<li>
<li>Ensure no Host Namespaces are used<li>
<li>Privilege Escalation should be disabled<li>

<pre class="wp-block-code"><code>
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: pod-security-practice
  name: pod-security-practice
spec:
  hostNetwork: false
  containers:
  - image: rxmllc/trash-levels:1.1
    name: pod-security-practice
</code></pre>

All settings needed must be explicitly declared in the pod spec.

Make the following changes:

<pre class="wp-block-code"><code>
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: pod-security-practice
  name: pod-security-practice
spec:
  securityContext:
    runAsNonRoot: true                   # Check that pod runs all containers as Non-root
    runAsUser: 1000                      # All containers must use
  hostNetwork: false                     # Already prevents sharing of host Network namespace
  hostPID: false                         # Prevent sharing of host PID Namespace
  hostIPC: false                         # Prevent sharing of host IPC Namespace
  containers:
  - image: rxmllc/trash-levels:1.1
    name: pod-security-practice
    securityContext:
      allowPrivilegeEscalation: False    # Privilege Escalation disabled
      seccompProfile:
        type: RuntimeDefault             # Seccomp profile set to the runtime default
</code></pre>



The pod must be deployable on your cluster and should be in the <code>Running</code> state for this drill to be considered complete.

<pre class="wp-block-code"><code>
~$ kubectl apply -f practicecks.yaml

pod/pod-security-practice created

~$ kubectl get pods

~$ kubectl describe pod pod-security-practice

Name:         pod-security-practice
Namespace:    default
Priority:     0
Node:         oct-b/192.168.229.157
Start Time:   Wed, 29 Sep 2021 12:20:49 -0700
Labels:       run=pod-security-practice
Annotations:  container.seccomp.security.alpha.kubernetes.io/pod-security-practice: runtime/default
Status:       Running
IP:           10.32.0.8
IPs:
  IP:  10.32.0.8
Containers:
  pod-security-practice:
    Container ID:   docker://dcad50f46595bfffd432457292567e2e3e22cdb879b5ee745e3418ca3883abb4
    Image:          rxmllc/trash-levels:1.1
    Image ID:       docker-pullable://433017611331.dkr.ecr.us-west-2.amazonaws.com/trash-levels@sha256:db1e123640af5ace68121ec4bd700fbf26f0c361e616cc79b5c29115bc6ce90d
    Port:           <none>
    Host Port:      <none>
    State:          Running
      Started:      Wed, 29 Sep 2021 12:20:50 -0700
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-h8jjq (ro)
Conditions:
  Type              Status
  Initialized       True
  Ready             True
  ContainersReady   True
  PodScheduled      True
Volumes:
  kube-api-access-h8jjq:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   BestEffort
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  14s   default-scheduler  Successfully assigned default/pod-security-practice to oct-b
  Normal  Pulled     13s   kubelet            Container image "rxmllc/trash-levels:1.1" already present on machine
  Normal  Created    13s   kubelet            Created container pod-security-practice
  Normal  Started    13s   kubelet            Started container pod-security-practice

~$
</code></pre>

RX-M can provide more help with preparing for the CKS exam in one of our CKS bootcamps; we offer open enrollments and private engagements for teams or organizations.

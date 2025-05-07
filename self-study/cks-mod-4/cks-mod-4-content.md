# CKS Self-Study Mod 4


# Use appropriate pod security standards

Kubernetes Pod Security Standards became stable in Kubernetes v1.25 and define different isolation levels for Pods. These standards let you define pod restrictions on a namespace by namespace basis. The Kubernetes built-in Pod Security admission controller enforces the Pod Security Standards when enabled.

Pod Security admission places requirements on a Pod's Security Context and other related fields according to the three levels defined by the Pod Security Standards:

<ul class="wp-block-list">
<li>privileged - unrestricted policy that provides the widest possible level of permissions</li>
<li>baseline - minimally restrictive policy that prevents known privilege escalations</li>
<li>restricted - heavily restricted policy following best practices for pod hardening</li>
</ul>

Kubernetes defines a set of labels that you can set to define which of the predefined Pod Security Standard levels you want to use for a namespace. The label you select defines what action the control plane takes if a potential violation is detected:


<ul class="wp-block-list">
<li>enforce - policy violations will cause the pod to be rejected</li>
<li>audit - policy violations will trigger the addition of an audit annotation to the event recorded in the audit log,<br>but are otherwise allowed</li>
<li>warn - policy violations will trigger a user-facing warning, but are otherwise allowed</li>
</ul>

A namespace can configure any or all modes, or even set a different level for different modes. Let's create a namespace and label it with "enforce" on the "baseline" policy:

<pre class="wp-block-code"><code>
$ kubectl create ns pss-lab

$ kubectl label ns pss-lab pod-security.kubernetes.io/enforce=baseline
</code></pre>

Test the policy by attempting to create a pod that violates the baseline standard:

<pre class="wp-block-code"><code>
$ kubectl apply -n pss-lab -f - &lt;&lt;EOF
apiVersion: v1
kind: Pod
metadata:
  name: bad-pod
spec:
  containers:
  - name: hostinfo
    image: rxmllc/hostinfo
    securityContext:
      capabilities:
        add: ["SYS_ADMIN"]
EOF

Error from server (Forbidden): error when creating "STDIN": pods "bad-pod" is forbidden: violates PodSecurity "baseline:latest": non-default capabilities (container "hostinfo" must not include "SYS_ADMIN" in securityContext.capabilities.add)
</code></pre>

The <code>SYS_ADMIN</code> capability violates the baseline security standard so the apiserver rejects it with an error message.

To get more details about about each profile, check out the <a href="https://kubernetes.io/docs/concepts/security/pod-security-standards/" target="_blank" rel="noreferrer noopener">pod security standards page</a> in the Kubernetes docs.


# Managing Kubernetes Secrets

Kubernetes secrets are resources that hold and distribute potentially confidential data inside the cluster. In almost every way, they are functionally identical to ConfigMaps, except Kubernetes secrets store their key's values in a base64 encoding.

<pre class="wp-block-code"><code>
apiVersion: v1
data:
  password: VEVTVCNwdw==
  username: dGVzdHVzZXJAZXhhbXBsZS5jb20=
kind: Secret
metadata:
  creationTimestamp: null
  name: test-db-secret
</code></pre>

You can then consume a secret as an environment variable or as files. A pod like this will mount the secret above as a series of environment variables:

<pre class="wp-block-code"><code>
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: secret-consumer
  name: secret-consumer
spec:
  containers:
  - command:
    - /bin/sh
    - -c
    - env
    env:
    - name: CRED
      valueFrom:
        secretMapKeyRef:
          name: test-db-secret
          key: username
    - name: CRED_KEY
      valueFrom:
        secretMapKeyRef:
          name: test-db-secret
          key: password
    image: busybox
    name: secret-consumer
</code></pre>

There are two other types of secrets that users can manage:

<ul>
<li><code>docker-registry</code> secrets allow users to provide container registry credentials to pods that need to pull images from private registries. These secrets are consumed by the <code>imagePullSecret</code> key in a pod spec.</li>
<li><code>tls</code> secrets store certificates and keys that can be consumed by pods or applications in the cluster. These are used by various resources, such as Ingresses and other third-party tools.></li>
</ul>

Base64 alone is not sufficient to help protect Kubernetes secrets. When a secret is consumed, its encoded values are presented in plaintext. Additionally, when stored in etcd, the secrets are decoded, and their contents are plainly visible to any client that has access to the etcd cluster.

<pre class="wp-block-code"><code>
~$ etcdctl get /registry/secrets/default/secret1 --cacert /etc/kubernetes/pki/etcd/ca.crt--cert /etc/kubernetes/pki/etcd/server.crt --key /etc/kubernetes/pki/etcd/server.key

/registry/secrets/default/secret1
k8s


v1Secret�
�
secret1default"*$02833b1c-fc3d-413a-a9b0-8c9ff4b9b4ef2��ňz�b
kubectl-createUpdatev��ňFieldsV1:.
,{"f:data":{".":{},"f:mykey":{}},"f:type":{}}B
mykeymydataOpaque"

</code></pre>

One way to protect secrets is to enable encryption, which involves providing an encryption service and passing it to the API Server:

<pre class="wp-block-code"><code>
$ nano enc-conf.yaml

apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
    - secrets
    providers:
    - aescbc:
        keys:
        - name: key1
          # Secret populated with the result of: head -c 32 /dev/urandom | base64
          secret: kKTjuD22oev0z+GLeF0/pHIL7T0rMLq+SCuU2R8JQ3E=
    - identity: {}
</code></pre>

In the example above we enabled a provider in the <code>EncryptionConfiguration</code> file. Best practices would be to use an integration with a KMS (like Hashicorp Vault) but because you are not allowed access to any KMS provider's documentation during the CKS exam, it can be considered out of scope.

Provide the config to the API server with the <code>--encryption-provider-config</code> (assuming you place the above yaml as <code>encconf.yaml</code> in a directory mounted by the API Server container at <code>/etc/kubernetes/hoststore</code>):

<pre class="wp-block-code"><code>
$ nano /etc/kubernetes/manifests/kube-apiserver.yaml

...

    - --encryption-provider-config=/etc/kubernetes/hoststore/encconf.yaml

...
</code></pre>

If you create a secret and try to retrieve it with an etcd client like <code>etcdctl</code>, the result will be encrypted:

<pre class="wp-block-code"><code>
$ etcdctl get /registry/secrets/default/secret --cacert /etc/kubernetes/pki/etcd/ca.crt--cert /etc/kubernetes/pki/etcd/server.crt --key /etc/kubernetes/pki/etcd/server.key

/registry/secrets/default/secret
...
</code></pre>

Enabling encryption helps protect the secret to an extent. The primary weaknesses of encryption include:

- No specific control over secret access
- Containers and applications consuming the secret still see the value
- Limited to no ability to see changes to the secret

The best secret protection comes from using external or third-party mechanisms. Key management systems (KMS)
provided by Cloud vendors and purpose-built secret management software go the extra mile by providing:

- Secret encryption at rest
- Off-disk secret storage
- Secret value rotation
- Access control

To learn more about K8s secrets, see [here](https://kubernetes.io/docs/concepts/configuration/secret/). For additional
information about how you can enable secret encryption at rest, go
[here](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/).


# Understand and implement isolation techniques (multi-tenancy, sandboxed containers, etc.)

Container technology allows applications to run within their own isolated spaces alongside a host operating system. This is possible because the containers themselves share the underlying kernel with the operating system.

This approach opens the Kernel as a potential attack vector, as anybody who can access the machine's kernel can potentially intercept calls made from the container runtime to the host kernel. This is where sandboxing container runtimes, like gVisor or Kata Containers, come in. These runtimes insert an abstraction layer between a container's perceived kernel communication channel and the host kernel itself.

To use different sandboxes, Kubernetes enables users to select specific container runtimes to run their applications with the RuntimeClass resource. RuntimeClass resources are similar to StorageClasses in that they define an integration with an external system. In the case of RuntimeClasses, the external integration is with a specific container runtime.

Assuming <code>gvisor</code> is <a href="https://gvisor.dev/docs/user_guide/install/#install-from-an-apt-repository" target="_blank" rel="noreferrer noopener">installed</a>, and the existing <a href="https://gvisor.dev/docs/user_guide/containerd/quick_start/" target="_blank" rel="noreferrer noopener">container runtime is configured</a> to use it with the <code>runsc</code> handler, the following <code>RuntimeClass</code> allows the pod to use it:

<pre class="wp-block-code"><code>
apiVersion: node.k8s.io/v1beta1
kind: RuntimeClass
metadata:
  name: gvisor
handler: runsc
</code></pre>

Runtime classes can then be called inside a pod spec using the <code>runtimeClassName</code> keyword:

<pre class="wp-block-code"><code>
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: gvisor-pod
  name: gvisor-pod
spec:
  runtimeClassName: gvisor
  containers:
  - image: nginx
    name: gvisor-pod
</code></pre>

To learn more about Runtime Classes and how to use them, see the
[Kubernetes Documentation Page on RuntimeClass](https://kubernetes.io/docs/concepts/containers/runtime-class/).


## Implement Pod-to-Pod encryption (Cilium, Istio)

If your applications do not implement TLS, then another way of doing so would be to use some kind of proxying sidecar container running alongside your application. The sidecar container can intercept communication from other pods in the cluster, initiating and terminating TLS, then passing the traffic to the application container. This approach is implemented by service meshes like Istio, which use Envoy as their sidecar containers which can be injected into pods.

Istio can be installed simply with the <code>istioctl</code> tool:

<pre class="wp-block-code"><code>
$ curl -L https://istio.io/downloadIstio | sh -

$ sudo cp istio-*/bin/istioctl /usr/local/bin/

$ istioctl install --set profile=demo -y
</code></pre>

Once Istio is installed, create and label a namespace called <code>lab</code> to enable auto-injection of Envoy sidecars:

<pre class="wp-block-code"><code>
$ kubectl create ns lab

$ kubectl label ns lab istio-injection=enabled
</code></pre>

Deploy the Httpbin sample service from the Istio project:

<pre class="wp-block-code"><code>
$ kubectl -n lab apply -f https://raw.githubusercontent.com/istio/istio/release-1.25/samples/httpbin/httpbin.yaml
</code></pre>

Because the httpbin service is not exposed outside the cluster you cannot <code>curl</code> it directly, however you can verify that it is working correctly using a <code>curl</code> command against <code>httpbin:8000</code> from inside the cluster.

<pre class="wp-block-code"><code>
$ kubectl -n lab run client -it --rm --image=rxmllc/tools

/ # curl httpbin:8000 | head

  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 12071    0 12071    0   &lt;!DOCTYPE html> 0 --:--:-- --:--:-- --:--:--     0
  0  286&lt;html>
&lt;head>
  &lt;meta http-equiv='content-type' value='text/html;charset=utf8'>
  &lt;meta name='generator' value='Ronn/v0.7.3 (http://github.com/rtomayko/ronn/tree/0.7.3)'>
  &lt;title>go-httpbin(1): HTTP Client Testing Service&lt;/title>
  &lt;style type='text/css' media='all'>
  /* style: man */
  body#manpage {margin:0;background:#fff;}
  .mp {max-width:100ex;padding:0 9ex 1ex 4ex}
6k      0 --:--:-- --:--:-- --:--:-- 3929k

/ # exit

Session ended, resume using 'kubectl attach client -c client -i -t' command when the pod is running
pod "client" deleted
</code></pre>

By default, Istio configures mTLS using <code>PERMISSIVE</code> mode, allowing services to accept both plaintext and mTLS traffic. We can enforce mTLS with an Istio <code>PeerAuthentication</code> config set to <code>STRICT</code> mode:

<pre class="wp-block-code"><code>
$ kubectl apply -n lab -f - &lt;&lt;EOF
apiVersion: security.istio.io/v1
kind: PeerAuthentication
metadata:
  name: default
spec:
  mtls:
    mode: STRICT
EOF
</code></pre>

Run another <code>client</code> pod:

<pre class="wp-block-code"><code>
$ kubectl -n lab run client -it --rm --image=rxmllc/tools

/ # curl httpbin:8000 | head

  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 12071    0 12071    0   &lt;!DOCTYPE html> 0 --:--:-- --:--:-- --:--:--     0
  0  286&lt;html>
&lt;head>
  &lt;meta http-equiv='content-type' value='text/html;charset=utf8'>
  &lt;meta name='generator' value='Ronn/v0.7.3 (http://github.com/rtomayko/ronn/tree/0.7.3)'>
  &lt;title>go-httpbin(1): HTTP Client Testing Service&lt;/title>
  &lt;style type='text/css' media='all'>
  /* style: man */
  body#manpage {margin:0;background:#fff;}
  .mp {max-width:100ex;padding:0 9ex 1ex 4ex}
6k      0 --:--:-- --:--:-- --:--:-- 3929k

/ # exit

Session ended, resume using 'kubectl attach client -c client -i -t' command when the pod is running
pod "client" deleted
</code></pre>

This works because both pods receive Envoy sidecars in the <code>lab</code> namespace which facilitate the mTLS connection.

Run another <code>client</code> pod in the <code>default</code> namespace and query the <code>httpbin.lab</code> service (remember, in order to resolve a service in a different namespace you have to use the pattern <code>service-name.namespace</code>), does it work?

<pre class="wp-block-code"><code>
$ kubectl -n default run client -it --rm --image=rxmllc/tools

/ # curl httpbin.lab:8000 | head

  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0
curl: (56) Recv failure: Connection reset by peer

/ # exit

Session ended, resume using 'kubectl attach client -c client -i -t' command when the pod is running
pod "client" deleted
</code></pre>

It cannot connect because it doesn't have a sidecar to manage the mTLS connection for it so it sends the request in plaintext which you restricted. You can confirm this by deleting the <code>PeerAuthentication</code> and trying <code>curl</code> again from the <code>default</code> namespace:

<pre class="wp-block-code"><code>
$ kubectl -n lab delete pa default

peerauthentication.security.istio.io "default" deleted

$ kubectl -n default run client -it --rm --image=rxmllc/tools

/ # curl httpbin.lab:8000 | head

  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0&lt;!DOCTYPE html>
&lt;html>
&lt;head>
  &lt;meta http-equiv='content-type' value='text/html;charset=utf8'>
  &lt;meta name='generator' value='Ronn/v0.7.3 (http://github.com/rtomayko/ronn/tree/0.7.3)'>
  &lt;title>go-httpbin(1): HTTP Client Testing Service&lt;/title>
  &lt;style type='text/css' media='all'>
  /* style: man */
  body#manpage {margin:0;background:#fff;}
100 12  .mp {max-width:100ex;padding:0 9ex 1ex 4ex}
071    0 12071    0     0  1727k      0 --:--:-- --:--:-- --:--:-- 1964k

/ # exit

Session ended, resume using 'kubectl attach client -c client -i -t' command when the pod is running
pod "client" deleted
</code></pre>

With <code>PERMISSIVE</code> mode back in place the <code>httpbin</code> service accepts the plaintext traffic.

Learn more about how service meshes like Istio can be used to enable mTLS between pods <a href="https://istio.io/latest/docs/tasks/security/authentication/mtls-migration/" target="_blank" rel="noreferrer noopener">here</a>.


## Practice Drill

<ul>
<li>Create a generic secret named <code>postgres</code> with the key-value pairs <code>pgu=blogdb</code> and <code>pgk=helloEmpanada</code></li>
<li>Configure a pod named <code>postgres-db</code> that runs <code>postgres:latest</code> that uses the secret to furnish the
  <code>POSTGRES_PASSWORD</code> environment variable with the <code>pgk</code> key's value</li>
</ul>


# CONTENT FROM BEFORE AUGUST 2024 REMAINS BELOW IN CASE THE EXAM INCLUDES IT IN THE FUTURE


# Setup appropriate OS-level security domains

Kubernetes can instruct its container runtime to implement various OS-level security features, such as AppArmor or SELinux. These are done using the <code>securityContext</code> related features under a pod's container spec.

<pre class="wp-block-code"><code>
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: privpod
  name: privpod
spec:
  containers:
  - command:
    - tail
    - -f
    - /dev/null
    image: public.ecr.aws/runecast/busybox:1.32.1
    name: privpod
    resources: {}
    securityContext:
      capabilities:
        drop:
        - all
        add:
        - CHOWN
  dnsPolicy: ClusterFirst
  restartPolicy: Always
</code></pre>

The policy-based security model has been expanded upon with the Open Policy Agent (OPA) project, which attempts to expand the scope of policy-based security beyond Kubernetes. The project hosts a cluster extension known as the OPA Gatekeeper, which takes open policy agent policies, like the following from the OPA website:

<pre class="wp-block-code"><code>
package kubernetes.admission

import data.kubernetes.namespaces

operations = {"CREATE", "UPDATE"}

deny[msg] {
	input.request.kind.kind == "Ingress"
	operations[input.request.operation]
	host := input.request.object.spec.rules[_].host
	not fqdn_matches_any(host, valid_ingress_hosts)
	msg := sprintf("invalid ingress host %q", [host])
}

valid_ingress_hosts = {host |
	whitelist := namespaces[input.request.namespace].metadata.annotations["ingress-whitelist"]
	hosts := split(whitelist, ",")
	host := hosts[_]
}

fqdn_matches_any(str, patterns) {
	fqdn_matches(str, patterns[_])
}

fqdn_matches(str, pattern) {
	pattern_parts := split(pattern, ".")
	pattern_parts[0] == "*"
	str_parts := split(str, ".")
	n_pattern_parts := count(pattern_parts)
	n_str_parts := count(str_parts)
	suffix := trim(pattern, "*.")
	endswith(str, suffix)
}

fqdn_matches(str, pattern) {
    not contains(pattern, "*")
    str == pattern
}
</code></pre>

It then executes them by acting as an admissions webhook, consuming the policy is stored inside a configmap:

<pre class="wp-block-code"><code>
Error from server (invalid ingress host "acmecorp.com"): error when creating "ingress-bad.yaml": admission webhook "validating-webhook.openpolicyagent.org" denied the request: invalid ingress host "acmecorp.com"
</code></pre>

For more information about how to implement the Open Policy Agent gateway on your Kubernetes cluster, see [here](https://www.openpolicyagent.org/docs/v0.12.2/kubernetes-admission-control/).
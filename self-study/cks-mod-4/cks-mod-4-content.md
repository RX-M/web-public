# CKS Self-Study Mod 4


## Setup appropriate OS-level security domains

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


## Managing Kubernetes Secrets

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

One way to protect secrets is to enable etcd encryption, which involves providing an encryption service and passing it to the API Server:

<pre class="wp-block-code"><code>
~$ cat enc-conf.yaml

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

And providing that config to the API server with the <code>--encryption-provider-config</code> (assuming you place the above yaml as <code>encconf</code> in a directory mounted by the API Server container at <code>/etc/kubernetes/hoststore</code>):

<pre class="wp-block-code"><code>
~$ cat /etc/kubernetes/manifests/kube-apiserver.yaml

...

    - --encryption-provider-config=/etc/kubernetes/hoststore/encconf.yaml

...
</code></pre>

If you create a secret and try to retrieve it with an etcd client like <code>etcdctl</code>, the result will be encrypted:

<pre class="wp-block-code"><code>
~$ etcdctl get /registry/secrets/default/secret --cacert /etc/kubernetes/pki/etcd/ca.crt--cert /etc/kubernetes/pki/etcd/server.crt --key /etc/kubernetes/pki/etcd/server.key

/registry/secrets/default/secret
...
</code></pre>

Enabling etcd encryption helps protect the secret to an extent. The primary weaknesses of etcd encryption include:

- No specific control over secret access
- Containers and applications consuming the secret still see the value
- Limited to no ability to see changes to the secret

The best secret protection comes from using external or third-party mechanisms. Key management systems (KMS)
provided by Cloud vendors and purpose-built secret management software go the extra mile by providing:

- Secret encryption at rest
- Off-disk secret storage
- Secret value rotation
- Access control

To learn more about the secrets, see [here](https://kubernetes.io/docs/concepts/configuration/secret/). For additional
information about how you can enable secret encryption at rest, go
[here](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/).


## Use Container Runtime Sandboxes in Multi-tenant environments

Container technology allows applications to run within their own isolated spaces alongside a host operating system. This is possible because the containers themselves share the underlying kernel with the operating system.

This approach opens the Kernel as a potential attack vector, as anybody who can access the machine's kernel can potentially intercept calls made from the container runtime to the host kernel. This is where sandboxing container runtimes, like gVisor or Kata Containers, come in. These runtimes insert an abstraction layer between a container's perceived kernel communication channel and the host kernel itself.

To use different sandboxes, Kubernetes enables users to select specific container runtimes to run their applications with the RuntimeClass resource. RuntimeClass resources are similar to StorageClasses in that they define an integration with an external system. In the case of RuntimeClasses, the external integration is with a specific container runtime.

Assuming <code>gvisor</code> is installed, and the existing container runtime is configured to use it with the <code>runsc</code> handler, the following <code>RuntimeClass</code> allows the pod to use it:

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


## Implement pod-to-pod encryption by use of mTLS

The simplest way to implement mTLS between your pods is to run applications that can handle TLS within the pod's containers. Both client and server certificates can be stored within <code>tls</code> type secrets, and all communications from one pod to another should occur on TLS-enabled communication channels (ports, etc.)

However, if your applications do not implement TLS, then another way of doing so would be to use some kind of proxying (or ambassador) container running alongside your application. The ambassador container can intercept communication from other pods in the cluster, initiating and terminating TLS, then passing the traffic to the application container. This approach is implemented by service meshes like Istio, which use Envoy as their ambassador containers which can be injected into existing pods.

To learn more about how service meshes like Istio and Linkerd can be used to enable mTLS between pods, check out [this blog post on using Linkerd on the Kubernetes blog](https://kubernetes.io/blog/2018/09/18/hands-on-with-linkerd-2.0/) and [this blog post on how to set up a demo application on an Istio-enabled cluster](https://kubernetes.io/blog/2017/05/managing-microservices-with-istio-service-mesh/).

One example of pod-to-pod encryption outside of the use of a service mesh like Istio exists within a standard kubeadm-backed Kubernetes cluster, between the API Server and etcd.

On a control plane node, checking the process of the kube-apiserver with <code>ps</code> reveals the certificates it uses to communicate with etcd:

<pre class="wp-block-code"><code>
~$ ps -ef | grep kube-apiserver

root        1127     880  3 16:52 ?        00:02:17 kube-apiserver --advertise-address=172.31.18.190 --allow-privileged=true --authorization-mode=Node,RBAC --client-ca-file=/etc/kubernetes/pki/ca.crt --enable-admission-plugins=NodeRestriction --enable-bootstrap-token-auth=true --etcd-cafile=/etc/kubernetes/pki/etcd/ca.crt --etcd-certfile=/etc/kubernetes/pki/apiserver-etcd-client.crt --etcd-keyfile=/etc/kubernetes/pki/apiserver-etcd-client.key --etcd-servers=https://127.0.0.1:2379 --kubelet-client-certificate=/etc/kubernetes/pki/apiserver-kubelet-client.crt --kubelet-client-key=/etc/kubernetes/pki/apiserver-kubelet-client.key --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname --proxy-client-cert-file=/etc/kubernetes/pki/front-proxy-client.crt --proxy-client-key-file=/etc/kubernetes/pki/front-proxy-client.key --requestheader-allowed-names=front-proxy-client --requestheader-client-ca-file=/etc/kubernetes/pki/front-proxy-ca.crt --requestheader-extra-headers-prefix=X-Remote-Extra- --requestheader-group-headers=X-Remote-Group --requestheader-username-headers=X-Remote-User --secure-port=6443 --service-account-issuer=https://kubernetes.default.svc.cluster.local --service-account-key-file=/etc/kubernetes/pki/sa.pub --service-account-signing-key-file=/etc/kubernetes/pki/sa.key --service-cluster-ip-range=10.96.0.0/12 --tls-cert-file=/etc/kubernetes/pki/apiserver.crt --tls-private-key-file=/etc/kubernetes/pki/apiserver.key
ubuntu      6439    6415  0 17:57 pts/0    00:00:00 grep --color=auto kube-apiserver
</code></pre>

In addition to the certificate options, the API Server has additional settings to help further secure the TLS communications:

<ul>
<li><code>--tls-cipher-suites</code> determines which TLS Cipher suites are supported by your API Server</li>
<li><code>--tls-min-version</code> determines the minimum version of TLS that is enforced by the API Server</li>
</ul>

Additional details for these options, including their values, are available on the Kubernetes documentation: <a href="https://kubernetes.io/docs/reference/command-line-tools-reference/kube-apiserver/">https://kubernetes.io/docs/reference/command-line-tools-reference/kube-apiserver/</a>.

etcd has a similar suite of settings. While the etcd documents are not listed as an allowed resource, you can always look to the help output for etcd by running <code>etcd --help</code> in your cluster's etcd pod.

<pre  class="wp-block-code"><code>
~$ kubectl exec -n kube-system etcd-$(hostname) -- etcd --help

...

Security:
  --cert-file ''
    Path to the client server TLS cert file.
  --key-file ''
    Path to the client server TLS key file.
  --client-cert-auth 'false'
    Enable client cert authentication.
  --client-crl-file ''
    Path to the client certificate revocation list file.
  --client-cert-allowed-hostname ''
    Allowed TLS hostname for client cert authentication.
  --trusted-ca-file ''
    Path to the client server TLS trusted CA cert file.
  --auto-tls 'false'
    Client TLS using generated certificates.
  --peer-cert-file ''
    Path to the peer server TLS cert file.
  --peer-key-file ''
    Path to the peer server TLS key file.
  --peer-client-cert-auth 'false'
    Enable peer client cert authentication.
  --peer-trusted-ca-file ''
    Path to the peer server TLS trusted CA file.
  --peer-cert-allowed-cn ''
    Required CN for client certs connecting to the peer endpoint.
  --peer-cert-allowed-hostname ''
    Allowed TLS hostname for inter peer authentication.
  --peer-auto-tls 'false'
    Peer TLS using self-generated certificates if --peer-key-file and --peer-cert-file are not provided.
  --self-signed-cert-validity '1'
    The validity period of the client and peer certificates that are automatically generated by etcd when you specify ClientAutoTLS and PeerAutoTLS, the unit is year, and the default is 1.
  --peer-crl-file ''
    Path to the peer certificate revocation list file.
  --cipher-suites ''
    Comma-separated list of supported TLS cipher suites between client/server and peers (empty will be auto-populated by Go).
  --cors '*'
    Comma-separated whitelist of origins for CORS, or cross-origin resource sharing, (empty or * means allow all).
  --host-whitelist '*'
    Acceptable hostnames from HTTP client requests, if server is not secure (empty or * means allow all).
</code></pre>

Of note, here are the last five options, all of which provide similar options (including TLS Cipher Suites) for helping secure TLS communication between etcd and its clients (the Kubernetes API server, in this case.)


## Practice Drill

<ul>
<li>Create a generic secret named <code>postgres</code> with the key-value pairs <code>pgu=blogdb</code> and <code>pgk=helloEmpanada</code></li>
<li>Configure a pod named <code>postgres-db</code> that runs <code>postgres:latest</code> that uses the secret to furnish the
  <code>POSTGRES_PASSWORD</code> environment variable with the <code>pgk</code> key's value</li>
</ul>

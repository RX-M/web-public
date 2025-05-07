# CKS Self-Study Mod 5


# Minimize Base Image Footprint

Container images encompass all of the code and dependencies needed to run an application, enabling reliable and repeatable deployment of an application across many nodes. All the tools, functionality, and other features provided and supported by the application get deployed when a new container is created from a container image.

There is such a thing as having too much in a container, which is often the case when creating images using an OS-based image. All the tools included within an OS-based image can be useful to regular users and bad actors alike.

A common solution to addressing "heavy" base images in containers is the multi-stage build. A multi-stage build performs an application build in an intermediate container, then copies the compiled code or binary to a lighter-weight base image that has none of the extraneous libraries, tools, and other features of the original build container.

Given the following Dockerfile:

<pre class="wp-block-code"><code>
FROM docker.io/golang:1.24
WORKDIR /go/src/hello
COPY ./hello.go /go/src/hello
CMD ["go","run","hello.go"]
</code></pre>

The image produced will lead to an image of this size:

<pre class="wp-block-code"><code>
$ docker image ls hello-build

REPOSITORY    TAG       IMAGE ID       CREATED         SIZE
hello-build   v1        9cc4ff41478e   2 minutes ago   839MB
</code></pre>

Much of that size comes from the base <code>go</code> image, which has all the tools necessary to build and compile a go application. Each time this container image needs to be pulled down to a node, it will also incur that cost with Network I/O.

Implementing a multi-stage build involves two things:

<ul>
<li>Naming the first container build portion with an easily referenced Alias</li>
<li>Adding a second <code>FROM</code> instruction that uses a lighter weight base image</li>
<li>Copying the compiled code or binary from the first container to the second container</li>
<li>Enabling the second container to run the code</li>
</ul>

<pre class="wp-block-code"><code>
FROM docker.io/golang:1.24 AS build-env
WORKDIR /go/src/hello/
COPY ./hello.go /go/src/hello/
RUN ["go","mod","init","example.com/hello"]
RUN ["go","build","-tags","netgo"]

FROM scratch
COPY --from=build-env /go/src/hello/hello hello
EXPOSE 8080
ENTRYPOINT ["./hello"]
</code></pre>

The <code>scratch</code> base image is practically an empty container filesystem that is sufficient to run a binary. No other tools are present - even if a bad actor compromised a container created from this image, there would be no tools to continue the attack.

After building the image, you will notice a container size difference between the two:

<pre class="wp-block-code"><code>
$ docker image ls hello-build

REPOSITORY    TAG       IMAGE ID       CREATED          SIZE
hello-build   v2        ec99759c61a6   47 seconds ago   6.39MB
hello-build   v1        6847f39c0f81   3 minutes ago    839MB

</code></pre>

The delta between the two container sizes depends on the base image used in the second stage of the build. Since <code>scratch</code> is practically empty compared to the original <code>golang</code> base image, the resulting application container went from almost 1GiB to less than 7MiB. There are cases where the second stage base image can be larger than the original build image.

Whenever this image needs to be deployed, your network and resulting device do not need to move as much data.

One important thing to remember about images with small footprints is that many tools that you may need to use, like <code>ls</code> or <code>cat</code>, will not be available for you in case you need to debug the container.

While you cannot use the Docker documentation on the exam as a reference, if you would like more information on using multi-staged Docker builds, see [here](https://docs.docker.com/develop/develop-images/multistage-build/).


# Understand your supply chain (e.g. SBOM, CI/CD, artifact repositories)


A lot of modern software relies on the use of existing libraries, modules, and even software to provide their functionality. Because of the compound nature of these kinds of programs, it is important to understand what exactly composes these applications. Understanding what goes into an application will make it easier to pinpoint which components may be out of date, compromised in any way, or simply to understand what exactly has gone into that software.

Thus, the concept of the "bill of materials" - a list of atomic pieces, or components, that makes up an assembled good - has been brought to the software world. The "software bill of materials" (SBOM) seeks to provide a structured list of all of the components that are included within a piece of software. Specifically, SBOMs try to detail:

<ul class="wp-block-list">
<li>Who created or supplied a component of software that is integrated with an assembled product</li>
<li>Who created or supplied the <em>components of those components</em></li>
<li>What version of each component is there</li>
</ul>

The <code>bom</code> CLI was created as an SBOM tool for the Kubernetes project that lets you create, view and transform Software Bills of Materials (SBOMs).

Install <code>bom</code> from its GitHub releases page:

<pre class="wp-block-code"><code>
$ curl -L  https://github.com/kubernetes-sigs/bom/releases/download/v0.6.0/bom-amd64-linux  -o bom

$ sudo mv ./bom /usr/local/bin/bom

$ sudo chmod +x /usr/local/bin/bom
</code></pre>

Once installed, <code>bom</code> can be used to process container images, this example pulls the <code>kube-apiserver</code> image, analyzes it, and describes it in the SBOM (be prepared for a lot of output):

<pre class="wp-block-code"><code>
$ bom generate -n http://example.com/ --image registry.k8s.io/kube-apiserver:v1.32.0

INFO bom v0.6.0: Generating SPDX Bill of Materials 
INFO Processing image reference: registry.k8s.io/kube-apiserver:v1.32.0 
INFO Reference registry.k8s.io/kube-apiserver:v1.32.0 points to an index 
INFO Reference image index points to 4 manifests  
INFO Adding image registry.k8s.io/kube-apiserver@sha256:fe1eb8fc870b01f4b1f470d2b179a1d1a86d6e2fa174bd10c01bf45bc5b03200 (amd64/linux) 
INFO Adding image registry.k8s.io/kube-apiserver@sha256:9ff42b586c0a57f3fc4a0689afe6db4d8f92f7f79bef3b47b2c75ab112e17de7 (arm64/linux) 
INFO Adding image registry.k8s.io/kube-apiserver@sha256:ea21094176bd09a82ec094146c7f7e1f3d4167a21709b3ed40a120607dfea9c2 (ppc64le/linux) 
INFO Adding image registry.k8s.io/kube-apiserver@sha256:bea18802acb8c43280a65eb9e88f252d7c309cacf1032712d476675a22796b57 (s390x/linux) 
INFO Generating SBOM for multiarch image registry.k8s.io/kube-apiserver@sha256:ebc0ce2d7e647dd97980ec338ad81496c111741ab4ad05e7c5d37539aaf7dc3b 
INFO Package describes image registry.k8s.io/kube-apiserver:9ff42b586c0a57f3fc4a0689afe6db4d8f92f7f79bef3b47b2c75ab112e17de7 
INFO Image manifest lists 13 layers               
INFO Scan of container layers found distroless base image 
INFO Scan of container image returned 3 OS packages in layer #12 
INFO Package describes image registry.k8s.io/kube-apiserver:fe1eb8fc870b01f4b1f470d2b179a1d1a86d6e2fa174bd10c01bf45bc5b03200 
INFO Image manifest lists 13 layers               
INFO Scan of container layers found distroless base image 
INFO Scan of container image returned 3 OS packages in layer #12 
INFO Package describes image registry.k8s.io/kube-apiserver:ea21094176bd09a82ec094146c7f7e1f3d4167a21709b3ed40a120607dfea9c2 
INFO Image manifest lists 13 layers               
INFO Scan of container layers found distroless base image 
INFO Scan of container image returned 3 OS packages in layer #12 
INFO Package describes image registry.k8s.io/kube-apiserver:bea18802acb8c43280a65eb9e88f252d7c309cacf1032712d476675a22796b57 
INFO Image manifest lists 13 layers               
INFO Scan of container layers found distroless base image 
INFO Scan of container image returned 3 OS packages in layer #12 
WARN Document has no name defined, automatically set to SBOM-SPDX-099bd45e-21ec-429b-830d-fddf898c42b9 
INFO Package SPDXRef-Package-sha256-ebc0ce2d7e647dd97980ec338ad81496c111741ab4ad05e7c5d37539aaf7dc3b has 4 relationships defined 
INFO Package SPDXRef-Package-registry.k8s.io-kube-apiserverC64sha256-ebc0ce2d7e647dd97980ec338ad81496c111741ab4ad05e7c5d37539aaf7dc3b-registry.k8s.io-kube-apiserverC64sha256-9ff42b586c0a57f3fc4a0689afe6db4d8f92f7f79bef3b47b2c75ab112e17de7 has 14 relationships defined 
INFO Package SPDXRef-Package-registry.k8s.io-kube-apiserver-9ff42b586c0a57f3fc4a0689afe6db4d8f92f7f79bef3b47b2c75ab112e17de7-sha256-eb08ddeccf9e5e88236da144a832b19a4225144c55a7ab5bad1a6bccbec355a1 has 3 relationships defined 
INFO Package SPDXRef-Package-registry.k8s.io-kube-apiserverC64sha256-ebc0ce2d7e647dd97980ec338ad81496c111741ab4ad05e7c5d37539aaf7dc3b-registry.k8s.io-kube-apiserverC64sha256-fe1eb8fc870b01f4b1f470d2b179a1d1a86d6e2fa174bd10c01bf45bc5b03200 has 14 relationships defined 
INFO Package SPDXRef-Package-registry.k8s.io-kube-apiserver-fe1eb8fc870b01f4b1f470d2b179a1d1a86d6e2fa174bd10c01bf45bc5b03200-sha256-890390ebdc76f0adb568dc7a78122514a4138ec665103e5c641b5c4aadf01507 has 3 relationships defined 
INFO Package SPDXRef-Package-registry.k8s.io-kube-apiserverC64sha256-ebc0ce2d7e647dd97980ec338ad81496c111741ab4ad05e7c5d37539aaf7dc3b-registry.k8s.io-kube-apiserverC64sha256-ea21094176bd09a82ec094146c7f7e1f3d4167a21709b3ed40a120607dfea9c2 has 14 relationships defined 
INFO Package SPDXRef-Package-registry.k8s.io-kube-apiserver-ea21094176bd09a82ec094146c7f7e1f3d4167a21709b3ed40a120607dfea9c2-sha256-2748249821e550babcbb250abc95c594c84f2efac60da86aca77edecec488a59 has 3 relationships defined 
INFO Package SPDXRef-Package-registry.k8s.io-kube-apiserverC64sha256-ebc0ce2d7e647dd97980ec338ad81496c111741ab4ad05e7c5d37539aaf7dc3b-registry.k8s.io-kube-apiserverC64sha256-bea18802acb8c43280a65eb9e88f252d7c309cacf1032712d476675a22796b57 has 14 relationships defined 
INFO Package SPDXRef-Package-registry.k8s.io-kube-apiserver-bea18802acb8c43280a65eb9e88f252d7c309cacf1032712d476675a22796b57-sha256-0df17086225e85bce46bf0d61c9084e00c56ff3499ba0949b183b51cc35689a5 has 3 relationships defined 
SPDXVersion: SPDX-2.3
DataLicense: CC0-1.0
SPDXID: SPDXRef-DOCUMENT
DocumentName: SBOM-SPDX-099bd45e-21ec-429b-830d-fddf898c42b9
DocumentNamespace: http://example.com/
Creator: Organization: Kubernetes Release Engineering
Creator: Tool: bom-v0.6.0
LicenseListVersion: 3.21
Created: 2025-05-07T00:04:31Z

...
</code></pre>

The <code>bom</code> <a href="https://kubernetes-sigs.github.io/bom/" target="_blank" rel="noreferrer noopener">documentation</a> is one of the allowed resources during the CKS exam.


# Perform static analysis of user workloads and container images (e.g. Kubesec, KubeLinter)

Static Analysis is the practice of checking code against some kind of standard and debugging it before running. There are many static analysis tools for Kubernetes workloads, such as Chekov, KubeScore, and KubeLinter. Static analysis tools adhere to defined standards that help users identify and correct common misconfigurations that lead to poor functionality and potential security issues.

<pre class="wp-block-code"><code>
$ kubectl run testpod -o yaml --dry-run=client --image nginx

apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: testpod
  name: testpod
spec:
  containers:
  - image: nginx
    name: testpod
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
</code></pre>

Given the very basic, generated <code>kubectl run</code> output, a tool like <code>kube-score</code> can provide insight into what configurations could be changed.

<pre class="wp-block-code"><code>
$ kubectl run testpod -o yaml --dry-run=client --image nginx | ./kube-score score -

v1/Pod testpod                                                                💥
    [CRITICAL] Container Security Context
        · testpod -> Container has no configured security context
            Set securityContext to run the container in a more secure context.
    [CRITICAL] Container Resources
        · testpod -> CPU limit is not set
            Resource limits are recommended to avoid resource DDOS. Set
            resources.limits.cpu
        · testpod -> Memory limit is not set
            Resource limits are recommended to avoid resource DDOS. Set
            resources.limits.memory
        · testpod -> CPU request is not set
            Resource requests are recommended to make sure that the application
            can start and run without crashing. Set resources.requests.cpu
        · testpod -> Memory request is not set
            Resource requests are recommended to make sure that the application
            can start and run without crashing. Set resources.requests.memory
    [CRITICAL] Container Image Tag
        · testpod -> Image with latest tag
            Using a fixed tag is recommended to avoid accidental upgrades
    [CRITICAL] Pod NetworkPolicy
        · The pod does not have a matching NetworkPolicy
            Create a NetworkPolicy that targets this pod to control who/what
            can communicate with this pod. Note, this feature needs to be
            supported by the CNI implementation used in the Kubernetes cluster
            to have an effect.
</code></pre>

On the exam, you may be called to implement some of the best practices without the help of any static analysis tools. These best practices may be presented on different levels of the workload, including the Dockerfile for a container or a Kubernetes YAML file.

Some of these best practices you may need to memorize for Dockerfiles include:

<ul>
<li>Avoid the use of "latest" tags for base images in Dockerfiles</li>
<li>Ensure a non-root user is running the application in the container</li>
<li>Try to use ports above the common "privileged" set of ports (those below 1000)</li>
<li>Prefer the use of multi-stage Dockerfiles (Dockerfiles with multiple stanzas using different base images)</li>
<li>Other practices on The Docker CIS Benchmark: <a href="https://www.aquasec.com/cloud-native-academy/docker-container/docker-cis-benchmark/">https://www.aquasec.com/cloud-native-academy/docker-container/docker-cis-benchmark/</a></li>
</ul>

<ul>
</ul>

For Kubernetes, some of the security standards you should be familiar with are available on the Kubernetes Documentation (in the case of the first two links) and also referred to during the test (in the last two links):

<ul>
<li>General Kubernetes best practices: <a href="https://kubernetes.io/docs/concepts/configuration/overview/">https://kubernetes.io/docs/concepts/configuration/overview/</a></li>
<li>The Kubernetes Pod Security Standard: <a href="https://kubernetes.io/docs/concepts/security/pod-security-standards/">https://kubernetes.io/docs/concepts/security/pod-security-standards/</a></li>
<li>The CIS Kubernetes Security Benchmark: <a href="https://www.cisecurity.org/benchmark/kubernetes/">https://www.cisecurity.org/benchmark/kubernetes/</a></li>
</ul>

Keep in mind that the best practices expected are not always security related; they could be more simple functional practices that need to be fixed. To check for changes, try building an image from the Dockerfile or applying the Kubernetes manifest before proceeding.


# Secure your supply chain (permitted registries, sign and validate artifacts, etc.)


## Permitted Registries

One of the primary threats to containerized workloads is a compromised supply chain. Container images will pull in dependencies from all manner of sources, often trusting those sources to provide the most up-to-date code possible. If a bad actor compromises one of those trusted sources with malicious code, then all container images built using that compromised supply chain will have that code.

A critical part of the Kubernetes software supply chain is the image registry. A compromised image registry guarantees that malicious software will be deployed into Kubernetes. Thus, a Kubernetes cluster should be able to validate and trust the image registry before it pulls and runs a container image.

The primary way of configuring your Kubernetes cluster to trust certain image registries is through an admissions webhook. This webhook should be supported by some kind of controller, whether it is custom code like the [Kube Image Bouncer](https://github.com/kainlite/kube-image-bouncer) or the Open Policy Agent and its subprojects, like Gatekeeper. The webhook allows the API server to contact the custom controller, which can connect to an external service that can validate an image against a policy or list of allowed container image registries.

For example, if you are using OPA, the policy would resemble something like the following:

<pre class="wp-block-code"><code>
package kubernetes.admission
deny[msg] {
    input.request.kind.kind == "Pod"
    image := input.request.object.spec.containers[_].image
    not startswith(image, "organization.io")
    msg := sprintf("image fails to come from trusted registry: %v", [image])
}
</code></pre>

Where all but the repository specified image repository is prevented from pulling the image. With the policy above being enforced, a ValidatingWebhook can then be put in place:

<pre class="wp-block-code"><code>
apiVersion: admissionregistration.k8s.io/v1
metadata:
  name: opa-validating-webhook
webhooks:
  - name: validating-webhook.openpolicyagent.org
    namespaceSelector:
      matchExpressions:
      - key: openpolicyagent.org/webhook
        operator: NotIn
        values:
        - ignore
    rules:
      - operations: ["CREATE", "UPDATE"]
        apiGroups: ["*"]
        apiVersions: ["*"]
        resources: ["*"]
    clientConfig:
      caBundle: $(cat ca.crt | base64 | tr -d '\n')
      service:
        namespace: opa
        name: opa
</code></pre>

Now, whenever a pod requests an image from repositories other than what is specified in the policy, the validation controller will prevent the creation of that pod.

For more information about enabling your Kubernetes cluster to trust and use private registries, see [here](https://kubernetes.io/docs/concepts/containers/images/#using-a-private-registry).


## Validate Artifacts

Further trust in images can be established by signing images with gpg keys. Container image authors can sign a container image using a private GPG key. Those application authors then release their public GPG key alongside the binaries, which users can use to verify the image.

<pre class="wp-block-code"><code>
$ sudo gpg --verify /var/lib/docker/image/overlay2/imagedb/content/sha256/8f80b775d1d271f60bd2505d3c21a6c55e21b94bc2ef698be960a74872ff28c4.gpg

gpg: Signature made Wed 17 Feb 2021 09:27:05 AM PST
gpg:                using RSA key 168A3BB89FC72851E67D7C55E478D8119B9052B9
gpg: checking the trustdb
gpg: marginals needed: 3  completes needed: 1  trust model: pgp
gpg: depth: 0  valid:   1  signed:   0  trust: 0-, 0q, 0n, 0m, 0f, 1u
gpg: next trustdb check due at 2023-02-17
gpg: Good signature from "Software Dev <software_dev@example.com>" [ultimate]
</code></pre>

In addition to GPG keys, tools like Docker can also sign images to help ensure the authenticity of container images:

<pre class="wp-block-code"><code>
$ docker trust sign rxmllc/hello-world:v1

Signing and pushing trust metadata for rxmllc/hello-worldo:v1
The push refers to a repository [docker.io/rxmllc/hello-world]
a3fbb648f0bd: Layer already exists
5eac2de68a97: Layer already exists
8d4d1ab5ff74: Layer already exists v2: digest: sha256:8f6f460abf0436922df7eb06d28b3cdf733d2cac1a185456c26debbff0839c56 size: 1787
Signing and pushing trust metadata
Enter passphrase for repository key with ID 36d4c36:
Successfully signed docker.io/rxmllc/hello-world:v1

$ docker trust inspect --pretty rxmllc/hello-world

SIGNED TAG    DIGEST                                                           SIGNERS
v1            8f6f460abf0436922df7eb06d28b3cdf733d2cac1a185456c26debbff0839c56 (Repo Admin)

Administrative keys for rxmllc/hello-world:
Repository Key: 36d4c3601102fa7c5712a343c03b94469e5835fb27c191b529c06fd19c14a942
Root Key: 246d360f7c53a9021ee7d4259e3c5692f3f1f7ad4737b1ea8c7b8da741ad980b
</code></pre>

You can learn more about Docker Content Trust [here](https://docs.docker.com/engine/security/trust/).


# Practice Drill

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

The pod must be deployable on your cluster and should be in the <code>Running</code> state for this drill to be considered complete.



































Container images are immutable point-in-time snapshots of an application. When a container is built, all the current fixes, features, bugs, and vulnerabilities are packaged into that image. If any part of that container image, whether an application, a library, or some other dependency, has some identified vulnerability, it is critical to update those components. Vulnerability scanning is the practice of checking an application, containerized or otherwise, for any security risks that may be associated with a given release of an application.

Tools like Trivvy will often pull the specified image and run a scan against a database to find any outstanding vulnerabilities, rating and describing them in the output of the command.

Here is an example of Trivvy output for an image:

<pre class="wp-block-code"><code>
~$ trivy image rxmllc/alpine-git:0.1

2021-09-22T14:24:35.838-0700	INFO	Need to update DB
2021-09-22T14:24:35.838-0700	INFO	Downloading DB...
24.04 MiB / 24.04 MiB [------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------] 100.00% 27.67 MiB p/s 1s
2021-09-22T14:24:40.589-0700	INFO	Detected OS: alpine
2021-09-22T14:24:40.589-0700	INFO	Detecting Alpine vulnerabilities...
2021-09-22T14:24:40.589-0700	INFO	Number of language-specific files: 0

rxmllc/alpine-git:0.1 (alpine 3.11.3)
=====================================
Total: 41 (UNKNOWN: 0, LOW: 4, MEDIUM: 12, HIGH: 22, CRITICAL: 3)

+--------------+------------------+----------+-------------------+---------------+---------------------------------------+
|   LIBRARY    | VULNERABILITY ID | SEVERITY | INSTALLED VERSION | FIXED VERSION |                 TITLE                 |
+--------------+------------------+----------+-------------------+---------------+---------------------------------------+
| apk-tools    | CVE-2021-36159   | CRITICAL | 2.10.4-r3         | 2.10.7-r0     | libfetch before 2021-07-26, as        |
|              |                  |          |                   |               | used in apk-tools, xbps, and          |
|              |                  |          |                   |               | other products, mishandles...         |
|              |                  |          |                   |               | -->avd.aquasec.com/nvd/cve-2021-36159 |
+              +------------------+----------+                   +---------------+---------------------------------------+
|              | CVE-2021-30139   | HIGH     |                   | 2.10.6-r0     | In Alpine Linux apk-tools             |
|              |                  |          |                   |               | before 2.12.5, the tarball            |
|              |                  |          |                   |               | parser allows a buffer...             |
|              |                  |          |                   |               | -->avd.aquasec.com/nvd/cve-2021-30139 |
+--------------+------------------+          +-------------------+---------------+---------------------------------------+
| busybox      | CVE-2021-28831   |          | 1.31.1-r9         | 1.31.1-r10    | busybox: invalid free or segmentation |
|              |                  |          |                   |               | fault via malformed gzip data         |
|              |                  |          |                   |               | -->avd.aquasec.com/nvd/cve-2021-28831 |
+--------------+------------------+          +-------------------+---------------+---------------------------------------+

...
</code></pre>

The vulnerability is described per library. Vulnerable and fixed version numbers are presented alongside a short description of the vulnerability.

After running a tool like Trivvy, it is up to the developer to create a container image that takes the identified vulnerabilities and applies the appropriate fixed versions.
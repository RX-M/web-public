<!-- CKA Self-Study Mod 2 -->


# Understand application deployments and how to perform rolling updates and rollbacks


## Deployments

Kubernetes uses a family of API resources known as controllers to enforce some kind of desired state on the Kubernetes cluster. The desired state could be a number of pods run across the cluster, or they can be other states like ensuring persistent volumes are bound to persistent volume claims. Controller resources on Kubernetes are managed by some kind of controller component (an example being the kube-controller-manager). These controller components watch the API for their managed resource and take actions to reconcile any differences between the desired state of the user and the current running state of the cluster.

A deployment is a controller resource that ensures a specified number and version of pods run on the Kubernetes cluster. Deployments create and control ReplicaSets, which create and remove pods according to the deployment’s desired state. Kubelets report the current state of the pods to the Kubernetes API server. The kube-controller-manager compares the current state to the desired state (based on the deployment spec). If the current and desired states differ, the kube-controller-manager makes requests to the API Server to reconcile the differences.

The deployment spec declares the desired state of pod configurations under the pod template. The following example is a deployment of 3 nginx pods using the nginx version 1.16 image:

<pre class="wp-block-code"><code>
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    run: nginx
  name: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      run: nginx
  template:
    metadata:
      labels:
        run: nginx
    spec:
      containers:
      - name: nginx
        image: docker.io/nginx:1.16
</code></pre>


## Rolling Updates

Updates to the deployment’s pod template trigger a rolling update. When a deployment’s pod template is updated, a new replicaSet is created that then creates new pods based on the updated pod spec. When the new pods are created, the previous version’s replicaSet is scaled to zero to remove the old pods. This strategy is known as a rolling update.

The following example creates a deployment of nginx pods with 3 replicas. The deployment’s rollout status and history are verified with <code>kubectl rollout</code> .

<pre class="wp-block-code"><code>
$ kubectl create deployment nginx --image=docker.io/nginx:1.16 --replicas=3

deployment.apps/nginx created

$ kubectl rollout status deploy nginx

deployment "nginx" successfully rolled out

$ kubectl rollout history deploy nginx

deployment.apps/nginx
REVISION    CHANGE-CAUSE
1           none

$
</code></pre>

Next, update the deployment to use the nginx version 1.17 image. This update will trigger a rolling update. A new replicaSet will be created and the pods under old replicaSets will be terminated (scaled to 0). After updating the deployment, check the rollout status immediately to capture the rolling update.

<pre class="wp-block-code"><code>
$ kubectl set image deployment nginx nginx=docker.io/nginx:1.17

deployment.apps/nginx image updated

$ kubectl rollout status deploy nginx

Waiting for deployment "nginx" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "nginx" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "nginx" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "nginx" rollout to finish: 1 old replicas are pending termination...
Waiting for deployment "nginx" rollout to finish: 1 old replicas are pending termination...
deployment "nginx" successfully rolled out

$
</code></pre>

[Learn more about deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/) and [updating deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#updating-a-deployment).


## Deployments Rollbacks

Kubernetes allows users to undo deployment updates. Deployments can be rolled back to a previous version with <code>kubectl rollout undo</code> or you can specify a specific revision.

Using the previous example, let’s look at the revisions available.

<pre class="wp-block-code"><code>
$ kubectl rollout history deploy nginx

deployment.apps/nginx
REVISION  CHANGE-CAUSE
1         none
2         none

$
</code></pre>

The deployment’s update is now under revision 2.

Next we undo the rollout to a specific revision, watch the status, and check the rollout history.

<pre class="wp-block-code"><code>
$ kubectl rollout undo deploy nginx --to-revision=1

deployment.apps/nginx rolled back

$ kubectl rollout status deploy nginx

Waiting for deployment "nginx" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "nginx" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "nginx" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "nginx" rollout to finish: 1 old replicas are pending termination...
Waiting for deployment "nginx" rollout to finish: 1 old replicas are pending termination...
deployment "nginx" successfully rolled out

$ kubectl rollout history deploy nginx

deployment.apps/nginx
REVISION  CHANGE-CAUSE
2         none
3         none

$
</code></pre>

The deployment is back to using the nginx 1.16 image. The revision number is stored as an annotation on both the deployment and the ReplicaSets.

[Learn more about rolling back deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#rolling-back-to-a-previous-revision)


# Use ConfigMaps and Secrets to configure applications

There are several ways to configure applications running under Kubernetes. One way is to change the command and arguments running in the container using the <code>command</code> and <code>args</code> arrays in a yaml file:

<pre class="wp-block-code"><code>
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: busybox
  name: busybox
spec:
  containers:
  - command:
    - /bin/sh
    args:
    - -c
    - tail -f /dev/null
    image: busybox
    name: busybox
</code></pre>

Application configurations and credentials can be stored in the cluster as ConfigMap or Secret resources. Containers running in pods can consume configMaps and secrets as volumes or environment variables.  ConfigMaps can be created from literal key-value pairs or from files. Below, we create a configmap from a redis configuration file on disk:

<pre class="wp-block-code"><code>
$ cat redis.conf

bind 127.0.0.1
protected-mode yes
port 6379
tcp-backlog 511
timeout 0
tcp-keepalive 300

$ kubectl create configmap --from-file redis.conf redisconf

configmap/redisconf created

$ kubectl describe configmap redisconf

Name:         redisconf
Namespace:    default
Labels:       <none>
Annotations:  <none>

Data
====
redis.conf:
----
bind 127.0.0.1
protected-mode yes
port 6379
tcp-backlog 511
timeout 0
tcp-keepalive 300

Events:  <none>

$
</code></pre>

The redis.conf file is now available for any pod to use and mount. ConfigMaps are a good way to make common configuration files available to applications running anywhere in a Kubernetes cluster. The example below shows a pod that runs redis using the redis.conf file stored as a configmap:

<pre class="wp-block-code"><code>
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: redis-dev
  name: redis-dev
spec:
  containers:
  - command:
    - redis-server
    - /config/redis.conf
    image: redis
    name: redis-dev
    volumeMounts:
    - name: redis
      mountPath: /config
  volumes:
  - name: redis
    configMap:
      name: redisconf
  restartPolicy: OnFailure
</code></pre>

If more sensitive data must be handled, then the <code>Secret</code> API resource is available. Secrets come in many types, from opaque secrets which act like ConfigMaps to more specialized user-facing secrets like the TLS (which store certificates for other API objects to use directly) or Registry credential secrets (which are supplied in a pod specification to enable a Kubelet to pass private registry credentials to its container runtime).

Be aware that, in most "out of the box" Kubernetes setups, there are few methods in place that secure secrets. The user is responsible for taking actions like restricting access to secrets through RBAC, enabling encryption for Secrets by the API Server (to have them encrypted at rest in etcd), or enabling more robust key management system based integrations to protect secret data.

Secrets in the cluster are consumed in the same way as ConfigMaps as either environment variable values or files. The preferred method is through files since file-level permissions and additional security controls (like AppArmor or SecComp) can help secure the contents of those secrets.

[Learn more about configuring applications using configMaps to run under Kubernetes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/) and [how to influence container commands](https://kubernetes.io/docs/tasks/inject-data-application/define-command-argument-container/).


# Configure workload autoscaling

Applications deployed using a controller like a Deployment or statefulSet can be scaled up or down by modifying the number of replicas.

Changing the <code>replicas</code> key value in the controller’s spec will trigger an update to the application’s current replicaSet that increases (or reduces) the number of pods that run the application. This is done imperatively using <code>kubectl scale</code>:

<pre class="wp-block-code"><code>
$ kubectl scale deploy redis-prod --replicas=3

deployment.apps/redis-prod scaled
</code></pre>

Or declaratively by making changes to the controller’s spec’s YAML and applying it to the cluster:

<pre class="wp-block-code"><code>
$ nano redis-prod.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: redis-prod
  name: redis-prod
spec:
  replicas: 5
  selector:
    matchLabels:
      app: redis-prod
  template:
    metadata:
      labels:
        app: redis-prod
    spec:
      containers:
      - image: docker.io/redis:4.0
        name: redis

$ kubectl apply -f redis-prod.yaml

deployment.apps/redis-prod configured
</code></pre>

[Learn more about scaling your applications using controllers like deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#scaling-a-deployment).

The Horizontal Pod Autoscaler (HPA) can be used to update a Deployment or StatefulSet with the goal of automatically scaling the workload to match demand, or load. The Controller Manager will query metrics from either the resource metrics API (for per-pod resource metrics), or the custom metrics API (for all other metrics) and compare the current resource utilization against the metrics threshold specified in an HPA definition to determine scaling.

The simplest way to express an HPA using the resource metrics API is using the <code>autoscale</code> command, for example:

<pre class="wp-block-code"><code>
$ kubectl autoscale deployment redis-prod --cpu-percent=80 --min=5 --max=10
</code></pre>

For more details about pod autoscaling, see the [Kubernetes documentation on HPAs](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/).


# Understand the primitives used to create robust, self-healing, application deployments

A self-healing application in the context of Kubernetes:

<ul>
  <li>Automatically recovers containers from an unhealthy state</li>
  <li>Ensures at least a single copy of the application is running at all times</li>
  <li>Maintain a consistent network identity</li>
</ul>

These three are achieved by creating the Kubernetes resources that declare the desired state of the cluster. For those three items above, those include:

<ul>
  <li>Pods which declare the desired state for containers. The Kubelet ensures any containers recover from an unhealthy state per the pod spec</li>
  <li>Pods themselves can have additional settings like health checks in the form of liveness and readiness probes, which help the Kubelet test for application functionality and readiness for client traffic</li>
  <li>Controller resources like deployments. Deployments declare the desired state for pods. The Controller Manager ensures the desired number of pods are always present in the cluster</li>
  <li>Services which declare stable network endpoints that can route to pods. kube-proxy ensures all service addresses and routing are in place within the cluster</li>
</ul>

Users can create a simple self-healing application using a controller to maintain a desired state (at least one running pod) and a service to maintain a consistent network identity in the face of pod deletion/recreation.

<pre class="wp-block-code"><code>
$ kubectl create deploy apache-prod --image httpd

deployment.apps/apache-prod created

$ kubectl expose deploy apache-prod --port 80

service/apache-prod exposed
</code></pre>

This creates a deployment, which ensures at least a single copy of the application runs at all times and a service that maintains the consistent network identity.

A liveness probe configured in the deployment spec provides the pod’s managing kubelet with a way to check if the application is alive.

<pre class="wp-block-code"><code>
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: apache-prod
  name: apache-prod
spec:
  progressDeadlineSeconds: 600
  replicas: 1

  selector:
    matchLabels:
      app: apache-prod
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: apache-prod
    spec:
      containers:
      - image: httpd
        imagePullPolicy: Always
        livenessProbe:
          failureThreshold: 1
          httpGet:
            path: /
            port: 80
            scheme: HTTP
          periodSeconds: 10
          successThreshold: 1
          timeoutSeconds: 1
        name: httpd
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      dnsPolicy: ClusterFirst
      restartPolicy: Always
</code></pre>

If this probe’s livenessProbe ever returns a failure, the kubelet tells the container runtime to restart the container and bring the application back from an unhealthy state.

Learn about:
<ul>
  <li><a href="https://kubernetes.io/docs/concepts/services-networking/service/">Services</a></li>
  <li><a href="https://kubernetes.io/docs/concepts/workloads/controllers/deployment/">Deployments</a></li>
  <li><a href="https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/">Details about application probes</a></li>
</ul>


# Configure Pod admission and scheduling (limits, node affinity, etc.)

Workloads in Kubernetes are configured in pods, which define how containers should run in the cluster. Before pods can run their containers, they must be scheduled, or assigned, to one of the nodes in the cluster. This decision is made by the kube-scheduler based on the following factors:

<ul>
<li>Availability of CPU, memory, and ephemeral storage (based on what is reported by the Kubelet)</li>
<li>The presence of labels (if the user specified a node selector or affinity)</li>
<li>The presence of taints on the nodes and whether the pod tolerates those taints</li>
</ul>

Users have the option of defining resource requests and limits for pods. Resource requests define a minimum amount of CPU, memory, or ephemeral storage that must be available on a node. Resource limits ensure the containers in the pod cannot use more than a given amount of those same types of resources.

The following pod will only run on the cluster if there are are least 4 cores (4000 millicores) of available CPU and 256MB of available ram being reported as available by any Kubelets in the cluster:

<pre class="wp-block-code"><code>
~$ cat testpod2.yaml

apiVersion: v1
kind: Pod
metadata:
  name: webserver-resources
  labels:
    app: website
    component: webserver
    vendor: rx-m
spec:
  volumes:
  - name: log
    emptyDir: {}
  containers:
  - name: webserver
    image: docker.io/nginx:1.23.3
    resources:
      requests:
        cpu: 4
        memory: 256M
</code></pre>

If none of the Kubelets can fulfill the resource request, the pod goes into a <code>Pending</code> state. This generates an event in the pod's namespace, which you is viewable with <code>kubectl describe pod</code> or <code>kubectl get events</code>.

<pre class="wp-block-code"><code>
~$ kubectl apply -f testpod2.yaml

pod/webserver-resources created

ubuntu@ip-172-31-18-190:~$ kubectl get pods

NAME                  READY   STATUS    RESTARTS   AGE
webserver-resources   0/1     Pending   0          3s

ubuntu@ip-172-31-18-190:~$ kubectl describe pod webserver-resources

Name:             webserver-resources
Namespace:        default
Priority:         0
Service Account:  default
Node:             <none>
Labels:           app=website
                  component=webserver
                  vendor=rx-m
Annotations:      <none>
Status:           Pending
IP:
IPs:              <none>
Containers:
  webserver:
    Image:      nginx:1.23.3
    Port:       <none>
    Host Port:  <none>
    Requests:
      cpu:        4
      memory:     256M
    Environment:  <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-fbdq4 (ro)
Conditions:
  Type           Status
  PodScheduled   False
Volumes:
  log:
    Type:       EmptyDir (a temporary directory that shares a pod's lifetime)
    Medium:
    SizeLimit:  <unset>
  kube-api-access-fbdq4:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   Burstable
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type     Reason            Age   From               Message
  ----     ------            ----  ----               -------
  Warning  FailedScheduling  10s   default-scheduler  0/1 nodes are available: 1 Insufficient cpu. preemption: 0/1 nodes are available: 1 No preemption victims found for incoming pod..
</code></pre>

The following procedures can help a cluster run a pod that cannot be scheduled due to lack of resources:

<ul>
<li>Adjust the pod resource requests to fit what is available in the cluster</li>
<li>Scale down (remove pods) other workloads to release resources</li>
<li>Add an additional node to the cluster to increase workload capacity by presenting more resources</li>
</ul>

Once a pod is scheduled, the Kubelet thin provisions the requested resources and subtracts the requested amounts from its allocatable resource pool. The only other process that takes requests into account at this point is autoscaling, which uses the resource request to help calculate the scaling threshold. Resource requests cannot be changed on an existing pod: the entire pod much be redeployed with new limits if a change is necessary.

During the lifetime of the pod and its containers, the Kubelet enforces any specified resource limits the pod. This ensures that that the cluster can automatically remove and reschedule the violating pod and its containers. Depending on the resource request and limit configuration, the Pod gets classified in one of three categories:

<ul>
<li>BestEffort - assigned to pods whose containers do not specify any resource requests or limits</li>
<li>Burstable - assigned to pods with at least one container that specific a resource request or limit</li>
<li>Guaranteed - assigned to pods where all containers specify identical resource request and limit values</li>
</ul>

Depending on this categorization, or Quality of Service (QOS) class, the Kubelet makes a decision on which pods get removed based on this priority: BestEffort -> Burstable -> Guaranteed.

[Learn more about managing container resources here](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/). There is also [a task page](https://kubernetes.io/docs/tasks/configure-pod-container/assign-cpu-resource/) on the Kubernetes docs that describe to to implement them.


# Practice Drill

Create a deployment with five replicas named <code>cicd</code> that creates pods that run the <code>docker.io/jenkins/jenkins:lts</code> image.


# CONTENT FROM BEFORE FEB 2025 REMAINS BELOW IN CASE THE EXAM INCLUDES IT IN THE FUTURE


# Manifest Management and Common Templating Tools

An application deployed on Kubernetes requires many different API resources to function properly, including:

<ul>
<li>Deployments or other controllers to run the application pods</li>
<li>RBAC resources to ensure that the application can communicate with the API server (if necessary)</li>
<li>ConfigMaps and secrets to help configure the application</li>
<li>Custom resource definitions (CRDs) and instances of those custom resources</li>
</ul>

If all documents specifying each of these resources, known as manifests, are deployed together then an application will run successfully. This is feasible for simple applications that are already configured to run in a specific environment, but for more complex and generally available applications, some kind of manifest management and templating tool is necessary.

Two popular choices for manifest management that are accessible through the test resources are Helm and Kustomize.


## Kustomize

Kustomize is a component of <code>kubectl</code> that enables users to generate a generic set of Kubernetes specifications and environment-specific overrides to deploy an application.

At the heart of Kustomize is the <code>kustomization.yaml</code>. This defines a variety of parameters and manifests to be combined:

<pre class="wp-block-code"><code>
$ cat kustomization.yaml 

namespace: test
resources:
- test-run.yaml
configMapGenerator:
- name: app-env-vars
  files:
  - test-vars.env
</code></pre>

This YAML file will generate manifests bound for the <code>test</code> namespace. It will insert these values into a manifest with the file name <code>test-run.yaml</code>. In addition to the namespace parameter, it will also generate a configmap with the contents of another file, <code>test-vars.env</code>, as its data.

The <code>test-run.yaml</code> defines a Job resource whose containers will mount the configmap app-env-vars (which will be generated by kustomize)

<pre class="wp-block-code"><code>
$ cat test-run.yaml 

apiVersion: batch/v1
kind: Job
metadata:
  name: test-run
spec:
  template:
    spec:
      containers:
      - image: reg.internal:4500/sampler:stable
        name: test-run
        command:
        - test
        - -c
        - /config/test-vars.env
        volumeMounts:
        - mountPath: /config
          name: config
      volumes:
      - configMap:
          name: app-env-vars
        name: config
</code></pre>

And the <code>test-vars.env</code> is the file to be stored in the generated configmap:

<pre class="wp-block-code"><code>
$ cat test-vars.env 
PARAM_1="true"
PARAM_2="outdoor"
PARAM_3="short"
</code></pre>

As long as all of the files mentioned are colocated in the same directory, the <code>kubectl kustomize</code> command can be invoked:

<pre class="wp-block-code"><code>
$ ls -l

total 12
-rw-rw-r-- 1 ubuntu ubuntu 111 Jul  5 18:01 kustomization.yaml
-rw-rw-r-- 1 ubuntu ubuntu 410 Jul  5 18:02 test-run.yaml
-rw-rw-r-- 1 ubuntu ubuntu  49 Jul  5 18:02 test-vars.env

$ kubectl kustomize ./

apiVersion: v1
data:
  test-vars.env: |
    PARAM_1="true"
    PARAM_2="outdoor"
    PARAM_3="short"
kind: ConfigMap
metadata:
  name: app-env-vars-777chm5mf2
  namespace: test
---
apiVersion: batch/v1
kind: Job
metadata:
  name: test-run
  namespace: test
spec:
  template:
    spec:
      containers:
      - command:
        - test
        - -c
        - /config/test-vars.env
        image: reg.internal:4500/sampler:stable
        name: test-run
        volumeMounts:
        - mountPath: /config
          name: config
      volumes:
      - configMap:
          name: app-env-vars-777chm5mf2
        name: config
</code></pre>

Here, two manifests have been generated: one for the ConfigMap (which received an automatically generated name) and a modified version of the Job manifest, which has the namespace inserted and the generated configmap name.

The main advantage of kustomize is that no other tools are necessary - it is built into the <code>kubectl</code> binary.

[Learn more about Kustomize from the Kubernetes Documentation here](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/).
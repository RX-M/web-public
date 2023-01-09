<!-- CKA Self-Study Mod 2 -->


# Deployments, Rolling Updates, and RollBacks


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
        image: nginx:1.16
</code></pre>


## Rolling Updates

Updates to the deployment’s pod template trigger a rolling update. When a deployment’s pod template is updated, a new replicaSet is created that then creates new pods based on the updated pod spec. When the new pods are created, the previous version’s replicaSet is scaled to zero to remove the old pods. This strategy is known as a rolling update.

The following example creates a deployment of nginx pods with 3 replicas. The deployment’s rollout status and history are verified with <code>kubectl rollout</code> .

<pre class="wp-block-code"><code>
$ kubectl create deployment nginx --image=nginx:1.16 --replicas=3

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
$ kubectl set image deployment nginx nginx=nginx:1.17

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


# Configure Applications

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


# Scale Applications

Applications deployed using a controller like a Deployment or statefulSet can be scaled up or down by modifying the number of replicas.

Changing the <code>replicas</code> key value in the controller’s spec will trigger an update to the application’s current replicaSet that increases (or reduces) the number of pods that run the application. This is done imperatively using <code>kubectl scale</code>:

<pre class="wp-block-code"><code>
$ kubectl scale deploy redis-prod --replicas=3

deployment.apps/redis-prod scaled

$
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
      - image: redis:4.0
        name: redis

$ kubectl apply -f redis-prod.yaml

deployment.apps/redis-prod configured

$
</code></pre>

[Learn more about scaling your applications using controllers like deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#scaling-a-deployment).


# Self-healing Applications

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

$
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
  <li>[Services](https://kubernetes.io/docs/concepts/services-networking/service/)</li>
  <li>[Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)</li>
  <li>[Details about application probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/).</li>
</ul>


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
-rw-rw-r-- 1 ubuntu ubuntu 111 Jan  9 17:03 kustomization.yaml
-rw-rw-r-- 1 ubuntu ubuntu 418 Jan  9 17:03 test-run.yaml
-rw-rw-r-- 1 ubuntu ubuntu  49 Jan  9 16:58 test-vars.env

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


## Helm

Helm is a very popular option for distributing Kubernetes applications. A series of templatized manifests are packaged into a single deployment artifact called a chart. This chart can be downloaded and distributed much more easily than a series of manifests. At install time, the chart can be supplied with a series of values meant to populate the template. This is driven by the <code>helm</code> command, which handles the rendering of chart manifests and submission to the Kubernetes API.

The most basic form of a Helm chart consists of the following directory structure:

<pre class="wp-block-code"><code>
$ ls -lR simplechart/

simplechart/:
total 8
-rw-rw-r-- 1 ubuntu ubuntu   48 Jan  9 17:27 Chart.yaml
drwxrwxr-x 2 ubuntu ubuntu 4096 Jan  9 17:27 templates

simplechart/templates:
total 4
-rw-rw-r-- 1 ubuntu ubuntu 368 Jan  9 17:27 deployment.yaml
</code></pre>

The Chart.yaml presents the identity and other metadata of the chart, allowing tools like <code>helm</code> to properly handle them.

<pre class="wp-block-code"><code>
$ cat simplechart/Chart.yaml 

apiVersion: v2
name: simplechart
version: 0.1.0
</code></pre>

A template directory is also present, which will hold one or more templatized manifests to be filled in, or rendered, by Helm at install time.

Here, there is only one template for a deployment:

<pre class="wp-block-code"><code>
$ cat simplechart/templates/deployment.yaml 

apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: simplechart
  name: simplechart
spec:
  replicas: {{ .Values.replicas }}
  selector:
    matchLabels:
      app: simplechart
  template:
    metadata:
      labels:
        app: simplechart
    spec:
      containers:
      - image: nginx
        name: nginx
        ports:
        - containerPort: 80
</code></pre>

This deployment's <code>replicas</code> key has been templatized to accept a parameter, or value, at install time. This template line gets replaced when the <code>helm install</code> command is passed with a value. 

Here, we run that command set use the <code>--set replicas=1</code> option to provide the value for the templatized <code>replicas</code> key. Values can also be provided declaratively using a file called <code>values.yaml</code>:

<pre class="wp-block-code"><code>
$ helm install simplerelease2 ./simplechart/ --set replicas=1 --debug

install.go:192: [debug] Original chart version: ""
install.go:209: [debug] CHART PATH: /home/ubuntu/simplechart

client.go:128: [debug] creating 1 resource(s)
NAME: simplerelease2
LAST DEPLOYED: Mon Jan  9 17:34:16 2023
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
USER-SUPPLIED VALUES:
replicas: 1

COMPUTED VALUES:
replicas: 1

HOOKS:
MANIFEST:
---
# Source: simplechart/templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: simplechart
  name: simplechart
spec:
  replicas: 1
  selector:
    matchLabels:
      app: simplechart
  template:
    metadata:
      labels:
        app: simplechart
    spec:
      containers:
      - image: nginx
        name: nginx
        ports:
        - containerPort: 80
</code></pre>

The manifest was rendered with <code>replicas: 1</code> per the template and supplied value. In addition to rendering the templates, Helm also submits those generated manifests to the Kubernetes API:

<pre class="wp-block-code"><code>
$ kubectl get all

NAME                               READY   STATUS    RESTARTS   AGE
pod/simplechart-85786b6496-2c4kd   1/1     Running   0          102s

NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
service/kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   67m

NAME                          READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/simplechart   1/1     1            1           102s

NAME                                     DESIRED   CURRENT   READY   AGE
replicaset.apps/simplechart-85786b6496   1         1         1       102s
</code></pre>

The install process creates a release of the Helm chart, which can be updated or removed as a single unit using Helm:

<pre class="wp-block-code"><code>
$ helm ls

NAME          	NAMESPACE	REVISION	UPDATED                                	STATUS  	CHART            	APP VERSION
simplerelease2	default  	1       	2023-01-09 17:34:16.622395997 +0000 UTC	deployed	simplechart-0.1.0	           
ubuntu@labsys:~$ 

$ helm uninstall simplerelease2

release "simplerelease2" uninstalled

ubuntu@labsys:~$ kubectl get all

NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
service/kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   68m
</code></pre>

[Become familiar with the Helm website, which you can access during the exam, here](https://helm.sh/).


# Practice Drill

Create a deployment with five replicas named <code>cicd</code> that creates pods that run the <code>jenkins/jenkins:lts</code> image.
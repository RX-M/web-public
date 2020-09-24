<!-- CKA Self-Study Mod 2 -->


# Deployments and Rolling Updates

A deployment is a controller that ensures an application’s pods run according to a desired state. Deployments create and control replicaSets, which create and remove pods according to the deployment’s desired state. Kubelets report the current state to the Kubernetes API server. The API server compares the current state to the desired state (stored in etcd). If the current and desired states differ, the Kubernetes API server tells the kubelet(s) to make deployment changes to match the desired state.

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

Updates to the deployment’s pod template trigger a gradual update. When a deployment’s pod template is updated, a new replicaSet is created that then creates new pods based on the updated pod spec. When the new pods are created, the previous version’s replicaSet is scaled to zero to remove the old pods. This strategy is known as a rolling update.

The following example creates a deployment of nginx pods with 3 replicas. The <code>--record</code> option annotates and saves the <code>kubectl </code> command for future reference. The deployment’s rollout status and history are verified with <code>kubectl rollout</code> .

<pre class="wp-block-code"><code>
$ kubectl run nginx --image=nginx:1.16 --replicas=3 --record

kubectl run --generator=deployment/apps.v1 is DEPRECATED and will be removed in a future version. Use kubectl run --generator=run-pod/v1 or kubectl create instead.
deployment.apps/nginx created

$ kubectl rollout status deploy nginx

deployment "nginx" successfully rolled out

$ kubectl rollout history deploy nginx

deployment.apps/nginx
REVISION    CHANGE-CAUSE
1                    kubectl run nginx --image=nginx:1.16 --replicas=3 --record=true

$
</code></pre>
Because the <code>--record</code> option was used to create the deployment, the annotation is listed under the <code>CHANGE-CAUSE</code> column. If <code>--record</code> was not used to annotate then <code>none</code> would appear under <code>CHANGE-CAUSE</code> for revision 1.

Next, update the deployment to use the nginx version 1.17 image. This update will trigger a rolling update. A new replicaSet will be created and the pods under old replicaSets will be terminated (scaled to 0). After updating the deployment, check the rollout status immediately to capture the rolling update.

<pre class="wp-block-code"><code>
$ kubectl set image deploy nginx nginx=nginx:1.17 --record

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


# Deployments and Rollbacks

Kubernetes allows users to undo deployment updates. Deployments can be rolled back to a previous version with <code>kubectl rollout undo deploy <deployment_name></code> or you can specify a specific revision.

Using the previous example, let’s look at the revisions available.

<pre class="wp-block-code"><code>
$ kubectl rollout history deploy nginx

deployment.apps/nginx
REVISION  CHANGE-CAUSE
1         kubectl run nginx --image=nginx:1.16 --replicas=3 --record=true
2         kubectl set image deploy nginx nginx=nginx:1.17 --record=true

$
</code></pre>

The deployment’s update is now under revision 2. Again, if <code>--record</code> was not used to annotate then <code>none</code> would be listed under the <code>CHANGE-CAUSE</code> column.

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
2         kubectl set image deploy nginx nginx=nginx:1.17 --record=true
3         kubectl run nginx --image=nginx:1.16 --replicas=3 --record=true

$
</code></pre>

The deployment is back to using the nginx 1.16 image.

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
Automatically recovers containers from an unhealthy state
Ensures at least a single copy of the application is running at all times
Maintain a consistent network identity

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
- [Services](https://kubernetes.io/docs/concepts/services-networking/service/)
- [Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [Details about application probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/).


# Practice Drill

Create a deployment with five replicas named <code>cicd</code> that creates pods that run the <code>jenkins/jenkins:lts</code> image.

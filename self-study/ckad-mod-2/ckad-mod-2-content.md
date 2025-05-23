<!-- CKAD Self-Study Mod 2 -->

# Application Deployment


# Use Kubernetes primitives to implement common deployment strategies (e.g. blue/green or canary)

Deployments when combined with services, can expose new application features and updates to users with little downtime. That combination allows a basic Kubernetes clusters to perform things like canary or blue/green deployments.

A canary deployment is a pattern where new features or updates are exposed to users gradually. A canary deployment can be achieved by taking a service backed by a deployment and adding additional pods with the updated code to that service (whether they are standalone pods or another deployment).

Given an existing webserver deployment with two pods and a service, each time clients send a request to the service IP address, one of the original webserver pods returns the response:

<pre class="wp-block-code"><code>$ kubectl get pods,svc -l app=webserver

NAME                                   READY   STATUS    RESTARTS   AGE
pod/webserver-557f5b46fd-bdsfs         1/1     Running   0          16s
pod/webserver-557f5b46fd-dm9rg         1/1     Running   0          16s
pod/webserver-canary-78c74979f-5nmwf   1/1     Running   0          106s

NAME                TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
service/webserver   ClusterIP   10.104.77.88   <none>        80/TCP    8s

$ curl 10.104.77.88

&lt;!DOCTYPE html&gt;
&lt;html&gt;
&lt;head&gt;
&lt;title&gt;Welcome to nginx!&lt;/title&gt;

...

</code></pre>

Creating another deployment whose pods have the <code>app=webserver</code> label adds those pods as endpoints to the <code>webserver</code> service:

<pre class="wp-block-code"><code>$ cat webserver-canary.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: webserver-canary
spec:
  replicas: 1
  selector:
    matchLabels:
      app: webserver
  template:
    metadata:
      labels:
        app: webserver
    spec:
      containers:
      - image: docker.io/httpd:latest
        name: httpd

$ kubectl apply -f webserver-canary.yaml

deployment.apps/webserver-canary created

$ kubectl get svc,pods -l app=webserver

NAME                TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
service/webserver   ClusterIP   10.104.77.88   <none>        80/TCP    60s

NAME                                   READY   STATUS    RESTARTS   AGE
pod/webserver-557f5b46fd-bdsfs         1/1     Running   0          68s
pod/webserver-557f5b46fd-dm9rg         1/1     Running   0          68s
pod/webserver-canary-78c74979f-kwhtm   1/1     Running   0          7s
</code></pre>

In the typical canary deployment, a majority of requests made to the service should go to the original pods. The canary deployment should have fewer active pods to enable a portion of that traffic (33% in the case of a service with 3 pods) to be routed to the canary:

<pre class="wp-block-code"><code>$ curl 10.104.77.88

&lt;!DOCTYPE html&gt;
&lt;html&gt;
&lt;head&gt;
&lt;title&gt;Welcome to nginx!&lt;/title&gt;

...

$ curl 10.104.77.88

&lt;html&gt;&lt;body&gt;&lt;h1&gt;It works!&lt;/h1&gt;&lt;/body&gt;&lt;/html&gt;
</code></pre>

If a gradual switch to a new version is not needed, then the blue/green strategy can be implemented instead. In a blue/green strategy, the network endpoint (the service) is instructed to send all traffic to new versions of the application that serve as the back end.

Given the following scenario, with a service backed by NGINX and a "new" webserver backed by Apache webserver:

<pre class="wp-block-code"><code>$ kubectl get svc,pods

NAME                TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
service/webserver   ClusterIP   10.104.77.88   <none>        80/TCP    2m54s

NAME                                   READY   STATUS    RESTARTS   AGE
pod/new-webserver-58bd87898c-59b8s     1/1     Running   0          13s
pod/new-webserver-58bd87898c-fmmhq     1/1     Running   0          13s
pod/new-webserver-58bd87898c-w64gm     1/1     Running   0          13s
pod/webserver-557f5b46fd-bdsfs         1/1     Running   0          3m2s
pod/webserver-557f5b46fd-dm9rg         1/1     Running   0          3m2s
pod/webserver-canary-78c74979f-kwhtm   1/1     Running   0          2m1s
</code></pre>

For a blue-green deployment, switching the service's selector (which determines which pods a service send traffic to) ensures that the service goes from the blue backend (NGINX) to the green backend (httpd).

<pre class="wp-block-code"><code>$ kubectl set selector svc webserver app=new-webserver

service/webserver selector updated
</code></pre>

Now, 100% of all requests go to the new, Apache webserver while the old NGINX websever can be safely retired:

<pre class="wp-block-code"><code>$ curl 10.104.77.88

&lt;html&gt;&lt;body&gt;&lt;h1&gt;It works!&lt;/h1&gt;&lt;/body&gt;&lt;/html&gt;

$ curl 10.104.77.88

&lt;html&gt;&lt;body&gt;&lt;h1&gt;It works!&lt;/h1&gt;&lt;/body&gt;&lt;/html&gt;

$ curl 10.104.77.88

&lt;html&gt;&lt;body&gt;&lt;h1&gt;It works!&lt;/h1&gt;&lt;/body&gt;&lt;/html&gt;
</code></pre>

Learn more about <a href="https://kubernetes.io/blog/2018/04/30/zero-downtime-deployment-kubernetes-jenkins/" target="_blank" rel="noreferrer noopener">blue/green deployments</a> and <a href="https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#canary-deployment" target="_blank" rel="noreferrer noopener">canary deployments</a>.


# Understand Deployments and how to perform rolling updates

A deployment is a controller that ensures an application’s pods run according to a desired state. Deployments create and control replicaSets, which create and remove pods according to the deployment’s desired state. Kubelets report the current state to the Kubernetes API server. The API server compares the current state to the desired state (stored in etcd). If the current and desired states differ, the Kubernetes API server tells the kubelet(s) to make deployment changes to match the desired state.

The deployment spec declares the desired state of pod configurations under the pod template. The following example is a deployment of 3 nginx pods using the nginx version 1.16 image:

<pre class="wp-block-code"><code>apiVersion: apps/v1
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

Updates to the deployment’s pod template trigger a gradual update. When a deployment’s pod template is updated, a new replicaSet is created that then creates new pods based on the updated pod spec. When the new pods are created, the previous version’s replicaSet is scaled to zero to remove the old pods. This strategy is known as a rolling update.

The following example creates a deployment of nginx pods with 3 replicas. The <code>--record</code> option annotates and saves the <code>kubectl</code> command for future reference. The deployment’s rollout status and history are verified with <code>kubectl rollout</code>.

<pre class="wp-block-code"><code>$ kubectl create deploy nginx --image=docker.io/nginx:1.16 --replicas=3

deployment.apps/nginx created

$ kubectl rollout status deploy nginx

deployment "nginx" successfully rolled out
</code></pre>

Next, update the deployment to use the nginx version 1.17 image. This update will trigger a rolling update. A new replicaSet will be created and the pods under old replicaSets will be terminated (scaled to 0). After updating the deployment, check the rollout status immediately to capture the rolling update.

<pre class="wp-block-code"><code>$ kubectl set image deploy nginx nginx=nginx:1.17

deployment.apps/nginx image updated

$ kubectl rollout status deploy nginx

Waiting for deployment "nginx" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "nginx" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "nginx" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "nginx" rollout to finish: 1 old replicas are pending termination...
Waiting for deployment "nginx" rollout to finish: 1 old replicas are pending termination...
deployment "nginx" successfully rolled out
</code></pre>

Learn more about <a href="https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#updating-a-deployment" target="_blank" rel="noreferrer noopener">deployments</a> and <a href="https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#updating-a-deployment">updating deployments</a>.


## Deployments and Rollbacks

Kubernetes allows users to undo deployment updates. Deployments can be rolled back to a previous version with <code>kubectl rollout undo deploy <deployment_name></code> or you can specify a specific revision.

Using the previous example, let’s look at the revisions available.

<pre class="wp-block-code"><code>$ kubectl rollout history deploy nginx

deployment.apps/nginx
REVISION  CHANGE-CAUSE
1         <none>
2         <none>
</code></pre>

The deployment’s update is now under revision 2. Again, if <code>--record</code> was not used to annotate then <code>none</code> would be listed under the <code>CHANGE-CAUSE</code> column.

Next we undo the rollout to a specific revision, watch the status, and check the rollout history.

<pre class="wp-block-code"><code>$ kubectl rollout undo deploy nginx --to-revision=1

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
2         <none>
3         <none>

$
</code></pre>

The deployment is back to using the nginx 1.16 image.

Learn more about <a href="https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#rolling-back-to-a-previous-revision" target="_blank" rel="noreferrer noopener">rolling back deployments</a>.


## Scaling Deployments

Applications deployed using a controller like a Deployment or statefulSet can be scaled up or down by modifying the number of replicas.

Changing the <code>replicas</code> key value in the controller’s spec will trigger an update to the application’s current replicaSet that increases (or reduces) the number of pods that run the application. This is done imperatively using <code>kubectl scale</code>:

<pre class="wp-block-code"><code>$ kubectl scale deploy redis-prod --replicas=3

deployment.apps/redis-prod scaled
</code></pre>

Or declaratively by making changes to the controller’s spec’s YAML and applying it to the cluster:

<pre class="wp-block-code"><code>$ nano redis-prod.yaml

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

Learn more about <a href="https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#scaling-a-deployment" target="_blank" rel="noreferrer noopener">scaling your applications using controllers like deployments</a>.


# Use the Helm package manager to deploy existing packages

A popular method of deploying applications on Kubernetes is through the use of templating tools like Helm. These tools allow users to specify all of the components of an application meant to run on Kubernetes in a series of templated YAML files. Tools like Helm take these templated YAML files, populate them with user-defined values, and deploy all of those resources together.

Helm take the concept of an application package and applies it to Kubernetes. The application's resource templates are grouped into packages known as charts. Charts include all of the YAML templates that represent Kubernetes resources deployed by Helm and supporting components like metadata, information on dependencies, custom resource definitions, and tests to be run.

Deploying an application with helm is straightforward. First, identify a chart that you want to install. Common chart repositories include artifacthub.io, which in provides as a central space to find charts.

If you do not already have Helm installed, use the instructions from their documentation (which is available to you during the exam!): https://helm.sh/docs/intro/install/

Once you have identified a chart, you need to add that chart as a source to your Helm instance: 

<pre class="wp-block-code"><code>$ helm repo add bitnami https://charts.bitnami.com/bitnami

"bitnami" has been added to your repositories

$ helm repo update

Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "bitnami" chart repository
Update Complete. ⎈Happy Helming!⎈
</code></pre>

After adding the chart repository to helm, your <code>helm</code> command can now search and install charts from the selected repository.

If you want to install, say, a NGINX using helm from the recently added repository, you use helm's <code>install</code> command to generate a release of that chart. The release is the grouping of Kubernetes resources generated from the template in the chart. The resources are populated with values provided by the user (using the <code>--set</code> flag or providing a yaml file with values) or the default values of provided by the chart during the installation:

<pre class="wp-block-code"><code>$ helm install self-study-nginx bitnami/nginx

NAME: self-study-nginx
LAST DEPLOYED: Sat Jul 13 00:28:53 2024
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
CHART NAME: nginx
CHART VERSION: 18.1.4
APP VERSION: 1.27.0

** Please be patient while the chart is being deployed **
NGINX can be accessed through the following DNS name from within your cluster:

    self-study-nginx.default.svc.cluster.local (port 80)

To access NGINX from outside the cluster, follow the steps below:

1. Get the NGINX URL by running these commands:

  NOTE: It may take a few minutes for the LoadBalancer IP to be available.
        Watch the status with: 'kubectl get svc --namespace default -w self-study-nginx'

    export SERVICE_PORT=$(kubectl get --namespace default -o jsonpath="{.spec.ports[0].port}" services self-study-nginx)
    export SERVICE_IP=$(kubectl get svc --namespace default self-study-nginx -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
    echo "http://${SERVICE_IP}:${SERVICE_PORT}"

WARNING: There are "resources" sections in the chart not set. Using "resourcesPreset" is not recommended for production. For production installations, please set the following values according to your workload needs:
  - cloneStaticSiteFromGit.gitSync.resources
  - resources
+info https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/

⚠ SECURITY WARNING: Original containers have been substituted. This Helm chart was designed, tested, and validated on multiple platforms using a specific set of Bitnami and Tanzu Application Catalog containers. Substituting other containers is likely to cause degraded security and performance, broken chart features, and missing environment variables.

Substituted images detected:
  - %!s(<nil>)/:%!s(<nil>)
</code></pre>

You can easily identify the resources generated by Helm because every resource bears the name of its release once deployed to the cluster:

<pre class="wp-block-code"><code>$ kubectl get all -l app.kubernetes.io/managed-by=Helm

NAME                                    READY   STATUS    RESTARTS   AGE
pod/self-study-nginx-7465568b8d-9w7xt   1/1     Running   0          46s

NAME                       TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
service/self-study-nginx   LoadBalancer   10.104.94.141   <pending>     80:32680/TCP,443:30438/TCP   46s

NAME                               READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/self-study-nginx   1/1     1            1           46s

NAME                                          DESIRED   CURRENT   READY   AGE
replicaset.apps/self-study-nginx-7465568b8d   1         1         1       46s
</code></pre>

Once a release is generated, you can change the parameters by changing values (again using the <code>--set</code> or providing an updated values yaml file) and using Helm's <code>upgrade</code> subcommand:

<pre class="wp-block-code"><code>$ helm upgrade self-study-nginx --set service.type=NodePort bitnami/nginx

Release "self-study-nginx" has been upgraded. Happy Helming!
NAME: self-study-nginx
LAST DEPLOYED: Sat Jul 13 00:29:57 2024
NAMESPACE: default
STATUS: deployed
REVISION: 2
TEST SUITE: None
NOTES:
CHART NAME: nginx
CHART VERSION: 18.1.4
APP VERSION: 1.27.0

** Please be patient while the chart is being deployed **
NGINX can be accessed through the following DNS name from within your cluster:

    self-study-nginx.default.svc.cluster.local (port 80)

To access NGINX from outside the cluster, follow the steps below:

1. Get the NGINX URL by running these commands:

    export NODE_PORT=$(kubectl get --namespace default -o jsonpath="{.spec.ports[0].nodePort}" services self-study-nginx)
    export NODE_IP=$(kubectl get nodes --namespace default -o jsonpath="{.items[0].status.addresses[0].address}")
    echo "http://${NODE_IP}:${NODE_PORT}"

WARNING: There are "resources" sections in the chart not set. Using "resourcesPreset" is not recommended for production. For production installations, please set the following values according to your workload needs:
  - cloneStaticSiteFromGit.gitSync.resources
  - resources
+info https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/

⚠ SECURITY WARNING: Original containers have been substituted. This Helm chart was designed, tested, and validated on multiple platforms using a specific set of Bitnami and Tanzu Application Catalog containers. Substituting other containers is likely to cause degraded security and performance, broken chart features, and missing environment variables.

Substituted images detected:
  - %!s(<nil>)/:%!s(<nil>)
</code></pre>

The releases themselves are managed directly by Helm, and can be uninstalled using Helm's <code>uninstall</code> command:

<pre class="wp-block-code"><code>$ helm uninstall self-study-nginx

release "self-study-nginx" uninstalled

$ kubectl get all -l app.kubernetes.io/managed-by=Helm

No resources found in default namespace.
</code></pre>


Learn more about the <a href="https://helm.sh/docs/" target="_blank" rel="noreferrer noopener">Helm package manager and its use</a>.


# Kustomize

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

The <code>test-run.yaml</code> defines a Job resource whose containers will mount the configmap app-env-vars (which will be generated by kustomize).

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
-rw-rw-r-- 1 ubuntu ubuntu 111 Jul  13 00:31 kustomization.yaml
-rw-rw-r-- 1 ubuntu ubuntu 410 Jul  13 00:31 test-run.yaml
-rw-rw-r-- 1 ubuntu ubuntu  49 Jul  13 00:31 test-vars.env

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


# Practice Drill

Create a deployment with five replicas named <code>cicd</code> that creates pods that run the <code>jenkins/jenkins:lts</code> image.
<!-- CKAD Self-Study Mod 3 -->

Create the <code>my-sa</code> ServiceAccount:

<pre class="wp-block-code"><code>
$ kubectl create serviceaccount my-sa

</code></pre>

Update the create and update a pod to use the <code>docker.io/nginx:latest</code> image:

<pre class="wp-block-code"><code>
$ kubectl run mypod --image docker.io/nginx:latest -o yaml --dry-run=client > mysapod.yaml

$ nano mysapod.yaml && cat $_

apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: mypod
  name: mypod
spec:
  containers:
  - image: docker.io/nginx:latest
    name: mypod
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
  serviceAccountName: my-sa     # Add this
status: {}
</code></pre>

Apply the pod and you can see the ServiceAccount assignment with <code>kubectl describe</code>:

<pre class="wp-block-code"><code>
$ kubectl apply -f mysapod.yaml

pod/mypod created

$ kubectl describe pod mysapod | grep -i service

Service Account:  my-sa
</code></pre>
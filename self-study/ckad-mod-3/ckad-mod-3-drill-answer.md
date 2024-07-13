<!-- CKAD Self-Study Mod 3 -->

Create the <code>my-sa</code> ServiceAccount:

<pre class="wp-block-code"><code>$ kubectl create serviceaccount my-sa

$
</code></pre>

Update the deployment to use the <code>nginx:latest</code> image:

<pre class="wp-block-code"><code>$ nano mysapod.yaml ; cat $_

apiVersion: v1
kind: Pod
metadata:
  labels:
    run: mysapod
  name: mysapod
spec:
  containers:
  - image: rxmllc/hostinfo:latest
    name: mysapod.yaml
  serviceAccountName: my-sa
 
$
</code></pre>

Apply the pod and you can see the ServiceAccoutn assignment under something like `kubectl describe`:

<pre class="wp-block-code"><code>$ kubectl describe pod mysapod | grep Service

Service Account:  my-sa
</code></pre>


As an additional exercise, try creating a pod that binds to a persistent volume claim and create the necessary Kubernetes objects.

RX-M can provide more help with preparing for the CKAD exam in one of our CKAD bootcamps; we offer open enrollments and private engagements for teams or organizations.

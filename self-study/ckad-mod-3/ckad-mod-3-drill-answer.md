<!-- CKAD Self-Study Mod 3 -->

Create the <code>nginx</code> deployment:

<pre class="wp-block-code"><code>$ kubectl create deployment nginx --image=nginx:1.9 --replicas=2 --record

deployment.apps/nginx created

$
</code></pre>

Update the deployment to use the <code>nginx:latest</code> image:

<pre class="wp-block-code"><code>$ kubectl set image deploy nginx nginx=nginx:latest --record

deployment.apps/nginx image updated

$
</code></pre>

Undo the image update and rollback the deployment to use the <code>nginx:1.9</code> image:

<pre class="wp-block-code"><code>
$ kubectl rollout undo deploy nginx
deployment.apps/nginx rolled back

$
</code></pre>


As an additional exercise, try creating a pod that binds to a persistent volume claim and create the necessary Kubernetes objects.

RX-M can provide more help with preparing for the CKAD exam in one of our CKAD bootcamps; we offer open enrollments and private engagements for teams or organizations.

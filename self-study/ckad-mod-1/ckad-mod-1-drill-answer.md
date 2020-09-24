<!-- CKAD Self-Study Mod 1 -->

Create the <code>my-sa</code> ServiceAccount:

<pre class="wp-block-code"><code>
$ kubectl create serviceaccount my-sa

serviceaccount/my-sa created

$
</code></pre>

Create a pod that runs the <code>nginx</code> image with the <code>my-sa</code> ServiceAccount:

<pre class="wp-block-code"><code>
$ kubectl run ckad-basic-pod --generator=run-pod/v1 --image=nginx --serviceaccount=my-sa

pod/ckad-basic-pod created

$
</code></pre>


As an additional exercise, try creating a configMap and/or secret and using it in a pod as a volume or environment variable.

RX-M can provide more help with preparing for the CKAD exam in one of our CKAD bootcamps; we offer open enrollments and private engagements for teams or organizations.

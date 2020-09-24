<!-- CKAD Self-Study Mod 4 -->

Create the deployment with 3 replicas using the <code>nginx</code> image:

<pre class="wp-block-code"><code>
$ kubectl create deployment nginx --image=nginx:latest --replicas=3
deployment.apps/nginx created

$
</code></pre>

Create a NodePort type Service exposing the deployment outside the cluster on port 80 of the Service:

<pre class="wp-block-code"><code>
$ kubectl expose deploy nginx --type=NodePort --port=80

service/nginx exposed

$ kubectl get services

NAME           TYPE         CLUSTER-IP     EXTERNAL-IP   PORT(S)           AGE
kubernetes   ClusterIP    10.96.0.1            &lt;none&gt              443/TCP            22d
nginx            NodePort    10.99.178.77      &lt;none&gt              80:32550/TCP   23s

$ wget -qO - 10.99.178.77:80 | awk NR==4

&lt;title&gt;Welcome to nginx!&lt;/title&gt

$
</code></pre>


As an additional exercise, try creating an nginx pod, a busybox pod, and a deny-all network policy then create an ingress network policy that uses a label from the busybox pod to allow traffic to the nginx pod

RX-M can provide more help with preparing for the CKAD exam in one of our CKAD bootcamps; we offer open enrollments and private engagements for teams or organizations.

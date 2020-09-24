<!-- CKAD Self-Study Mod 2 -->

Create the yaml manifest to use as a template for the <code>ckad-side</code> pod using the <code>-o yaml --dry-run=client</code> options:

<pre class="wp-block-code"><code>
$ kubectl run ckad-sidecar --restart=Never --image=busybox:latest -o yaml --dry-run=client > ckad-sidecar.yaml --command -- /bin/sh -c "sleep 15 && wget -qO - http://ckad-sidecar | awk NR==4 && tail -f /dev/null"

$
</code></pre>
Edit <code>ckad-sidecar.yaml</code>, rename the <code>busybox</code> container appropriately, and add a <code>nginx</code> container.

<pre class="wp-block-code"><code>
$ nano ckad-sidecar.yaml && cat ckad-sidecar.yaml
apiVersion: v1
kind: Pod
metadata:
  name: ckad-sidecar
spec:
  containers:
  - name: nginx
    image: nginx:latest
  - name: busybox
    image: busybox:latest
    command:
    - /bin/sh
    - -c
    - sleep 15 && wget -qO - http://ckad-sidecar | awk NR==4 && tail -f /dev/null
  restartPolicy: OnFailure

$ kubectl apply -f ckad-sidecar.yaml

pod/ckad-sidecar created

$ kubectl logs ckad-sidecar -c busybox

&lt;title&gt;Welcome to nginx!&lt;/title&gt;

$
</code></pre>


As an additional exercise, try creating a pod with an Apache HTTP container (httpd) with a liveness probe that uses httpGet.

RX-M can provide more help with preparing for the CKAD exam in one of our CKAD bootcamps; we offer open enrollments and private engagements for teams or organizations.

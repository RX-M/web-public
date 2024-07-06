<!-- CKA Self-Study Mod 4 -->

First, create a pod spec imperatively using <code>kubectl run</code>

<pre class="wp-block-code"><code>
$ kubectl run --restart Never --image docker.io/centos/httpd --dry-run -o yaml mod4drillpod  > mod4drillpod.yaml

$
</code></pre>


Then, add a <code>hostPath</code> entry under the <code>volumes</code> array of the pod spec:

<pre class="wp-block-code"><code>
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: mod4drillpod
  name: mod4drillpod
spec:
  containers:
  - image: docker.io/centos/httpd
    name: mod4drillpod
  volumes:
  - name: apache-logs
    hostPath:
      path: /tmp/httpd/
</code></pre>


Now add a <code>volumeMounts</code> array to the container spec that mounts the <code>hostPath</code> volume to <code>/var/log/httpd</code>:

<pre class="wp-block-code"><code>
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: mod4drillpod
  name: mod4drillpod
spec:
  containers:
  - image: docker.io/centos/httpd
    name: mod4drillpod
    volumeMounts:
    - name: apache-logs
      mountPath: /var/log/httpd
  volumes:
  - name: apache-logs
    hostPath:
      path: /tmp/httpd/
</code></pre>

Now create the pod:

<pre class="wp-block-code"><code>
$ kubectl apply -f mod4drillpod.yaml

pod/mod4drillpod created

$
</code></pre>

And use <code>ls</code> to confirm that the volume mounted successfully:

<pre class="wp-block-code"><code>
$ ls -l /tmp/httpd/

total 4
-rw-r--r-- 1 root root   0 Feb 27 10:44 access_log
-rw-r--r-- 1 root root 767 Feb 27 10:44 error_log

$
</code></pre>


As an additional exercise, create a storage class that uses the local storage class plugin and a PVC that uses that storage class.

RX-M can provide more help with preparing for the CKA exam in one of our CKA bootcamps; we offer open enrollments and private engagements for teams or organizations.

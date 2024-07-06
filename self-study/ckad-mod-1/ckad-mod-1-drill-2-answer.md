<!-- CKAD Self-Study Mod 1 -->

If you built the image in Docker and run containerd as your container runtime, run the following commands to transfer the image:

<pre class="wp-block-code"><code>sudo docker save docker.io/self-study/webserver:v1 -o chlg-img.tar
sudo ctr -n k8s.io image import chlg-img.tar
</code></pre>

Create the pod:

<pre class="wp-block-code"><code>$ nano self-study-pod-1.yaml && cat $_

apiVersion: v1
kind: Pod
metadata:
  labels:
    run: self-study-pod-1
  name: self-study-pod-1
spec:
  containers:
  - image: self-study/webserver:v1
    name: primary
    volumeMounts:
    - name: logs
      mountPath: /var/log/httpd
  - image: docker.io/fluent/fluent-bit:1.9.2
    name: logger
    volumeMounts:
    - name: logs
      mountPath: /httpd
    command:
    - /fluent-bit/bin/fluent-bit
    - -i
    - tail
    - -p
    - path=/httpd/access_log
    - -o
    - stdout
  volumes:
  - name: logs
    emptyDir: {}

$ kubectl apply -f self-study-pod-1.yaml

pod/self-study-pod-1 created

$
</code></pre>

To test it, curl the IP of the <code>self-study-pod-1</code> pod and retrieve the logs from the its <code>logger</code> container:

<pre class="wp-block-code"><code>
$ kubectl get pods self-study-pod-1 -o wide

NAME               READY   STATUS    RESTARTS   AGE    IP          NODE              NOMINATED NODE   READINESS GATES
self-study-pod-1   2/2     Running   0          9m3s   10.0.0.92   ip-172-31-21-39   <none>           <none>

$ curl 10.0.0.92

welcome

$ kubectl logs self-study-pod-1 logger

Fluent Bit v1.9.2
* Copyright (C) 2015-2022 The Fluent Bit Authors
* Fluent Bit is a CNCF sub-project under the umbrella of Fluentd
* https://fluentbit.io

[2024/07/06 00:01:32] [ info] [fluent bit] version=1.9.2, commit=27a63c11d3, pid=1
[2024/07/06 00:01:32] [ info] [storage] version=1.1.6, type=memory-only, sync=normal, checksum=disabled, max_chunks_up=128
[2024/07/06 00:01:32] [ info] [cmetrics] version=0.3.0
[2024/07/06 00:01:32] [ info] [sp] stream processor started
[2024/07/06 00:01:32] [ info] [output:stdout:stdout.0] worker #0 started
[2024/07/06 00:10:31] [ info] [input:tail:tail.0] inotify_fs_add(): inode=2368357 watch_fd=1 name=/httpd/access_log
[0] tail.0: [1720224641.451816104, {"log"=>"10.0.0.246 - - [06/Jul/2024:00:10:41 +0000] "GET / HTTP/1.1" 403 8 "-" "curl/8.5.0""}]

$
</code></pre>

RX-M can provide more help with preparing for the CKAD exam in one of our CKAD bootcamps; we offer open enrollments and private engagements for teams or organizations.

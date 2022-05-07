<!-- CKAD Self-Study Mod 1 -->

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
  - image: fluent/fluent-bit:1.9.2
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

NAME               READY   STATUS    RESTARTS   AGE    IP          NODE               NOMINATED NODE   READINESS GATES
self-study-pod-1   2/2     Running   0          4m1s   10.32.0.4   ip-172-31-57-184   <none>           <none>

$ curl 10.32.0.4

welcome

$ kubectl logs self-study-pod-1 logger
Fluent Bit v1.9.2
* Copyright (C) 2015-2022 The Fluent Bit Authors
* Fluent Bit is a CNCF sub-project under the umbrella of Fluentd
* https://fluentbit.io

[2022/04/19 00:14:32] [ info] [fluent bit] version=1.9.2, commit=27a63c11d3, pid=1
[2022/04/19 00:14:32] [ info] [storage] version=1.1.6, type=memory-only, sync=normal, checksum=disabled, max_chunks_up=128
[2022/04/19 00:14:32] [ info] [cmetrics] version=0.3.0
[2022/04/19 00:14:32] [ info] [sp] stream processor started
[2022/04/19 00:14:32] [ info] [output:stdout:stdout.0] worker #0 started
[2022/04/19 00:14:32] [ info] [input:tail:tail.0] inotify_fs_add(): inode=777359 watch_fd=1 name=/httpd/access_log
[0] tail.0: [1650327514.381999019, {"log"=>"10.32.0.1 - - [19/Apr/2022:00:18:34 +0000] "GET / HTTP/1.1" 403 8 "-" "curl/7.68.0""}]

$
</code></pre>

RX-M can provide more help with preparing for the CKAD exam in one of our CKAD bootcamps; we offer open enrollments and private engagements for teams or organizations.

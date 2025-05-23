<!-- CKAD Self-Study Mod 4 -->

First, run the command to create the pod:

<pre class="wp-block-code"><code>
$ kubectl run --image nginx nginx-drill

pod/nginx-drill created
</code></pre>

Then, use <code>kubectl expose</code> with the <code>--type NodePort</code> flag to create a nodePort service imperatively. Make sure to expose the pod since that is the was created by the initial run command:

<pre class="wp-block-code"><code>
$ kubectl expose --type NodePort --port 80 pod nginx-drill

service/nginx-drill exposed
</code></pre>

After exposing the pod, list the services. You will see the nginx-drill NodePort service maps port 80 to a port within the 30000 range:

<pre class="wp-block-code"><code>
$ kubectl get svc

NAME               TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
kubernetes         ClusterIP   10.96.0.1        <none>        443/TCP        55m
nginx-drill        NodePort    10.106.33.204    <none>        80:32072/TCP   53s
</code></pre>

Finally, try to send a curl request to the nginx pod using your machine IP address:

<pre class="wp-block-code"><code>$ ip a s | head

1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host noprefixroute 
       valid_lft forever preferred_lft forever
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:85:82:8f brd ff:ff:ff:ff:ff:ff
    inet 10.0.2.15/24 metric 100 brd 10.0.2.255 scope global dynamic enp0s3
       valid_lft 84885sec preferred_lft 84885sec

$ curl 10.0.2.15:32072

&lt;!DOCTYPE html&gt;
&lt;html&gt;
&lt;head&gt;
&lt;title&gt;Welcome to nginx!&lt;/title&gt;
&lt;style&gt;
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
&lt;/style&gt;
&lt;/head&gt;
&lt;body&gt;
&lt;h1&gt;Welcome to nginx!&lt;/h1&gt;
&lt;p&gt;If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.&lt;/p&gt;

&lt;p&gt;For online documentation and support please refer to
&lt;a href="http://nginx.org/"&gt;nginx.org&lt;/a&gt;.&lt;br/&gt;
Commercial support is available at
&lt;a href="http://nginx.com/"&gt;nginx.com&lt;/a&gt;.&lt;/p&gt;

&lt;p&gt;&lt;em&gt;Thank you for using nginx.&lt;/em&gt;&lt;/p&gt;
&lt;/body&gt;
&lt;/html&gt;
</code></pre>


As another exercise, create a ClusterIP service called <code>other-svc</code> using <code>kubectl create</code> and use a label selector to associate it with the nginx-drill deployment created above.
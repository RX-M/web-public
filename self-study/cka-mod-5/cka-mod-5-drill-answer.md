<!-- CKA Self-Study Mod 5 -->

First, run the command:

<pre class="wp-block-code"><code>
$ kubectl run --restart Never --image redis:0.2 redispod

pod/redispod created
$
</code></pre>

Kubernetes reports that it successfully created the object. Now list the events in the current namespace with <code>kubectl get events</code>:

<pre class="wp-block-code"><code>
$ kubectl get events | grep redispod

<unknown>   Normal    Scheduled   pod/redispod   Successfully assigned default/redispod to ubuntu
3s          Normal    Pulling     pod/redispod   Pulling image "redis:0.2"
2s          Warning   Failed      pod/redispod   Failed to pull image "redis:0.2": rpc error: code = Unknown desc = Error response from daemon: manifest for redis:0.2 not found: manifest unknown: manifest unknown
2s          Warning   Failed      pod/redispod   Error: ErrImagePull
16s         Normal    BackOff     pod/redispod   Back-off pulling image "redis:0.2"
16s         Warning   Failed      pod/redispod   Error: ImagePullBackOff

$
</code></pre>

We see that the node’s container runtime could not find an image redis:0.2. As instructed, write these events to a file:

<pre class="wp-block-code"><code>
$ kubectl get events | grep redispod > /tmp/troubleshooting-answer.txt

$ cat /tmp/troubleshooting-answer.txt

<unknown>   Normal    Scheduled   pod/redispod   Successfully assigned default/redispod to ubuntu
58s         Normal    Pulling     pod/redispod   Pulling image "redis:0.2"
57s         Warning   Failed      pod/redispod   Failed to pull image "redis:0.2": rpc error: code = Unknown desc = Error response from daemon: manifest for redis:0.2 not found: manifest unknown: manifest unknown
57s         Warning   Failed      pod/redispod   Error: ErrImagePull
45s         Normal    BackOff     pod/redispod   Back-off pulling image "redis:0.2"
33s         Warning   Failed      pod/redispod   Error: ImagePullBackOff

$
</code></pre>


As an additional exercise, fix the <code>redispod</code> above using the <code>redis:latest</code> image.

RX-M can provide more help with preparing for the CKA exam in one of our CKA bootcamps; we offer open enrollments and private engagements for teams or organizations.

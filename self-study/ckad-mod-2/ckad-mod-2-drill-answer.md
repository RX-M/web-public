<!-- CKAD Self-Study Mod 2 -->

Create a deployment named <code>cicd</code> with the <code>jenkins/jenkins:lts</code> image:
<pre class="wp-block-code"><code>
$ kubectl create deploy cicd --image jenkins/jenkins:lts

deployment.apps/cicd created

$ kubectl scale deploy cicd --replicas=5

deployment.apps/cicd scaled

$
</code></pre>

Or

<pre class="wp-block-code"><code>
$ kubectl create deploy cicd --image jenkins/jenkins:lts -o yaml --dry-run > cicddep.yaml

$ nano cicddep.yaml
</code></pre>

Change the <code>replicas</code> key’s value to 5

<pre class="wp-block-code"><code>

$ kubectl apply -f cicddep.yaml

$
</code></pre>


As an additional exercise, try to change the command in the pods that the cicd deployment creates to output the jenkins logs to a file.

RX-M can provide more help with preparing for the CKAD exam in one of our CKAD bootcamps; we offer open enrollments and private engagements for teams or organizations.

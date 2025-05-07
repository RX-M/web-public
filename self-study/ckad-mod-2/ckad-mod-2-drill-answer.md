<!-- CKAD Self-Study Mod 2 -->

<pre class="wp-block-code"><code>
$ kubectl create deploy cicd --image jenkins/jenkins:lts

deployment.apps/cicd created

$ kubectl scale deploy cicd --replicas=5

deployment.apps/cicd scaled
</code></pre>

Or

<pre class="wp-block-code"><code>
$ kubectl create deploy cicd --image jenkins/jenkins:lts -o yaml --dry-run=client > cicddep.yaml

$ nano cicddep.yaml
</code></pre>

Change the <code>replicas</code> key’s value to 5 and apply it.

<pre class="wp-block-code"><code>

$ kubectl apply -f cicddep.yaml
</code></pre>

As an additional exercise, try to change the command in the pods that the cicd deployment creates to output the jenkins logs to a file.
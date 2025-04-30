<!-- CKA Self-Study Mod 1 -->

Start by imperatively creating the role:

<pre class="wp-block-code"><code>
$ kubectl create role --verb get,list --resource ingresses,networkpolicy webdrillrole

role.rbac.authorization.k8s.io/webdrillrole created

$
</code></pre>

Then create a rolebinding that binds the <code>webdrillrolle</code> to the <code>networker</code> user:

<pre class="wp-block-code"><code>
$ kubectl create rolebinding --user networker --role webdrillrole webdrillrolebinding

rolebinding.rbac.authorization.k8s.io/webdrillrolebinding created

$
</code></pre>

You can confirm this by using <code>kubectl auth can-i --as</code> and see if the networker user can perform get or list on ingress and network policy:

<pre class="wp-block-code"><code>
$ kubectl auth can-i get ingress --as networker

yes

$ kubectl auth can-i get ingress --as networker -n kube-system

no

$
</code></pre>


As another exercise, try to create another role that can create network policies and bind the role to the same <code>networker</code>user.

RX-M can provide more help with preparing for the CKA exam in one of our CKA bootcamps or test prep sessions; we offer open enrollments and private engagements for teams or organizations.
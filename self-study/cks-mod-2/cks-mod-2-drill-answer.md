<!-- CKAD Self-Study Mod 2 -->

Create a new namespace in your cluster named <code>secure-ns</code>

<pre class="wp-block-code"><code>
$ kubectl create ns secure-ns

namespace/secure-ns created
</code></pre>

<li>In the <code>secure-ns</code> namespace, create a service account named <code>app-api-ops</code></li>

<pre class="wp-block-code"><code>
$ kubectl create sa app-api-ops -n secure-ns

serviceaccount/app-api-ops created

$ kubectl get sa -n secure-ns

NAME          SECRETS   AGE
app-api-ops   1         10s
default       1         88s
</code></pre>

<ul>
<li>Create the RBAC resources (naming them <code>app-api-rbac</code>) needed to grant the <code>app-api-ops</code> service account under the <code>secure-ns</code> namespace to do the following:</li>
  <ul>
  <li><code>get</code> and <code>list</code> all pods in the <code>secure-ns</code> namespace only</li>
  <li><code>create</code>, <code>update</code>, <code>get</code>, and <code>list</code> deployments in the <code>seucred-ns</code> namespace only</li>
  <li><code>get</code> and <code>list</code> all configmaps and secrets in the <code>secure-ns</code> namespace only</li>
  </ul>
</ul>

Create the role imperatively to minimize yaml editing:

<pre class="wp-block-code"><code>
$ kubectl create role app-api-rbac -n secure-ns \
--resource pods,deployments,configmaps,secrets \
--verb get,list,create,update \
-o yaml --dry-run=client > app-api-rbac-role.yaml
</code></pre>

Remove the <code>create</code> and <code>update</code> verbs from the <code>""</code> apiGroup rules and apply the role:

<pre class="wp-block-code"><code>
$ nano app-api-rbac-role.yaml && cat $_

apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: app-api-rbac
  namespace: secure-ns
rules:
- apiGroups:
  - ""
  resources:
  - pods
  - configmaps
  - secrets
  verbs:
  - get
  - list
- apiGroups:
  - apps
  resources:
  - deployments
  verbs:
  - get
  - list
  - create
  - update

$ kubectl apply -f app-api-rbac-role.yaml

role.rbac.authorization.k8s.io/app-api-rbac created
</code></pre>

Create the rolebinding:

<pre class="wp-block-code"><code>
$ kubectl create -n secure-ns rolebinding app-api-rbac \
--serviceaccount secure-ns:app-api-ops --role app-api-rbac

rolebinding.rbac.authorization.k8s.io/app-api-rbac created
</code></pre>

You can check the role and rolebinding using <code>kubectl auth can-i</code>:

<pre class="wp-block-code"><code>
$ kubectl auth can-i get pods --as system:serviceaccount:secure-ns:app-api-ops

no

$ kubectl auth can-i get pods --as system:serviceaccount:secure-ns:app-api-ops --namespace secure-ns

yes

$ kubectl auth can-i update deployments --as system:serviceaccount:secure-ns:app-api-ops --namespace secure-ns

yes
</code></pre>
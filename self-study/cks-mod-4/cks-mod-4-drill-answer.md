<!-- CKAD Self-Study Mod 4 -->

# Practice Drill

<li>Create a generic secret named <code>postgres</code> with the key-value pairs <code>pgu=blogdb</code> and <code>pgk=helloEmpanada</code></li>

<pre class="wp-block-code"><code>
~$ kubectl create secret generic postgres --from-literal pgu=blogdb --from-literal pgk=helloEmpanada -o yaml

apiVersion: v1
data:
  pgk: aGVsbG9FbXBhbmFkYQ==
  pgu: YmxvZ2Ri
kind: Secret
metadata:
  creationTimestamp: "2021-09-21T17:13:33Z"
  name: postgres
  namespace: default
  resourceVersion: "2146"
  uid: f84e78fa-b00f-420d-8c68-7a9e4db0b49c
type: Opaque

~$
</code></pre>

<li>Configure a pod named <code>postgres-db</code> that runs <code>postgres:latest</code> that uses the secret to furnish the <code>POSTGRES_PASSWORD</code> environment variable with the <code>pgk</code> key's value</li>

Add an <code>nev</code> section to the pod spec defining the <code>POSTGRES_PASSWORD</code> environment variable to use the <code>valueFrom</code> a <code>secretKeyRef</code> for the <code>postgres</code> secret's <code>pgk</code> key:

<pre class="wp-block-code"><code>
~$ kubectl run postgres-db --dry-run=client -o yaml --image postgres:latest > postgres-db.yaml
</code></pre>

<pre class="wp-block-code"><code>
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: postgres-db
  name: postgres-db
spec:
  containers:
  - image: postgres:latest
    name: postgres-db
    env:
    - name: POSTGRES_PASSWORD
      valueFrom:
        secretKeyRef:
          name: postgres
          key: pgk
</code></pre>

Apply the pod spec:

<pre class="wp-block-code"><code>
~$ kubectl apply -f postgres-db.yaml

pod/postgres-db created

~$
</code></pre>

Once the pod is up, you can use <code>env<code> to check if the environment variable was successfully declared:

<pre class="wp-block-code"></code>
~$ kubectl exec postgres-db -- env

PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/lib/postgresql/13/bin
HOSTNAME=postgres-db
POSTGRES_PASSWORD=helloEmpanada

..

~$
</code></pre>

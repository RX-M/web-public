<!-- CKS Self-Study Mod 6 -->

Create a deployment named <code>secure-webserver</code> that:

<ul>
<li>Deploys 3 pods</li>
<li>Each pod should have an empty directory volume named <code>bootstrap</code></li>
<li>Each pod should have two containers:</li>
  <ul>
  <li>One thats run the <code>rxmllc/trash-levels:1.1</code> image named webserver that mounts the <code>bootstrap</code> volume at <code>/bootstrap</code></li>
  <li>An init container that runs <code>alpine:3.13.6</code> which mounts the <code>bootstrap</code> volume at <code>/trash-levels/boot</code></li>
    <ul>
    <li>The init container should run the command: <code>/bin/sh -c "touch /trash-levels/boot/bootfile"</code></li>
    </ul>
  </ul>
<li>Container filesystems should be immutable where possible</li>
</ul>

You can optionally create most of the specification using <code>kubectl create</code>:

<pre class="wp-block-code"><code>
~$ kubectl create deploy secure-webserver --replicas=3 --image=rxmllc/trash-levels:1.1,alpine:3.13.6 -o yaml --dry-run=client

apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: secure-webserver
  name: secure-webserver
spec:
  replicas: 3
  selector:
    matchLabels:
      app: secure-webserver
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: secure-webserver
    spec:
      containers:
      - image: rxmllc/trash-levels:1.1
        name: trash-levels
        resources: {}
      - image: alpine:3.13.6
        name: alpine
        resources: {}
status: {}
</code></pre>

Make the following changes:

<li>Remove any empty <code>resources</code> and <code>creationTimestamp</code> keys</li>
<li>Add an empty directory named <code>bootstrap</code> under <code>template.spec.volumes</code></li>
<li>Mount the <code>bootstrap</code> volume to the <code>trash-levels</code> container at <code>/bootstrap</code></li>
<li>For the trash-levels container only, set the <code>bootstrap</code> volume to <code>readOnly</code> <code>true</code></li>
<li>Place the <code>alpine</code> container under the <code>initContainers</code> key</li>
<li>Mount the <code>bootstrap</code> volume to the <code>alpine</code> container at <code>/trash-levels/boot</code></li>
  <ul>
  <li>Do not set <code>readOnly</code> or set <code>readOnly: false</code></li>
  </ul>
<li>For the alpine container only, set the <code>command</code> to <code>/bin/sh -c "touch /trash-levels/boot/bootfile"</code></li>
<li>Set <code>securityContext.readOnlyRootFilesystem</code> to <code>true</code> on both containers</li>
</ul>

<pre class="wp-block-code"><code>
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: secure-webserver
  name: secure-webserver
spec:
  replicas: 3
  selector:
    matchLabels:
      app: secure-webserver
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: secure-webserver
    spec:
      volumes:
      - name: bootstrap
        emptyDir: {}
      containers:
      - image: rxmllc/trash-levels:1.1
        name: trash-levels
        volumeMounts:
        - name: bootstrap
          mountPath: /bootstrap
          readOnly: true
        securityContext:
          readOnlyRootFilesystem: true
      initContainers:
      - image: alpine:3.13.6
        name: alpine
        volumeMounts:
        - name: bootstrap
          mountPath: /trash-levels/boot
        command:
        - /bin/sh
        - -c
        - "touch /trash-levels/boot/bootfile"
        securityContext:
          readOnlyRootFilesystem: true
</code></pre>

Remember that volumes must be secured separately from the root filesystem, so do not forget to set <code>readOnly</code> on them separately if your application does not need to write to them!

RX-M can provide more help with preparing for the CKS exam in one of our CKS bootcamps; we offer open enrollments and private engagements for teams or organizations.

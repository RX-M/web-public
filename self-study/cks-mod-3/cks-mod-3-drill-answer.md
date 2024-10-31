<!-- CKAD Self-Study Mod 3 -->

# Practice Drill

<ul>
<li>Create a pod named <code>seccomp-drill</code> that uses the following SecComp profile:</li>
  <ul>
  <li>Name the seccomp profile <code>auditing.json</code></li>
  <li>The pod should use the <code>httpd:latest</code> image</li>

<pre class="wp-block-code"><code>
{
    "defaultAction": "SCMP_ACT_ERRNO"
}
</code></pre></ul>
</ul>

<pre class="wp-block-code"><code>
{
    "defaultAction": "SCMP_ACT_ERRNO"
}
</code></pre>

Create <code>/var/lib/kubelet/seccomp/profiles</code> and the <code>auditing.json</code> file under it:

<pre class="wp-block-code"><code>
~$ sudo mkdir -p /var/lib/kubelet/seccomp/profiles

~$ sudo nano /var/lib/kubelet/seccomp/profiles/auditing.json
</code></pre>

Then create the pod:

<pre class="wp-block-code"><code>
apiVersion: v1
kind: Pod
metadata:
  name: seccomp-drill
  labels:
    app: seccomp-drill
spec:
  securityContext:
    seccompProfile:
      type: Localhost
      localhostProfile: profiles/auditing.json
  containers:
  - name: seccomp-drill
    image: httpd:latest
</code></pre>

This seccomp profile prevents all syscalls, so the pod will not start, which is good for an untrusted workload.

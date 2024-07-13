<!-- CKAD Self-Study Mod 5 -->

After you apply the pod, use <code>kubectl get pods</code> to view the status. You will see that the pod is not ready, and has a status of Init:ImagePullBackOff. Note that the "Init:" prefix implies that this is a failure with the init container not one of the normal containers in the pod.

<pre class="wp-block-code"><code>$ kubectl get pods debug-pod1

NAME         READY   STATUS              RESTARTS   AGE
debug-pod1   0/1     Init:ErrImagePull   0          17s

$
</code></pre>

Run <code>kubectl describe pod debug-pod1</code> and look at the events. You will see that the Kubelet was unable to pull the specified image using the container manager:

<pre class="wp-block-code"><code>$ kubectl describe pods debug-pod1

Name:             debug-pod1
Namespace:        default
Priority:         0
Service Account:  default
Node:             labsys/10.0.2.15
Start Time:       Sat, 13 Jul 2024 01:16:48 +0000
Labels:           app=debug-pod1
Annotations:      <none>
Status:           Pending
IP:               10.0.0.92

...

Events:
  Type     Reason     Age                From               Message
  ----     ------     ----               ----               -------
  Normal   Scheduled  54s                default-scheduler  Successfully assigned default/debug-pod1 to labsys
  Normal   BackOff    25s (x2 over 52s)  kubelet            Back-off pulling image "alpinelatest"
  Warning  Failed     25s (x2 over 52s)  kubelet            Error: ImagePullBackOff
  Normal   Pulling    11s (x3 over 53s)  kubelet            Pulling image "alpinelatest"
  Warning  Failed     10s (x3 over 53s)  kubelet            Failed to pull image "alpinelatest": failed to pull and unpack image "docker.io/library/alpinelatest:latest": failed to resolve reference "docker.io/library/alpinelatest:latest": pull access denied, repository does not exist or may require authorization: server message: insufficient_scope: authorization failed
  Warning  Failed     10s (x3 over 53s)  kubelet            Error: ErrImagePull

$
</code></pre>

Take a close look at the init container image name. If you check Docker Hub you will see that no such image exists. The init  container image name is incorrect. Change the image name to <code>alpine</code> or <code>alpine:latest</code> to correct the issue.

<pre class="wp-block-code"><code>$ nano pod-debug-1.yaml ; cat $_

apiVersion: v1
kind: Pod
metadata:
  name: debug-pod1
  labels:
    app: debug-pod1
spec:
  initContainers:
  - name: init-container
    image: alpine:latest

    command: ["/bin/sh", "-c"]
    args: ["echo hello"]
  containers:
  - name: myapp-container
    image: busybox
    command: ["/bin/sh", "-c"]
    args: ["tail -f /dev/null"]

$
</code></pre>

You can only edit some of the fields in running pods, and image is not one of them. To apply the fix you will need to delete the pod and recreate it. If this pod were created by a deployment or another controller, updating the pod spec in the parent resource would perform the update for you.

<pre class="wp-block-code"><code>$ kubectl delete -f pod-debug-1.yaml

pod "debug-pod1" deleted

$ kubectl apply -f pod-debug-1.yaml

pod/debug-pod1 created

$
</code></pre>

Check the pod status with <code>kubectl get pods</code> and you should see the pod running:

<pre class="wp-block-code"><code>$ kubectl get pods debug-pod1

NAME         READY   STATUS    RESTARTS   AGE
debug-pod1   1/1     Running   0          36s

$
</code></pre>

You can display the pod logs with <code>kubectl logs</code> to verify the log output from the init container.

You can visit RX-M's own <strong><a href="https://kubernetes.io/docs/tasks/debug-application-cluster/">bust-a-kube</a></strong> repository to get additional practice while you prepare.

RX-M can provide more help with preparing for the CKAD exam in one of our CKAD bootcamps; we offer open enrollments and private engagements for teams or organizations.

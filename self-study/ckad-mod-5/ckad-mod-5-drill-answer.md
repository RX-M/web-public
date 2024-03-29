<!-- CKAD Self-Study Mod 5 -->

After you apply the pod, use <code>kubectl get pods</code> to view the status. You will see that the pod is not ready, and has a status of Init:ImagePullBackOff. Note that the "Init:" prefix implies that this is a failure with the init container not one of the normal containers in the pod.

<pre class="wp-block-code"><code>$ kubectl get pods

NAME         READY   STATUS                  RESTARTS   AGE
debug-pod1   0/1     Init:ImagePullBackOff   0          43s

$
</code></pre>

Run <code>kubectl describe pod debug-pod1</code> and look at the events. You will see that the Kubelet was unable to pull the specified image using the container manager:

<pre class="wp-block-code"><code>$ kubectl describe pods debug-pod1

Name:         debug-pod1
Namespace:    default
Priority:     0
Node:         sept-b/192.168.229.155
Start Time:   Thu, 10 Sep 2020 10:36:05 -0700
Labels:       app=debug-pod1
Annotations:  <none>
Status:       Pending

...

Events:
  Type     Reason     Age                From               Message
  ----     ------     ----               ----               -------
  Normal   Scheduled  83s                default-scheduler  Successfully assigned default/debug-pod1 to sept-b
  Normal   Pulling    42s (x3 over 82s)  kubelet, sept-b    Pulling image "alpinelatest"
  Warning  Failed     40s (x3 over 80s)  kubelet, sept-b    Failed to pull image "alpinelatest": rpc error: code = Unknown desc = Error response from daemon: pull access denied for alpinelatest, repository does not exist or may require 'docker login': denied: requested access to the resource is denied
  Warning  Failed     40s (x3 over 80s)  kubelet, sept-b    Error: ErrImagePull
  Normal   BackOff    12s (x4 over 79s)  kubelet, sept-b    Back-off pulling image "alpinelatest"
  Warning  Failed     12s (x4 over 79s)  kubelet, sept-b    Error: ImagePullBackOff

$
</code></pre>

Take a close look at the init container image name. If you check Docker Hub you will see that no such image exists. The init  container image name is incorrect. Change the image name to <code>alpine</code> or <code>alpine:latest</code> to correct the issue.

<pre class="wp-block-code"><code>$ nano pod-debug-1.yaml && cat pod-debug-1.yaml

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

RX-M can provide more help with preparing for the CKAD exam in one of our CKAD bootcamps; we offer open enrollments and private engagements for teams or organizations.

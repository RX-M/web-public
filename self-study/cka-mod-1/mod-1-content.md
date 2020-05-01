<!-- CKA Self-Study Mod 1 -->

## Label Selectors and Scheduling

Labels are key/value pairs attached to Kubernetes objects. Labels allow users to organize and select multiple objects bearing the same labels. Many Kubernetes resources, like services or networkPolicies for example, use labels to determine what pods receive their functionality.

The Kubernetes scheduler uses labels through nodeSelectors and node/pod affinity/anti-affinity. Pod nodeSelectors determine which nodes will run certain workloads based on labels on nodes. To do this, nodes need to be labeled:

<pre class="wp-block-code"><code>
$ kubectl label node knode1 disk=fast
</code></pre>

Then, in a pod template or spec, a <code>nodeSelector</code> key tells the scheduler to assign the pod(s) to node(s) that bear the desired label and value:

<pre class="wp-block-code"><code>
apiVersion: v1
kind: Pod
metadata:
  name: cka-label-selector-pod
  labels:
    name: cka-label-selector-pod
spec:
  nodeSelector:
    disk: fast
  containers:
  - name: cka-label-selector-container
    Image: nginx
</code></pre>

Using this spec the Kubernetes scheduler will only assign the pod to a node bearing the disk=fast label.

Learn more about labels [here](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/#label-selectors).


## DaemonSets

A DaemonSet is a controller that ensures a single copy of a pod runs on every node in the cluster. They are popular for running workloads tied to a host--like storage, logging, or monitoring agents in a Kuberentes cluster. Due to the permissions required to run DaemonSets, Kubernetes platform administrators and operators are the primary DaemonSet consumers.

DaemonSets are very similar to deployments and use a nearly identical spec: Both use pod templates to declare how they run their workloads, with the primary difference being the lack of a <code>replicas</code> key, since DaemonSets only run one pod on all nodes in the cluster (that they tolerate). An example DaemonSet:

<pre class="wp-block-code"><code>
apiVersion: apps/v1
kind: DaemonSet
metadata:
  labels:
    app: fluentd-logger
  name: fluentd-logger
spec:
  selector:
    matchLabels:
      app: fluentd-logger
  template:
    metadata:
      labels:
        app: fluentd-logger
    spec:
      containers:
      - image: fluent/fluentd
        name: fluentd
        volumeMounts:
        - name: varlog
          mountPath: /var/log
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      tolerations:
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
</code></pre>

The spec above runs a Fluentd log aggregation container that mounts the host’s <code>/var/log</code> directory in order to forward system logs--a common use case for the DaemonSet!

Learn more about Daemonsets [here](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/).


## Container Resource Requests and Limits

Resource requests and limits are set on a per-container basis within a pod. By specifying a resource request we tell the Kubernetes scheduler the minimum amount of each resource (CPU and memory) a container will need. By specifying limits, we set up cgroup constraints on the node where the process runs. An example of setting requests/limits looks like:

<pre class="wp-block-code"><code>
apiVersion: v1
kind: Pod
metadata:
  name: cka-resource-pod
spec:
  containers:
  - name: cka-resource-container
    image: some-app-img:v4.1
    resources:
      limits:
        cpu: "1"
        memory: “1Gi”
      requests:
        cpu: "0.5"
        memory: “500Mi”
</code></pre>

Containers that exceed their resource limits are liable to be evicted or restarted by their managing kubelet. Conversely, schedulers will not assign pods to kubelets that do not report enough resources to fulfill pod resource requests.

Learn more about pod resource requests/limits [here](https://kubernetes.io/docs/tasks/configure-pod-container/assign-cpu-resource/)


## Multiple Schedulers


Kubernetes clusters can have multiple schedulers running at the same time. In most cases each scheduler runs as a deployment in the cluster itself, and must possess the proper RBAC permissions to operate as a scheduler in the cluster.

In clusters with multiple schedulers, Pod manifests use the <code>spec.schedulerName</code> key to define which scheduler in a cluster will assign the pod to a kubelet.

<pre class="wp-block-code"><code>
apiVersion: v1
kind: Pod
metadata:
  name: cka-multischeduler-example-pod
  labels:
    name: cka-multischeduler-example-pod
spec:
  schedulerName: default-scheduler
  containers:
  - name: nginx
    image: nginx
</code></pre>

Learn more about deploying and configuring Multiple schedulers in your cluster [here](https://kubernetes.io/docs/tasks/administer-cluster/configure-multiple-schedulers/).


## Manually Scheduling a Pod

There are several ways to influence pod scheduling directly from the pod manifest. One example covered earlier is the <code>nodeSelector</code> that you place within a pod spec or template to select nodes based on a label.

One way to “schedule” pods manually is by placing a pod manifest inside a kubelet’s manifest directory. Any valid pod manifests placed inside this directory are created and managed by the kubelet directly.

<pre class="wp-block-code"><code>
$ sudo cat /var/lib/kubelet/config.yaml | grep -i staticPodPath

staticPodPath: /etc/kubernetes/manifests

$ ls -l /etc/kubernetes/manifests/

total 16
-rw------- 1 root root 1764 Feb 18 10:57 kubelet-assigned-pod.yaml

$
</code></pre>

Another option is <code>nodeName</code>, in which you name a specific node directly in the pod spec or template. This option tells the scheduler to place the pod only on that node:

<pre class="wp-block-code"><code>
apiVersion: v1
kind: Pod
metadata:
  name: cka-scheduled-pod
spec:
  containers:
  - name: cka-scheduled-pod
    image: redis
  nodeName: master-1
</code></pre>

The spec above tells the scheduler to place this pod on a node called master-1. Pod will run as long as there is a ready node in the cluster with that name.

Learn more about how you can manually schedule pods [here](https://kubernetes.io/docs/concepts/configuration/assign-pod-node/) and [here](https://kubernetes.io/blog/2017/03/advanced-scheduling-in-kubernetes/)


## Displaying Scheduler Events

In Kubernetes clusters bootstrapped with kubeadm, the scheduler runs as a pod in the kube-system namespace. Events associated with the scheduler are retrievable using <code>kubectl get events</code> or <code>kubectl describe</code> on your cluster’s kube-scheduler pod. The kube-scheduler pod outputs its logs to standard out, so you can use <code>kubectl logs</code> on it like any other pod:

<pre class="wp-block-code"><code>
$ kubectl logs  -n kube-system kube-scheduler
</code></pre>

Any cluster-related events generated by the kube-scheduler are also viewable using <code>kubectl get events</code>:

<pre class="wp-block-code"><code>
$ kubectl get events -o wide -A --sort-by lastTimestamp | grep scheduler

7m21s        1       kube-scheduler.15f6761d8d3f60d5
kube-system   7m21s       Normal    LeaderElection            endpoints/kube-scheduler                                                        default-scheduler         ubuntu_ae3b2864-fa00-4e84-ad4f-2a24aa388868 became leader                                                                                            7m21s        1       kube-scheduler.15f6761d8d3f3d73

$
</code></pre>

See an example of how to confirm scheduling behavior using events [here](https://kubernetes.io/docs/tasks/administer-cluster/configure-multiple-schedulers/#verifying-that-the-pods-were-scheduled-using-the-desired-schedulers).


## Configuring Schedulers

Schedulers select nodes to run pods using a 2-step process: filtering and scoring. These steps are tunable by users by adjusting the <code>predicates</code> and <code>priorities</code> lists. In the past, these were done using scheduler config files and later a configmap:

<pre class="wp-block-code"><code>
{
"kind" : "Policy",
"apiVersion" : "v1",
"predicates" : [
	{"name" : "PodFitsHostPorts", "order": 1},
	{"name" : "PodFitsResources", "order": 3},
	{"name" : "NoDiskConflict",  "order": 4},
	{"name" : "NoVolumeZoneConflict",  "order": 6},
	{"name" : "MatchNodeSelector",  "order": 5},
	{"name" : "HostName",  "order": 2}
	],
"priorities" : [
	{"name" : "LeastRequestedPriority", "weight" : 1},
	{"name" : "BalancedResourceAllocation", "weight" : 1},
	{"name" : "ServiceSpreadingPriority", "weight" : 1},
	{"name" : "EqualPriority", "weight" : 1}
	]
}
</code></pre>

The Kubernetes scheduler makes use of predicates to identify nodes to which a pod may be scheduled. The scheduler then uses policies to rank the nodes that pass the predicate tests. To use this policy, pass the <code>--use-legacy-policy-config</code> and <code>--policy-config-file</code> flags to the kube-scheduler in its manifest. This method of configuring schedulers was deprecated starting in version 1.13, but still works in version 1.17.

The latest versions of Kubernetes are moving toward modifying the default kube-scheduler with plugins created around the [scheduling framework](https://kubernetes.io/docs/concepts/configuration/scheduling-framework/).

Learn more about configuring scheduler priorities and predicates [here](https://kubernetes.io/docs/concepts/scheduling/kube-scheduler/#kube-scheduler-implementation)


## Monitoring Cluster Components

The Kubernetes core components are the API Server, Controller Manager, Scheduler, and etcd. In kubeadm-bootstrapped clusters, the core components run as pods in the kube-system namespace.

Cluster components like the scheduler or controller manager create events in the cluster to report their activities and any warnings or errors:

<pre class="wp-block-code"><code>
$ kubectl get events -o wide

LAST SEEN   TYPE      REASON              OBJECT                               SUBOBJECT                     SOURCE                  MESSAGE                                                                                                                                              FIRST SEEN   COUNT   NAME
12m         Normal    SuccessfulCreate    replicaset/nginx-prod-66df7bf77f                                   replicaset-controller   Created pod: nginx-prod-66df7bf77f-7vwqg                                                                                                             12m          1       nginx-prod-66df7bf77f.15f6b09fe23c1520
11m         Normal    SuccessfulCreate    replicaset/nginx-prod-66df7bf77f                                   replicaset-controller   Created pod: nginx-prod-66df7bf77f-fmtq2                                                                                                             11m          1       nginx-prod-66df7bf77f.15f6b0a62d896e27
11m         Normal    SuccessfulCreate    replicaset/nginx-prod-66df7bf77f                                   replicaset-controller   Created pod: nginx-prod-66df7bf77f-4xvs5                                                                                                             11m          1       nginx-prod-66df7bf77f.15f6b0a62ed46213
11m         Normal    SuccessfulCreate    replicaset/nginx-prod-66df7bf77f                                   replicaset-controller   Created pod: nginx-prod-66df7bf77f-qlg9n                                                                                                             11m          1       nginx-prod-66df7bf77f.15f6b0a62ef90d27
11m         Normal    SuccessfulCreate    replicaset/nginx-prod-66df7bf77f                                   replicaset-controller   Created pod: nginx-prod-66df7bf77f-ck2mk                                                                                                             11m          1       nginx-prod-66df7bf77f.15f6b0a63270f7e2
12m         Normal    ScalingReplicaSet   deployment/nginx-prod                                              deployment-controller   Scaled up replica set nginx-prod-66df7bf77f to 1                                                                                                     12m          1       nginx-prod.15f6b09fe1a3bdf8
11m         Normal    ScalingReplicaSet   deployment/nginx-prod                                              deployment-controller   Scaled up replica set nginx-prod-66df7bf77f to 5                                                                                                     11m          1       nginx-prod.15f6b0a62cd478c4

$
</code></pre>

If you have a metrics provider like the [Kubernetes metrics server](https://github.com/kubernetes-sigs/metrics-server), you can view the resource use reported by the kubelets on each pod with <code>kubectl top pods</code>:

<pre class="wp-block-code"><code>
$ kubectl top pods -n kube-system

NAME                             CPU(cores)   MEMORY(bytes)   
coredns-6955765f44-dlw95         2m           8Mi             
coredns-6955765f44-kj4fw         2m           15Mi            
etcd-ubuntu                      10m          68Mi            
kube-apiserver-ubuntu            22m          237Mi           
kube-controller-manager-ubuntu   8m           38Mi            
kube-proxy-gx2rc                 1m           16Mi            
kube-scheduler-ubuntu            2m           15Mi            
metrics-server-c84799697-2mtr5   1m           12Mi            
weave-net-6r2ds                  1m           52Mi            

$
</code></pre>

Learn more about the tools used to monitor cluster components [here](https://kubernetes.io/docs/tasks/debug-application-cluster/resource-usage-monitoring/) and to learn about the cluster logging architecture look [here](https://kubernetes.io/docs/concepts/cluster-administration/logging/#cluster-level-logging-architectures).


## Managing Cluster Component Logs

Use commands like <code>kubectl logs</code> and <code>kubectl get events</code> in the <code>kube-system</code> namespace to view cluster component logs like any other pod in the cluster:

<pre class="wp-block-code"><code>
$ kubectl logs -n kube-system kube-apiserver-ubuntu

I0224 22:09:55.658818       1 controller.go:127] OpenAPI AggregationController: action for item v1beta1.metrics.k8s.io: Rate Limited Requeue.
I0224 22:10:16.978806       1 controller.go:107] OpenAPI AggregationController: Processing item v1beta1.metrics.k8s.io
I0224 22:11:17.927354       1 controller.go:107] OpenAPI AggregationController: Processing item v1beta1.metrics.k8s.io
I0224 22:11:35.982245       1 controller.go:606] quota admission added evaluator for: deployments.apps
I0224 22:11:35.984983       1 controller.go:606] quota admission added evaluator for: replicasets.apps

$
</code></pre>

Logs from cluster components running as systemd services outside the cluster, like the Kubelet, are viewable through host-based commands like <code>journalctl</code> or <code>systemctl status</code>:

<pre class="wp-block-code"><code>
$ sudo systemctl status kubelet

● kubelet.service - kubelet: The Kubernetes Node Agent
   Loaded: loaded (/lib/systemd/system/kubelet.service; enabled; vendor preset: enabled)
  Drop-In: /etc/systemd/system/kubelet.service.d
           └─10-kubeadm.conf
   Active: active (running) since Mon 2020-02-24 14:09:14 PST; 18h ago
     Docs: https://kubernetes.io/docs/home/
 Main PID: 1161 (kubelet)
    Tasks: 19
   Memory: 140.0M
      CPU: 6min 49.221s
   CGroup: /system.slice/kubelet.service
           └─1161 /usr/bin/kubelet --bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf --config=/var/lib/kubelet/config.yaml --cgroup-driver=cgroupfs --network-plugin=cni --pod-infra-container-image=k8s.gcr.io/pause:3.1 --rotate-server-certificates

$
</code></pre>

Learn more about cluster component logging and the initial steps to troubleshooting [here](https://kubernetes.io/docs/tasks/debug-application-cluster/debug-cluster/#looking-at-logs)



## Monitoring Applications

Kubernetes exposes several ways to monitor applications at the cluster level. You can view events within the context of applications by using <code>kubectl describe</code> on an application's pods or deployment:

<pre class="wp-block-code"><code>
$ kubectl describe pod/nginx-prod-66df7bf77f-qlg9n

Name:         nginx-prod-66df7bf77f-qlg9n
Namespace:    default
Priority:     0
Node:         ubuntu/192.168.229.134
Start Time:   Tue, 25 Feb 2020 08:02:26 -0800
Labels:       app=nginx-prod
              pod-template-hash=66df7bf77f
Annotations:  <none>
Status:       Running
IP:           10.32.0.10
IPs:
  IP:           10.32.0.10
Controlled By:  ReplicaSet/nginx-prod-66df7bf77f
Containers:
  nginx:
    Container ID:   docker://7672f6b790cea9c277d7de5d019f23a2d803effe02f9efe1b665889b0e9aefb9
    Image:          nginx
    Image ID:       docker-pullable://nginx@sha256:ad5552c786f128e389a0263104ae39f3d3c7895579d45ae716f528185b36bc6f
    Port:           <none>
    Host Port:      <none>
    State:          Running
      Started:      Tue, 25 Feb 2020 08:02:34 -0800
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-27w2z (ro)
Conditions:
  Type              Status
  Initialized       True
  Ready             True
  ContainersReady   True
  PodScheduled      True
Volumes:
  default-token-27w2z:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-27w2z
    Optional:    false
QoS Class:       BestEffort
Node-Selectors:  <none>
Tolerations:     node.kubernetes.io/not-ready:NoExecute for 300s
                 node.kubernetes.io/unreachable:NoExecute for 300s
Events:
  Type    Reason     Age        From               Message
  ----    ------     ----       ----               -------
  Normal  Scheduled  <unknown>  default-scheduler  Successfully assigned default/nginx-prod-66df7bf77f-qlg9n to ubuntu
  Normal  Pulling    20m        kubelet, ubuntu    Pulling image "nginx"
  Normal  Pulled     20m        kubelet, ubuntu    Successfully pulled image "nginx"
  Normal  Created    20m        kubelet, ubuntu    Created container nginx
  Normal  Started    20m        kubelet, ubuntu    Started container nginx

$
</code></pre>

A variety of probes are also available to use to tell Kubernetes whether an application is up (liveness), ready for traffic (readiness), or still starting (startup). Probes are set up under a pod spec and test if an application is alive, ready, or still starting by running a script or sending a request to a web endpoint. A basic probe that checks for an application’ looks like this:

<pre class="wp-block-code"><code>
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: nginx-prod
    spec:
      containers:
      - image: nginx
        imagePullPolicy: Always
        name: nginx
        readinessProbe:
          failureThreshold: 1
          httpGet:
            path: /
            port: 80
            scheme: HTTP
          periodSeconds: 10
          successThreshold: 1
          timeoutSeconds: 1
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
</code></pre>

Learn more about gaining insight on your applications through and other troubleshooting techniques [here](https://kubernetes.io/docs/tasks/debug-application-cluster/)


## Managing Application Logs

Kubernetes retrieves application logs through the container runtime’s logging engine. Many container runtimes publish logs to stdout by default, so application logs are readily available using <code>kubectl logs</code>:

<pre class="wp-block-code"><code>
$ kubectl logs nginx-prod-7894c755fc-cmwm7

10.32.0.1 - - [25/Feb/2020:16:44:26 +0000] "GET / HTTP/1.1" 200 612 "-" "kube-probe/1.17" "-"
10.32.0.1 - - [25/Feb/2020:16:44:36 +0000] "GET / HTTP/1.1" 200 612 "-" "kube-probe/1.17" "-"

$
</code></pre>

The applications themselves must send their logs to STDOUT too. If an application sends its logs to files within the container’s filesystem, then another option is to use <code>kubectl exec</code> and use the container’s built-in shell (if it has one) to read the file:

<pre class="wp-block-code"><code>
$ kubectl exec nginx-prod-6d6bbd4db4-vl7w4 -c nginx -- cat /tmp/nginx.log

10.32.0.1 - - [25/Feb/2020:17:13:34 +0000] "GET / HTTP/1.1" 200 612 "-" "kube-probe/1.17" "-"
10.32.0.1 - - [25/Feb/2020:17:13:44 +0000] "GET / HTTP/1.1" 200 612 "-" "kube-probe/1.17" "-"
10.32.0.1 - - [25/Feb/2020:17:13:54 +0000] "GET / HTTP/1.1" 200 612 "-" "kube-probe/1.17" "-"
10.32.0.1 - - [25/Feb/2020:17:14:04 +0000] "GET / HTTP/1.1" 200 612 "-" "kube-probe/1.17" "-"
10.32.0.1 - - [25/Feb/2020:17:14:14 +0000] "GET / HTTP/1.1" 200 612 "-" "kube-probe/1.17" "-"

$
</code></pre>

[Learn more about application logging strategies in Kubernetes](https://kubernetes.io/docs/concepts/cluster-administration/logging/)


## Practice Drill

Create a Daemonset that runs Fluent Bit.
The Fluent Bit container spec should resemble this:

<pre class="wp-block-code"><code>
spec:
  containers:
  - command:
    - /fluent-bit/bin/fluent-bit
    - -i
    - cpu
    - -o
    - stdout
    image: fluent/fluent-bit
    name: fluent-bit
</code></pre>

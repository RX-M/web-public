# CKS Self-Study Mod 3


# Minimize host OS footprint

This topic pertains to ensuring that the Kubernetes cluster does not tie itself too closely to its host infrastructure, exposing it to threats that specifically target the machines (virtual or otherwise). The pattern most susceptible to host OS-based attacks is the use of the <code>hostPath</code> volume. HostPath volumes expose host-level directories to the containers running within pods. This has two threats:

<ul>
<li>Any bad actors that gain access to the host machine will be able to access directories used by applications within the containers</li>
<li>Containers deployed by bad actors can affect host machines by writing potentially malicious or damaging data into the mounted directories.</li>
</ul>

The primary way to mitigate threats posed by hostPaths is to mount them as readOnly on the container side and readable only by the kubelet user on the host machine. This ensures that only the directories managed by the kubelet are writable and that containers being deployed on the machines cannot affect the host.

An example of a hostPath mounted as read-only in a container can be found on the manifest in the <code>kube-apiserver</code> in a cluster deployed using the <code>kubeadm</code> tool:

<pre class="wp-block-code"><code>
    volumeMounts:
    - mountPath: /etc/ssl/certs
      name: ca-certs
      readOnly: true
    - mountPath: /etc/ca-certificates
      name: etc-ca-certificates
      readOnly: true
    - mountPath: /etc/pki
      name: etc-pki
      readOnly: true
    - mountPath: /etc/kubernetes/pki
      name: k8s-certs
      readOnly: true
    - mountPath: /usr/local/share/ca-certificates
      name: usr-local-share-ca-certificates
      readOnly: true
    - mountPath: /usr/share/ca-certificates
      name: usr-share-ca-certificates
      readOnly: true
</code></pre>

HostPaths are only one aspect of securing your workloads on Kubernetes and are one aspect of the pod security standard. The pod security standard provides a series of recommendations that categorize pods into three categories: <code>privileged</code>, <code>baseline</code>, and <code>restricted</code>. Some of the methods that directly pertain to this topic include:

<ul>
<li>Disabling the ability of pods to share namespaces with the host, such as the <code>network</code>, <code>PID</code>, and <code>IPC</code> namespaces</li>
<li>Disallowing various root capabilities within the root context, preventing users within pods from making certain privileged calls to the host filesystem</li>
<li>Preventing a container from using the <code>/proc</code> filesystem from the host</li>
</ul>

Additional pod security standard settings can be found on the [Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/) page.


# Use least-privilege identity and access management

This topic refers to managing access control on the infrastructure level, specifically for Kubernetes clusters with components that interact with the infrastructure. These include:

<ul>
<li>Storage classes that use CNI plugins to dynamically provision storage media
<li>KMS systems integrated with the API Server to provide better secret handling</li>
<li>Hosted Kubernetes clusters that use the provider's IAM system to grant in-cluster permissions</li>
</ul>

The primary concept to understand in this topic is the principle of least privilege. Least privileges describe the bare minimum privileges a subject needs to perform its functions. The privileges required by any given component will differ
based on the system it integrates with and the provider enforcing the IAM roles. You will find the recommended permissions on the documentation for such components.

Below is the example IAM role required for the AWS EBS CSI driver that allows the Kubernetes persistent volume subsystem to dynamically create EBS block storage devices for applications:

<pre class="wp-block-code"><code>
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:CreateSnapshot",
        "ec2:AttachVolume",
        "ec2:DetachVolume",
        "ec2:ModifyVolume",
        "ec2:DescribeAvailabilityZones",
        "ec2:DescribeInstances",
        "ec2:DescribeSnapshots",
        "ec2:DescribeTags",
        "ec2:DescribeVolumes",
        "ec2:DescribeVolumesModifications"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "ec2:CreateTags"
      ],
      "Resource": [
        "arn:aws:ec2:*:*:volume/*",
        "arn:aws:ec2:*:*:snapshot/*"
      ],
      "Condition": {
        "StringEquals": {
          "ec2:CreateAction": [
            "CreateVolume",
            "CreateSnapshot"
          ]
        }
      }
    },
    {
      "Effect": "Allow",
      "Action": [
        "ec2:DeleteTags"
      ],
      "Resource": [
        "arn:aws:ec2:*:*:volume/*",
        "arn:aws:ec2:*:*:snapshot/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "ec2:CreateVolume"
      ],
      "Resource": "*",
      "Condition": {
        "StringLike": {
          "aws:RequestTag/ebs.csi.aws.com/cluster": "true"
        }
      }
    },
    {
      "Effect": "Allow",
      "Action": [
        "ec2:CreateVolume"
      ],
      "Resource": "*",
      "Condition": {
        "StringLike": {
          "aws:RequestTag/CSIVolumeName": "*"
        }
      }
    },
    {
      "Effect": "Allow",
      "Action": [
        "ec2:CreateVolume"
      ],
      "Resource": "*",
      "Condition": {
        "StringLike": {
          "aws:RequestTag/kubernetes.io/cluster/*": "owned"
        }
      }
    },
    {
      "Effect": "Allow",
      "Action": [
        "ec2:DeleteVolume"
      ],
      "Resource": "*",
      "Condition": {
        "StringLike": {
          "ec2:ResourceTag/ebs.csi.aws.com/cluster": "true"
        }
      }
    },
    {
      "Effect": "Allow",
      "Action": [
        "ec2:DeleteVolume"
      ],
      "Resource": "*",
      "Condition": {
        "StringLike": {
          "ec2:ResourceTag/CSIVolumeName": "*"
        }
      }
    },
    {
      "Effect": "Allow",
      "Action": [
        "ec2:DeleteVolume"
      ],
      "Resource": "*",
      "Condition": {
        "StringLike": {
          "ec2:ResourceTag/kubernetes.io/cluster/*": "owned"
        }
      }
    },
    {
      "Effect": "Allow",
      "Action": [
        "ec2:DeleteSnapshot"
      ],
      "Resource": "*",
      "Condition": {
        "StringLike": {
          "ec2:ResourceTag/CSIVolumeSnapshotName": "*"
        }
      }
    },
    {
      "Effect": "Allow",
      "Action": [
        "ec2:DeleteSnapshot"
      ],
      "Resource": "*",
      "Condition": {
        "StringLike": {
          "ec2:ResourceTag/ebs.csi.aws.com/cluster": "true"
        }
      }
    }
  ]
}
</code></pre>

Note that the principle of least privilege also applies to the Kubernetes role based access control (RBAC) system.

For more information on the principle of least privilege, check out the AWS documentation on the principle of least privilege [here](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html#grant-least-privilege). For a concrete example of controlling IAM for your cluster, the Kubernetes KOPS provisioning tool gives a [good breakdown of using IAM rules and granting permissions within specified boundaries](https://github.com/kubernetes/kops/blob/master/docs/iam_roles.md).


# Minimize external access to the network

The infrastructure that runs the Kubernetes cluster presents its own set of attack vectors, namely access to the machines. Common machine attacks that occur after network infiltration include:

- Deploying payloads that "steal" system resources, like CryptoMiners
- Exfiltrating sensitive data like credentials or proprietary data
- Sabotaging the infrastructure by deploying a malicious payload that prevents the system from functioning normally

Minimizing external access to the network can help mitigate these threats. One of the universal changes is modifying firewalls to restrict traffic to only the communication channels necessary for applications and the Kubernetes cluster to work.

These communication channels include:

- SSH (Port 22)
- DNS (Port 53)
- The infrastructure's VM CIDR (enabling VMs in the same network to communicate)
- The expected pod network CIDRs (usually 10.0.0.0/8 or 192.168.0.0/16)
- The ports necessary for cluster components to communicate

Opening such rules requires some knowledge of the firewall configuration software. Implementing some of the rules above using <code>iptables</code> involves:

<pre class="wp-block-code"><code>
~$ sudo iptables -A INPUT -p tcp -m tcp -m multiport --dports 22,53 -j ACCEPT

~$ sudo iptables -A INPUT -p udp -m udp -m multiport --dports 53 -j ACCEPT

~$ sudo iptables -A INPUT -s 10.0.0.0/8 -j ACCEPT

~$ sudo iptables -A INPUT -j DROP
</code></pre>

All other traffic should then be blocked by default, which is what the last command above does.

Here is an example of a firewall blocking incoming connections for a Kubernetes cluster using Linux <code>iptables</code>:

<pre class="wp-block-code"><code>
~$ sudo iptables -L INPUT --line-numbers -n

Chain INPUT (policy ACCEPT)
num  target     prot opt source               destination
1    KUBE-NODEPORTS  all  --  0.0.0.0/0            0.0.0.0/0            /* Kubernetes health check service ports */
2    KUBE-EXTERNAL-SERVICES  all  --  0.0.0.0/0            0.0.0.0/0            ctstate NEW /* Kubernetes externally-visible service portals */
3    KUBE-FIREWALL  all  --  0.0.0.0/0            0.0.0.0/0
4    ACCEPT     tcp  --  0.0.0.0/0            0.0.0.0/0            tcp multiport dports 22,53
5    ACCEPT     udp  --  0.0.0.0/0            0.0.0.0/0            udp multiport dports 53
6    ACCEPT     all  --  10.0.0.0/8           0.0.0.0/0
7    ACCEPT     all  --  10.0.0.0/8           0.0.0.0/0
8    ACCEPT     tcp  --  172.31.0.0/16        0.0.0.0/0
9    ACCEPT     all  --  0.0.0.0/0            0.0.0.0/0
10   ACCEPT     tcp  --  0.0.0.0/0            0.0.0.0/0            tcp multiport dports 80,443
11   ACCEPT     tcp  --  0.0.0.0/0            0.0.0.0/0            tcp multiport dports 6443,10250
12   ACCEPT     all  --  0.0.0.0/0            0.0.0.0/0            ctstate RELATED,ESTABLISHED
13   DROP       all  --  0.0.0.0/0            0.0.0.0/0
</code></pre>

It is important to consider the possibility of data exfiltration, so it is reasonable to establish firewall rules that only allow outbound traffic (egress) for those same communication channels:

<pre class="wp-block-code"><code>
~$ sudo iptables -A OUTPUT -p tcp -m tcp -m multiport --dports 22,53 -j ACCEPT

~$ sudo iptables -A OUTPUT -p udp -m udp -m multiport --dports 53 -j ACCEPT

~$ sudo iptables -A OUTPUT -p tcp -m tcp -m multiport --dports 6443,10250 -j ACCEPT

~$ sudo iptables -A OUTPUT -d 10.0.0.0/8 -j ACCEPT

~$ sudo iptables -A OUTPUT -d 172.31.0.0/16 -j ACCEPT

~$ sudo iptables -A OUTPUT -o lo -j ACCEPT

~$ sudo iptables -A OUTPUT -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT

~$ sudo iptables -A OUTPUT -p tcp -m tcp -m multiport --dports 443 -j ACCEPT

~$ sudo iptables -A OUTPUT -j DROP
</code></pre>

To find more information on securing a Kubernetes cluster, see the Kubernetes documentation page on [securing a Kubernetes cluster](https://kubernetes.io/docs/tasks/administer-cluster/securing-a-cluster/).


# Appropriately use kernel hardening tools such as AppArmor, seccomp

Container-based technologies rely on the ability to access resources on the host machine and make system calls (syscalls) to the host kernel. There are two technologies that can help improve a system's security stance: AppArmor and Seccomp profiles. Kubernetes supports the use of AppArmor profiles and seccomp capabilities as defined within the Pod security context.


## AppArmor

AppArmor is a system-level module that must be enabled for workloads to take advantage of them. Many Linux distributions deployed on cloud infrastructure will have their own set of AppArmor modules. Container runtimes also have a default profile that is installed when the runtime is installed:

<pre class="wp-block-code"><code>
$ cat /sys/module/apparmor/parameters/enabled

Y

$ sudo aa-status

apparmor module is loaded.
36 profiles are loaded.
34 profiles are in enforce mode.
   /snap/snapd/14066/usr/lib/snapd/snap-confine
   /snap/snapd/14066/usr/lib/snapd/snap-confine//mount-namespace-capture-helper
   /snap/snapd/15177/usr/lib/snapd/snap-confine
   /snap/snapd/15177/usr/lib/snapd/snap-confine//mount-namespace-capture-helper
   /snap/snapd/15534/usr/lib/snapd/snap-confine
   /snap/snapd/15534/usr/lib/snapd/snap-confine//mount-namespace-capture-helper
   /usr/bin/man
   /usr/lib/NetworkManager/nm-dhcp-client.action
   /usr/lib/NetworkManager/nm-dhcp-helper
   /usr/lib/connman/scripts/dhclient-script
   /usr/lib/snapd/snap-confine
   /usr/lib/snapd/snap-confine//mount-namespace-capture-helper
   /usr/sbin/tcpdump
   /{,usr/}sbin/dhclient
   docker-default
   lsb_release
   man_filter
   man_groff
   nvidia_modprobe
   nvidia_modprobe//kmod
   snap-update-ns.amazon-ssm-agent
   snap-update-ns.lxd
   snap.lxd.activate
   snap.lxd.benchmark
   snap.lxd.buginfo
   snap.lxd.check-kernel
   snap.lxd.daemon
   snap.lxd.hook.configure
   snap.lxd.hook.install
   snap.lxd.hook.remove
   snap.lxd.lxc
   snap.lxd.lxc-to-lxd
   snap.lxd.lxd
   snap.lxd.migrate
2 profiles are in complain mode.
   snap.amazon-ssm-agent.amazon-ssm-agent
   snap.amazon-ssm-agent.ssm-cli
39 processes have profiles defined.
37 processes are in enforce mode.
   /pause (9551) docker-default
   /pause (9591) docker-default
   /pause (9651) docker-default
   /pause (9686) docker-default
   /usr/local/bin/kube-controller-manager (9760) docker-default
   /usr/local/bin/kube-scheduler (9797) docker-default
   /usr/local/bin/kube-apiserver (9855) docker-default
   /usr/local/bin/etcd (9857) docker-default
   /pause (11235) docker-default
   /pause (11245) docker-default
   /coredns (11424) docker-default
   /coredns (11432) docker-default
   /usr/local/apache2/bin/httpd (26335) docker-default
   /usr/local/apache2/bin/httpd (26366) docker-default
   /usr/local/apache2/bin/httpd (26367) docker-default
   /usr/local/apache2/bin/httpd (26369) docker-default
   /pause (428185) docker-default
   /usr/sbin/nginx (428296) docker-default
   /usr/sbin/nginx (428333) docker-default
   /usr/sbin/nginx (428334) docker-default
   /pause (430406) docker-default
   /pause (430438) docker-default
   /pause (430454) docker-default
   /usr/sbin/nginx (430673) docker-default
   /usr/sbin/nginx (430710) docker-default
   /usr/sbin/nginx (430711) docker-default
   /usr/sbin/nginx (430744) docker-default
   /usr/sbin/nginx (430784) docker-default
   /usr/sbin/nginx (430785) docker-default
   /usr/sbin/nginx (430826) docker-default
   /usr/sbin/nginx (430863) docker-default
   /usr/sbin/nginx (430864) docker-default
   /pause (692260) docker-default
   /usr/bin/dash (692398) docker-default
   /pause (693545) docker-default
   /metrics-server (693645) docker-default
   /usr/bin/sleep (780885) docker-default
2 processes are in complain mode.
   /snap/amazon-ssm-agent/5163/amazon-ssm-agent (40540) snap.amazon-ssm-agent.amazon-ssm-agent
   /snap/amazon-ssm-agent/5163/ssm-agent-worker (40775) snap.amazon-ssm-agent.amazon-ssm-agent
0 processes are unconfined but have a profile defined.
</code></pre>

AppArmor profiles can be specified at the pod level or the container level using the <code>securityContext</code> object in a pod spec by specifying the type of profile (and optionally the name of the profile):

<pre class="wp-block-code"><code>
  securityContext:
    appArmorProfile:
      type: &lt;profile&gt;     # Only required when the type is "Localhost"
</code></pre>

<code>&lt;profile></code> specifies which policy (1 of 3 forms):

<ul>
<li><code>RuntimeDefault</code> - use the runtime's default profile</li>
<li><code>Localhost</code>&nbsp;- use a profile loaded on the host<!-- wp:list -->
<ul class="wp-block-list"><!-- wp:list-item -->
<li><code>localhostProfile</code> - the name of a profile loaded on the node that should be used</li>
<!-- /wp:list-item --></ul>
<!-- /wp:list --><!-- wp:list -->
<ul class="wp-block-list"><!-- wp:list-item -->
<li><code>localhostProfile</code> - the name of a profile loaded on the node that should be used</li>
<!-- /wp:list-item --></ul>
<!-- /wp:list --></li>
<li><code>Unconfined</code> - no profile applied</li>
</ul>

The following spec describes a pod named <code>apparmor-pod</code> with a container in it named <code>apparmor-container</code>. To the <code>apparmor-container</code> container the runtime's default AppArmor profile found on the host will be applied:

<pre class="wp-block-code"><code>
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: apparmor-pod
  name: apparmor-pod
spec:
  containers:
  - command:
    - tail
    - -f
    - /dev/null
    image: ubuntu:24.04
    name: apparmor-container
    securityContext:
      appArmorProfile:
        type: RuntimeDefault
</code></pre>

For more information on applying AppArmor to your pods, see the [Kubernetes Documentation page on applying AppArmor profiles](https://kubernetes.io/docs/tutorials/clusters/apparmor/). Also, the [GitLab Apparmor documentation](https://gitlab.com/apparmor/apparmor/-/wikis/Documentation) can be referred to for additional information on using AppArmor in general.


## Seccomp

In addition to AppArmor, Seccomp profiles can be declared for pods to restrict the system calls that can be made by containers. Seccomp profiles are described in JSON objects. Most container runtimes have some kind of default seccomp profile, which is fairly permissive. The simplest seccomp profiles can be as short as the following example presented by the Kubernetes documentation:

<pre class="wp-block-code"><code>
{
    "defaultAction": "SCMP_ACT_LOG"
}
</code></pre>

This seccomp profile will simply log any syscall activity made by the container under <code>/var/log/syslog</code>. The seccomp profiles must be made available to the Kubelet by placing a seccomp profile within the Kubelet's directory. In a <code>kubeadm</code> cluster, that would be <code>/var/lib/kubelet/seccomp/profiles</code>:

<pre class="wp-block-code"><code>
$ sudo ls -l /var/lib/kubelet/seccomp/profiles

total 4
-rw-r--r-- 1 root root 40 Sep 22 09:41 audit.json
</code></pre>

In versions 1.19 and above, Seccomp profiles are implemented by describing a seccomp profile under the pod's <code>securityContext</code> spec:

<pre class="wp-block-code"><code>
apiVersion: v1
kind: Pod
metadata:
  name: seccomp-pod
  labels:
    app: seccomp-pod
spec:
  securityContext:
    seccompProfile:
      type: Localhost
      localhostProfile: profiles/audit.json
  containers:
  - name: seccomp-pod
    image: alpine:latest
    command:
    - top
</code></pre>

This pod will run an Alpine Linux container that runs the <code>top</code> monitoring tool repeatedly, which uses the example
seccomp profile mentioned above. The seccomp profile is being loaded from the local host, in this case, the Kubelet
directory. If a seccompProfile configuration is omitted, the container runtime will default to its own seccomp
profile if it has one enabled.

Once the pod is created, the policy can be checked for effect by viewing <code>/var/log/syslog</code>:

<pre class="wp-block-code"><code>
~$ tail /var/log/syslog

Sep 22 09:52:39 oct-b kernel: [37037.870465] audit: type=1326 audit(1632329559.168:1163): auid=4294967295 uid=0 gid=0 ses=4294967295 pid=103055 comm="top" exe="/bin/busybox" sig=0 arch=c000003e syscall=11 compat=0 ip=0x7f4d632c05a1 code=0x7ffc0000
Sep 22 09:52:39 oct-b kernel: [37037.870473] audit: type=1326 audit(1632329559.168:1164): auid=4294967295 uid=0 gid=0 ses=4294967295 pid=103055 comm="top" exe="/bin/busybox" sig=0 arch=c000003e syscall=16 compat=0 ip=0x7f4d632beb7d code=0x7ffc0000
Sep 22 09:52:39 oct-b kernel: [37037.870477] audit: type=1326 audit(1632329559.168:1165): auid=4294967295 uid=0 gid=0 ses=4294967295 pid=103055 comm="top" exe="/bin/busybox" sig=0 arch=c000003e syscall=2 compat=0 ip=0x7f4d632df3ad code=0x7ffc0000

</code></pre>

To see more information about implementing seccomp profiles on your containers, see [the Kubernetes Documentation on restricting syscalls with Seccomp](https://kubernetes.io/docs/tutorials/clusters/seccomp/).


## Practice Drill

<ul>
<li>Create a pod named <code>seccomp-drill</code> that uses the following SecComp profile:</li>
  <pre class="wp-block-code"><code>
  {
      "defaultAction": "SCMP_ACT_ERRNO"
  }
  </code></pre>
  <ul>
  <li>Name the seccomp profile <code>auditing.json</code></li>
  <li>The pod should use the <code>httpd:latest</code> image</li>
  </ul>
</ul>

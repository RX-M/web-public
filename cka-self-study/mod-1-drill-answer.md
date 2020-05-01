<!-- CKA Self-Study Mod 1 -->

Create a deployment spec to work from:

<pre class="wp-block-code"><code>
$ kubectl create --dry-run -o yaml deploy fluentbit-ds --image fluent/fluent-bit > fluentbit-ds.yaml
</code></pre>

Make the following changes to fluentbit-ds.yaml:

- Change the kind to Daemonset
- Remove the replicas and strategy keys
- Change the Fluent-bit container entry under spec.template.spec.containers to resemble the example -provided

Example:

<pre class="wp-block-code"><code>
apiVersion: apps/v1
kind: DaemonSet
metadata:
  labels:
    app: fluentbit-ds
  name: fluentbit-ds
spec:
  selector:
    matchLabels:
      app: fluentbit-ds
  template:
    metadata:
      labels:
        app: fluentbit-ds
    spec:
      containers:
      - image: fluent/fluent-bit
        name: fluent-bit
        command:
        - /fluent-bit/bin/fluent-bit
        - -i
        - cpu
        - -o
        - stdout
</code></pre>

Apply fluentbit-ds.yaml:

<pre class="wp-block-code"><code>
$ kubectl apply -f fluentbit-ds.yaml

daemonset.apps/fluentbit-ds created

$
</code></pre>

View the logs on the resulting pod:

<pre class="wp-block-code"><code>
$ kubectl logs fluentbit-ds-dpvdn

Fluent Bit v1.3.8
Copyright (C) Treasure Data

[2020/02/25 17:31:46] [ info] [storage] version=1.0.0, initializing...
[2020/02/25 17:31:46] [ info] [storage] in-memory
[2020/02/25 17:31:46] [ info] [storage] normal synchronization mode, checksum disabled, max_chunks_up=128
[2020/02/25 17:31:46] [ info] [engine] started (pid=1)
[2020/02/25 17:31:46] [ info] [sp] stream processor started
[0] cpu.0: [1582651907.000610077, {"cpu_p"=>1.000000, "user_p"=>0.000000, "system_p"=>1.000000, "cpu0.p_cpu"=>1.000000, "cpu0.p_user"=>1.000000, "cpu0.p_system"=>0.000000, "cpu1.p_cpu"=>2.000000, "cpu1.p_user"=>0.000000, "cpu1.p_system"=>2.000000}]
[1] cpu.0: [1582651908.000351847, {"cpu_p"=>23.500000, "user_p"=>15.500000, "system_p"=>8.000000, "cpu0.p_cpu"=>25.000000, "cpu0.p_user"=>17.000000, "cpu0.p_system"=>8.000000, "cpu1.p_cpu"=>20.000000, "cpu1.p_user"=>13.000000, "cpu1.p_system"=>7.000000}]
[2] cpu.0: [1582651909.000499973, {"cpu_p"=>21.500000, "user_p"=>16.500000, "system_p"=>5.000000, "cpu0.p_cpu"=>21.000000, "cpu0.p_user"=>16.000000, "cpu0.p_system"=>5.000000, "cpu1.p_cpu"=>23.000000, "cpu1.p_user"=>18.000000, "cpu1.p_system"=>5.000000}]
[3] cpu.0: [1582651910.000399424, {"cpu_p"=>9.500000, "user_p"=>6.000000, "system_p"=>3.500000, "cpu0.p_cpu"=>10.000000, "cpu0.p_user"=>6.000000, "cpu0.p_system"=>4.000000, "cpu1.p_cpu"=>10.000000, "cpu1.p_user"=>6.000000, "cpu1.p_system"=>4.000000}]
...

$
</code></pre>

As an additional exercise, try to label the fluent-bit daemonset with function=logger, then list the logs for the daemonset pod using that label.

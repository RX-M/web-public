<!-- CKAD Self-Study Mod 1 -->

<pre class="wp-block-code"><code>$ nano Dockerfile && cat $_

FROM centos/httpd
RUN /bin/sh -c "echo welcome" > /usr/share/httpd/noindex/index.html

$ docker build -t self-study/webserver:v1 .

Sending build context to Docker daemon  3.584kB
Step 1/2 : FROM centos/httpd
 ---> 2cc07fbb5000
Step 2/2 : RUN /bin/sh -c "echo welcome" > /usr/share/httpd/noindex/index.html
 ---> Running in 2addf467f100
Removing intermediate container 2addf467f100
 ---> 3c1011807b54
Successfully built 3c1011807b54
Successfully tagged self-study/webserver:v1

$
</code></pre>

<!-- CKAD Self-Study Mod 1 -->

<pre class="wp-block-code"><code>$ nano Dockerfile && cat $_

FROM centos/httpd
RUN /bin/sh -c "echo welcome" > /usr/share/httpd/noindex/index.html

$ docker build -t self-study/webserver:v1 .

[+] Building 10.9s (6/6) FINISHED                                                                                                                                       docker:default
 => [internal] load build definition from Dockerfile                                                                                                                              0.0s
 => => transferring dockerfile: 123B                                                                                                                                              0.0s
 => [internal] load metadata for docker.io/centos/httpd:latest                                                                                                                    1.0s
 => [internal] load .dockerignore                                                                                                                                                 0.0s
 => => transferring context: 2B                                                                                                                                                   0.0s
 => [1/2] FROM docker.io/centos/httpd:latest@sha256:26c6674463ff3b8529874b17f8bb55d21a0dcf86e025eafb3c9eeee15ee4f369                                                              8.3s
 => => resolve docker.io/centos/httpd:latest@sha256:26c6674463ff3b8529874b17f8bb55d21a0dcf86e025eafb3c9eeee15ee4f369                                                              0.0s
 => => sha256:20c0ca1c0cd55f78dc154784428f788c05e79011029108f3911e4e82aa9ff386 296B / 296B                                                                                        0.3s
 => => sha256:26c6674463ff3b8529874b17f8bb55d21a0dcf86e025eafb3c9eeee15ee4f369 1.16kB / 1.16kB                                                                                    0.0s
 => => sha256:2cc07fbb5000234e85b7ef63b6253f397491959af2a24251b6ae20c207beb814 4.10kB / 4.10kB                                                                                    0.0s
 => => sha256:a02a4930cb5d36f3290eb84f4bfa30668ef2e9fe3a1fb73ec015fc58b9958b17 75.17MB / 75.17MB                                                                                  0.9s
 => => sha256:628eaef4a9e0d47120551b449696b7600ea63e26d7db34ea368a88e9a6d3f6fe 32.93MB / 32.93MB                                                                                  0.6s
 => => sha256:30cf2fb1a57e65075ab3025d15317005b598a10f0444b63e02c57c113bff8fba 293B / 293B                                                                                        0.4s
 => => extracting sha256:a02a4930cb5d36f3290eb84f4bfa30668ef2e9fe3a1fb73ec015fc58b9958b17                                                                                         5.4s
 => => extracting sha256:628eaef4a9e0d47120551b449696b7600ea63e26d7db34ea368a88e9a6d3f6fe                                                                                         0.8s
 => => extracting sha256:20c0ca1c0cd55f78dc154784428f788c05e79011029108f3911e4e82aa9ff386                                                                                         0.0s
 => => extracting sha256:30cf2fb1a57e65075ab3025d15317005b598a10f0444b63e02c57c113bff8fba                                                                                         0.0s
 => [2/2] RUN /bin/sh -c "echo welcome" > /usr/share/httpd/noindex/index.html                                                                                                     1.4s
 => exporting to image                                                                                                                                                            0.0s
 => => exporting layers                                                                                                                                                           0.0s
 => => writing image sha256:0e3e267ad364b147d4aa6e6fe6bcdb3e78333aba8b03854a47d92f2d0ba40d70                                                                                      0.0s
 => => naming to docker.io/self-study/webserver:v1                                                                                                                                0.0s
$
</code></pre>

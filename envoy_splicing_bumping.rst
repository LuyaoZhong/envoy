How to Verify Envoy TLS Splicing/Bumping Functionalities
========================================================

Download and Build Envoy
------------------------
#. Clone the code to local host::

    $ git clone --branch tls-splicing-bumping-v1.21 https://github.com/LuyaoZhong/envoy.git

#. Build Envoy (Follow the official documentation)::

    $ bazel build --jobs 64 -c dbg --copt=-fno-limit-debug-info envoy

Prepare Test Environment
------------------------
#. Create new user “test” (assume there is already a default user “ubuntu”)::

    ubuntu@node1:~/envoy$ sudo useradd test -m -s /bin/bash -u 10000

#. Set up iptables rule, redirect the traffic from user “test” to envoy::

    ubuntu@node1:~/envoy$ sudo iptables -t nat -A OUTPUT -p tcp  -j REDIRECT --to-ports 1234 -m owner --uid-owner 10000

#. Add a CA to Ubuntu::

    ubuntu@node1:~/envoy$ sudo cp root-ca.pem /etc/ssl/certs/

   There are a root CA certificate and a root CA private key we have generated in advance under envoy directory, copy the .crt file into /etc/ssl/certs/. Envoy uses this CA cert/key to mimic server certificates, this makes the curl client trust the certs signed by the specified CA.

#. Start Envoy listening at port 1234::

    ubuntu@node1:~/envoy$ bazel-bin/source/exe/envoy-static -c envoy_splicing_bumping.yaml --concurrency 1 --log-level trace


Test TLS splicing and bumping
-----------------------------
#. TLS splicing without HTTP CONNECT::

    test@node1:~/envoy$ curl -v https://www.usbank.com/

   The traffic will be redirect to Envoy since we have iptables rule applied, Envoy works like a TCP proxy.

#. TLS splicing with HTTP CONNECT::

    ubuntu@node1:~/envoy$ curl -v -x 127.0.0.1:1234 https://www.usbank.com/

   “-x” specify the front proxy(Envoy) when accessing the website, Envoy handles the HTTP CONNECT first and let the traffic go through a TCP proxy without decryption.

#. TLS bumping without HTTP CONNECT::

    test@node1:~/envoy$ curl --v https://www.baidu.com/

   The traffic will be redirect to envoy since we have iptables rule applied, Envoy mimics the server certificate and does TLS handshake with downstream.

#. TLS bumping with HTTP CONNECT::

    ubuntu@node1:~/envoy$ curl -v -x 127.0.0.1:1234 https://www.baidu.com/

   “-x” specify the front proxy(Envoy) when accessing the website, Envoy handles the HTTP CONNECT first and mimics the server certificate afterwards.

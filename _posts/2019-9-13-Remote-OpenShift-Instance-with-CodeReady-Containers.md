---
layout: post
title: Running a Remote OpenShift Instance with CodeReady Containers
---
# Overview
This document pulls from multiple sources and shows how to run an OpenShift instance on a server with CodeReady Containers (crc) and configure a client machine to access it remotely. The size of the VM created by crc can be controlled so it's possible to create a persistent development instance of OpenShift that meets your needs.

The document also contains a few tips on debugging issues with the crc instance and adding insecure registries.

The instructions here were tested with Fedora on both the server (F30) and a latop (F29).

# Thanks to

Thanks to Marcel Wysocki from Red Hat for the haproxy solution and the entire CodeReady Containers team for crc!

# Useful links
[Red Hat blog article on CodeReady Containers](https://developers.redhat.com/blog/2019/09/05/red-hat-openshift-4-on-your-laptop-introducing-red-hat-codeready-containers/)

[Download page on cloud.redhat.com](https://cloud.redhat.com/openshift/install/crc/installer-provisioned?intcmp=7013a000002CtetAAC)

[CRC documentation on github.io](https://code-ready.github.io/crc/)

[Project sources on github](https://github.com/code-ready/crc)

# Download and setup CRC on a server

Go to the [download page]((https://cloud.redhat.com/openshift/install/crc/installer-provisioned?intcmp=7013a000002CtetAAC) and get crc for Linux. You’ll also need the pull secret listed there during the installation process.
Make sure to copy the *crc* binary to */usr/local/bin* or somewhere on your path.

The initial setup command only needs to be run once, and it creates a *~/.crc* directory:

```
$ crc setup
```
*Note: occasionally this may fail with “Failed to restart NetworkManager”. Just rerun crc setup a few times until it works*

# Create an OpenShift Instance with CRC

```
$ crc start
```
*Note: this is going to ask for the pull secret from the download page, paste it at the prompt.*

Optionally, use the -m and -c flags to increase the VM size, for example a 32GiB with 8 cpus:
 
```
$ crc start -m 32768 -c 8
```
See the [documentation](https://code-ready.github.io/crc/) or *crc -h* for other things you can do


If you want to just use crc locally on this machine, you can stop here, you’re all set!

# Make sure you have haproxy and a few other things

```
sudo dnf -y install haproxy policycoreutils-python-utils jq
```

# Modify the firewall on the server

```
$ sudo systemctl start firewalld
$ sudo firewall-cmd --add-port=80/tcp --permanent
$ sudo firewall-cmd --add-port=6443/tcp --permanent
$ sudo firewall-cmd --add-port=443/tcp --permanent
$ sudo systemctl restart firewalld
$ sudo semanage port -a -t http_port_t -p tcp 6443
```

# Configure haproxy on the server

```
$ sudo cp /etc/haproxy/haproxy.cfg /etc/haproxy/haproxy.cfg.bak
$ cat <<EOT > haproxy.cfg
global
        debug

defaults
        log global
        mode    http
        timeout connect 5000
        timeout client 5000
        timeout server 5000

frontend apps
    bind SERVER_IP:80
    bind SERVER_IP:443
    option tcplog
    mode tcp
    default_backend apps

backend apps
    mode tcp
    balance roundrobin
    option ssl-hello-chk
    server webserver1 CRC_IP:443 check

frontend api
    bind SERVER_IP:6443
    option tcplog
    mode tcp
    default_backend api

backend api
    mode tcp
    balance roundrobin
    option ssl-hello-chk
    server webserver1 CRC_IP:6443 check
EOT

$ export SERVER_IP=$(hostname --ip-address)
$ export CRC_IP=$(crc ip)
$ sed -i "s/SERVER_IP/$SERVER_IP/g" haproxy.cfg
$ sed -i "s/CRC_IP/$CRC_IP/g" haproxy.cfg
$ sudo cp haproxy.cfg /etc/haproxy/haproxy.cfg
$ sudo systemctl start haproxy
```

# Setup NetworkManager on the client machine

```
$ cat <<EOT > external-crc.conf
address=/apps-crc.testing/SERVER_IP
address=/api.crc.testing/SERVER_IP
EOT

$ export SERVER_IP=”your server’s external IP address”
$ sed -i "s/SERVER_IP/$SERVER_IP/g" external-crc.conf
$ sudo cp external-crc.conf /etc/NetworkManager/dnsmasq.d/external-crc.conf
$ sudo systemctl reload NetworkManager
```

# Login to the OpenShift instance from the client machine

The password for the kubeadmin account was printed when crc started, but if you don't have it
handy you can do this on the server from the home directory of the user running crc:

```
$ find . -name kubeadmin-password
./.crc/cache/crc_libvirt_4.1.11/kubeadmin-password
$ more ./.crc/cache/crc_libvirt_4.1.11/kubeadmin-password
```

Now just login to OpenShift from your client machine using the stand crc url

```
$ oc login -u kubeadmin -p <kubeadmin password>  https://api.crc.testing:6443
```

The OpenShift console will be available at https://console-openshift-console.apps-crc.testing

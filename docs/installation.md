---
layout: normal
---
<h1>Installation</h1>

* TOC
{:toc}

# Via Setup Script

You can install reactor automatically via a [setup script](https://raw.github.com/gridcentric/reactor-core/master/setup-server.sh).

    curl https://raw.github.com/gridcentric/reactor-core/master/setup-server.sh | sudo bash -

Note that you can similarly install the client packages via a [setup script](https://raw.github.com/gridcentric/reactor-core/master/setup-client.sh).

    curl https://raw.github.com/gridcentric/reactor-core/master/setup-client.sh | sudo bash -

# Via cloud-init

Reactor installed via a cloud-init, using the [setup script](https://raw.github.com/gridcentric/reactor-core/master/setup-server.sh).

Simply download this file, and pass it as the user-data to a new instance.

    nova boot ... --user-data=setup-server.sh reactor-instance

NOTE: You'll want to ensure that security groups are set up appropriately for
the Reactor instance. The default API port is 8080, but by default this is
connected through default http and https endpoints.

# From Packages

Reactor is normally installed from cloud-init, but you may choose to install it
manually in your system directly from packages. Keep in mind that in such case
you will need to configure bindings with your load balancer.

The following two sections synthesize the download instructions found elsewhere
for the two main distros.

## Ubuntu / Debian

First get our public key.

    wget -O - http://downloads.gridcentric.com/packages/gridcentric.key | sudo apt-key add -

Second, configure a new APT repo.

    echo deb http://downloads.gridcentric.com/packages/reactor/reactor-core/ubuntu/ gridcentric multiverse | sudo tee /etc/apt/sources.list.d/reactor.list

Now let apt do its job.

    sudo apt-get update
    sudo apt-get install -y reactor-server

## Centos 6.x / RHEL

In this case, you need to enable the EPEL repository to pull in additioinal dependencies.

    rpm --import http://dl.fedoraproject.org/pub/epel/RPM-GPG-KEY-EPEL-6
    rpm -Uvh http://fedora.mirror.nexicom.net/epel/6/i386/epel-release-6-8.noarch.rpm

Also import our public key.

    rpm --import http://downloads.gridcentric.com/packages/gridcentric.key

Now create a yum repo for reactor, in e.g. `/etc/yum.repos.d/reactor.repo`.

    [reactor]
    name=reactor
    baseurl=http://downloads.gridcentric.com/packages/reactor/reactor-core/centos
    enabled=1
    gpgcheck=1

Now unleash yum. Several dependencies will be pulled in, including the JRE for Zookeeper's benefit.

    yum install -y nginx haproxy dnsmasq zookeeper socat
    yum install -y reactor-server

# From Source

You can install the Reactor packages directly from source.

You can either use `pip` to install the packages.

    sudo pip install https://github.com/gridcentric/reactor-core/archive/master.zip

Or, you can clone the repo and run `setup.py`.

    git clone https://github.com/gridcentric/reactor-core
    cd reactor-core && sudo python setup.py install

---
layout: normal
---
<h1>Tutorial</h1>

* TOC
{:toc}

# Introduction

This tutorial is a simple walk-through for using Reactor to auto-scale a
classic LAMP stack on OpenStack. In this case, we'll use
[WordPress](http://wordpress.org) backed by [MySQL](http://mysql.org).

To keep this simple, we'll just keep a static database for the moment, and
limit the setup to an auto-scaling front-end.

This tutorial assumes that you're working with Ubuntu 12.04 Precise images.

# Reactor Installation

If you haven't already, you should setup a Reactor instance in the OpenStack
cloud where you will be managing applications. See the [Installation
instructions](installation.html) for more information. You should be able to
log in to this instance and run the `reactor` command.

# Application Setup

The first step is to install our applications.

This is something that Reactor can't help with, and for your own applications
we highly recommend leveraging available configuration management tools to
ensure that you can rebuild your stack quickly and easily.

In this case, we'll simply be using a couple of `cloud-init` scripts to install
WordPress and MySQL. Let's start with MySQL.

## Requisites

Prioring to starting VMs, let's setup the necessary security groups as it is
easier now than later.

First, we create one for MySQL.

    nova secgroup-create mysql "Mysql instances"
    nova secgroup-add-rule mysql tcp 3306 3306 0.0.0.0/0

Next we create one for WordPress.

    nova secgroup-create wordpress "Wordpress instances"
    nova secgroup-add-rule wordpress tcp 80 80 0.0.0.0/0

## Installing

To install your backend, boot an instance using this `cloud-init` script.

    #!/bin/bash

    # Install the server.
    apt-get update
    apt-get -y -o Dpkg::Options::="--force-confdef" install mysql-server mysql-client

    # Create the wordpress database.
    mysql --defaults-extra-file=/etc/mysql/debian.cnf <<EOF
    CREATE DATABASE IF NOT EXISTS wordpress;
    GRANT SELECT,INSERT,UPDATE,DELETE,CREATE,DROP,ALTER
    ON wordpress.*
    TO wordpress@'%'
    IDENTIFIED BY 'topsecretpassword';
    EOF

    # Ensure mysql is open.
    sed -e 's/^bind-address\s*=.*$/bind-address = 0.0.0.0/g' -i /etc/mysql/my.cnf
    restart mysql

For example, suppose you save this script in a file call
`create-mysql-server`. I would then start an instance as follows, but note
that my `key_name`, `nic` and `image` are all unique to my setup.

    nova boot --image=precise-server-cloudimg-amd64.img --flavor=1 --security_groups=mysql --key_name=amscanne --nic net-id=773075d2-3df6-45e6-8c62-251a49fb8e1f --user-data=create-mysql-server mysql-server

Next, we can save and edit the following `cloud-init` file for our
WordPress frontend.

    #!/bin/bash

    apt-get update
    apt-get -y -o Dpkg::Options::="--force-confdef" install apache2 wordpress

    cat > /etc/apache2/sites-available/wordpress <<EOF
    Alias /blog /usr/share/wordpress
    Alias /blog/wp-content /var/lib/wordpress/wp-content
    <Directory /usr/share/wordpress>
        Options FollowSymLinks
        AllowOverride Limit Options FileInfo
        DirectoryIndex index.php
        Order allow,deny
        Allow from all
    </Directory>
    <Directory /var/lib/wordpress/wp-content>
        Options FollowSymLinks
        Order allow,deny
        Allow from all
    </Directory>
    EOF

    cat > /etc/wordpress/config-default.php <<EOF
    <?php
    define('DB_NAME', 'wordpress');
    define('DB_USER', 'wordpress');
    define('DB_PASSWORD', 'topsecretpassword');
    define('DB_HOST', '$mysql_server');
    define('WP_CONTENT_DIR', '/var/lib/wordpress/wp-content');
    ?>
    EOF

    a2ensite wordpress
    service apache2 restart

In the above file, we'll have to change the references to `$mysql_server` to
the IP address of the MySQL server that we just booted. (You could also change
the password in both if you like, but this is just a demo).

Next, we'll use that script to boot a WordPress instance. In my case, I would do so as follows.

    nova boot --image=precise-server-cloudimg-amd64.img --flavor=1 --security_groups=wordpress --key_name=amscanne --nic net-id=773075d2-3df6-45e6-8c62-251a49fb8e1f --user-data=create-wordpress-server wordpress-server

## Your New Blog!

If everything worked, you should be able to go the IP address of your new
WordPress instance and setup your new blog. Replace `$wordpress_server` in
the following URL to get to your instance.

    http://$wordpress_server/blog/wp-admin/install.php

# Preparing an Endpoint

We're ready to create an endpoint for this WordPress installation. There are
lots of ways to do this, but a pretty easy way is to simply create an image
from the instance. You'll need to figure out the `instance_id` for your server
and run the `nova image-create` command to create a new image.

    nova image-create aa14f7a1-815e-406f-aa2f-1391e96c303a wp-image

This produced the image `wp-image`. Now we can generate the appropriate
configuration for the endpoint. Below is my configuration.

    [endpoint]
    url = http://example.com
    port = 80
    cloud = osapi
    loadbalancer = nginx

    [cloud:osapi]
    auth_url = http://10.1.1.1:5000/v2.0/
    username = thisismyusername
    password = thisismypassword
    tenant_name = thisismytenant
    image_id = f52f71ea-7489-402d-9b4d-ce5c0d9e5fd5
    flavor_id = 1
    instance_name = wordpress-server
    network_conf = net-id=773075d2-3df6-45e6-8c62-251a49fb8e1f
    security_groups = wordpress
    filter_instances = False

The `image_id` above is the image created by the `nova image-create` command.

Once you've tuned this configuration to match your own setup, you can create
the endpoint. I saved this file as `wordpress.conf`.

    reactor create wordpress < wordpress.conf

You should now see the `wordpress` endpoint in `reactor list`.

## Virtual Memory Streaming

If the target OpenStack cloud is [VMS-enabled](http://www.gridcentric.com),
instead of using `nova image-create`, you can use `nova live-image-create`.

You would then provide the `instance_id` of the live-image instead of the
`flavor_id` or `image_id` in the `cloud:osvms` section. Then set the `cloud`
key to `osvms`.

In this case, the configuration would look similar to the following.

    [endpoint]
    url = http://example.com
    port = 80
    cloud = osvms
    loadbalancer = nginx

    [cloud:osvms]
    auth_url = http://10.1.1.1:5000/v2.0/
    username = thisismyusername
    password = thisismypassword
    tenant_name = thisismytenant
    instance_id = f52f71ea-7489-402d-9b4d-ce5c0d9e5fd5
    instance_name = wordpress-server
    security_groups = wordpress
    filter_instances = False

By using [VMS](http://www.gridcentric.com), instances run in a fraction of the
memory and start instantly -- allowing for much better elasiticity.

# Associating Instances

Although Reactor generally manages instances for you, it's possible to
associate existing instances with an endpoint. To do this, we simply take the
`instance_id` of the running WordPress server and use `reactor associate`.

    reactor associate wordpress aa14f7a1-815e-406f-aa2f-1391e96c303a

We then need to register the IP address when we manually associate an instance.
This was mine, but your IP will be different.

    reactor register 172.16.0.131

At this point, if everything has been setup correctly your Reactor manager
should have generated an Nginx configuration that points to the current
instance. You can check on the system.

    cat /etc/nginx/sites-enabled/reactor.89dce6a446a69d6b9bdc01ac75251e4c322bcdff.conf

# Accessing Your Blog

In order to access your blog properly, you'll need to access it through
Reactor. There are many different ways that you can do this, however for the
purposes of this tutorial we will stick with the simplest.

## Edit /etc/hosts

Grab the IP address of your Reactor instance and add it to /etc/hosts with the
name example.com (or whatever name you chose for the URL in the configuration).

    172.16.0.172 example.com

## Configure Your Blog

Once you're hosts file has been set up, hit the login (i.e.
http://example.com/blog/wp-login.php) and change the general settings on your
site. Make sure that you blog address is set to whatever you set the URL to
above (i.e. example.com).

# Setting Scaling Rules

At this point everything should be working. If you want to see Reactor's
decisions in managing your blog, you can use `reactor log` to access the
endpoint log.

    reactor log wordpress

To see the current metrics collected from the loadbalancer, use `reactor
get-metrics`.

    reactor get-metrics wordpress

Now let's make this more interesting. At the moment, this endpoint is paused.
This means that it won't add or remove any instances (it's a neutral state).
Before we unpause it, let's set approximate minimums and maximums.

    reactor set wordpress scaling min_instances 1
    reactor set wordpress scaling max_instances 5

Looking at the metrics above, set some interesting metrics. Reactor will
attempt to maintain the appropriate number of instances based on your metrics.
Let's set something a bit ridiculous so we can see the scaling in action.

    reactor set wordpress scaling rules '1 < active < 10'

That looks good, let's start it up.

    reactor start wordpress

Now hit your site with a lot of clicks, and watch new instances come and go.

---
layout: normal
---
<h1>Docker</h1>

* TOC
{:toc}

Docker can be used to run containers across different managers. Each manager
runs their own docker instances, but the cluster will co-ordinate and schedule
instances appropriately.

## Manager Options

* slots

The available docker scheduling slots on this host.

## Endpoint Options

* slots

The number of scheduled slots required.

* image

The docker image name.

* command

The docker command to run in the given image.

* user

The user to use for running the command.

* environment

The environment for the command.

* mem_limit

The memory limit for the container.

* dns

The DNS server for the container.

* hostname

The hostname of the container.

## Example

### Setting Up Docker

In order to use Reactor with Docker, you'll first need to install Docker. This
is straight-forward if you follow the instructions
[here](https://www.docker.io/gettingstarted/).

You'll also need to ensure that
[docker-py](https://github.com/dotcloud/docker-py) is installed. You can
generally it this as follows.

    sudo pip install https://github.com/dotcloud/docker-py/archive/master.zip

Once all the necessary components are installed, restart reactor.

    /etc/init.d/reactor restart

### Creating an Image

For this example, we assume you've setup the python example as specified in the
Docker tutorial [here](http://docs.docker.io/en/latest/examples/python_web_app/).

You can specify any image to use with Reactor, in the same way you would
specify an image on the command line when running Docker directly.

### Create an Endpoint

To keep things simple, unmanage the default HTTP endpoints if they still exist.

    reactor unmanage apihttp
    reactor unmanage apihttps

Then, create a simple configuration for our docker application. Note that your
`image` parameter will be equal to the `$BUILD_IMG` in the above tutorial.

    [endpoint]
    url=http://
    cloud=docker
    loadbalancer=nginx
    port=5000

    [scaling]
    min_instances=1
    max_instances=5

    [cloud:docker]
    image=eb2b727f547d
    command=/usr/local/bin/runapp

Next, create an endpoint for our docker application.

    reactor manage pyex < docker.conf
    reactor start pyex

Finally, check that you can access your auto-scaled, distributed docker application.

    curl http://127.0.0.1

### Using Auto-scaling

As an example, we'll set the above endpoint to maintain an average of one
connection per second per docker instance.

    reactor set pyex scaling rules '0 < rate < 1'

Check the current metrics.

    reactor get-metrics pyex

You should see:

    {u'active': 0.0}

Now, hit the endpoint a lot.

    while true; do curl http://localhost; done

After a while, stop and you check the metrics.

    reactor get-metrics pyex

You should see something like:

    {u'active': 3.6, u'rate': 33.01126731369651, u'bytes': 396.1352077643581, u'response': 0.002031185031185033}

Shortly, Reactor will start more instances. And after the decomission period, they will be killed.

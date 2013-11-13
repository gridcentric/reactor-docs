---
layout: normal
---
<h1>User Guide</h1>

* TOC
{:toc}

# Architecting the Application

Two main decisions need to be made when architecting a Reactor-managed application:

1. Which Reactor load balancing mechanism (if any) should be used?

1. Which scaling metrics should Reactor use to scale the application?

Insight into making these decisions is detailed below:

## Deciding Which Load Balancing Mechanism to Use

Reactor provides few built-in load balancing mechanisms, including: HTTP
reverse-proxy load balancing (provided by [nginx](http://wiki.nginx.org)) and
DNS round-robin load balancing (provided by
[dnsmasq](http://www.thekelleys.org.uk/dnsmasq/doc.html)).

In some cases, the application itself provides load balancing capabilities. For
example, a Hadoop cluster headnode is typically responsible for tasking out
work to the slave nodes in its cluster. In these cases, Reactor does not
perform any load balancing; instead, it performs scale management and makes
instance information available to the application so that the application can
make its own decisions about distributing load.

### HTTP-based (Nginx)

The Nginx load balancing mechanism is the preferred approach to load-balancing
HTTP-based applications. It provides session "stickiness", and is optimized for
delivery of HTTP-based applications. The Nginx load balancing mechanism is
not appropriate for applications that are not based on HTTP.

The HTTP-based load balancing mechanism has the following configurable properties:

* `sticky_sessions` - This property indicates whether clients need to be sent to the same virtual machine instance over the course of their interaction with the application. For example, if the application makes use of PHP sessions, this property should be set to `true`. If it is not required for clients to be sent to the same virtual machine instance over the course of their intraction with the application, this property can be set to `false`. The default value is `true`.

* `keepalive` - This property specifies the amount of time, in seconds, that the load balancer should maintain a connection with the client. The default value is `0`.

### DNS-based (Dnsmasq)

The Dnsmasq load balancing mechanism is used in cases where clients connect to
a single application endpoint (e.g. a SIP proxy) that is not HTTP-based. The
DNS-based load balancing mechanism works by returning different virtual machine
instances in response to DNS queries on the hostname component of the endpoint
URI. The DNS-based load balancing mechanism should be used in cases where the
HTTP-based load balancing mechanism is not applicable, but where the
application endpoint can be addressed by a single URI.

The DNS-based load balancing mechanism has no configurable properties.

### Application-managed

In cases where the application manages the balancing of load among its
resources, Reactor performs no load balancing functions; instead, it performs
scale management and makes instance information available to the application so
that the application can make its own decisions about distributing load.

## Using Metrics to Construct Scaling Policies

Application metrics are used to tell Reactor when to add or remove virtual
machine instances from a Reactor-managed application.

### Built-in metrics

The following metrics are built-in for all applications:

* `instances` - The number virtual machine instances servicing the endpoint.
* `active` - The average number of active connections per virtual machine instance.

The following metrics are built-in for applications that use the Nginx load
balancing mechanism:

* `response` - The average response time (in milliseconds) per HTTP transaction.
* `rate` - The number of HTTP transactions processed per virtual machine instance, per second.
* `bytes` - The amount of HTTP traffic processed (in bytes) per virtual machine instance, per second.

### Defining Scaling Rules

Scaling rules are defined in the form:

    [<min> (<|<=)] <metric> [(<|<=) <max>]

For example, the scaling rule:

    10 <= active <= 30

tells Reactor to keep the average number of active connections per instance between 10 and 30.

At least one minimum clause should be specified or else Reactor will never
scale the number of virtual machine instances down. Likewise, at least one
maximum clause should be specified or else Reactor will never scale the number
of virtual machine instances up.

Multiple scaling rules can be specified by separating them with a comma. For
example, the rule set:

    response < 300, 10 <= active <= 30

tells Reactor to maintain response times below 300ms, and keep the average
number of active connections per instance between 10 and 30.

If two scaling rules conflict with each other, the rule that is earlier in the
list takes precidence. For example, the rule set:

    2 <= instances, 10 <= active <= 30

tells Reactor to maintain at least two instances, and keep the number of active
connections per instance between 10 and 30. If the average number of active
connections drops below 10, Reactor will still keep two instances ready.

On the other hand, the rule set:

    10 <= active <= 30, response < 300

tells Reactor to keep the number of active connections per instance between 10
and 30 and maintain response times below 300ms. If the average number of active
connections is already 10, Reactor will not increase the number of instances
even if average response times rise above 300ms.

To construct a scaling policy where:

* We always maintain at least two instances and at most ten instances
* As long as the above holds, we maintain response times below 300ms
* As long as the above holds, we maintain active connections between 10 and 30

we would use the rule set:

    2 <= instances <= 10, response < 300, 10 <= active <= 30

### Custom metrics

Applications can also use custom metrics to define scaling policies. For
example, a SIP application may be required to maintain a dropped call rate of
less than 1%. This would be represented by a scaling rule such as:

    dropped_calls < 0.01

Each application instance would then report the number of calls it had dropped,
along with a *weight*. The weight is used by Reactor to calculate the average
value of the custom metric. In the SIP example, the weight would be the total
number of calls processed by the instance in the same time period. For example,
if an instance handled 1000 calls during a period of time and dropped eight of
them, it would report the following to Reactor:

    `{ "dropped_calls" : [ 8, 1000 ] }`

If this were the only instance in the system, Reactor would calculate an
average `dropped_calls` rate of 0.008 and thus the above scaling rule is
satisfied. If, however, an additional instance also reported to Reactor:

    `{ "dropped_calls" : [ 12 , 1000 ] }`

then Reactor would calculate an average `dropped_calls` rate of 0.010 and scale
up the number of instances in the system.

For more information on instance reporting see the [Reactor API
Reference](api-reference.html).

# Deploying a Reactor System

## Endpoint configuration

### Creating an endpoint

Endpoints are created by pushing a configuration file to Reactor. The
configuration file has the following format:

    [endpoint]
    url=<URL of endpoint>
    loadbalancer=<loadbalancer: nginx or dnsmasq, for example>
    cloud=<cloud: osapi or osvms, for example>

    [scaling]
    rules=<list of scaling rules>
    min_instances=<integer>
    max_instances=<integer>

    [cloud:osvms]
    instance_id=<VMS-enabled instance ID>
    auth_url=<OpenStack API URL>
    user=<OpenStack username>
    password=<OpenStack password>
    tenant_name=<OpenStack tenant>

The parameters have the following meanings:

* General parameters:

  * `url` - The URL of the endpoint.

* Scaling parameters:
  * `rules` - The list of scaling rules used to scale the endpoint up and down.
  * `min_instances` - The absolute minimum number of instances to create, regardless of metrics and scaling rules.
  * `max_instances` - The absolute maximum number of instances to create, regardless of metrics and scaling rules.
* VMS parameters (OpenStack-specific):
  * `instance_id` - The VMS-enabled VM template to use for creating new instances.
  * `auth_url` - The URL used to access the OpenStack API for the cloud this endpoint resides in.
  * `user` - The OpenStack username for the cloud this endpoint resides in.
  * `password` - The OpenStack password for the cloud this endpoint resides in.
  * `tenant_name` - The OpenStack project for the cloud this endpoint resides in.

The configuration file is pushed to Reactor using the `reactor create` command:

    reactor create <endpoint name> < <path to config file>

For example:

    reactor create www-production < www-production.conf

### Starting an endpoint

Endpoints are started using the `reactor start` command:

    reactor start <endpoint name>

For example:

    reactor start www-production

### Stopping an endpoint

Endpoints are stopped using the `reactor stop` command:

    reactor stop <endpoint name>

For example:

    reactor stop www-production

### Updating an endpoint

An endpoint configuration is updated using the `reactor update` command:

    reactor update <endpoint name> < <path to config file>

For example:

    reactor update www-production < www-production.conf

### Removing an endpoint

An endpoint is removed using the `reactor remove` command:

    reactor remove <endpoint name>

For example:

    reactor remove www-production

---
layout: normal
---
<h1>Key Concepts</h1>

* TOC
{:toc}

# Applications

A Reactor-managed application is a cloud-based application that is load
balanced and managed by one or more Reactor instances. The application can be
an HTTP-based web service, an arbitrary network socket-based service (such as a
memcached service), or even a non-service-based networked application (such as
a Hadoop map-reduce cluster).

Reactor-managed applications have the following properties:

* **Scalable** - The application is built from functionally-equivalent virtual *infrastructure blocks* - for example, web servers, memcached servers, or Hadoop workers. Infrastructure blocks can be added and removed from the system dynamically, and are based off of a common *template* - they differ from each other only by their runtime identification (e.g. hostnames and IP addresses) and runtime working sets (e.g. active users, active sessions, or active applications).

* **Measurable** - Load on the application is definable and measurable by *metrics* - for example, number of current connections, number or active sessions, number of active applications, etc.

* **Responsive** - There is a mapping between application load and the number of infrastructure blocks needed to service that load. Put another way: increasing the number of infrastructure blocks increases the capacity of the application to handle load.

An application is composed of one or more endpoints. Each endpoint How you
define the exact boundaries of your application is up to you. For example, an
application may consist of multiple endpoints or a single one.

# Reactor Server

Reactor servers provide the Reactor API. Typically these are co-located with
the Reactor managers.

# Reactor Manager

Load balancers and instances are managed by Reactor managers. The managers pull
metrics from the load balancers, query the clouds via their APIs and launch or
kill instances as per the scaling specifications.

You may run any number of managers, and they can support different load
balancers. The appropriate endpoints will be picked up by the appropriate
managers based on load balancers indicated in the configuration. The number of
endpoints managered by an individual manager is determined by the `keys`
configuration value for the manager.

Typically managers are co-located with Reactor servers.

# Endpoints

Endpoints are public entry points into Reactor-managed applications. An
endpoint can correspond to an actual addressable resource (for example, an HTTP
URL for HTTP-based applications) or can be purely symbolic (for example, the
name of a Hadoop cluster). A Reactor-managed application can have multiple
defined endpoints - for example, an HTTP-based application may have a
production endpoint (e.g. http://www.example.com) and a staging endpoint (e.g.
http://staging.example.com). Each endpoint will be independently load-balanced
and scaled, but one endpoint may rely on metrics from another endpoint within
the same Reactor-managed application.

# Instances

Instances are the virtual infrastructure blocks (i.e. VMs) that power
Reactor-managed applications. Reactor creates instances from the Virtual Memory
Streaming (VMS)-enabled VM template specified by the application administrator.
Instances register with the Reactor system and may provide metrics (e.g. CPU
load, number of active connections) to Reactor to help it make scaling
decisions. Instances within an application endpoint will all be based off of
the same VMS-enabled template.

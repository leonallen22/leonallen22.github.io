---
layout: post
title: "Timeseries and Timeseries again"
---
This will be a high-level overview of the metrics stack that I use and maintain at work. Due to the complexity of the tools I won't go into too much detail and will omit some pieces of the architecture that aren't relevant to the topic.

We currently gather host metrics using a stack including Netdata, Cloudwatch, Prometheus, Consul, and Grafana. Netdata is a great tool for collecting a large amount of information on a given host - we use it for all hosts that are not EC2 instances. We use CloudWatch to provide metrics for EC2 instances which is an Amazon Web Services (AWS) offering. The data must be consumed and aggregated, though, since Netdata is only focused on the gathering of metrics and has a short default retention period. For this purpose, we use Prometheus. As we grew our infrastructure, we began introducing dynamically allocated agents using several different cloud providers, one of which is an Openstack implementation. To handle this we added Consul to the picture so that we could have new nodes automatically join a cluster which Prometheus could then use to populate its targets list. This is especially useful since our Openstack hosts are automatically deleted when they are no longer needed: having long-lived metrics helps troubleshooting problems with our infrastructure. Then, for convenience, we added Grafana to visualize the data we collected in Prometheus. Luckily Grafana supports CloudWatch as a datasource, so we didn't need to do much more work for our EC2 instances. The first iteration of our metrics stack was pretty ad-hoc: Prometheus, Consul, and Grafana were run on the same host. That looked something like this:

```
+-----------------+---------------+
|                 |               |
|                 |               |
|   +-----------+ | +-----------+ | +-----------+
|   |ConsulAgent<-+->ConsulAgent<-+->ConsulAgent|
|   +-----------+   +-----------+   +-----------+
|   |  Netdata  |   |  Netdata  |   |  Netdata  |
|   +-----+-----+   +-----+-----+   +-----+-----+
|         |               |               |
|         |               |               |
|         |               |               |
|         +---------------+-------+-------+
|                                 |
|                                 |                          +--+
|                   +-----------  |                          |VM|
+------------------->  Consul  |  |        +----------+      +--+
                    +----v-----+  |        |CloudWatch<------+VM|
                    |Prometheus<--+        +----+-----+      +--+
                    +----v-----+                |            |VM|
                    | Grafana  <----------------+            +--+
                    +----------+
```

It's a bit difficult to depict the Consul cluster due to its complexity - just know that it uses a gossip protocol, which means that each Consul node can share knowledge with all other nodes.


I decided to re-architect a bit this past week, to help ease any scaling we may need in the future. It will also be a bit easier to debug if/when something goes wrong with the components of the stack itself. I moved each service to its own host which makes it easier to swap out each service for upgrades, during service outages, or behind a load balancer. The new stack looks something like this:

```
        +----------------+---------------+
        |                |               |
        |                |               |
        |  +-----------+ | +-----------+ | +-----------+
        |  |ConsulAgent<-+->ConsulAgent<-+->ConsulAgent|
        |  +-----------+   +-----------+   +-----------+
        |  |  Netdata  |   |  Netdata  |   |  Netdata  |
        |  +-----+-----+   +-----+-----+   +-----+-----+
        |        |               |               |
        |        |               |               |
        |        |               |               |
        |        |               |               |
        |        |               |               |
        |        +---------------+-------+-------+
        |                                |
        |                                |
        |                                |
        |                                |
        |     +-------+            +-----v-----+
        +----->Consul1+------------>Prometheus1|
+--+          +-------+            +-----+-----+
|VM|                                     |
+--+      +----------+                   |
|VM+------>CloudWatch|                   |
+--+      +----+-----+                   |
|VM|           |            +--------+   |
+--+           +------------>Grafana1<---+
                            +--------+
```

With this architecture we should be able to provision additional instances (to be labeled 'Grafana2', 'Prometheus2', etc.) of each service as needed and configure a load-balancer to automatically route traffic between instances, increasing reliability. Upgrades should also be easier within this configuration. And of course, revisiting this stack was useful in verifying our current settings are still appropriate, automating some of the service deployment, and resizing the hosts running the services.

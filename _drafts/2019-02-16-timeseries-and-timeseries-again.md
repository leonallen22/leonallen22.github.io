---
layout: post
title: "Timeseries and Timeseries again"
---
We currently gather host metrics using a stack including Netdata, Cloudwatch, Prometheus, Consul, and Grafana. Netdata is a great tool for collecting a large amount of information on a given host - we use it for all hosts that are not EC2 instances. We use CloudWatch for EC2 instances. The data must be consumed and aggregated, though, since Netdata is only focused on the gathering of metrics and has a short default retention period. For this purpose, we use Prometheus. Having the data in one place is much more convenient, and Prometheus has many other benefits besides. As we grew our infrastructure, we began introducing dynamically allocated agents in a several different cloud providers, one of which is an Openstack implementation. To handle this we added Consul to the picture so that we can automatically add new nodes to Prometheus' targets list. This is especially useful since our Openstack hosts are automatically deleted when they are no longer needed, so having long-lived metrics helps troubleshooting problems with our infrastructure. Then, for convenience, we added Grafana (authentication handled using LDAP) to visualize the data we collected in Prometheus. Luckily Grafana supports CloudWatch as a datasource, so we didn't need to do much more work for our EC2 instances. We also used Nginx as a reverse proxy. The first iteration of our metrics stack was pretty ad-hoc: Prometheus, Consul, Grafana, and Nginx were run on the same host. That looked something like this:

```
+-----------+   +-----------+   +-----------+
|Target Node|   |Target Node|   |Target Node|
+-----+-----+   +----+------+   +----+------+
      ^              ^               ^
      |              |               |
      |              |               |
      +------------------------------+
                     |
                     |
                +----+-----+
                |Prometheus|
                +----------+
                |  Consul  |
                +----------+
                | Grafana  +--------+
                +----------+        |
                |  Nginx   |        v
                +----------+      +-+--+
                                  |LDAP|
                                  +----+

```


I decided to re-architect a bit this past week, to help ease any scaling we may need in the future, and help reliability a bit. I moved each service to its own host, which makes more sense because each of them supports high-availability and/or clustering. The new stack now looks like this:

```
+-----------+   +-----------+   +-----------+
|Target Node|   |Target Node|   |Target Node|
+-----+-----+   +----+------+   +------+----+
      ^              ^                 ^
      |              |                 |
      |              |                 |
      |       +------+-----+           |
      |       |            |           |
      |  +----+--+      +--+--------+  |
      +--+Consul1+----->+Prometheus1+--+
         +-------+      +-----+-----+
                              ^
                              |
                              |
                              |
   +----+        +--------+   |
   |LDAP+<-------+Grafana1+---+
   +----+        +----+---+
                      ^
                      |
                      |
                   +--+--+
                   |Nginx|
                   +-----+
```

With this architecture we should be able to provision additional instances of each service as needed, and configure a load-balancer to automatically route traffic between instances and prevent overloading any one instance, increasing reliability. Revisiting this stack was also useful in verifying our current settings are decent, automating some of the service deployment, and resizing the hosts running the services.

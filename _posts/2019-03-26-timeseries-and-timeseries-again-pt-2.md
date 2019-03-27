---
layout: post
title: "Timeseries and Timeseries Again (pt. 2)"
---

One new development I have been pretty excited about is the addition of Loki to our monitoring stack.Developed by the same individuals who develop for Grafana, Loki is a simple log aggregation system designed to be horizontally scalable, cost-effective, and easy to operate. It styles itself as "like Prometheus, but for logs", and that's really all I've wanted for the past year - that you can switch back and forth between metrics and logs within the same interface is just a bonus. Loki relies on promtail to tag and send logs from hosts you care about. Grafana can then use the Loki server as a datasource to present logs to the user, who can narrow down a subset of data based on labels (I decided on hostname, ip address, and filename as a starting set of labels) and perform simple greps on the relevant collection of log events. Loki is still in an alpha stage and as such, there is not a ton of documentation out there have successfully deployed Loki and Promtail instances that are good enough as a minimum viable product. Here is an updated diagram of the monitoring architecture we're working with:

```
      +----------------+---------------+
      |                |               |
      |                |               |
      |           +--+ | +--------+--+ | +--------+----+
      |           |    |          |    |          |    |
      |  +--------+--+ | +--------+--+ | +--------+--+ |
      |  |Promtail   | | |Promtail   | | |Promtail   | |
      |  +-----------+ | +-----------+ | +-----------+ |
      |  |ConsulAgent+-+-+ConsulAgent+-+-+ConsulAgent| |
      |  +-----------+   +-----------+   +-----------+ |
      |  |  Netdata  |   |  Netdata  |   |  Netdata  | |
      |  +-----+-----+   +-----+-----+   +-----+-----+ |
      |        |               |               |       |
      |        |               |               |       |
      |        |               |               |       |
      |        |               |               |       |
      |        |               |               |       |
      |        +---------------+-------+-------+       |
      |                                |               |
      |                                |               |
      |                                |               |
      |                                |               |
      |     +-------+            +-----v-----+      +--v--+
      +-----+Consul1+------------>Prometheus1|      |Loki1|
+--+        +-------+            +-----+-----+      +--+--+
|VM+-+                                 |               |
+--+ |    +----------+                 |               |
|VM+------>CloudWatch|                 |               |
+--+ |    +----+-----+                 |               |
|VM+-+         |          +--------+   |               |
+--+           +---------->Grafana1<---+---------------+
                          +--------+
```

I did run into a couple of hurdles: due to recent rearchitecting and 'right-sizing' some of our hosts, we have some hosts with very limited resources and I made the mistake of writing deployment automation that deployed promtail by building it from source on each host - that was a bad time. At this stage in the projects development, options for deployment are slim, but luckily docker images are provided, which I was able to make some tweaks to for our use case. I also would have liked to configure our Loki server to use Amazon Web Services (AWS) Simple Storage Service (S3) for its storage, but after poking around a bit and following the few suggestions I could find in the repository, I couldn't get it working and went forward with a local storage configuration instead. I ran into a weird [issue](https://github.com/grafana/loki/issues/310) with our AWS secret access key which messed with the Loki configuration since it included a '/' as part of the key, causing Loki to resolve the URI incorrectly. I was able to generate a new key, but ran into more issues that caused me to pivot due to time constraints. I do look forward to following the project and watching it mature, and I have high hopes since Grafana has proven to be useful and reliable so far.

Finally, why Loki? While it is a very young project, it has proven easy enough to deploy and I get the feeling it will incur very little overhead to maintain. As such, it will cost us little to pivot toward another solution if the need arises. We have relied on Splunk, a very mature, well known, widely used (and Expensive) solution for log aggregation in the past. It has fancy features such as its very own query language that I did not have time to learn, and Artificial Intelligence (AI). To be fair, Splunk was useful on multiple occasions as a log aggregator for Jenkins. But, in the middle of a sequence of dumpster fires, our Splunk server died and we were stuck waiting on it to be rebuilt. Oh, and by the way, historical data was not going to be available. In that time I was able to prove out Loki as a viable alternative (still waiting on the new Splunk instance, by the way) and I'm pretty close to having deployment fully automated. To be clear, Loki is a simple system, and not a replacement for something like Splunk. But sometimes all you need are aggregated logs and grep, and, at this stage, I find myself in that situation more often than I need all the bells and whistles Splunk comes with.

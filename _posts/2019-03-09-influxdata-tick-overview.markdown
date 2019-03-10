---
layout: post
title:  "InfluxData TICK stack overview"
tags: [monitoring, influxdata]
excerpt: "<p>How to collect metrics from your systems using InfluxData TICK stack ? 
Let's see what are the components of the stack</p>"
---

Monitoring relies a lots on metrics. It allows you to build dashboards and alerting to setup your SLI.
It exists lots of Monitoring platforms and they are often built around the following components:
* Collectors for retrieving the data to monitor 
* Time series databases to store the collected metrics
* Dashboards to visualize the metrics
* Alerting system to trigger alerts

Such a monitoring platform is [Prometheus][prometheus]{:target="_blank"} which I usually work with.
But this time I had to collect metrics from an Activemq broker using an existing TICK stack.

Let's start by a presentation of the InfluxData TICK stack. 
In the next article, I will put this in practice by explaining in details how you can monitor an ActiveMQ broker.

## InfluxData stack overview

The editor InfluxData has built a full monitoring stack
called TICK stack. TICK stands for :
* **T**elegraf : metric's collector
* **I**nfluxDB : metric's time series database
* **C**hronograf : data visualization
* **K**apacitor : alerting system

## Telegraf

Telegraf is a `go` binary you can install as agent on each server you want to monitor. 

Once installed, Telegraf start collecting metrics. 
Default collected metrics are :
* cpu
* disk
* diskio
* mem
* swap

But there are lots of [input plugins][telegraf-input-plugins]{:target="_blank"} you can activate to gather additional metrics.
[Collected metrics can be processed][telegraf-processor-plugins]{:target="_blank"}
and [sent to an external datasource][telegraf-output-plugins]{:target="_blank"}.

Telegraf defines itself as ***plugin oriented***. 
Sure it contains lots of different inputs, processors and outputs. 
And you can [contribute your own plugin][telegraf-contribute-plugin].
But it is not **modular** : all the plugins are included in the telegraf binary.
If you want to add your custom plugin without contributing it, you need to rebuild the binary from source.
Same thing if you want to run a lightweight telegraf binary with the only plugins you need, you'll have to rebuild it removing the unneeded plugins.

So instead of writing new plugins, it is better to take advantage of already existing ones.
And fortunately, there is a very flexible plugin when it comes to retrieve metrics from jvm systems : [jolokia2 plugin][telegraf-jolokia-plugin]{:target="_blank"}. 


## InfluxDB

InfluxDB is an opensource time series **database** : it is used to store metrics values over time.

Other popular time series databases : 
* [Prometheus][prometheus]{:target="_blank"}
* [Graphite][graphite]{:target="_blank"}
* [OpenTSDB][openTSDB]{:target="_blank"}
* [Elastic][elastic]{:target="_blank"} (With elastic beats)

InfluxDB is the core component of InfluxData platform. 

InfluxDB metrics can be queried with [InfluxQL][influx-ql]{:target="_blank"}, an SQL-like query language. 

## Chronograf

Chronograf is the visualization tool of the stack. Use it to explore your data, and to create your dashboards.
It can also be linked to Kapacitor to define alert rules.

## Kapacitor

Kapacitor is the influxData alerting engine.
Kapacitor can forwards alerts on multiple systems : smtp, pager duty, slack, ...

## Conclusion

We have introduced TICK stack components to understand how they can be used to implement a full monitoring stack.
Now it is time to see how to use it with a concrete example. This will be the subject of the next article.

[prometheus]: https://prometheus.io/
[graphite]: https://graphiteapp.org/
[openTSDB]: http://opentsdb.net/
[elastic]: https://www.elastic.co/solutions/metrics

[influx-ql]: https://docs.influxdata.com/influxdb/latest/query_language/

[telegraf-input-plugins]: https://github.com/influxdata/telegraf#input-plugins
[telegraf-output-plugins]: https://github.com/influxdata/telegraf#output-plugins
[telegraf-processor-plugins]: https://github.com/influxdata/telegraf#processor-plugins
[telegraf-contribute-plugin]: https://github.com/influxdata/telegraf/blob/master/CONTRIBUTING.md
[telegraf-jolokia-plugin]: https://github.com/influxdata/telegraf/tree/master/plugins/inputs/jolokia2


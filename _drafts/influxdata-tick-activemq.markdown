---
layout: post
title:  "ActiveMQ monitoring with TICK"
tags: [monitoring, influxdata, activemq]
excerpt: <p>ActiveMQ monitoring with TICK stack and Jolokia plugin.</p>
---

Let's see how to monitor an ActiveQM Broker using InfluxData TICK stack.
To know more about TICK stack, you can read [my first post][tick-overview]{:target="_blank"} about it.

## Prerequisite

Run a TICK stack along with an activemq broker.

Here is a full working example based on docker and docker-compose.
Once cloned, you can start the stack by running this command :

```shell
docker-compose up -d
```

It will start the following services : 

```shell
➜  tick git:(master) ✗ docker-compose ps
      Name                     Command               State                                              Ports                                           
--------------------------------------------------------------------------------------------------------------------------------------------------------
tick_activemq_1     /bin/sh -c bin/activemq co ...   Up       1883/tcp, 5672/tcp, 61613/tcp, 61614/tcp, 0.0.0.0:61616->61616/tcp, 0.0.0.0:8161->8161/tcp
tick_chronograf_1   /entrypoint.sh chronograf        Up       0.0.0.0:8888->8888/tcp                                                                                                                                                                 
tick_influxdb_1     /entrypoint.sh influxd           Up       0.0.0.0:8086->8086/tcp                                                                    
tick_kapacitor_1    /entrypoint.sh kapacitord        Up       0.0.0.0:9092->9092/tcp                                                                    
tick_telegraf_1     /entrypoint.sh telegraf          Up       0.0.0.0:8092->8092/udp, 0.0.0.0:8094->8094/tcp, 0.0.0.0:8125->8125/udp     
```

## What do we want to monitor ?

First, let's introduce some basic concepts around Activemq messaging :

* ActiveMQ is a messaging broker. 
It is in charge of relaying messages from producers to consumers in an asynchronous way.
Messages can be persisted while waiting to be consumed.

* ActiveMQ implements 2 types of destinations to transmit messages : 
  * queue : point to point messaging. Only one receiver can consume the message 
  * topic : receiver can subscribe a topic. All suscribed receivers will receive produced messages.

* If there are no consumers on a queue, messages are persisted on disk waiting to be consumed.

* If there is an error during the consumption of a message it is a good pratice to forward it on a 
dedicated error queue (DLQ : dead letter queue).

* Messages received by the broker are handled on memory

* Broker is configured with a max connection setting. Each producer and consumer use a connection.


That said, we may need to be alerted in the following cases :
* DLQ are not empty : it means some messages could not be consumed (timeout ? data error ?).
  We may need to replay this messages or visualize them to check if everything is ok.
* Broker consumes too much memory : as we said, with default configuration, messages are stored in memory
during a transfer for faster processing. If messages are big it may consumes a lot of memory. 
We then need to either optimize the configuration, either upgrade the memory allocated to the broker
* Too many connections on the broker : with default configuration, each `transportConnector` is configured
with a `maximumConnections=1000` parameter. We need to be alerted if the current connection number approaches 
this threshold.

## Activemq available metrics

Activemq supports [JMX][wikipedia-jmx]{:target="_blank"} to allow monitoring. 
You can find [on the official documentation page][activemq-documentation-jmx]{:target="_blank"}
which are the available monitoring operations.

In this article, we will use the following mbeans :
* Broker : to retrieve broker metrics (Memory usage, total connections)
* Destination : to retrieve metrics about each destination (queue or topic) : enqueueCount, MemoryPercentageUsage...

All these JMX operations are exposed through http thanks to [Jolokia][jolokia]{:target="_blank"}.
Jolokia is remote JMX with JSON over HTTP.
It means that you can execute all JMX operations and get json response with http.

Jolokia can be deployed as agent on all jvm applications, [either by deploying a war, an osgi bundle or a jvm agent][jolokia-agents]{:target="_blank"}.
In our case, Jolokia is embedded by default in activemq, so you don't need to install it.

## Activemq with Tick

### Telegraf configuration

As we said in previous paragraph, Activemq metrics are exposed over http by jolokia.
So the most suitable telegraf plugin to retrieve activemq metrics is
[jolokia2 plugin][telegraf-jolokia-plugin]{:target="_blank"}

You can enable the jolokia2 plugin by editing the `etc/telegraf/telegraf.conf` file.

If you check this sample config from the project you cloned:

```
[[inputs.jolokia2_agent]]
  # Add agents URLs to query
  urls = ["http://activemq:8161/api/jolokia"]
  username = "admin"
  password = "admin"

  ## This collect data about activemq queues
  [[inputs.jolokia2_agent.metric]]
    name = "activemq_broker_queues"
    mbean  = "org.apache.activemq:type=Broker,brokerName=*,destinationType=Queue,destinationName=*"
    paths = ["QueueSize", "MemoryPercentUsage", "EnqueueCount", "DequeueCount"]
    tag_keys = ["destinationName"]

  [[inputs.jolokia2_agent.metric]]
    name = "activemq_broker_topics"
    mbean  = "org.apache.activemq:type=Broker,brokerName=*,destinationType=Topic,destinationName=*"
    tag_keys = ["destinationName"]

  [[inputs.jolokia2_agent.metric]]
    name = "activemq_broker"
    mbean  = "org.apache.activemq:type=Broker,brokerName=*"
    tag_keys = ["brokerName"]

```

This configuration enables the input `inputs.jolokia2_agent` with 3 `inputs.jolokia2_agent.metric`: 
* `activemq_broker_queues` : retrieve queue metrics based 
on mbean `org.apache.activemq:type=Broker,brokerName=*,destinationType=Queue,destinationName=*`.
We do not need to retrieve all queue metrics, so we filter them with the property `paths = ["QueueSize", "MemoryPercentUsage", "EnqueueCount", "DequeueCount"]`.
Metrics are grouped by `destinationName`
* `activemq_broker_topics` : same as for `activemq_broker_queues` but for topics so mbean is `org.apache.activemq:type=Broker,brokerName=*,destinationType=Topic,destinationName=*`
* `activemq_broker` : retrieve broker metrics grouped by `brokerName` (in case we have multiple brokers)

All these metrics are collected and stored in influxdb.

In our project `etc/telegraf/telegraf.conf`, we have :

```
[[outputs.influxdb]]
  ...
  urls = ["http://influxdb:8086"]
  ...
  database = "telegraf" # required
  ...
```

So all metrics will be stored in the `telegraf` database on our docker `influxdb` service.

### Chronograf visualisation and dashboards

Now that we have metrics stored in influxdb, we can use `chronograf` to setup dashboards.

Open chronograf on your browser : http://localhost:8888/

You can use the chronograf data explorer to build your queries using [influx QL][influx-ql].

![Chronograf data explorer]({{ "/assets/images/posts/tick-activemq/chronograf_data_explorer.png" | absolute_url }})

Here are some samples queries :

TODO.

Now you can build dashboard based on those queries :

TODO.

### Alerting with Kapacitor

We now have a functional dashboard, the final step is to ensure we can be alerted in case of issues.



[activemq-documentation-jmx]: https://activemq.apache.org/jmx.html
[influx-ql]: https://docs.influxdata.com/influxdb/v1.7/query_language/spec/
[jolokia]: https://jolokia.org/
[jolokia-agent]: https://jolokia.org/reference/html/agents.html
[tick-overview]: http://matthieure.me/2019/03/09/influxdata-tick-overview.html
[telegraf-jolokia-plugin]: https://github.com/influxdata/telegraf/tree/master/plugins/inputs/jolokia2
[wikipedia-jmx]: https://en.wikipedia.org/wiki/Java_Management_Extensions

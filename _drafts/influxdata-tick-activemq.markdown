---
layout: post
title:  "ActiveMQ monitoring with TICK"
tags: [monitoring, influxdata, activemq]
excerpt: <p>ActiveMQ monitoring with TICK stack and Jolokia plugin.</p>
---

Let's see how to monitor an ActiveQM Broker using InfluxData TICK stack.
To know more about TICK stack, you can read [my first post][tick-overview] about it.

## Prerequisite

Run a TICK stack and an artemis broker.

If you use docker, you can use this `docker-compose` example : 

<script src="https://gist.github.com/remyma/83f8c2e360309414420373fba0613166.js"></script>

Once you write this `docker-compose.yaml` file, just run it with following command :

```shell
docker-compose up -d
```




## Telegraf Jolokia input plugin

Telegraf defines itself as ***plugin oriented***. 
Sure it contains lots of different inputs, processors and outputs. 
And you can [contribute your own plugin](https://github.com/influxdata/telegraf/blob/master/CONTRIBUTING.md).
But it is not **modular** : all the plugins are included in the telegraf binary.
If you want to add your custom plugin without contributing it, you need to rebuild the binary from source.
Same thing if you want a lightweight telegraf binary with the only plugins you need, you'll have to rebuid it removing the uneeded plugins.

So instead of writing new plugins, try to take advantage of already existing ones.
And fortunately, there is a very flexible plugin when it comes to retrieve metrics from jvm systems : [jolokia2 plugin](https://github.com/influxdata/telegraf/tree/master/plugins/inputs/jolokia2). 


### What is Jolokia ?

According to [Jolokia homepage](https://jolokia.org/), Jolokia is remote JMX with JSON over HTTP.
It means that you can execute all JMX operations and get json response with http. 

Jolokia can be deployed as agent on all jvm applications, [either by deploying a war, an osgi bundle or a jvm agent](https://jolokia.org/reference/html/agents.html).

So if you need a telegraf plugin to consume metrics from a jvm system, it is the right plugin to choose.
And to go further : even if it exists an existing plugin for your jvm system (let's say activemq, tomcat, kafka...), 
you may consider using it instead as it may be less limited for the mectrics to retrieve.

### How to use telegraf jolokia plugin

To enable this plugin, you have to declare it in `telegraf` configuration.
Then you can get inspired by the documented examples from their github repository :
[https://github.com/influxdata/telegraf/tree/master/plugins/inputs/jolokia2/examples](https://github.com/influxdata/telegraf/tree/master/plugins/inputs/jolokia2/examples)

[tick-overview]: http://matthieure.me/2019/03/09/influxdata-tick-overview.html

---
layout: post
title:  "Use Hystrix circuit-breaker in your camel flows" 
categories: karaf camel hystrix
---

When developing services in a distributed environment, you often need to interact with remote services. Those remote services
can affect your system health for example if the remote service is unavailable or unresponsive. Hystrix has been design to protect your services by isolating your them from the outside world. In this post we'll see how to integrate hystrix inside your camel routes.

## Hystrix

Hystrix use the `circuit-breaker` pattern.

### Circuit breaker

## Camel-hystrix

Let's say we have an e-commerce platform where we want to send each order detail to a remote ERP by calling a Http service.
The ERP expose a Http post service `order` on url `http://erp/orders` that accepts xml payloads.

The route use to send the data to the ERP could be like this :

{% highlight java %}
    from("direct:sendOrder")
        .setHeader(Exchange.HTTP_METHOD, constant(HttpMethods.POST))
        .setHeader(Exchange.CONTENT_TYPE, constant("application/xml"))

        .toD("http://erp/orders")
{% endhighlight %}

In this configuration, each order will be sent to the order service no matter if this service is alive or not.
Let's say that this service becomes unavailable, we want it to fail fast to prevent cascading failures.
The first step is to protect the service call by using Hystrix in your camel route :

{% highlight java %}
    from("direct:sendOrder")
        .hystrix()
            .id("POST-order")
            .setHeader(Exchange.HTTP_METHOD, constant(HttpMethods.POST))
            .setHeader(Exchange.CONTENT_TYPE, constant("application/xml"))

            .toD("http://erp/orders")
        .end();
{% endhighlight %}

With this configuration using default values your circuit will break if 20+ request fail during a delay of 5 seconds.
It means that if you send 25 requests in less than 5 seconds and the first 20 request failed, then the circuit break and the next 5 request will fail directly without calling the remote service.

### hystrix fallback

Now what happens if the ERP order service becomes unavailable ? How do you deal with errors ?
You may want to provide a fallback to store the orders you were unable to send

{% highlight java %}
    from("direct:sendOrder")
        .hystrix()
            .id("POST-order")
            .hystrixConfiguration()
                .executionTimeoutInMilliseconds(2000)
            .end()

            .setHeader(Exchange.HTTP_METHOD, constant(HttpMethods.POST))
            .setHeader(Exchange.CONTENT_TYPE, constant("application/xml"))
            
            .toD("http://erp/orders")
        .onFallback()
        .end();
{% endhighlight %}

### hystrix 

## Monitoring Hystrix

To monitor your hystrix services, it is possible to expose metrics and to visualize them through a [dashboard (deprecated)][hystrix-dashboard-wiki].
Metrics are exposed via Hystrix EventStream.

### Hystrix event stream

Here is an example of a event stream :
```
```

### Hystrix dashboard

The Hystrix dashboard is a simple webapp to visualize the metrics.

If you want to monitor a cluster of server, you can gather each app events stream via *Netflix Turbine* and connect directly Hystrix dashboard webapp to it.

An other way to gather hystrix event stream could be to use a telegraf

## Run on karaf server

## References

[hystrix-dashboard-wiki] https://github.com/Netflix-Skunkworks/hystrix-dashboard/wiki
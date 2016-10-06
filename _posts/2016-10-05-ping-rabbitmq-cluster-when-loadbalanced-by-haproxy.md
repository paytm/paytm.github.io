---
layout: post
title:  "Ping RabbitMq cluster when loadbalanced by Haproxy"
date:   2016-10-05 07:09:05
categories: rabbitmq engineering
author: shrey
---

We use RabbitMq for a lot of operations. It forms bus/queueing system/pub-sub base for a lot of sub-systems. Mostly all rabbitMq installations are in cluster mode , and we use Haproxy to loadbalancer / HA.

Google for `rabbitmq timeout haproxy` , and you will find a lot of issues related to rabbitmq connections timing out when using with Haproxy. Most of solutions ask you to change haproxy config which do not work. Alternate solutions are changing kernel limits on TCP keepalive, which I believe should be the last resort.

Now, haproxy demands that some packets be sent on the connection which forcefully keeps the connection open. What can be those packets ? [RabbitMq heartbeat](http://www.rabbitmq.com/heartbeats.html) does not do the trick ( still figuring out why). We need some more robust ping mechanism from client to server.

RabbitMq protocol ( AMQP) does not have Ping. We came out with solution where we open a channel every x seconds and close it immediately. Channels are way to multiplex multiple operations over a single connection and very lightweight. Its holding out so far.
You can find the implementation [here](https://github.com/shreyagarwal/elasty)



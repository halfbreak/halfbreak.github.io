---
layout: post
title:  "Event Dispatcher"
date:   2020-12-20 13:29:15 +0000
categories: microservices
---

This post describes a solution I've seen applied on how to process events in a ordered way (while still processing them in paralel) on the consumer level.

In the previous post, we discussed why we would want to process events in a ordered way, how we would approach it and also the difficulties we might face when doing it with certain messaging systems. The given example was RabbitMQ, but it can happen with many more, for example JMS systems like Tibco EMS.

# The solution

The strategy is illustrated in the following image:

![image](\assets\images\article_02_diagram.jpg)

For this strategy to work we have a single consumer per queue (represented as _Rabbit Consumer_) which grabs events from the _RabbitMQ_ queue and puts it into an in-memory queue, implemented as a _LinkedBlockingQueue_ here. 

The dispatcher which runs on another thread, so as to not block the _Rabbit Consumer_, picks events from the queue and dispatches it to its partition, by writing it to the partition queue. The partition is computed using a pre-configured field from the event payload and a classic hashing/modulo technique. 

Then there are as much handlers as there are partitions, each consuming from its own queue and processing events as they come.

The example code is available [here](https://github.com/halfbreak/rabbit-dispatcher).
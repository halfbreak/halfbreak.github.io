---
layout: post
title:  "Ordered events"
date:   2020-11-20 18:29:15 +0000
categories: microservices
---

Or how to process events in order in a microservices architecture.

# The problem

Whenever you join a new company, besides getting accostumed to a team and its processes, the most interesting challenge is to understand its architecture. Both its advantages and disadvantages. The following example is from a recent company I have worked. The system follows an event-driven architecture, where multiple microservices produce events whenever a notable change in its state occurs. These events are published into a message broker, in this case _RabbitMQ_. Each event type has an associated exchange (a topic in _RabbitMQ_, for publish-subscribe workloads) and each service has its own queue. Whenever a service is interested in a certain event, it binds its queue to the event's exchange.

Let's illustrate this in a classic Car Dealership example:

![image](\assets\images\article_01_diagram.jpg)

There are several problems with this approach. The first one is the sheer number of exchanges that this approach generates, in a very big system there will be hundreds of events and therefore hundreds of exchanges. But the bigger problem is ordering. A service, say _AdvertisingService_, consuming messages in the _AdvertisingServiceQueue_ might see events that belong to the same entity in a different order. A _CustomerUpdated_ event can be received before a _CustomerCreated_ event, which will cause the first event to be discard (the customer can't be updated because it wasn't created yet!).

_RabbitMQ_ ordering guarantees are the following: "messages published in one channel, passing through one exchange and one queue and one outgoing channel will be received in the same order that they were sent". And many things can go wrong either while publishing messages or consuming them, and events can get out of order. 

# The bad solutions

I have seen a few attempts at solutions, most going around the true problem.

## Retries

Whenever you consume a _CustomerUpdated_ you retry processing the event a certain amount of times in hopes that its _CustomerCreated_ has been processed in the meantime. This method assumes that the _CustomerCreated_ event is going to be processed quickly, if it is not, _CustomerUpdated_ will be discarded anyway. Retry policies should also be reserved for individual error types, a transient error, like a database connection exception, should make the event be retried, but a persistent error, like an encoding error, should not be retried as it will never succeed. That means that the dependence between events must be explicitly coded in the receiving system. There can be more challenging dependance between events than simply the first event not being processed. Imagine for example the order between two _CustomerUpdated_, or some type of payment invoice that is dependent on a certain type of customer information being updated.

## In-memory ordering

Some systems solve this issue by ordering events after receiving them. But this only works in batch systems as they can order events by timestamp before their processing. In stream processing in-memory ordering is very difficult, because you don't know whether you still have to wait for some other event to arrive.

Attempts at this approach are very popular, however there are many limitations that come with it.

A _CustomerUpdated_ is consumed, since no customer exists yet, then it holds the message in memory until the _CustomerCreated_ is received and processed. However consistency is lost since the event is in memory and there are no guarantees when the _CustomerCreated_ event will be received.

To keep consistency, manual acknowledge mode can be used, but not only is that tricky to implement properly (a badly handled exception case can keep a message unacked until a timeout or until a restart) but also it could potentially keep many messages unacked and many connections to the message broker with it until a _CustomerCreated_ is processed.

Another approach would be to persist the _CustomerUpdated_ in a database table, until the _CustomerCreated_ is processed and then proceed with that message and others that might have been received, ordered by timestamp. But that is basically implementing Event Sourcing (at a really smaller scale), and add multiple IO operations to any message processing, slowing down your application by a big degree.

# The good solution

### So is there any solution for this problem?

The easiest solution is to go back to the beginning. All the events that belong to the same exchange are ordered, therefore any events that must be ordered must in the same exchange. Most commonly all events belonging to the same entity (e.g. Customer) but not always, for example if the _CarBought_ event requires a customer to exist it might be necessary to put _CustomerCreated_ and _CarBought_ in the same exchange, and since _CustomerCreated_ and _CustomerUpdated_ are already in the same exchange then the three events would all be in the correct order.

### Does this mean that all events need to be in the same exchange?

Of course not, only related events that are required to be ordered. If an event has no such relation with the Customer events then it is better to just split them in different exchange, then they can be subscribed only by the services that need them.

### How about parallelism?

If there is a queue per service bound to a single exchange, but there are multiple competing consumers on it, then ordering will also be lost. So is it impossible to have high throughput and ordering simultaneously? The key to have both of this is to add data partitioning, unfortunately _RabbitMQ_ does not support it. Partitioning works by choosing a message field (partition key) and splitting the messages into different queues by this key's value (how to split them depends on partition strategy). There can be one single consumer per queue, but parallelism is achieved by the number of partitions.

Although _RabbitMQ_ doesn't support partitioning it is possible to simulate it, the Spring Framework for example does this, the publisher must tag the messages it sends into an exchange with a routing key that indicates which is its partition and it will reach the appropriate queue.

### But this seems like too much work.

Yes. But there is a system that already does all of this natively, Apache Kafka. It is heavier and more difficult to setup, it brings zookeeper together as well, but it fulfills all the requirements without needing to limit concurrent consumers or creating virtual partitions. It just works.

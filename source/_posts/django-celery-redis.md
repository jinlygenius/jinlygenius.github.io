---
title: django celery redis
date: 2017-07-18 14:42:51
tags:
- django
- celery
- redis
---

这篇真的好全啊。。。

http://docs.celeryproject.org/en/latest/django/first-steps-with-django.html

这里就不禁想好好讲讲celery，其实是 Advanced Message Queueing Protocol (AMQP) 了。我自己也是参考了文章 https://www.abhishek-tiwari.com/amqp-rabbitmq-and-celery-a-visual-guide-for-dummies/ 和 https://www.cloudamqp.com/blog/2015-09-03-part4-rabbitmq-for-beginners-exchanges-routing-keys-bindings.html

再次感谢网络上这么多大神分享的这么多好的文章。很多原理都是在网上看了很多文章琢磨和理解透彻的。


#### 生产者 ####
生产者就是主业务服务里，产生异步消息或者任务的角色。生产者产生消息任务以后就需要下一个角色 Broker来发挥作用了。


#### Broker ####
Broker由 Exchange，Binding 和 Queue 组成。（我们最早用redis做Broker。）Broker主要用来接收和存储生产者发出的消息任务 —— Exchange会接收消息任务，通过Bindings等因素分发给某些Queue，然后消息任务会存储在被选择的Queue里。

Exchange是一个消息分发代理，会根据Bindings，消息的 header attributes, routing keys，决定什么消息任务进什么队列。

Binding其实就是Exchange和Queue之间的link。

我们可以配置多个Queue，每个队列有自己的binding key。

当一条消息任务发来后，这个任务会被打上一个routing key。而Exchange会把消息任务分发给和这个routing key完全一致的binding key的队列。


#### 消费者 ####
消费者由很多Workers组成。每个worker也可以起多个并发进程。每个worker会根据自己被配置的routing key对应的Queue去获取任务并执行。



#### Exchange Types ####




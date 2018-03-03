---
layout: post
title:  "Redis lock vs Consul lock"
date:   2018-02-27 12:00:00
categories: [redis,consul,lock,kafka,spring]
comments: true
---

![Redis lock vs Consul Lock]({{ "/img/goals.jpg" | absolute_url }})

A month or so ago, I read a post in Slack Engineering's blog about how [Slack handles billions of tasks in miliseconds](https://slack.engineering/scaling-slacks-job-queue-687222e9d100). 

In that post, Slack engineers speaks about how they have redesigned their queue architecture and how they uses redis, consul and kafka. 

It called my attention they used consul as a locking mechanism instead of redis because I have developed several locks with redis and it's works great. Besides, they have integrated a redis cluster and I thnik this fact should facilitate its implementation.

After I read this post, I got doubts about which locking mechanism would have better performance so I develop both strategies of locking and a kafka queue to determine which gets the best performance.

My aim was to test the performance offered by sending thousands of messages to kafka and persisting them in mongo through a listener.

First of all, I built an environment with docker consisting of consul, redis, kafka, zookeeper and mongo. I wanted to test the performance of send milles of messages to kafka and store it in mongo.  
{% highlight yml %}
version: "3"
services:

  consul:
    image: consul:latest
    command: agent -server -dev -client 0.0.0.0 -log-level err
    ports:
    - 8500:8500
    healthcheck:
      test: "exit 0"
  
  redis:
    image: redis
    ports:
        - "6379:6379"

  mongo:
    image: mongo:3.2.4
    ports:
    - 27017:27017
    command: --smallfiles

  zookeeper:
    image: wurstmeister/zookeeper
    ports:
      - "2181:2181"
      
  kafka:
    image: wurstmeister/kafka
    ports:
      - "9092:9092"
    hostname:
      "kafka"
    environment:
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_PORT: 9092
	  KAFKA_AUTO_CREATE_TOPICS_ENABLE: 'false'

{% endhighlight %}

We can set up our environment with docker-compose.

~~~ shell
docker-compose -up
~~~

Once we have our environment up, we need to create a kafka topic to channel the messages. For this we can rely on this tool, [Kafka Tool](http://www.kafkatool.com/).

![Kafka Topic Creation]({{ "/img/kafka-post/kafkaTopic-creation.png" | absolute_url }})
![Kafka Topic Creation Successfully]({{ "/img/kafka-post/kafkaTopic-creation-successfully.png" | absolute_url }})

Our environment is fine, we have to implement a sender and a listener.

#### Sender:





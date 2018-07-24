---
layout: post
title:  "How to test Kafka Streams"
date:   2018-07-24 19:00:00
categories: [kafka, kafka-streams, test]
comments: true
---

![Apache Kafka]({{ "/img/apache-kafka.png" | absolute_url }})

## How to test Kafka Streams

### 1. Introduction

After a couple of months learning and researching about kafka streams, i wasn't able to find much information about how to test my kafka streams so I would like to share how a kafka stream could be tested with unit or integration tests.


We have the following scenarios:

1. Bank Balance: Extracted from [udemy](https://www.udemy.com/kafka-streams/). Process all incoming transactions and accumulate them in a balance.
2. Customer purchases dispatcher: Process all incoming purchases and dispatch to the specific customer topic informed in the purchase code received.
3. Menu preparation: For each customer, the stream receives several recipes and this recipes must be grouped into a menu and sent by email to the customer. A single email should be received by each customer

For the above scenarios, we have unit and/or integration tests.
1. Unit tests has been developed with [kafka-streams-test-utils](https://kafka.apache.org/documentation/streams/developer-guide/testing.html) library.
2. Integration tests has been developed with [spring-kafka-test](https://github.com/spring-projects/spring-kafka/blob/master/src/reference/asciidoc/testing.adoc) library.

### 2. Setup

Testing a kafka stream is only available on version 1.1.0 or higher, so we need to set this version for all our kafka dependencies.

{% highlight xml %}


		<properties>
			<spring-kafka.version>2.1.7.RELEASE</spring-kafka.version>
			<kafka.version>1.1.0</kafka.version>
		</properties>

		<!-- spring-kafka -->
		<dependency>
			<groupId>org.springframework.kafka</groupId>
			<artifactId>spring-kafka</artifactId>
			<version>${spring-kafka.version}</version>
			<exclusions>
				<exclusion>
					<groupId>org.apache.kafka</groupId>
					<artifactId>kafka-clients</artifactId>
				</exclusion>
			</exclusions>
		</dependency>
		<dependency>
			<groupId>org.apache.kafka</groupId>
			<artifactId>kafka-streams</artifactId>
			<version>${kafka.version}</version>
		</dependency>

		<dependency>
			<groupId>org.apache.kafka</groupId>
			<artifactId>kafka-clients</artifactId>
			<version>${kafka.version}</version>
		</dependency>

		<dependency>
			<groupId>org.apache.kafka</groupId>
			<artifactId>kafka-clients</artifactId>
			<version>${kafka.version}</version>
			<classifier>test</classifier>
		</dependency>

		<dependency>
			<groupId>org.apache.kafka</groupId>
			<artifactId>kafka_2.11</artifactId>
			<version>${kafka.version}</version>
		</dependency>

		<!-- Testing -->
		<!-- Spring tests -->
		<dependency>
			<groupId>org.springframework.kafka</groupId>
			<artifactId>spring-kafka-test</artifactId>
			<version>${spring-kafka.version}</version>
			<exclusions>
				<exclusion>
					<groupId>org.apache.kafka</groupId>
					<artifactId>kafka-clients</artifactId>
				</exclusion>
			</exclusions>
			<scope>test</scope>
		</dependency>

		<!-- Kafka tests -->
		<dependency>
			<groupId>org.apache.kafka</groupId>
			<artifactId>kafka_2.11</artifactId>
			<version>1.1.0</version>
			<classifier>test</classifier>
		</dependency>

		<dependency>
			<groupId>org.apache.kafka</groupId>
			<artifactId>kafka-streams-test-utils</artifactId>
			<version>${kafka.version}</version>
			<scope>test</scope>
		</dependency>

{% endhighlight %}

Some dependencies, like junit, mockito, etc.. has been omitted to avoid verbosity

### 3. Unit tests

### 4. Integration tests

### 5. Conclusion








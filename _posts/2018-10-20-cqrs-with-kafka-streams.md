---
layout: post
title:  "CQRS with Kafka Streams"
date:   2018-10-20 14:00:00
categories: [Kafka, Kafka-streams, CQRS, Spring, Tutorial, Event Sourcing, Talks, Twitter Streams, Docker, PostgreSQL, Kafka Connect, Spring Boot, Kotlin]
comments: true
---

![CQRS with Kafka Streams](https://david-romero.github.io/img/cqrs-with-kafka-streams.png){:height="500px" width="500px"}

## CQRS with Kafka Streams

### 1. Introduction

Last September, my coworker [Iván Gutiérrez](https://es.linkedin.com/in/ivangutierrezrodriguez) and me, spoke to our cowokers how to implement [Event sourcing with Kafka](https://drive.google.com/open?id=1TrcvgfnO7Dqg96JYSp881i0zdr2gQJ4xHR_BeOLexIM) and in this talk, I developed a demo with the goal of strengthen the theoretical concepts. In this demo, I developed a Kafka Stream that reads the tweets containing "Java" word from Twitter, group tweets by username and select the tweet with the most likes. The pipeline ends sending the recolected information to PostgreSQL

As we have received positive feedback and we have learned a lot of things, I want to share this demo in order to it will be available to anyone who wants to take a look.

The demo was divided into 5 steps due to simplicity.
1. [The stack](the-stack)
2. [Producer (Writer App)](producer)
3. [Kafka Stream](kafka-stream)
4. [Kafka Connector](kafka-connector)
5. [Reader App](reader-app)

### 2. Implementation

#### The Stack

The whole stack has been implemented in Docker for its simplicity when integrating several tools and for its isolation level. 
The stack is composed of 

{% highlight yml %}


version: '3.1'
services:

  #############
  # Kafka
  #############
  zookeeper:
    image: confluentinc/cp-zookeeper
    container_name: zookeeper
    network_mode: host
    ports:
      - "2181:2181"
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181

  kafka:
    image: confluentinc/cp-kafka
    container_name: kafka
    network_mode: host
    depends_on:
      - zookeeper
    ports:
      - "9092:9092"  
    environment:
      KAFKA_ZOOKEEPER_CONNECT: localhost:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1 # We have only 1 broker, so offsets topic can only have one replication factor.

  connect:
    image: confluentinc/cp-kafka-connect
    container_name: kafka-connect
    network_mode: host
    ports:
      - "8083:8083"
    depends_on:
      - zookeeper
      - kafka
    volumes:
      - $PWD/connect-plugins:/etc/kafka-connect/jars
    environment:
      CONNECT_BOOTSTRAP_SERVERS: kafka:9092
      CONNECT_REST_PORT: 8083 # Kafka connect creates an endpoint in order to add connectors
      CONNECT_REST_ADVERTISED_HOST_NAME: "kafka-connect"
      CONNECT_GROUP_ID: kafka-connect
      CONNECT_ZOOKEEPER_CONNECT: zookeeper:2181
      CONNECT_CONFIG_STORAGE_TOPIC: kafka-connect-config
      CONNECT_OFFSET_STORAGE_TOPIC: kafka-connect-offsets
      CONNECT_STATUS_STORAGE_TOPIC: kafka-connect-status
      CONNECT_CONFIG_STORAGE_REPLICATION_FACTOR: 1 # We have only 1 broker, so we can only have 1 replication factor.
      CONNECT_OFFSET_STORAGE_REPLICATION_FACTOR: 1 
      CONNECT_STATUS_STORAGE_REPLICATION_FACTOR: 1 
      CONNECT_KEY_CONVERTER: "org.apache.kafka.connect.storage.StringConverter" # We receive a string as key and a json as value
      CONNECT_VALUE_CONVERTER: "org.apache.kafka.connect.json.JsonConverter"
      CONNECT_INTERNAL_KEY_CONVERTER: "org.apache.kafka.connect.storage.StringConverter"
      CONNECT_INTERNAL_VALUE_CONVERTER: "org.apache.kafka.connect.json.JsonConverter"
      CONNECT_PLUGIN_PATH: /usr/share/java,/etc/kafka-connect/jars

  #############
  # PostgreSQL
  #############
  db:
    container_name: postgresql
    network_mode: host
    image: postgres
    restart: always
    ports:
    - "5432:5432"
    environment:
      POSTGRES_DB: influencers
      POSTGRES_USER: user
      POSTGRES_PASSWORD: 1234

{% endhighlight %}

Some dependencies, like junit, mockito, etc.. has been omitted to avoid verbosity

#### Producer

Unit tests for kafka streams are available from version 1.1.0 and it is the best way to test the topology of your kafka stream. The main advantage of unit tests over the integration ones is that they do not require the kafka ecosystem to be executed, therefore they are faster to execute and more isolated.

Let's suppose that we have the following scenario:

We have a topic to which all the purchases made in our application and a topic for each customer. Each purchase has an identification code which includes the code of the customer who made the purchase. We have to redirect this purchase to the customer's own topic. To know the topic related to each client we receive a map where the key will be the customer code and the value of the target topic. 
Besides, we have to replace spanish character 'ñ' by 'n'.

The solution provided is the following:

{% highlight java %}

		// Kafka config
		Properties properties = new Properties();
		properties.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092"); //Kafka cluster hosts.
		properties.put(ProducerConfig.CLIENT_ID_CONFIG, "demo-twitter-kafka-application-producer"); // Group id
		properties.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName()); 
		properties.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());
		Producer<String, String> producer = new KafkaProducer<>(properties);

		// Twitter Stream
		final TwitterStream twitterStream = new TwitterStreamFactory().getInstance();
		final StatusListener listener = new StatusListener() {
			public void onStatus(Status status) {
				final long likes = getLikes(status);
				final String tweet = status.getText();
				final String content = status.getUser().getName() + "::::" + tweet + "::::" + likes;
				log.info(content);
				producer.send(new ProducerRecord<>("tweets", content));
			}

			//Some methods have been omitted for simplicity.

			private long getLikes(Status status) {
				return status.getRetweetedStatus() != null ? status.getRetweetedStatus().getFavoriteCount() : 0;
			}
		};
		twitterStream.addListener(listener);
		final FilterQuery tweetFilterQuery = new FilterQuery();
		tweetFilterQuery.track(new String[] { "Java" });
		twitterStream.filter(tweetFilterQuery);
		SpringApplication.run(DemoTwitterKafkaProducerApplication.class, args);
		Runtime.getRuntime().addShutdownHook(new Thread(() -> producer.close())); //Kafka producer should close when application finishes.
	}
}


{% endhighlight %}

Now, we can test our solution.

Following the [documentation](https://kafka.apache.org/documentation/streams/developer-guide/testing.html), we need to create a TestDriver and a consumer factory if we want to read messages.

{% highlight java %}

	



{% endhighlight %}

The driver configuration should be the same that we have in our kafka environment.

Once we have our TestDrive we can test our topology.

{% highlight java %}

	
	


{% endhighlight %}

Once we have our test finished we can verify that everything is fine

![Success](https://david-romero.github.io/img/kafkaUnitTest.png)


#### Kafka Stream

In the same way that the unit tests help us verify that our topology is well designed, the integration tests also help us in this task by adding the extra to introduce the kafka ecosystem in our tests.
This implies that our tests will be more "real" but in the other hand, they will be much slower.

Spring framework has developed a very useful library that provides all necessary to develop a good integration tests. Further information could be obtained [here](https://github.com/spring-projects/spring-kafka)

Let's suppose that we have the following scenario:

We have a topic with incoming transactions and we must group them by customer and create a balance of these transactions. This balance will have the sum of the transaction amounts, the transaction count, and the last timestamp.

The solution provided is the following:

{% highlight java %}

	@Configuration
	@EnableKafkaStreams
	static class KafkaConsumerConfiguration {
		
		final Serde<Influencer> jsonSerde = new JsonSerde<>(Influencer.class);
		final Materialized<String, Influencer, KeyValueStore<Bytes, byte[]>> materialized = Materialized.<String, Influencer, KeyValueStore<Bytes, byte[]>>as("aggregation-tweets-by-likes").withValueSerde(jsonSerde);
		
		@Bean
		KStream<String, String> stream(StreamsBuilder streamBuilder){
			final KStream<String, String> stream = streamBuilder.stream("tweets");
			stream
				.selectKey(( key , value ) -> String.valueOf(value.split("::::")[0]))
				.groupByKey()
				.aggregate(Influencer::init, this::aggregateInfoToInfluencer, materialized)
				.mapValues(InfluencerJsonSchema::new)
				.toStream()
				.peek( (username, jsonSchema) -> log.info("Sending a new tweet from user: {}", username))
				.to("influencers", Produced.with(Serdes.String(), new JsonSerde<>(InfluencerJsonSchema.class)));
			return stream;
		}
		
		private Influencer aggregateInfoToInfluencer(String username, String tweet, Influencer influencer) {
			final long likes = Long.valueOf(tweet.split("::::")[2]);
			if ( likes >= influencer.getLikes() ) {
				return new Influencer(influencer.getTweets()+1, username, String.valueOf(tweet.split("::::")[1]), likes);
			} else {
				return new Influencer(influencer.getTweets()+1, username, influencer.getContent(), influencer.getLikes());
			}
		}
		
		@Bean(name = KafkaStreamsDefaultConfiguration.DEFAULT_STREAMS_CONFIG_BEAN_NAME)
		public StreamsConfig kStreamsConfigs(KafkaProperties kafkaProperties) {
			Map<String, Object> props = new HashMap<>();
			props.put(StreamsConfig.APPLICATION_ID_CONFIG, "demo-twitter-kafka-application");
			props.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, kafkaProperties.getBootstrapServers());
			props.put(StreamsConfig.DEFAULT_KEY_SERDE_CLASS_CONFIG, Serdes.String().getClass());
			props.put(StreamsConfig.DEFAULT_VALUE_SERDE_CLASS_CONFIG, Serdes.String().getClass());
			return new StreamsConfig(props);
		}
		
	}


{% endhighlight %}



{% highlight java %}
	
	@RequiredArgsConstructor
	@Getter
	public static class Influencer {
		
		final long tweets;
		
		final String username;
		
		final String content;
		
		final long likes;
		
		static Influencer init() {
			return new Influencer(0, "","", 0);
		}
		
		@JsonCreator
		static Influencer fromJson(@JsonProperty("tweets") long tweetCounts, @JsonProperty("username") String username, @JsonProperty("content") String content, @JsonProperty("likes") long likes) {
			return new Influencer(tweetCounts, username, content, likes);
		}
		
	}


{% endhighlight %}



#### Kafka Connector

{% highlight java %}

/**
 * https://gist.github.com/rmoff/2b922fd1f9baf3ba1d66b98e9dd7b364
 * 
 */
@Getter
public class InfluencerJsonSchema {

	Schema schema;
	Influencer payload;

	InfluencerJsonSchema(long tweetCounts, String username, String content, long likes) {
		this.payload = new Influencer(tweetCounts, username, content, likes);
		Field fieldTweetCounts = Field.builder().field("tweets").type("int64").build();
		Field fieldContent = Field.builder().field("content").type("string").build();
		Field fieldUsername = Field.builder().field("username").type("string").build();
		Field fieldLikes = Field.builder().field("likes").type("int64").build();
		this.schema = new Schema("struct", Arrays.asList(fieldUsername,fieldContent,fieldLikes,fieldTweetCounts));
	}
	
	public InfluencerJsonSchema(Influencer influencer) {
		this(influencer.getTweets(),influencer.getUsername(),influencer.getContent(),influencer.getLikes());
	}

	@Getter
	@AllArgsConstructor
	static class Schema {

		String type;
		List<Field> fields;

	}

	@Getter
	@Builder
	static class Field {

		String type;
		String field;

	}
}

{% endhighlight %}

{% highlight json %}

{
    "name": "jdbc-sink",
    "config": {
        "connector.class": "io.confluent.connect.jdbc.JdbcSinkConnector",
        "tasks.max": "1",
        "topics": "influencers",
	    "table.name.format": "influencer",
        "connection.url": "jdbc:postgresql://postgresql:5432/influencers?user=user&password=1234",
        "auto.create": "true",                                                   
        "insert.mode": "upsert",                                                 
        "pk.fields": "username",                                                       
        "pk.mode": "record_key"                                                
    }
}

{% endhighlight %}

```console
foo@bar:~$ curl -i -X POST -H "Accept:application/json" -H  "Content-Type:application/json" http://localhost:8083/connectors/ -d @connect-plugins/jdbc-sink.json
```

#### Reader App

### 3. Conclusion

If we want to develop a quality kafka streams we need to test the topologies and for that goal we can follow two approaches: kafka-tests and/or spring-kafka-tests. 
In my humble opinion, we should develop both strategies in order to tests as cases as possible always maintaining a balance between both testing strategies.
In this [Github Repo]((https://github.com/david-romero/kafka-streams-tests)), there is available the tests for scenario 3.

The full source code for this article is available over on [GitHub](https://github.com/david-romero/demo-twitter-kafka).









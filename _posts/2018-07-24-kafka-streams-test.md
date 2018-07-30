---
layout: post
title:  "How to test Kafka Streams"
date:   2018-07-24 19:00:00
categories: [Kafka, Kafka-streams, Testing, Spring, Tutorial]
comments: true
---

![Apache Kafka](./img/apache-kafka.png=500x500)

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
			<version>${kafka.version}</version>
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

Unit tests for kafka streams are available from version 1.1.0 and it is the best way to test the topology of your kafka stream. The main advantage of unit tests over the integration ones is that they do not require the kafka ecosystem to be executed, therefore they are faster to execute and more isolated.

Let's suppose that we have the following scenario:

We have a topic to which all the purchases made in our application and a topic for each customer. Each purchase has an identification code which includes the code of the customer who made the purchase. We have to redirect this purchase to the customer's own topic. To know the topic related to each client we receive a map where the key will be the customer code and the value of the target topic. 
Besides, we have to replace spanish character 'ñ' by 'n'.

The solution provided is the following:

{% highlight java %}

	public Topology build() {
		final KStream<String, String> stream = streamBuilder.stream(topic);

		final KStream<String, String>[] streams = stream
			.filter(this::hasLengthUpper20)
			.mapValues(s -> s.replace("ñ", "n"))
			.branch(createKafkaPredicates());

		final List<String> targetTopics = new ArrayList<>(symbolTopicMap.values());
		for (int streamIndex = 0; streamIndex < symbolTopicMap.size(); streamIndex++) {
			streams[streamIndex].to(targetTopics.get(streamIndex));
		}

		return streamBuilder.build();

	}

	private boolean hasLengthUpper20(String key, String value) {
		return StringUtils.hasLength(value) && value.length() > 20;
	}

	private KafkaPredicate[] createKafkaPredicates() {
		final List<KafkaPredicate> predicates = symbolTopicMap.keySet().stream().map(symbolToKafkaPredicateFuncition)
				.collect(Collectors.toList());
		KafkaPredicate[] array = new KafkaPredicate[predicates.size()];
		return predicates.toArray(array);
	}
	


	@RequiredArgsConstructor
	class KafkaPredicate implements Predicate<String, String> {

		final BiPredicate<String, String> predicate;

		@Override
		public boolean test(String key, String value) {
			return predicate.test(key, value);
		}

	}


{% endhighlight %}

Now, we can test our solution.

Following the [documentation](https://kafka.apache.org/documentation/streams/developer-guide/testing.html), we need to create a TestDriver and a consumer factory if we want to read messages.

{% highlight java %}

	
	TopologyTestDriver testDriver;
	
	ConsumerRecordFactory<String, String> factory = new ConsumerRecordFactory<>(TOPIC, new StringSerializer(), new StringSerializer());
	
	@BeforeEach
	public void setUp() {
		final Properties config = new Properties();
		config.put(StreamsConfig.APPLICATION_ID_CONFIG, "test");
		config.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, "dummy:1234");
		config.setProperty(StreamsConfig.DEFAULT_KEY_SERDE_CLASS_CONFIG, Serdes.String().getClass().getName());
	    config.setProperty(StreamsConfig.DEFAULT_VALUE_SERDE_CLASS_CONFIG, Serdes.String().getClass().getName());
		StreamsBuilder streamBuilder = new StreamsBuilder();
		DispatcherKStreamBuilder builder = new DispatcherKStreamBuilder(streamBuilder, TOPIC, SYMBOL_TOPIC_MAP); //Our topology builder. 
		testDriver = new TopologyTestDriver(builder.build(), config);
	}


{% endhighlight %}

The driver configuration should be the same that we have in our kafka environment.

Once we have our TestDrive we can test our topology.

{% highlight java %}

	
	@Test
	@DisplayName("Given Ten Codes And A Map With Customer Codes And Target Topics When The Stream Receives Ten Codes Then Every Target Topic Should Receive Its PurchaseCode")
	public void givenTenCodesAndAMapWithCustomerCodeAndTargetTopicWhenTheStreamReceivesTenCodeThenEveryTargetTopicShouldReceiveItsPurchaseCode() throws InterruptedException {
		// given
		String[] purchaseCodes = Accounts.accounts;
		// purchase code format: salkdjaslkdajsdlajsdklajsdaklsjdfyhbeubyhquy12345kdalsdjaksldjasldjhvbfudybdudfubdf. ascii(0-15) + Customer_Code(15-20) + ascii(20+)
		// purchases code and topic map format: { "12345" : "TOPIC_CUSTOMER_1" , "54321" : "TOPIC_CUSTOMER_2" }
		
		// when
		stream(purchaseCodes).forEach(this::sendMessage);

		// then
		assertCodeIsInTopic(purchaseCodes[0], TOPIC_CUSTOMER1);
		assertCodeIsInTopic(purchaseCodes[1], TOPIC_CUSTOMER2);
		assertCodeIsInTopic(purchaseCodes[2], TOPIC_CUSTOMER3);
		assertCodeIsInTopic(purchaseCodes[3], TOPIC_CUSTOMER3);
		assertCodeIsInTopic(purchaseCodes[4], TOPIC_CUSTOMER4);
		assertCodeIsInTopic(purchaseCodes[5], TOPIC_CUSTOMER5);
		assertCodeIsInTopic(purchaseCodes[6], TOPIC_CUSTOMER1);
		assertCodeIsInTopic(purchaseCodes[7], TOPIC_CUSTOMER2);
		assertCodeIsInTopic(purchaseCodes[8], TOPIC_CUSTOMER1);
		assertCodeIsInTopic(purchaseCodes[9], TOPIC_CUSTOMER4);
	}

	private void assertCodeIsInTopic(String code, String topic) {
		OutputVerifier.compareKeyValue(readMessage(topic), null, code);
	}
	
	private void sendMessage(final String message) {
		final KeyValue<String,String> kv = new KeyValue<String, String>(null, message);
		final List<KeyValue<String,String>> keyValues = java.util.Arrays.asList(kv);
		final List<ConsumerRecord<byte[], byte[]>> create = factory.create(keyValues);
		testDriver.pipeInput(create);
	}
	
	private ProducerRecord<String, String> readMessage(String topic) {
		return testDriver.readOutput(topic, new StringDeserializer(), new StringDeserializer());
	}


{% endhighlight %}

Once we have our test finished we can verify that everything is fine

![Success](./img/kafkaUnitTest.png)


### 4. Integration tests

In the same way that the unit tests help us verify that our topology is well designed, the integration tests also help us in this task by adding the extra to introduce the kafka ecosystem in our tests.
This implies that our tests will be more "real" but in the other hand, they will be much slower.

Spring framework has developed a very useful library that provides all necessary to develop a good integration tests. Further information could be obtained [here](https://github.com/spring-projects/spring-kafka)

Let's suppose that we have the following scenario:

We have a topic with incoming transactions and we must group them by customer and create a balance of these transactions. This balance will have the sum of the transaction amounts, the transaction count, and the last timestamp.

The solution provided is the following:

{% highlight java %}

	
	public Topology build() {
		final KStream<String, Transaction> stream = streamBuilder.stream(inputTopic);
		
		stream
				.groupByKey(Serialized.with(keySerde, new TransactionJsonSerde()))
				.aggregate(Balance::init, (key, transaction, balance) -> applyTransaction(balance, transaction))
				.toStream()
				.to(outputTopic, Produced.with(keySerde, valueSerde));

		return streamBuilder.build();
	}

	Balance applyTransaction(final Balance balance, final Transaction transaction) {
		final BigDecimal amount = balance.getAmount().add(BigDecimal.valueOf(transaction.getAmount())).setScale(4, RoundingMode.HALF_UP);
		final int count = balance.getTransactionCounts() + 1;
		final long timestamp = Math.max(balance.getTimestamp(), transaction.getTimestamp());
		return new Balance(amount, timestamp, count);
	}

	@Getter
	@RequiredArgsConstructor
	static class Balance {

		final BigDecimal amount;
		final long timestamp;
		final int transactionCounts;

		static Balance init() {
			return new Balance(BigDecimal.ZERO, 0, 0);
		}

	}


{% endhighlight %}

Now, we can test our solution.

Following the [documentation](https://github.com/spring-projects/spring-kafka/blob/master/src/reference/asciidoc/testing.adoc), we have to define some configuration beans.

{% highlight java %}

	
@Configuration
@EnableKafkaStreams
public class KafkaStreamsConfiguration {

	@Value("${" + KafkaEmbedded.SPRING_EMBEDDED_KAFKA_BROKERS + "}")
	private String brokerAddresses;

	@Bean
	public ProducerFactory<String, Transaction> producerFactory() {
		return new DefaultKafkaProducerFactory<>(producerConfig());
	}

	@Bean
	public Map<String, Object> producerConfig() {
		Map<String, Object> senderProps = KafkaTestUtils.senderProps(this.brokerAddresses);
		senderProps.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
		senderProps.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, JsonSerializer.class);
        // producer acks
		senderProps.put(ProducerConfig.ACKS_CONFIG, "all"); // strongest producing guarantee
		senderProps.put(ProducerConfig.RETRIES_CONFIG, "3");
		senderProps.put(ProducerConfig.LINGER_MS_CONFIG, "1");
		
        // leverage idempotent producer from Kafka 0.11 !
		//senderProps.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, "true"); // ensure we don't push duplicates
		
		senderProps.put(StreamsConfig.COMMIT_INTERVAL_MS_CONFIG, "100");
		return senderProps;
	}

	@Bean
	public KafkaTemplate<String, Transaction> kafkaTemplate() {
		return new KafkaTemplate<>(producerFactory());
	}

	@Bean(name = KafkaStreamsDefaultConfiguration.DEFAULT_STREAMS_CONFIG_BEAN_NAME)
	public StreamsConfig kStreamsConfigs() {
		Map<String, Object> props = new HashMap<>();
		props.put(StreamsConfig.APPLICATION_ID_CONFIG, "testStreams-once-bank");
		props.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, this.brokerAddresses);
		props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
		props.put(StreamsConfig.DEFAULT_KEY_SERDE_CLASS_CONFIG, Serdes.String().getClass());
		props.put(StreamsConfig.DEFAULT_VALUE_SERDE_CLASS_CONFIG, TransactionJsonSerde.class.getName());
		// Exactly once processing!!
		//props.put(StreamsConfig.PROCESSING_GUARANTEE_CONFIG, StreamsConfig.EXACTLY_ONCE);
		props.put(StreamsConfig.COMMIT_INTERVAL_MS_CONFIG, "100");
		return new StreamsConfig(props);
	}

	@Bean
	public Topology bankKStreamBuilder(BankBalanceKStreamBuilder streamBuilder) {
		return streamBuilder.build();
	}

	@Bean
	public BankBalanceKStreamBuilder bankstreamBuilder(StreamsBuilder streamBuilder) {
		return new BankBalanceKStreamBuilder(streamBuilder, BankBalanceKStreamBuilderTest.INPUT_TOPIC,
				BankBalanceKStreamBuilderTest.OUTPUT_TOPIC);
	}

	@Bean
	Consumer<String, String> consumerInput() {
		final Properties props = new Properties();
		props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, this.brokerAddresses);
		props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
		props.put(ConsumerConfig.GROUP_ID_CONFIG, "KafkaExampleConsumerInput-transactions");
		props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
		props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
		props.put(StreamsConfig.COMMIT_INTERVAL_MS_CONFIG, "100");
		// Create the consumer using props.
		final Consumer<String, String> consumer = new KafkaConsumer<>(props);
		// Subscribe to the topic.
		consumer.subscribe(Collections.singletonList(BankBalanceKStreamBuilderTest.INPUT_TOPIC));
		return consumer;
	}
	
	@Bean
	Consumer<String, String> consumerOutput() {
		final Properties props = new Properties();
		props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, this.brokerAddresses);
		props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
		props.put(ConsumerConfig.GROUP_ID_CONFIG, "KafkaExampleConsumerOutput-balance");
		props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
		props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
		props.put(StreamsConfig.COMMIT_INTERVAL_MS_CONFIG, "100");
		// Create the consumer using props.
		final Consumer<String, String> consumer = new KafkaConsumer<>(props);
		// Subscribe to the topic.
		consumer.subscribe(Collections.singletonList(BankBalanceKStreamBuilderTest.OUTPUT_TOPIC));
		return consumer;
	}

}


{% endhighlight %}


Once we have our configuration class fine, we can create our integration tests.

{% highlight java %}

	
@ExtendWith(SpringExtension.class)
@ContextConfiguration(classes = KafkaStreamsConfiguration.class)
@EmbeddedKafka(partitions = 1, topics = { BankBalanceKStreamBuilderTest.INPUT_TOPIC, BankBalanceKStreamBuilderTest.OUTPUT_TOPIC })
public class BankBalanceKStreamBuilderTest {

	public static final String INPUT_TOPIC = "input";
	public static final String OUTPUT_TOPIC = "output";

	@Autowired
	private KafkaTemplate<String, Transaction> template;

	@Autowired
	Consumer<String, String> consumerInput;

	@Autowired
	Consumer<String, String> consumerOutput;

	private final Executor executor = Executors.newCachedThreadPool();

	String[] names = new String[] { "David", "John", "Manuel", "Carl" };

	@Test
	@DisplayName("Given A Large Number Of Transactions In Concurrent Mode When The Stream Process All Messages Then All Balances Should Be Calculated")
	public void givenALargeNumberOfTransactionsInConcurrentModeWhenTheStreamProcessAllMessagesThenAllBalancesShouldBeCalculated()
			throws InterruptedException {
		int numberOfTransactions = 600;
		executor.execute(() -> {
			for (int i = 0; i < numberOfTransactions; i++) {
				Transaction t = new Transaction();
				t.setAmount(ThreadLocalRandom.current().nextDouble(0.0, 100.0));
				final String name = names[ThreadLocalRandom.current().nextInt(0, 4)]; //we have only 4 mocked customers.
				t.setName(name);
				t.setTimestamp(System.nanoTime());
				template.send(INPUT_TOPIC, name, t);
			}
		}); //we send the trasaction in a secondary thread.

		Awaitility.await().atMost(Duration.FIVE_MINUTES).pollInterval(new Duration(5, TimeUnit.SECONDS)).until(() -> {
			int messages = consumerInput.poll(1000).count();
			return messages == 0;
		}); // We assert al messages have been sent.

		assertEquals(4, consumerOutput.poll(1000).count()); //As we have 4 different customers, we should have only four messages in the output topic.
	}

}


{% endhighlight %}

Our tests is done, so we can verify everything is fine!

![Success](./img/kafkaIntegrationTest.png)

As we talked before, integration tests are very slower, and we can check this issue in the previous test. It takes 16 seconds on my machine, a huge amount of time.


### 5. Conclusion

If we want to develop a quality kafka streams we need to test the topologies and for that goal we can follow two approaches: kafka-tests and/or spring-kafka-tests. 
In my humble opinion, we should develop both strategies in order to tests as cases as possible always maintaining a balance between both testing strategies.
In this [Github Repo]((https://github.com/david-romero/kafka-streams-tests)), there is available the tests for scenario 3.

The full source code for this article is available over on [GitHub](https://github.com/david-romero/kafka-streams-tests).









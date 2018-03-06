---
layout: post
title:  "How to test a System.exit(1)"
date:   2018-03-06 12:00:00
categories: [Redis,Testing,Mockito,Spring]
comments: true
---

### Problem:

Let's suppose you have to implement a publisher with Redis and Spring Boot and you have to bringing down the instance when redis is unavailable.

The code could be something like that:

{% highlight java %}
import org.springframework.data.redis.RedisConnectionFailureException;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.stereotype.Service;

import lombok.extern.slf4j.Slf4j;

@Service
@Slf4j
public class MessagePublisherImpl implements MessagePublisher {

    private final StringRedisTemplate redisTemplate;

    public MessagePublisherImpl(final StringRedisTemplate redisTemplate) {
        this.redisTemplate = redisTemplate;
    }

    @Override
    public void publish(final String message) {
    	try {
    		redisTemplate.convertAndSend("myTopic", message);
    	} catch (RedisConnectionFailureException e) {
    		log.error("Unable to connect to redis. Bringing down the instance");
    		System.exit(1);
    	}
    }

}
{% endhighlight %}

This implementation is fine and It works as we expected but now, we have a problem...We want to test the code, specifically, we want to test booth branches in the publish method.

For that mision, first of all, I created two tests:

1. Given a message When is published Then is published by redis

{% highlight java %}

	@Test
	@DisplayName("Given a message When is published Then is published by redis")
	public void givenAMessageWhenIsPublishedThenIsPublishedByRedis() {
		// given
		final String message = "Hi!";

		// when
		messagePublisher.publish(message);

		// then
		verify(redisTemplate, times(1)).convertAndSend(anyString(), eq(message));
	} 
{% endhighlight %}

2. Given the Redis instance down When a message is published Then the container is bringing down

{% highlight java %}
	@Test
	@DisplayName("Given the Redis instance down When a message is published Then the container is bringing down")
	public void givenTheRedisInstanceDownWhenAMessageIsPublishedThenTheContainerIsBringingDown() {
		// given
		willThrow(new RedisConnectionFailureException("")).given(redisTemplate)
				.convertAndSend(anyString(), any());
		final String message = "Hi!";

		// when
		messagePublisher.publish(message);

		// then
		// ???
	}
{% endhighlight %}

That's right, It's seems to be fine, now, when we executed the tests something weird occurrs. 

![Unit tests are not ending]({{ "/img/test-system-post/exitedTests.png" | absolute_url }})

Our tests execution are not ending due to the System.exit introduced in the publisher class.

How could I fix this issue?

### Solution:

The most obvious answer is trying to mocking System.exit, so, let's go.

As the [javadoc](https://docs.oracle.com/javase/7/docs/api/java/lang/System.html#exit(int)) shows us, System.exit method calls the exit method in class Runtime, thus, we should put the focus on that method.

We could create a Runtime Spy and exchange it with the static variable of the Runtine class and when the second test ends, we could leave everything as it was before. 

Our test will be as following:

{% highlight java %}
	@Test
	@DisplayName("Given the Redis instance down When a message is published Then the container is bringing down")
	public void givenTheRedisInstanceDownWhenAMessageIsPublishedThenTheContainerIsBringingDown() throws Exception {
		// given
		willThrow(new RedisConnectionFailureException("")).given(redisTemplate).convertAndSend(anyString(), any());
		final String message = "Hi!";
		Runtime originalRuntime = Runtime.getRuntime();
		Runtime spyRuntime = spy(originalRuntime);
		doNothing().when(spyRuntime).exit(eq(1));
		setField(Runtime.class, "currentRuntime", spyRuntime);

		// when
		messagePublisher.publish(message);

		// then
		verify(spyRuntime, times(1)).exit(eq(1));
		setField(Runtime.class, "currentRuntime", originalRuntime);
	}

	private void setField(Class<?> clazz, String name, Object spy) throws Exception {
		final Field field = clazz.getDeclaredField(name);
		field.setAccessible(true);
		field.set(null, spy);
	}
{% endhighlight %}

With this improve, our test suite runs fine and we have added an assert to the tests something important and that it did not have before.

![Unit tests are fine]({{ "/img/test-system-post/successTests.png" | absolute_url }})


### Conclusions:

Thanks to Mockito and Java Reflection we can test almost all the cauisics that we find in our day to day.

The full source code for this article is available over on [GitHub](https://github.com/david-romero/testing-cases).


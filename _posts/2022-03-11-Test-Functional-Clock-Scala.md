---
layout: post
title:  "How to test Clock[F].realTimeInstant in Cats Effects 3.X"
date:   2022-03-11 09:00:00
categories: [Testing, Scala, Functional Programming, Cats, Cats Effects, Tutorial]
comments: true
---

## Introduction

Cats Effects has redesigned the type classes provided in version 3, and this aspect affects us in the way that we use the type class Clock[F].

![Type-Classes]({{ "/img/cats-type-classes.png" | absolute_url }})


## Context


In the previous version, if we wanted to use a Clock, we need to add in the bounded context in order to have a new implicit parameter. 
```scala
  private def failed[F[_]: Clock: Functor](error: Throwable): F[Item] = {
    Clock[F].instantNow.map { now =>
      copy(
        error = Error(error.getMessage, now)),
        state = EXCEPTION,
      )
    }
  }
```  

And, how to test the above code to verify that the output of the function returns the desired result?

Effectively, using mocks.

```scala

    it("should include the time and message when marked as failed") {
      val item          = ItemBuilder().succeeded().build()
      implicit val clock = mock[Clock[IO]]

      clock.realTime(*) returnsF TimeUnit.NANOSECONDS.convert(50, TimeUnit.SECONDS)

      val failedItem = item.failed(new RuntimeException("Test error")).unsafeRunSync()

      failedItem.state shouldBe EXCEPTION
      val error = failedItem.error
      error.message shouldBe "Test error"
      error.dateTime shouldBe LocalDateTime.from(Instant.ofEpochSecond(50).atZone(ZoneId.systemDefault()))
    }
```  


## Migrating to Cats Effects 3


Now, Let’s go to check how to use the Clock type class with the new version of the library.

```scala
class PurchaseAnItemUseCase[F[_]: Sync] {  

  private def generateItemPurchasedEvent(item: Item): F[PurchaseAnItem] = for {
      now <- Clock[F].realTimeInstant
      event = ItemPurchasedEvent(
          itemReference = item.identifier(),
          at = now
      )
    } yield event

}
```  

The looks quite similar but with a significant change, the Clock type class is not present in the bounded context.

Now, we have the problem, How I can inject a mocked clock?

Cats Effects provide a runtime that can be mocked to use with fibers, callbacks, and time support.

You need to add the following dependency to take advantage of this feature:

`libraryDependencies += "org.typelevel" %% "cats-effect-testkit" % "3.3.7" % Test`

With the TestControl class, you can “advance in the time” so, taking this into account, if you go to a certain period of time and use the `Clock[F].realTimeInstant`, in fact, we are mocking a timer.

Let’s check an example:

#### Scenario 1:


*Given an item, when the item is purchased, an event should be published.*

```scala

      private val duration: FiniteDuration = FiniteDuration(Instant.parse("2021-01-01T00:00:00Z").toEpochMilli, TimeUnit.MILLISECONDS)

      it("should publish an ItemPurchased event") {
        //given
        val item     = anItem()
        val expected = itemPurchasedEvent(item)
        gateway.doRequest(item) returnsF item.asRight[Throwable]
        eventPublisher.publish(expectedEvent) returnsF (())

        //when
        (TestControl.execute(subject.doPurchase(item)) flatMap { control =>
          for {
            _ <- control.advanceAndTick(duration)
            _ <- IO {
              //then
              eventPublisher.publish(expectedEvent) was called
            }
          } yield ()
        }).unsafeRunSync()
      }
```      

We’ll put the focus on the lines after the //when

* Decorate our subject or program with the test runtime provided by Cats.

* We have the power to do a journey through time. We are going to travel to 2021-01-01T00:00:00Z

* Add an assert wrapped into the IO context since we are using the IO context in the whole test.

* Execute the subject or program plus the time journey plus the assert.

**The main aspect to take into account is the TestControl class and the adavanceAndTick function.**

Okay, that seems a little verbose but okay, we can test our code to verify that an external component is invoked.

But now, let’s go further and verify the output of a function.

#### Scenario 2:

*Given an item, when the item is purchased, it should return the purchased item*


```scala
  private val duration: FiniteDuration = FiniteDuration(Instant.parse("2021-01-01T00:00:00Z").toEpochMilli, TimeUnit.MILLISECONDS)
  private val kleisli                  = new (Id ~> IO) { def apply[E](e: E): IO[E] = IO.pure(e) }     

     it("should return a confirmed item") {
        //given
        val item          = anItem()
        val expected      = itemPurchasedEvent(item)
        val confirmedItem = item.confirmed()

        gateway.doRequest(item) returnsF item.asRight[Throwable]
        eventPublisher.publish(expectedEvent) returnsF (())

        //when
        (TestControl.execute(subject.doPurchase(item)) flatMap { control =>
          for {
            _     <- control.advanceAndTick(duration)
            item  <- control.results
            _ <- IO {
              //then
              item.value.mapK(kleisli).embed(IO.pure(item.failed(TestingPurposeError))) shouldBe IO.pure(confirmedItem)
            }
          } yield ()
        }).unsafeRunSync()
      }
```

We’ll put the focus on the lines after the // when

* Decorate our subject or program with the test runtime provided by Cats.

* We have the power to do a journey through time. We are going to travel to 2021-01-01T00:00:00Z

* Obtain the result of our subject. The type of the item variable is Option[Outcome[Id, Throwable, Item]]

* Line 20: The magic is here: I’m going to describe each operation step by step.

    - Extract the value of the Option using OptionValues trait.

    - Using Kleisli, map the monad from Id to IO since we are using IO.

    - At this point, we have the following type: `Outcome[IO, Throwable, Item]` and we want to extract the Item object. The Outcome type class is designed to model fibers execution status.

    ```scala
    sealed trait Outcome[F[_], E, A]
    final case class Succeeded[F[_], E, A](fa: F[A]) extends Outcome[F, E, A]
    final case class Errored[F[_], E, A](e: E) extends Outcome[F, E, A]
    final case class Canceled[F[_], E, A]() extends Outcome[F, E, A]
    ```

    - The Outcome type class provides an embed function with the aim of returning the result of the fiber or a fallback provided just in case the fiber is canceled. So, with the following statement: `item.value.mapK(kleisli).embed(IO.pure(item.failed(TestingPurposeError)))` we are extracting the item from the fiber result wrapped into IO.

    - The final step is to add an assert to verify that the output is the desired one.


## TL;DR


- Using `TestControl.execute(F[A])` and `control.advanceAndTick(duration)` you can mock the time of the test execution.

- The test is decorated with an IO, so you have to execute unsafeRunSync at the end of the TestControl statement.

- To extract the value from the fiber could be a little tedious and verbose. `item.value.mapK(kleisli).embed(IO.pure(item.failed(TestingPurposeError)))`

- Kleisli is like a Monad transformer.
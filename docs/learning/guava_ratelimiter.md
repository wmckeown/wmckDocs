# Guava Ratelimiter

## Introduction
The `RateLimiter` class is a construct that allows us to regulate the rate at which some processing happens. If we create a `RateLimiter` with N permits - it means that process can issue at most N permits per second.

## Maven Dependency
We'll be using Guava's library:

```xml
<dependency>
    <groupId>com.google.guava</groupId>
    <artifactId>guava</artifactId>
    <version>32.1.3-jre</version>
</dependency>
```

 ## Creating an using `RateLimiter`
 Let's say that we want to limit the rate of execution of the `doSomeLimitedOperation()` to 2 times per second.

 We can create a `RateLimiter` instance using its `create()` factory method:
 ```java
 RateLimiter rateLimiter = RateLimiter.create(2);
 ```

Next, in order to get an execution permit from the `RateLimiter`, we need to call the `acquire()` method:
```java
rateLimiter.acquire(1);
```
In order to check that works, we'll make 2 subsequent calls to the throttled method:

```java
long startTime = ZonedDateTime.now().getSecond();
rateLimiter.acquire(1);
doSomeLimitedOperation();
rateLimiter.acquire(1);
doSomeLimitedOperation();
long elapsedTimeSeconds = ZonedDateTime.now().getSecond() - startTime;

```

To simplify our testing, let's assume that doSomeLimitedOperation() method is completing immediately.

In such a case, both invocations of the acquire() method should not block and the elapsed time should be less or below one second - because both permits can be acquired immediately:

```java
assertThat(elapsedTimeSeconds <= 1); 
```

Additionally we can acquire all permits in one `acquire()` call. 

```java
@Test
public void givenLimitedResource_whenRequestOnce_thenShouldPermitWithoutBlocking()  {
    // given
    var rateLimiter = RateLimiter.create(100);

    // when
    var startTime = ZonedDateTime.now().getSecond();
    rateLimiter.acquire(100);
    doSomeLimitedOperation();
    var elapsedTimeSeconds = ZonedDateTime.now().getSecond() - startTime;

    // then
    assertThat(elapsedTimeSeconds <= 1);
}

```

## Acquiring Permits in a Blocking Way
Now, let's consider a slightly more complex example.

We'll create a `RateLimiter` with 100 permits. Then we'll execute an action that needs to acquire 1000 permits.

According to the specification of the `RateLimiter`, such an action will need at least 10 seconds to complete because we're only able to execute 100 units of action per second:

```java
@Test
public void givenLimitedResource_whenUseRateLimiter_thenShouldLimitPersists()   {
        // given
    var rateLimiter = RateLimiter.create(100);

    // when
    var startTime = ZonedDateTime.now().getSecond();
    rateLimiter.acquire(100);
    IntStream.range(0,1000).forEach(i -> {
        rateLimiter.acquire();
        doSomeLimitedOperation();
    });
    doSomeLimitedOperation();
    var elapsedTimeSeconds = ZonedDateTime.now().getSecond() - startTime;

    // then
    assertThat(elapsedTimeSeconds <= 1);
}

```
Note, how we're using the `acquire()` method here - this is a blocking method and we should be cautious when using it. When the `acquire()` method gets called, it blocks the executing thread until a permit is available.

Calling the `acquire()` method without an argument is the same as calling it with a one as an argument (`acquire(1)`) - it will try to acquire one permit.

## Acquiring Permits with a Timeout
The `RateLimiter` API also has a very useful `acquire()` method that accepts a timeout and `TimeUnit` as arguments.

Calling this method when there are no available permits will cause it to wait for a specified time and then time out - if there are not not enough available permits within the timeout.

When there are no available permits within the given timeout, it returns `false`. If an `acquire()` succeeds, it returns `true`.

```java
@Test
public void givenLimitedResource_whenTryAcquire_shouldNotBlockIndefinitely()    {
    // given
    var rateLimiter = RateLimiter.create(1);

    // when 
    rateLimiter.acquire();
    var result = rateLimiter.tryAcquire(2,10,TimeUnit.MILLISECONDS);

    // then
    assertThat(result).isFalse();
}

```

We created a `RateLimiter` with one permit so trying to acquire two permits will always cause `tryAcquire()` to return false.

## Conclusion
We've had a quick look at the `RateLimiter` construct from the Guava library.

We learned how to use the `RateLimiter` to limit the number of permits per second. We saw how to use its blocking API and we also used an explicit timeout to acquire the permit.


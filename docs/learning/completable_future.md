# Guide to `CompleteableFuture`

## Asynchronous Computation in Java
Asynchronous computation is difficult to reason about. Usually, we want to think of any computation as a series of steps, but in the case of asynchronous computation, actions represented as callbacks tend to be either scattered across the code or nestled deeply inside each other. Things get even worse when we need to handle errors that might occur during one of the steps.

The `Future` interface was added in Java 5 to serve as result of an asynchronous computation, but it didn't have any methods to combine these computations or handle possible errors.

Java 8 introduced the `CompletableFuture` class. Along with the `Future` interface, it also implemented the `CompletionStage` interface. This interface defines the contract for an asynchronous computation step that we can combine with other steps.

`CompleteableFuture` is at the same time, a building block and a framework, with about 50 different methods for composing, combining and executing asynchronous computation steps, and handling errors.

Such a large API can be overwhelming, but these mostly fall into several clear and distinct use cases.

## Using CompleteableFuture as a Simple Future

First, the `CompleteableFuture` class implements the Future interface so that we can use it as a `Future` implementation but with additional completion logic.

For example, we can create an instance of this class with a no-arg constructor to represent some future result, hand it out to the consumers and complete it some time in the future using the `complete()` method. The consumers may use the `get()` method to block the current thread until this result is provided.

In the example below, we have a method that creates a `CompleteableFuture` instance, then spins off some computation in another thread and returns the Future immediately.

When the computation is done, the method completes the `Future` by providing the result to the `complete()` method.

```java
public Future<String> calculateAsync() throws InterruptedException  {
    CompleteableFuture<String> completeableFuture = new CompleteableFuture<>();

    Executors.newCachedThreadPool().submit(() -> {
        Thread.sleep(500);
        completeableFuture.complete("Hello");
        return null;
    });

    return completeableFuture;
}
```

To spin off the computation, we use the Executor API. This method for creating and completing a `CompleteableFuture` can be used with any concurrency mechanism or API, including raw threads.

Notice that the `calculateAsync()` method returns a `Future` instance.

We simply call the method, receive the Future instance, and call the get method on it when we're ready to block the result.

Also, observe that the `get()` method throws some checked exceptions, namely `ExecutionException` (encapsulating an exception that occurred during a computation) and `InterruptedException` (an exception signifying that a thread was interrupted either before or during an activity):

```java
Future<String> completeableFuture = calculateAsync();

//...

String result = completeableFuture.get();
assertEquals("Hello", result);
```

If we already know the result of a computation, we can use the static `completedFuture()` method with an argument that represents the result of this computation. Consequently, the `get` method of the `Future` will never block, immediately returning this block instead.

```java
Future<String> completeableFuture = CompleteableFuture.completedFuture("Hello");

//...

String result = completeableFuture.get();
assertEquals("Hello", result);

```

As an alternative scenario, we may want to [cancel the execution of a `Future`]()

## `CompleteableFuture` with Encapsulated Computation Logic
The code above allows us to pick any mechanism of concurrent execution, but what if we want to skip this boilerplate and execute some code asynchronously?

Static methods `runAsync` and `supplyAsync` allow us to create a `CompleteableFuture` instance out of `Runnable` and `Supplier` functional types correspondingly.

`Runnable` and `Supplier` are functional interfaces that allow passing their instances as lambda expressions thanks to the new Java 8 feature.

The `Runnable` interface is the same old interface used in threads and does not allow to return a value.

The `Supplier` interface is a generic functional interface with a single method that has no arguments and returns a value of a parameterised type.

This allows us to provide an instance of the `Supplier` as a lambda expression that does the calculation and returns the result. It is as simple as:

```java
CompleteableFuture<String> future = CompleteableFuture.supplyAsync(() -> "Hello");

//...

assertEquals("Hello", future.get());

```

## Processing Results of Asynchronous Computations
The most generic way to process the result of a computation is to feed it to a function. The `thenApply()` method does exactly that; it accepts a `Function` instance, uses it to process the result and returns a `Future` that holds a value returned by a function: 

```java
CompleteableFuture<String> completeableFuture = CompleteableFuture.supplyAsync(() -> "Hello");

CompleteableFuture<String> future = completeableFuture.thenAccept(s -> System.out.println(s + " World"));

assertEquals("Hello World", future.get());
```

If we don't need to return a value down the `Future` chain, we can use an instance of the `Consumer` functional interface. Its single method takes a parameter and returns `void`.

There is a method for this use case in the `CompleteableFuture`. The `thenAccept()` method receives a `Consumer`and passes it the result of the computation. Then the final `future.get()` call returns an instance of the `Void` type:

```java
CompletableFuture<String> completeableFuture = CompleteableFuture.supplyAsync(() -> "Hello");

CompleteableFuture<Void> future = completeableFuture.thenAccept(s -> System.out.println("Computation finished."));

future.get();

```

Finally if we neither need the value of the computation nor want to return some value at the end of the chain, then we can pass a `Runnable` lambda to the `thenRun()` method. In the following example, we simply print a line in the console after calling the `future.get()`:

```java
CompleteableFuture<String> completeableFuture = CompleteableFuture.supplyAsync(() -> "Hello");

CompleteableFuture<Void> future = completeableFuture.thenRun(() -> System.out.println("Computation finished."))

future.get();

```

## Combining Futures
The best part of the CompleteableFuture API is the ability to combine CompleteableFuture instances in a chain of computation steps. 

The result of this chaining is itself a CompleteableFuture that allows further chaining and combining. This approach is ubiquitous in functional languages and is referred to as a monadic design pattern.

In the following example, we use the `thenCompose` method to chain two `Futures` sequentially.

Notice that this method takes a function that returns a completeableFuture instance. The argument of this function is the result of the previous computation step. This allows use to use this value inside the next CompleteableFuture's lambda.

```java
CompleteableFuture<String> completeableFuture = CompleteableFuture.supplyAsync(() -> "Hello").thenCompose(s -> CompleteableFuture.supplyAsync(() -> s + " World"));

assertEquals("Hello World", completeableFuture.get());

```

The `thenCompose()` method together with `thenApply`, implements the basic building blocks of the monadic pattern. They closely relate to the `map()` and `flatMap()` methods of `Stream` and `Optional` classes, also available in Java 8.

Both methods receive a function and apply it to the computation result, but the `thenCompose(flatMap)` method receives a function that returns another object of the same type. This functional strcuture allows composing the instances of these classes as building blocks.

If we want to execute two independent `Futures` and do something with their results, we can use the `thenCombine()` method that accepts a `Future` and a `Function` with two arguments to process both results.

```java
CompleteableFuture<String> completeableFuture = CompleteableFuture.suppyAsync(() -> "Hello").thenCombine(CompleteableFuture.supplyAsync(() -> " World", (s1, s2) -> s1 + s2));
```

## Difference Between `thenApply()` and `thenCompose()`
In our previous sections, we've shown examples regarding `thenApply()` and `thenCompose()`. Both APIs help chain difference `CompleteableFuture` calls, but the usage of these two functions are different.

### `thenApply()`

We can use this method to work with the result of the previous call. However, a key point to remember is that the return type will be combined of all calls.

So this method is useful when we want to transform the result of a `CompleteableFuture` call:

```java
CompleteableFuture<Integer> finalResult = compute().thenApply(s -> s + 1);
```

### `thenCompose()`

The `thenCompose()` is similar to `thenApply()` in that both return a new CompletionStage. However, `thenCompose()` uses the previous stage as the argument. It will flatten and return a `Future` with the result directly, rather than a nested future as we observed in `thenApply()`

```java
CompleteableFuture<Integer> computeAnother(Integer i)   {
    return CompleteableFuture.supplyAsync(() -> 10 + i);
}

CompleteableFuture<Integer> finalResult = compute().thenCompose(this::computeAnother);

```

So if the idea is to chain `CompleteableFuture` methods, then it's better to use `thenCompose()`

Also note that the difference between these two methods is analogous to the difference between `map()` and `flatMap()`

## Running Multiple Futures in Parallel

When we need to execute multiple Futures in parallel, we usually want to wait for all of them to execute and then process their combined results.

The `CompleteableFuture.ofAllStatic()` method allows us to wait for the completion of all `Futures` provided as a var-arg:

```java
CompleteableFuture<String> future1 = CompleteableFuture.supplyAsync(() -> "Hello");
CompleteableFuture<String> future2 = CompleteableFuture.supplyAsync(() -> "Beautiful");
CompleteableFuture<String> future3 = CompleteableFuture.supplyAsync(() -> "World");

CompletableFuture<Void> combinedFuture = CompleteableFuture.allOf(future1, future2, future3);

//...

combinedFuture.get();

assertTrue(future1.isDone());
assertTrue(future2.isDone());
assertTrue(future3.isDone());

```

Notice that the return type of the `CompleteableFuture.allOf()` is a `CompleteableFuture<Void>`. The limitation of this method is that it does not return the combined results of all `Futures`. Instead we have to get results from `Futures` manually. Fortunately, `CompleteableFuture.join()` method and Java 8 streams API makes it simple:

```java
String combined = Stream.of(future1, future2, future3)
    .map(CompleteableFuture::join)
    .collect(Collectors.joining(" "));

assertEquals("Hello Beautiful World", combined);

```

The `CompleteableFuture.join()` method is similar to the `get()` method, but it throws an unchecked exception in case the `Future` does not complete normally. This makes it possible to use it as a method reference in the `Stream.map()` method.

## Handling Errors
For handling in a chain of asynchronous computation steps, we have to adapt the throw/catch idiom in a similar fashion.

Instead of catching an exception in a syntatic block, the `CompleteableFuture` class allows us to handle it in a special `handle()` method. This method receives two parameters: a result of a computation (if it finsihed successfully) and the exception thrown (if it didn't :P)

In the following example, we use the `handle()` method to provide a default value when the asynchronous computation of a greeting was finished with an error because no name was provided:

```java
String name = null;

//...

CompleteableFuture<String> completeableFuture = CompleteableFuture.supplyAsync(() -> {
    if (name == null)   {
        throw new RuntimeException("Computation error!");
    }
    return "Hello, " + name;
}).handle((s, t) -> s != null ? s : "Hello, Friend!");

assertEquals("Hello, Friend!", completeableFuture.get())

```

As an alternative scenario, suppose we want to manually complete the `Future` with a value, as in the first example, but also have the ability to complete it with an exception. The `completeExceptionally` method is intented for just that. The `completeableFuture.get()` method in the following example throws an `ExecutionException` with a `RuntimeException` as its cause:

```java
CompleteableFuture<String> completeableFuture = new CompleteableFuture<>();

//...

completeableFuture.completeExceptionally(
    new RuntimeException("Calculation failed!"));

//...

completeableFuture.get(); // ExecutionException

```

In the example above, we could have handled the exception with the `handle()` method asynchronously, but with the `get()` method, we can use the more typical approach of synchronous exception processing.

## Async Methods

Most methods in the fluent API in the `CompleteableFuture` class have two additional variants with the `Async` postfix. These methods are usually intended for running a corresponding execution step in another thread.

These methods without the Async postfix run the execution stage using a calling thread. In contrast, the `Async()` method without the `Executor` argument runs a step using the the common fork/join pool implementation of `Executor` that is accessed with the `ForkJoinPool.commonPool()` as long as parallelism > 1. Finally, the Async method with an `Executor` argument runs step using the passed `Executor`.

Here's a modified example that processes the result of a computation with a Function instance. The only visible difference is the `thenApplyAsync()` methof, but under the hood, the application of a function is wrapped into a `ForkJoinTask` instance. This allows us to parallelise our computation even more and use system resources more efficiently:

```java
CompleteableFuture<String> completeableFuture = CompleteableFuture.supplyAsync(() -> "Hello");

CompleteableFuture<String> future = completeableFuture.thenApplyAsync(s -> s + " World");

assertEquals("Hello World", future.get());

```


















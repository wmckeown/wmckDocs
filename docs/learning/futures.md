# Guide to `java.util.concurrent.future`
*Adapted from [https://www.baeldung.com/java-future](https://www.baeldung.com/java-future)*

`Future` is an interface that can be useful when working with asynchronous calls and concurrent processing.

## Creating Futures
The `Future` class represents a future result of an asynchronous computation. This result will eventually appear in the `Future` after the processing is complete.

Let's see how to write methods that create and return a `Future` instance.

Long-running methods are good candidates for asynchronous processing and the `Future` interface because we can execute other processes while we are waiting for the task encapsulated in the Future to complete.

Some examples of operations that would leverage the async nature of `Future` are:

* Computational intensive processes (mathematical and scientific calculations)
* Manipulating large data structures (big data)
* Remote method calls (downloading files, HTML scrapping, web services)

### Implementing Futures with FutureTask
For our example, we're going to create a very simple class that calculates the square of an Integer. This definitely doesn't fit in the long-running methods category, but we're going to put a `Thread.sleep()` call in it so that it lasts 1 second before completing.

```java
public class SquareCalculator   {

    private ExecutorService executor = Executors.newSingleThreadExecutor();

    public Future<Integer> calculate(Integer input) {
        return executor.submit(() -> {
            thread.sleep(1000)
            return input * input;
        });
    }
}
```

The bit of code that actually performs the calculation is contained within the `call()` method, and supplied as a lambda expression. As we can see, there's nothing special about it, except for the `sleep()` call mentioned earlier.

It gets more interesting when we direct our attention to the use of `Callable` and `ExecutorService`

`Callable` is an interface reprsenting a task that returns a result, and has a single `call()` method. Here, we've created an instance of it using a lambda expression.

Creating an instance of `Callable` doesn't actually take us anywhere, we still have to pass this instance to an executor that will take care of starting the task in a new thread, and give us back the valuable `Future` object. That's where the `ExecutorService` comes in.

There are a few ways we can access an ExecutorService instance, and most of them are provided by the utility class `Executor`'s static factory methods. In this example, we used the basic `newSingleThreadExecutor()`, which gives us an `ExecutorService` capable of handling a single thread at a time.

Once we have an `ExecutorService` object, we just need to call `submit()` passing our `Callable` as an argument. Then `submit()` will start the task and return a `FutureTask` object, which is an implementation of the `Future` interface.

## Consuming Futures

### Using `isDone()` and `get()` to Obtain Results
Now we need to call `calculate, and use the returned Future to get the resulting Integer. Two methods from the Future API will help us with this task.

`Future.isDone()` tells us if the executor has finished processing the task. If the task is complete, it will return true; otherwise it returns false.

The method that returns the actual result from the calculation is `Future.get()`. We can see that this method blocks the execution until the task is complete. However, this won't be an issue in our example because we'll check if the task is complete by calling `isDone()`.

By using these two methods, we can run other code while we wait for the main task to finish:

```java
Future<Integer> future = new SquareCalculator.calculate(10);

while(!future.isDone()) {
    System.out.println("Calculating...")
    Thread.sleep(300);
}

Integer result = future.get();
```

In this example, we'll write a simple message on the output to let the user know the program is performing the calculation.

The method `get()` will block the execution until until the task is complete. Again, this won't be an issue because in our example, `get()` will only be called after making sure the task has finished. So in this scenario, `future.get()` will always return immediately.

It's worth mentioning that `get()` has an overloaded version that takes a timeout and a `TimeUnit` as arguments:

```java
Integer result = future.get(500, TimeUnit.MILLISECONDS);
```

The difference between `get(long, TimeUnit)` and `get()` is that the former will throw a `TimeOutException` if the task doesn't return before the specified timeout period.

### Cancelling a Future with `cancel()`

Suppose we triggered a task, but for some reason, we don't care about the result anymore. We can use `Future.cancel(boolean)` to tell the executor to stop the operation and interrupt its underlying thread:

```java
Future<Integer> future  = new SquareCalculator().calculate(4);
boolean cancelled = future.cancel();
```

Our instance of Future, from the code above, will never complete its operation. In fact, if we try to call get() from that instance, after the call to cancel, the outcome would be a `CancellationException`. `Future.isCancelled` will tell us if a Future was already cancelled. This can be quite useful to avoid getting a `CancellationException`.

It's also possible that a call to `cancel()` fails. In that case, the returned value will be false. It's important to note that `cancel()` takes a boolean value as an argument. This controls whether the thread executing the task should be interrupted or not.

## More Multithreading with Thread Pools
Our current `ExecutorService` is single-threaded, since it was obtained with the `Executors.newSingleThreadExecutor`. To highlight this single thread, let's trigger two calculations simulataneously.

```java
SquareCalculator squareCalculator = new SquareCalculator();

Future<Integer> future1 = squareCalculator.calculate(10);
Future<Integer> future1 = squareCalculator.calculate(100);

while(!(future.isDone() && future2.isDone()))   {
    System.out.println(
        String.format(
            "future1 is %s & future2 is %s",
            future1.isDone() ? "done" : "not done",
            future2.isDone() ? "done" : "not done",
        )
    );
    Thread.sleep(300);
}

Integer result1 = future1.get();
Integer result2 = future2.get();

System.out.println(result1 + " and " + result2);

squareCalculator.shutdown();
```

Now let's look at the output for this code:

```bash
calculating square for: 10
future1 is not done and future2 is not done
future1 is not done and future2 is not done
future1 is not done and future2 is not done
future1 is not done and future2 is not done
calculating square for: 100
future1 is done and future2 is not done
future1 is done and future2 is not done
future1 is done and future2 is not done
100 and 10000
```

It's clear that the process isn't parallel. We can see that the second task only starts once the first task is complete making the whole process take around w seconds to finish.

To make our program really multithreaded, we should use a different flavour of `ExecutorService`. Let's see how the behaviour of our example changes if we use a thread pool provided by the factory method `Executors.newFixedThreadPool()`.

```java
public class SquareCalculator   {
    private ExecutorService executor = Executors.newFixedThreadPool(2);

    //...
}

```

With a simple change in our `SquareCalculator` class, we now have an executor which is able to use two simultaneous threads.

If we run the same client code again, we'll get the following output:

```bash
calculating square for: 10
calculating square for: 100
future1 is not done and future2 is not done
future1 is not done and future2 is not done
future1 is not done and future2 is not done
future1 is not done and future2 is not done
100 and 10000
```

This is looking much better now. We can see that the two tasks start and finish running simultaneously, and the whole process takes around one second to complete.

There are other factory methods that can be used to create thread pools, like `Executors.newCachedThreadPool()` which reused previously used Threads when they're available, and `Executors.newScheduledThreadPool()`, which schedules commands to run after a given delay.

## Overview of ForkJoinTask
`ForkJoinTask` is an abstract class which implements `Future`, and is capable of running a large number of tasks hosted by a small number of actual threads in `ForkJoinPool`.

The main characteristic of a `ForkJoinTask` is that it will usually spawn new subtasks as part of the work required to complete its main task. It generates new tasks by calling `fork()` and it gathers all results with `join()`, thus the name of the class.

There are two abstract classes that implement `ForkJoinTask`. `RecursiveTask`, which returns a value upon completion, and `RecursiveAction` which doesn't return anything. As their names imply, these classes are to be used for recursive tasks, such as file-system navigation or complex mathematical computation.

Let's expand our previous example to create a class that, given an Integer, will calculate the sum of squares for all of its factorial elements. So for instance, if we pass the number 4 into our calculator, we should get the result of 4^2 + 3^2 + 2^ + 1^2 which is 30.

First we need to create a concrete implementation of `RecursiveTask` and implement its `compute()` method. This is where we'll write our business logic:

```java
public class FactorialSquareCalculator extends RecursiveTask<Integer>   {
    private Integer n;

    public FactorialSquareCalculator(Integer n) {
        this.n = n;
    }

    @Override
    protected Integer compute() {
        if (n <= 1) {
            return n;
        }
    }

    FactorialSquareCalculator calculator = new FactorialSqaureCalculator(n - 1);

    calculator.fork();

    return n * n + calculator.join();
}
```

Notice how we achieve recursiveness by creating a new instance of `FactorialSquareCalculator` within `compute()`. By calling `fork()` a non-blocking method, we ask `ForkJoinPool` to initiate the execution of this subtask.

The `join()` method will return the result from that calculation, to which we'll add the square of the number we're currently visiting.

Now we just need to create a `ForkJoinPool` to handle the execution and thread management.

```java
ForkJoinPool forkJoinPool = new ForkJoinPool();

FactorialSquareCalculator calculator = new FactorialSquareCalculator(10);

forkJoinPool.execute(calculator);
```

## Conclusion

We have comprehensively explored the `Future` interface, touching on all of its methods. We also learned how to leverage the power of thread pools to trigger multiple parallel operations. The main methods from the `ForkJoinTask` class: `fork()` and `join()`, were briefly covered as well.


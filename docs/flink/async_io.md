# Flink Asynchronous I/O for External Data Access

## The need for Asynchronous I/O operations
When interacting with external systems (e.g. when enriching stream events with data stored in a database), one needs to take care of that communication delay with the external system does not dominate the streaming application's total work.

Natively accessing data in the external database, for example in a `MapFunction`, typically means synchronous interaction: A request is sent to the database and the `MapFunction` waits until the response has been received. In many cases, this waiting makes up the vast majority of the function's time.

Asynchronous interaction with the database means that a single parallel function instance can handle many requests concurrently. That way, the waiting time can be overlaid with the sending other requests and receiving responses. At the very least, the waiting time is amortized over many requests. This leads in most cases to much higher streaming output.

// Diagram Goes Here

!!! Note
        Improving throughput by just scaling the `Map Function` to a very high parallelism is in some cases possible as well, but usually comes at a high resource cost: Having many more parallel parallel `MapFunction` instances means more tasks, threads, Flink internal-network connections, connections to the database, buffers and general internal bookkeeping overhead.

## Pre-requisites
As illustrated above, implemented proper asynchronous I/O to a database (or key/value) store requires a client to that database that supports asynchronous requests. Many popular databases offer such a client.

In the absence of such a client, one can try and turn a synchronous client into a limited concurrent client by creating multiple clients and handling the synchronous calls with a thread pool. However, this approach is usually less efficient than a proper asynchronous client.

## Async I/O API
Flink's Async I/O API allows users to use asynchronous request clients with data streams. The API handles the integration with data streams, as well as handling order, event time, fault tolerance, retry support, etc.

Assuming one has an asynchronous client for the target database, three parts are needed to implement a stream transformation with asynchronous I/O against the database:

* An implementation of `AsyncFunction` that dispatches the requests
* A `callback` that takes the result of the operation and hands it to the `ResultFuture`
* Applying the async I/O operation on a DataStream as a transformation with or without retry

The following code illustrates the basic pattern:

```java
// This example implements the asynchronous request and callback with Futures that have the interface
// of Java 8 Futures (which is the same as Flink Futures)

/**
* An implementation of the `AsyncFunction` that sends requests and sets the callback
*/
class AsyncDatabaseRequest extends RichAsyncFunction<String, Tuple2<String, String>>    {

    /** The database-specific client that can issue concurrent requests within callbacks */
    private transient DatabaseClient client;

    @Override
    public void open(Configuration parameters) throws Exception {
        client = new DatabaseClient(host, post, credentials);
    }

    @Override
    public void close() throws Exception    {
        client.close()
    }

    @Override
    public void asyncInvoke(String key, final ResultFuture<Tuple2<String, String>> resultFuture) throws Exception   {
        
        // issue the asynchronous request, recieve a future for result
        final Future<String> result = client.query(key);

        // Set the callback to executed once the request by the client is complete
        // the callback simply forwards the result to the ResultFuture
        CompleteableFuture.supplyAsync(new Supplier<String>()   {

            @Override
            public String get() {
                try {
                    return result.get();
                } catch (InterruptedException | ExecutionException e)   {
                    // Normally handled explicitly
                    return null;
                }
            }
        }).thenAccept((String dbResult) -> {
            resultFuture.complete(Collections.singleton(new Tuple2<>(key, dbResult)));
        });
    }

    // Create the original stream
    DataStream<String> stream = ...;

    // Apply the Async I/O transformation without retry
    DataStream<Tuple2<String,String>> resultStream = 
        AsyncDataStream.unorderedWait(stream, new AsyncDatabaseRequest(), 1000, TimeUnit.MILLISECONDS, 100);

    // Or Apply the async I/O transformation with retry
    // create an async retry strategy via utility class or a user defined strategy
    AsyncRetryStrategy asyncRetryStrategy = 
        new AsyncRetryStrategies.FixedDelayRetryStrategyBuilder(3, 100L) // maxAttempts=3 fixedDelay=100ms
            .ifResult(RetryPredicates.EMPTY_RESULT_PREDICATE)
            .ifException(RetryPredicates.HAS_EXCEPTION_PREDICATE)
            .build()

    // Apply the async I/O transformation with retry
    DataStream<Tuple2<String, String>> resultStream = 
        AsyncDataStream.unorderedWaitWithRetry(stream, new AsyncDatabaseRequest(), 1000, TimeUnit.MILLISECONDS, 100 asyncRetryStrategy);
}
```

!!! Important
        The `ResultFuture` is completed with the first call of the `ResultFuture.complete()`. All subsequent `complete()` calls will be ignored.

The following parameters control the asynchronous operations:

* **Timeout**: The timeout defines how long an async operation can take before it is finally considered failed, may include multiple retry requests, if retry is enabled. The parameter guards against dead/failed requests
* **Capacity**: Capacity defines how many async requests may be in progress at the same time,. Even though the async I/O approach typically leads to better throughput, the operator can still be the bottleneck in the streaming application. Limiting the number of concurrent requests ensures that the operator will not accumulate an ever-growing backlog of pending requests, but that will trigger backpressure once the capacity is exhausted.
* **AsyncRetryStrategy**: The `asyncRetryStrategy` defines what conditions will trigger a delayed retry and the delay strategy e.g. fixed-delay, exponential-backoff-delay, custom, etc.

### Timeout Handling
When an async I/O request times out, by default an exception is thrown and the job is restarted. If you want to handle timeouts, you can override the `AsyncFunction#Timeout` method. Make sure you call `ResultFuture.complete()` or `ResultFuture.completeExceptionally()` when overriding in order to indicate to Flink that the processing of this input record is completed. You can call `ResultFuture.complete(Collections.emptyList())` if you do not want to emit any record when timeouts happen.

### Order of Results
The concurrent requests issued by the `AsyncFunction` frequently complete in some undefined order, based on which request finished first. To control in which order the resulting records are emitted, Flink offers two modes.

* **Unordered**: Watermarks do not overtake records and vice-versa, meaning watermarks establish an order boundary. Records are emitted unordered only between watermarks. A record occurring after a certain watermark will be emitted only after that watermark was emitted. The watermark in turn will be emitted only after all result records from inputs before that watermark were enabled.
    * That means that in the presence of watermarks, the *unordered* mode introduces some of the the same latency and management overhead that the ordered mode does. The amount of overhead depends on the watermark frequency.
* **Ordered**: Order of watermarks and records is preserved just like the order between records is preserved. There is no significant change in overhead, compared to working processing time.

Recall that *Ingestion Time* is a special case of *event time* with automatically generated watermarks that are based on the source's processing time.

### Fault Tolerance Guarantees
The asynchronous I/O operator offers full exactly-once fault tolerance guarantees. It stores the records for in-flight asynchronous requests in checkpoints and restores/re-triggers the requests when recovering from a failure.

### Retry Support
The retry support introduces a built-in mechanism for async operator which exists transparently to the user's Async Function.

* **AsyncRetryStrategy**: The `AsyncRetryStrategy` contains the definition of the retry condition `AsyncRetryPredicate` and the interfaces to determine whether to consume retry and the retry interval based on the current attempt number. Note that after the trigger retry condition is met, it is possible to abandon the retry because the current attempt number exceeds the preset limit, or to be forced to terminate the retry at the end of the task (in this case, the system takes the last execution result or exception as the final state).
* **AsyncRetryPredicate**: The retry condition can be triggered based on the return result or the execution exception.

### Implementation Tips
For implementations with `Future`s that have an `Executor` for callbacks, we suggest using a `DirectExecutor`, because the callback typically does minimal work, and a `DirectExecutor` avoids an additional thread-to-thread handover overhead. The callback typically only hands the result to the `ResultFuture`, which adds it to the output buffer. From there, the heavy logic that includes record emission and interaction with the checkpoint bookkeeping happens in a dedicated thread pool anyways.

A `DirectExecutor` can be obtained via `org.apache.flink.util.concurrent.Executors.directExecutor()` or `com.google.common.util.concurret.MoreExectors.directExecutor()`.

### Caveats
#### The `AsyncFunction` is not multi-threaded
A common confusion that we want to explicitly point out here is that the `AsyncFunction` is not called in a multi-threaded fashion. There exists only one instance of `AsyncFunction` and it is called sequentially for each record in the respective partition of the stream. Unless the `asyncInvoke(...)` method returns fast and relies on a callback (by the client), it will not result in proper asynchronous I/O.

For example, the following patterns result in a blocking `asyncInvoke(...)` functions and thus void the asynchronous behaviour:

* Using a database client whose lookup/query method call blocks until the result is received back
* Blocking/waiting on the future-type objects by an asynchronous client inside the `asyncInvoke(...)` method

**An AsyncFunction(AsyncWaitOperator) can be used anywhere in the job graph, except that it cannot be chained to a `SourceFunction/SourceStreamClass`.**

#### May Need Larger Queue Capacity if Retry Enabled
The new retry feature may result in larger queue capacity requirements, the maximum number can be approximately valued as below:

```bash
inputRate * retryRate * averageRetryDuration
```

For example, for a task with inputRate = 100 records/s, where 1$ of the elements will trigger 1 retry on average, and the average retry time is 60s, the additional queue capacity requirement will be:

```
100 records/s * 0.01 * 60s = 60
```

That is, adding 60 more capacity to the work queue may not affect the throughput in unordered output mode, in the case of ordered mode, the head element is the key point, and the longer it stays uncompleted, the longer the processing delay provided by the operator, the retry feature may increase the incomplete time of the head element, if in fact more retries are obtained with the same timeout constraint.

When the queue capacity grows (common way to ease backpressure), the risk of OOM increases. Though in fact, for `ListState` storage, the theoretical upper limit is `Integer.MAX_VALUE`, so the queue capacity's limit is the same, but we can't increase the queue capacity too big in production. Increasing task parallelism may be a more viable way.
    






# Learn Flink: Hands-On Training

This page covers:

* How to implement streaming data processing pipelines
* How and why Flink manages state
* How to use event time to consistently compute accurate analytics
* How to build event-driven applications on continuous streams
* How Flink is able to provide fault-tolerant, stateful stream processing with exactly once semantics

The main focus is on four critical concepts: continuous processing of streaing data, event time, stateful stream processing and state snapshots.

## Stream Processing
Streams are data's natural habitat. Whether it is events from web servers, trades from a stock exchange, or sensor readings from a machine on a factrt floor, data is created as part of a stream. But when you analyse data, you either organise your processing around `bounded` or `unbounded` streams and which of these paradigms you choose has profound consequences.

![Flink Stream Diagram](img/flink/stream.png)

Batch processing is the paradigm at work when you process a bounded data stream. In this mode of operation, you can choose to ingest the entire dataset before producing any results, which mean that it is possible, for example, to sort the data, compute global statistics, or produce a final report that summarises all of the input. 

Stream Processing involves unbounded data streams streams. Conceptually, at least, the input may never end, and so you are forced to continuously process the data as it arrives.

In Flink, applications are composed of streaming dataflows that may be defined by user-defined operators. These dataflows form directed graphs that start with one-or-more sources and end in one-or-more sinks.

```java
DataStream<String> lines = env.addSource(new FlinkKafkaConsumer<>(...)); // Source

DataStream<Event> events = lines.map((line) -> parse(line));             // Transformation

DataStream<Statistics> stats = events                                    // Transformation
    .keyBy(event -> event.id)
    .timeWindow(Time.seconds(10))
    .apply(new MyWindowAggregationFunction());

stats.addSink(new mySink(...));                                          // Sink
```

![Flink Streaming Dataflow](img/flink/streaming_dataflow.png)

Often there is a one-to-one correspondence between the transformations in the program and the operators in the dataflow. Sometimes however, one transfomration may consist of multiple operators.

An application may consume real-time data from streaming sources such as message queues or distributed logs, like Apache Kafka or Kinesis. But Flink can also consume bounded, historic data from a variety of data sources. Similarly, the streams of results being produced by a Flink application can be sent to a wide variety of systems that can be connected as sinks.

## Parallel Dataflows
Programs in Flink are inherently parallel and distrubuted. During execution, a stream has one or more stream partitions, and each operator has one or more operator subtasks. The operator subtasks are independent of one-another, and execute in different threads and possibly on different machines or containers.

The number of operator subtasks is the parallelism of that particular operator, different operators of the same program may have different levels of parallelism.

![Flink Streaming Parallelism](img/flink/stream_parallelism.png)

Streams can transport data between two operators in a one-to-one (or forwarding) pattern, or in a redistributing pattern:

* **One-to-one** streams (for example between the Source and the map() operators in the figure above) preserve the partitioning and ordering of the elements. That means that subtask[1] of the map() operator will see the same elements in the same order as they were produced by subtask[1] of the Source operator.
* **Redistributing** streams (as between map() and keyBy/window above, as well as between keyBy/window and Sink) change the partitioning of streams. Each operator subtask sends data to different target subtasks, depending on the selected transformation. Examples are keyBy() (which re-partitions by hashing the key), broadcast() or rebalance() (which re-partitions randomly). In a redistributing exchange the ordering among the elements is only preserved within each pair of sending and receiving subtasks (for example, subtask[1] of map() and subtask[2] of keyBy/window). So, for example, the redistribution between the keyBy/window and the Sink operators shown above introduces non-determinism regarding the order in which the aggregated results for different keys arrive at the Sink.

## Timely Stream Processing
For most streaming applications it is very valuable to be able to re-process historic data with the same code that is used to porcess live data - and to produce deterministic, consistent results regardless.

It can also be crucial to pay attention to the order in which events occurred, rather than the order in which they are delivered for processing, and to be able to reason about when a set of events is (or should be) complete. For example, consider the set of events involved in an e-commerce transaction or a financial trade.

These requirements for timely stream processing can be met by using event time timestamps that are recorded in the data stream, rather than using the clocks of the machines processing the data

## Stateful Stream Processing
Flink's operations can be stateful. This means that how one event is handled can depend on the accumulated effect of all events that came before it. State may be used to for something simple, such as counting events per minute to display on a dashboard, or for something more complex, such as computing features for a fraud-detection model.

A flink application is run in parallel on a distributed cluster. The various parallel instances of a given operator will execute independently, In seperate threads, and in general, will be running on different machines.

The set of parallel instances of a stateful operator is effectively a sharded key-value store. Each parallel instance is responsible for handling events for a specific group of keys, and the state of those keys is kept locally.

The diagram below shows a job running with a parallelism of two across the first three operators in the job graph, terminiating in a sink that has a parallelism of one. The third operator is stateful,a nd you can see that a full-connected network shuffle is occurring between the second and third operators. This is being done to partition the stream by some key, so that all the events that need to be processed together, will be.

![Stateful Processing High-Level Overview](img/flink/stateful_processing.png)

State is always accessed locally, which helps Flink applications achieve high throughput and low-latency. You can choose to keep state on the JVM heap, or if it is too large, in efficiently organised on-disk data structures.

![Stateful Processing Detail Overview](img/flink/stateful_processing_2.png)

## Fault Tolerance via State Snapshots

Flink is able to provide fault-tolerant, excatly once semantics through a combination of state snapshots and stream replay. These snapshots capture the entire state of the distributed pipeline, recording offsets into the input queues as well as the state throughout the job graph that has resulted from having ingested the data up to that point. When a failure occurs, the sources are rewound, the state is restored and processing is resumed. As depicted above, these state snapshots are captured asynchronously, without impeding the ongoing processing.

# Intro to the DataStream API

## What can be streamed?
Flink's DataStream APIs for Java and Scala will ley you stream anything you can serialise. Flink's own serialiser is used for:

* basic types i.e. String, Long, Boolean, Integer, Array
* composite types i.e Tuples, POJOs and Scala use classes

and Flink falls back to Kryo for other types.

### Java tuples and POJOs
Flink's native serialiser can operate efficiently on Tuples and POJOs

#### Tuples
For Java, Flink defines its own `Tuple0` thru `Tuple25` types.

```java
Tuple2<String, Integer> person = Tuple2.of("Fred", 35);

// zero based index
String name = person.f0;
Integer age = person.f1;
```

#### POJOs
Flink recognises a data type as a POJO type (and allows "by-name" field referencing) if the following conditions are fulfilled:

* The class is pubic and standalone (no non-static inner class)
* The class has a public no-argument constructor
* All non-static, non-transient fields in the class (and all superclasses) are either public (and non-final) or have public getter- and setter- methods that follow the Java beans naming conventions for getters and setters.

Example
```java
public class Person {
    public String name;
    public Integer age;
    public Person() {}
    public Person(String name, Integer age) {
        ...
    }
}

Person person = new Person("Eliot Aldersen", 32);
```

## A Complete Example
This example takes a string of records about people as input, and filters it to only include the adults.

```java
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.api.common.functions.FilterFunction;

public class Example {

    public static void main(String[] args) throws Exception {
        final StreamExecutionEnvironment env =
                StreamExecutionEnvironment.getExecutionEnvironment();

        DataStream<Person> flintstones = env.fromElements(
                new Person("Fred", 35),
                new Person("Wilma", 35),
                new Person("Pebbles", 2));

        DataStream<Person> adults = flintstones.filter(new FilterFunction<Person>() {
            @Override
            public boolean filter(Person person) throws Exception {
                return person.age >= 18;
            }
        });

        adults.print();

        env.execute();
    }

    public static class Person {
        public String name;
        public Integer age;
        public Person() {}

        public Person(String name, Integer age) {
            this.name = name;
            this.age = age;
        }

        public String toString() {
            return this.name.toString() + ": age " + this.age.toString();
        }
    }
}
```
### Stream execution environment
Every Flink application needs an execution environment, `env` in this example. Streaming applications need to use a `StreamExecutionEnvironment`

The DataStream API calls made in your application build a job graph tat is attached to the `StreamExecutionEnvironment`. When `env.execute()` is called, the graoh is packaged up and sent to the JobManager which parallelises the job and distributes slices of it to the Task Managers for execution. Each parallel slice of your job will be executed in a _task slot_.

Note that if you don't call execute, your application won't run.

![Flink Stream Execution Environment](img/flink/stream_execution_env.png)

The distributed runtime depends on your application being serialisable. It also requires that all dependencies are available to each node in the cluster

### Basic stream sources
The example above constructs a `DataStream<Person>` using `env.FromElements(...)`. This is a convenient way to thrw together a simple stream for use in a prototype or test. There is also a `fromCollection(Collection)` method on `StreamExecutionEnvironment`. So instead, you could do this:

```java
List<Person> people = new ArrayList<Person>();

people.add(new Person("Fred", 35));
people.add(new Person("Wilma", 35));
people.add(new Person("Pebbles", 2));

DataStream<Person> flintstones = env.fromCollection(people);
```

Another convenient way to get some data into a stream whilst prototyping is to use a socket:

```java
DataStream<String> lines = env.socketTextStream("localhost", 9999);
```

or a file: 
```java
DataStream<String> lines = env.readTextFile("file:///path");
```

In real applications, the most commonly used data sources are those that support low-latency, high throughput parallel reads in combination with rewinf and replay - the pre-requisites for high performance and fault tolerance - such as Apache Kafka, Kinesis and various filesystems. REST APIs and databases are also frequently used for stream encrichment.

### Basic stream sinks

The example above uses `adults.print()` to print its results to the task manager logs (which will appear in your IDE's console when running on an IDE). This will call `toString()` on each element of the stream.

The output looks something like this:
```
1> Fred: age 35
2> Wilma: age 35
```
Where `1>` and `2>` indicate which sub-task (i.e. thread) produced the output.
In production, commonly used sinks include the `StreamingFileSink`, various databases and several publish-subscribe (pub-sub) services.

### Debugging

In production, your application will run a remote clusteror a set of containers. And if it fails, it will fail remotely. The JobManager and TaskManager logs can be very helpful in debugging such failures, but it is much easier to do locsl debugging inside an IDE, which is something that Flink supports. You can set breakpoints, examine local variables and step through your code. You can also step into Flink's code, which can be a great way to learn more about its internals if you are curious to see how Flink works.


### Understanding the Flink jobs


Reactor map reduce over multiple threads

Publishers
Mono 0 - 1 items
Flux 0 - N itens

Cleaner completeable futures

FUtures

ETL Etract Transform Load

Hadoop - Old school ETL
isseue is it's file based

Spark (memory based micro batches)

Flink (Memory based, streaming near realitmime, never better syntax)

Segment replacemengt protocol
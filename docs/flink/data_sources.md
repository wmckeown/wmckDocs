# Data Sources
This page describes Flink's Data Source API and the concepts and architecture behind it.

## Data Source Concepts

### Core Components
A Data Source has three core components: Splits, the SplitEnumerator, and the SourceReader.

* A **Split** is a portion of data consumed by the source, like a file or a log partition. Splits are the granularity by which the source distributes the work and parallelises reading data.
* The **SourceReader** requests *Splits* and processes them, for example by reading the file or log partition represented by the Split. The *SourceReaders* run in parallel on the TaskManagers in the `SourceOperators` and produce the parallel stream of events/records
* The **SplitEnumerator** generates the *Splits* and assigns them to the *SourceReaders*. It runs as a single instace on the Job Manager and is responsible for maintaining the backlog of pending Splits and assigning them to the readers in a balanced manner.

The `Source` class is the API entry point that ties the above three components together.

### Unified across streaming and batch
The Data Source API supports both bounded batch sources (what we use) and unbounded streaming sources in a unified way.

In the bounded batch case, the enumerator generates a fixed source, and each split is necessarily finite.

## Examples
### Bounded File Source
The source has a URI/Path of a directory to read, and a Format that defines how to parse the files.

* A *Split* is a file, or a region of a file (if the data format supports splitting the file)
* The *SplitEnumerator* lists all files under a given directory path. It assigns Splits to the next reader that requests a Split. Once all Splits are assigned, it responds to requests with *NoMoreSplits*
* The *SourceReader* requests a Split and reads the assigned Split (file or file region) and parses it using the given Format. If it does not get another Split, but a NoMoreSplits message, it finishes. 

## The Data Source API

### Source
The Source API is a factory style interface create the following components.

* Split Enumerator
* Source Reader
* Split Serializer
* Enumerator Checkpoint Serializer

The Source also provides the boundedness atribute of the source (In our case, a `BOUNDED` stream is a stream with finite records), so that Flink can choose the appropriate mode to run the Flink jobs.

The Source implementations should be serializable as the Source instances are serialized and uploaded to the Flink cluster at runtime.

## SplitEnumerator
The SplitEnumerator is expected to be the "brain" of the Source. Typical implementations of the `SplitEnumerator` do the following:

* SourceReader registration handling
* SourceReader failure handling
  * The `addSplitsBack()` method will be invoked when a `SourceReader` fails. The `SplitEnumerator` should take back the split assignments that have not yet been acknowledged by the failed `SourceReader`
* `SourceEvent` handling
  * SourceEvents are custom events sent between SplitEnumerator and SourceReader. The implementation can leverage this mechanism to perform sophisticated coordination.
* Split discovery and assignment
  * The `SplitEnumerator` can assign splits to the SourceReaders in response to various events, including discovery of new splits, new `SourceReader` registration, `SourceReader` failure, etc.

A `SplitEnumerator` can accompish the above work with the help of the `SplitEnumeratorContext` which is provided to the `Source` on creation or restore of the `SplitEnumerator`. The `SplitEnumeratorContext` allows a `SplitEnumerator` to retrieve necessary information of the readers and perform coordination actions. The `Source` implementation is expected to pass the `SplitEnumeratorContext` to the `SplitEnumeratorInstance`.

While a `SplitEnumerator` implementation can work well in a reactive way by only taking the coordination actions when its method is invoked, some `SplitEnumerator` implementations might want to take actions actively. For example, a `SplitEnumerator` may want to periodically run split doscovery and assign new splits to the `SourceReaders`. Such implementations may find the `callAsync()` method in the SplitEnumeratorContext handy.

The below code snippet shows how the `SplitEnumerator` implementation can achieve that without maintaining its own threads:

```java
class MySplitEnumerator implements SplitEnumerator<MySplit, MyCheckpoint> {
    private final long DISCOVER_INTERVAL = 60_000L;

    private final SplitEnumeratorContext<MySplit> enumContext            ;

    /** The Source creates instances of SplitEnumerator and provides the context. */
    MySplitEnumerator(SplitEnumeratorContext<MySplit> enumContext) {
        this.enumContext = enumContext;
    }

    /**
     * A method to discover the splits.
     */
    private List<MySplit> discoverSplits() {...}
    
    @Override
    public void start() {
        ...
        enumContext.callAsync(this::discoverSplits, (splits, thrown) -> {
            Map<Integer, List<MySplit>> assignments = new HashMap<>();
            int parallelism = enumContext.currentParallelism();
            for (MySplit split : splits) {
                int owner = split.splitId().hashCode() % parallelism;
                assignments.computeIfAbsent(owner, s -> new ArrayList<>()).add(split);
            }
            enumContext.assignSplits(new SplitsAssignment<>(assignments));
        }, 0L, DISCOVER_INTERVAL);
        ...
    }
    ...
}

```

### SourceReader
The `SourceReader` is a component running in the Task Managers to consumer the records from the Splits.

The `SourceReader` exposes a pull-based consumption interface. A Flink task keeps calling `pollNext(ReaderOutput)` in a loop to poll records from the `SourceReader`. The return value of the `pollNext(ReaderOutput)` method indicates the status of the source reader.

* MORE_AVAILABLE - The SourceReader has more records available immediately
* NOTHING_AVILABLE - The SourceReader does not have more records avialble at this point, but may have more records in the future.
* END_OF_INPUT - The SourceReader has exhausted all the records and reached the end of data. This means the SourceReader can be closed.

In the interest of performance, a `ReaderOutput` is provided to the `pollNext(ReaderOutput)` method, so a `SourceReader` can emit multiple records in a single call of `pollNext()` if it has to. For example, sometimes the external system works on the granularity of blocks.  A block may contain multiple records but the source can only checkpoint at the block boundaries. In this case, the `SourceReader` can emit all the records in one block at a time to the `ReaderOutput`. However, the `SourceReader` implementation should avoid emitting multiple records in a single `pollNext(ReaderOutput)` invocation unless necessary. This is because the task thread that is polling from the `SourceReader` works in an event-loop and cannot block.

All the state of a `SourceReader` should be maintained inside the `SourceSplits` which are returned at the `snapshotState()` invocation. Doing this allows the `SourceSplits` to be reassigned to other `SourceReader`s when needed.

A `SourceReaderContext` is provided to the `Source` upon a `SourceReader` creation. It is expected that the `Source` will pass the context to the `SourceReader` instance. The `SourceReader` instance can send `SourceEvent` to its `SplitEnumerator` through the `SourceReaderContext`. A typical design pattern of the `Source` is letting the `SourceReaders` report their local information to the `SplitEnumerator` who has a global view to make decisions.

The `SourceReader` API is a low-level API that allows users to deal with the splits manually and have their own threading model to fetch and handover the records. To facilitate the `SourceReader` implementation, Flink has provided a `SourceReaderBase` class which significantly cuts down on the boilerplate code needed to write a `SourceReader`. It's highly recommended for the connector developers to take advantage of the `SourceReaderBase` instead of writing the `SourceReader`s from scratch.

### Luke, Use the Source!

In order to create a `DataStream` from a `Source`, one needs to pass the `Source` to a `StreamExecutionEnvironment`:

```java
final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

Source mySource = new MySource(...);

DataStream<Integer> stream = env.fromSource(
    mySource,
    WatermarkStrategy.noWatermarks(),
    "MySourceName"
);

```

## The Split Reader API

The core `SourceReader` API is fully asynchronous and requires implementations to manually manage reading splits asynchronously. However, in practice most sources perform blocking operations, like blocking `poll()` calls on clients (for example the `KafkaConsumer`), or blocking I/O operations on distributed file systems (HDFS, S3, ...). To make this compatible with the asynchronous Source API, these blocking (synchronous) operations need to happen in seperate threads, which hand over data to the asynchronous part of the reader.

The `SplitReader` is the high-level API for simple synchronous reading/polling-based source implementations, like file-reading, Kafka etc.

The core is the `SourceReaderBase` class, which takes a `SplitReader` and creates fetcher threads running the `SplitReader`, supporting different consumption threading models.

### SplitReader
The `SplitReader` API only has three methods:

* A blocking fetch to return a `RecordsWithSplitIds`
* A non-blocking method to handle split changes
* A non-blocking wake-up method to wake-up the blocking fetch operation

The `SplitReader` only focuses on reading the records from the external system, therefore is much simpler compared with `SourceReader`.

### SourceReaderBase

It is quite common that a `SourceReader` implementation does the following:

* Have a pool of threads fetching from splits of the external system in a blocking way
* 









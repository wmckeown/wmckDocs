## Trino Server Types
There are two types of Trino servers: **Coordinators** and **Workers**

### Coordinator
The Trino coordinator is the server that is responsible for parsing statements, planning queries and managing Trino worker nodes. It is the "brain" of a Trino installation and is also the node to which a client connects to submit statements for execution. Every Trino installation must have a Trino coordinator alongside one or more Trino workers. For development or testing purposes, a single instance of Trino can be configured to perform both roles.

The coordinator keeps track of the activity of each worker and coordinates the execution of a query. The coordinator creates a logical model of a query invloving a series of stages, which is then translated into a series of connected tasks running on a cluster of Trino workers.

Coordinators communicate with workers and clients using a REST API.

### Worker
A Trino worker is a server in a Trino installation, which is responsible for executing tasks and processing data. Worker nodes fetch data from connectors and exchange intermediate data with each-other. The coordinator is responsible for fetching results from the workers and returnign the final results to the client. 

When a Trino worker process starts up, it advertises itself to the discovery server in the coordinator, which makes it available to the Trino coordinator for task execution.

Workers communicate with other workers and Trino coordinators using a REST API

## Data Sources
These fundamental concepts below cover Trino's model of a particular data source.

### Connector
A connector adapts Trino to a data source such as Hive or a relational database. You can think of a connector the same way you think of a driver for a database. It is an implementation of Trino's Service Provider Interface (SPI), which allows Trino to interact with a resource using a standard API.

Trino contains several built-in connectors. Many 3rd-party developers have contributed connectors so that Trino can access data in a variety of data sources (For example - Pinot).

Every catalog is associated with a specific connector. If you examine a catalog configuration file, you'll see that each contains a mandatory property, `connector.name`, which is used by the catalog manager to create a connector for a given catalog. It is possible to have one more than one catalog and use the same connector to access two different instances of a similar database. For example, if you have two Hive clusters, you can configure two catalogs in a single Trino cluster that both use the Hive connector, allowing you to query data from both Hive clusters even with the same SQL query.

### Catalog
A Trino catalog contains schemas and references a data source via a connector. Do we use catalogs in Pinot-Trino land?

### Schema
Schemas are a way to organise tables. Together, a catalog and schema define a set of tables that can be queried. When accessing Hive or a releational database such as MySQL with Trino, a schema translates to the same conecept in the target database. Other types of connectors may choose to organise tables into schemas in a way that makes sense to the unerlying data source.

### Table
A table is a set of unordered rows, which are organised into named columns with types. This is the same as in any relational database. The mapping from source data to tables is dedined in the connector.

## Query Execution Model
Trino Executes SQL statements and turns these statements into queries, that are executed under a distributed cluster of coordinator and workers.

### Statement
Trino executes ANSI-compatible SQL statements. When the Trino docs refers to a statement, it is referring to statements as defined in the ANSI SQL standard, which consists of clauses, expressions and predicates.

In Trino, statements refer to the textutal representation of a SQL statement. When a statement is executed, Trino creates a query along with a query plan that is then distributed across a series of Trino workers.

### Query
When Trino parses a statement, it converts it to a query and creates a distribites query plan, which is then realised a series of intconnected stages running on Trino workers. When you retrieve information about a query in Trino, you receive a snapshot of every component that is involved in producing a result set in response to a statement.

The difference between a statement and a query is simple, a statement can be thought of as the SQL text that is passed to Trino, while a query refers to the configuration and components instantiated to execute that statement. A query encompasses stages, tasks, splits, connectors and other components and data sources working in concert to produce a result.

### Stage
When Trino executes a query, it does so by breaking up the execution into a hiercarchy of stages. For example, if Trino needs to aggregate data from one-billion rows stored in Hive, it does so by producing a root stage to aggregate the output of several other stages, all of which are designed to implement different sections of a distributed query plan.

The hierarchy of stages that composes a query resembles a tree. Every query has a root stage, which is responsible for aggregating the output from other stages. Stages are what the coordinator uses to model a distributed query plan, but stages themselves do not run on Trino workers

### Tasks
As mentioned previously, stages model a particular section of a distributed query plan, but stages themselves don't execute on Trino workers. To understand how a stage is executed, you need to understand that a stage is implemented as a series of tasks distributed over a network of Trino workers.

Tasks are the "work-horse" in the Trino architecture as a distributed query plan is a deconstructed into a series of stages, which are then translated into tasks, which then act upon or process splits. A Trino task has inputs and outputs, and just a stage can be executed in parallel by a series of tasks, a task is executing in parallel with a series of drivers.

### Split
Tasks operate on splits, which are sections of a larger data set. Stages at the lowest level of a distributed query plan retrieve data via splits from connectors and intermediate stages at a higher level of a distributed query plan retrieve data from other stages.

When Trino is scheduling a query, the coordinator queries a connector for a list of all splits that are available for a table. The coordinator keeps track of which machines are running which tasks, and what splits are being processed by which tasks.

### Driver
Tasks contain one or more parallel drivers. Drivers act upon data and combine operators to produce output that is then aggregated by a task and then delivered to another task in another stage. A driver is a sequence of operato instances, or you can think of a driver as a physical set of operators in memory. It is the lowest level of parallelism in the Trino architecture. A driver has one input and one output.

### Operator
An operator consumes, transforms and produces data. For example, a table scan fetches data from a connector and produces data than can be consumed by other operators, and a filter operator consumes data and produces a subset by applying a predicate over the input data.

### Exchange
Exchanges transfer data between Trino nodes for different stages of a query. Tasks produce data into an output buffer and consume data from other tasks using an exchange client.

## Resource Management With Trino

### `memory.heap-headroom-per-node`

* Default Value - (JVM Max Memory * 0.3)
  
This is the amount of memory set aside as headroom/buffer in the JVM heap for allocations that are no tracked by Trino

### `query.max-cpu-time`

* Default value: 1_000_000_000d
  
This is the max amount of COU time that a query can use across the entire cluster. Queries that exceed this limit are killed.

For our Trino config, this is controlled by the 

## Spill to Disk

### Overview
In the case of memory-intensive operations, Trino allows offloading intermediate operation results to disk. The goal of this mechanism is to enable execution of queries that require amounts of memory exceeding per-query or per-node limits.

The mechanism is similar to OS-level page swapping (a process of swapping a process temporarily to a secondary memory from the main memory which is fast than compared to secondary memory). However, it is implemented on the application level to address the specific needs of Trino.

Properties related to spilling are described in the [Spilling Properties]() below.

### Memory management and spill
By default, Trino kills queries, if the memory requested by the query execution exceeds session properties `query_max_memory` or `query_max_memory_per_node`. This mechanism ensures fairness in alocation of memory to queries, and prevents deadlock caused by memory allocation. It is effcient when there is a lot of small queries in the cluster, but leads to killing large queries that don't fit within the limits.

To overcome this inefficiency, the concept of revocable memeory was introduced. A query can request memory that does not count towards the limits, but this memory can be revoked by the memory manager at any time. When memory is revoked, the query runner *spills* intermediate data from memory to disk and continues to process it later.

In practice, when the cluster is idle, and all memory is available, a memory intensive query may use all of the memory in the cluster. On the other hand, when the cluster does not have much free memory, the same query may be forced to use disk as storage for intermediate data. A query that is forced to spill to disk, may have a longer execution time by orders of magnitude than a query that runs completely in memory.

!!! Note

    Enabling spill-to-disk does not guarantee execution of all memory intensive queries. It is still possible that the query runner fails to divide intermediate data into chunks small enough so that every chunk fits into memory, leading to OOM errors while loading the data from disk.

### Spill disk space
Spilling intermediate results to disk, and retrieving them back is exoensive in terms of IO operations, this queries that use spill likely become throttled by disk. To increase query perfromance, it is recommended to provide multiple paths on deparate local devices for spill (property `spiller-spill-path` in [Spilling Properties]())

The system drive should not be used for spilling, esepcially not the drive where the JVM is running and writing logs. Doing so may lead to cluster instability. Additionally, it is recommended to monitor the disk saturation of comnfigured spill paths.

Trino treats spill paths as independent disks, so there is no need to use RAID for spill.

### Spill compression
When spill compression is enabled (`spill-compression-enabled` property in [Spilling Properties](), spilled pages are compressed, before being written to disk. Enabling this feature can reduce disk IO at the cost of extra CPU load to compress and decompress spilled pages.)

### Spill encryption
When spill encryption is enabled (`spill-encryption-enabled` property in [Spilling Properties]()), spill contents are encrypted with a randomly generated (per spill file) secret key. Enabling this increases CPU load and reduces throughput of spilling to disk, but can protect spilled data from being recovered from spill files. Consider reducing the value of `memory-revoking0-threshold` when spill encryption is enabled, to account for the latency increase of spilling.

### Supported Operations
Not all mechanism support spilling to disk, and each handles spilling differently. Currently, the mechanism is implemented for the following operations.

#### Joins
During the join operation, one of the tables being joined is stored in memory. This table is called the build table. The rows from the other table stream through and are passed onto the next operation if they match the rows in the build table. The most memory-intensive part of the join is the build-table.

When the task concurrency is greater that one, the build table is partitioned. The number of partitions is equal to the value of the `task.concurrency` configuration parameter.

When the build table is partitioned, the spill-to-disk mechanism can decrease the peak memory usage needed by the join operation. When a query approaches the memory limit, a subset of the partitions of the build table gets spilled to disk, along with rows from the other table that fall into those same partitions. The number of partitions that get spilled influences the amount of disk space needed.

Afterward, the spilled partitions are read back one-by-one to finish the join operation.

With this mechanism, the peak memory used by the join operator can be decreased to the size of the largest build table partition. Assuming no data skew, this is `1 / task.concurrency (default: 16)`  times the size of the whole build table.

#### Aggregations



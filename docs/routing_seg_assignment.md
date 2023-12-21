# Pinot Routing & Segment Assignment

## Routing

### Optimsing Scatter & Gather
When the use-case has a very hugh Query Per Second (QPS) and low-latency requirements, we need to consider optimsing scatter and gather.

| Problem               | Impact                         | Solution                             |
| --------------------- | ------------------------------ | ------------------------------------ |
| Querying all servers  | Bad tail latency, not scalable | Control number of servers to fan out |
| Querying all segments | More CPU work on server        | Minimize the number of segments      |

#### Querying all servers
By default, Pinot uniformly distrbutes all the segments of a table, to all of the pinot servers. When scattering and gathering query requests, the Pinot Broker also uniformly distributes the workload among servers for each segment. As a result, each query will span out to all the servers with a balanced workload. This approach works well when the QPS is low and you have a small number of servers in the cluster. However, as we add more servers, or have more QPS, the probability of hitting slow servers (e.g. through garbage collection) increases steeply and Pinot will suffer from a long tail latency (high latencies that clients see fairly infrequently).

To address this, Pinot has `replicaGroups`, that allow us to control the number of servers to fan out for each query.

##### Replica group segment assignment and query routing
A `Replica Group` is a set of servers that contains a 'complete' set of segments of a table. Once we assign the segment based on replica group, each query can be answered by fanning out to a single replica group instead of all servers.

[Replica Group Diagram]

`Replica Groups` can be configured by settingg the `InstanceAssignmentConfig` in the table config. Replica group based routin can be configured by setting `replicaGroup` as the `instanceSelectorType` in the `RoutingConfig`.


```json
{
  ...
  "instanceAssignmentConfigMap": {
    "OFFLINE": {
      ...
      "replicaGroupPartitionConfig": {
        "replicaGroupBased": true,
        "numReplicaGroups": 3,
        "numInstancesPerReplicaGroup": 4
      }
    }
  },
  ...
  "routing": {
    "instanceSelectorType": "replicaGroup"
  },
  ...
}
```

As seen above, you can use `numReplicaGoups` to control the number of replica groups (replications) and use `numInstancesPerReplicaGroup` to control the number of servers to span. For instance, let's say you have 12 servers in the cluster. The config shown above will generate 3 replica groups (`numReplicaGroups=3`), and each replica group will contain 4 servers (`numInstancesPerPartition=4`). In this example, each query will span to a single replica group (4 servers).

Replica groups gives you the control on the number of servers to span for each query. When deciding the proper values for `numReplicaGroups` and `numInstancesPerReplicaGroup`, you should consider the trade-off between throughput and latency, as summarised in the table below:

| Approach                                                               | Pros                                                             | Cons                                                             |
| ---------------------------------------------------------------------- | ---------------------------------------------------------------- | ---------------------------------------------------------------- |
| Increase `numReplicaGroups` and Decrease `numInstancesPerReplicaGroup` | Increased throughput. Each server processes fewer queries        | Increased latency. Each server processes more segments per query |
| Increase `numInstancesPerReplicaGroup` and Decrease `numReplicaGroups` | Decreased Latency. Each server processes less segments per query | Decreased throughput. Each server has to process more queries    |

#### Querying all segments
By default, the Pinot Broker will distribute all segments for query processing and segment pruning is happening in the Pinot Server. In other words, the Pinot Server will look at the segment metadata such as the min/max time value and discard the segment if it does not contain any data that the query is asking for. Server pruning works best when the QPS is low. However, it becomes the bottleneck if the QPS is very high (1000s of queries per second) because unnecessary segments still need to be scheduled for processing which consumes CPU resources.

##### Partitioning
When data is partitioned on a [dimension](), each segment will contain all the rows with the same partition value for a partitioning dimension. In this case, a lot of segments can be pruned if a query requires to look at a single partition to compute the result. Below diagram gives the example of data partioned on member id while the query includes an equality filter on member id.

[Diagram goes here]

`Partitioning` can be enabled by setting the following configuration in the table config.

```json
{
  ...
  "tableIndexConfig": {
    ...
    "segmentPartitionConfig": {
      "columnPartitionMap": {
        "memberId": {
          "functionName": "Modulo",
          "numPartitions": 4
        }
      }
    },
    ...
  },
  ...
  "routing": {
    "segmentPrunerTypes": ["partition"]
  },
  ...
}
```

Pinot currently supports `Modulo`, `Murmur`, `ByteArray` and `HashCode` hash functions. After setting the above config, data needs to be partitioned with the same partition function and number of partitions before running the Pinot segment build and push job for offline push. REALTIME partitioning depends on Kafka for partitioning. When emitting an event to Kafka, a user needs to feed the partitioning key and partition function for the Kafka producer API.

When applied correctly, partition information should be avialable in the segment metadata.

```bash
column.memberId.partitionFunction = Module
column.memberId.numPartitions = 4
column.memberId.partitonValues = 1
```

Broker-side pruning for partitioning can be configured by setting the `segmentPrunerTypes` in the `RoutingConfig`. Note that the currrent implementation for partititioning only works for EQUALITY amd IN filter (e.g. `memberId = xx`, `member IN (x, y, z)`).

## Segment Assignment

Segment assignment refers to the strategy of assigning each statement from a table to the servers hosting the table. Picking the best segment assignment strategy can help reduce the overhead of the query routing, thus providing better performance.

### Balanced Segment Assignment
Balanced segment assignment is the default assignment strategy, where each segment is assigned to the server with the least segments already assigned. With this strategy, each server will have balanced query load, and each query will be routed to all the servers. It requires minimum configuration, and works well for small use cases.

[DIAGRAM]

### Replica Group Segment Assignment
Balanced Segment Assignment is ideal for small use cases with a small number of servers, but as the number of servers increases, routing each query to all the servers could harm the query performance due to the overhead of the increased fanout.

Replica-Group Segment Assignment is introduced to solve the horizontal scalability problem of the large use-cases, which makes Pinot linearly scalable. This strategy breaks the servers into multiple replica-groups, where each relica-group contains a full copy of all the segments.

When executing queries, each query will only be routed to the servers within the same replica group. In order to scale up the cluster, more replica groups can be added without affecting the fanout of the query, thus not impacting the query performance but increasing the overall throughput linearly.

[DIAGRAM]

### Partitioned Replica-Group Segment Assignment
In order to further increase the query performance, we can reduce the number of segments processed for each query by partitioning the data and using Partitioned Replica-Group Segment Assignment.

Partitioned Replica-Group Segment Assignment extends the Replica-Group Segment Assignment by assigning the segemnts from the same partition to the same set of servers. To solve a query which hits only one partition (e.g. `SELECT * FROM myTable WHERE memberId = 123` where `myTable` is partitioned with the `memberId` column), the query only needs to be routed to the servers for the targeting partition, which can significantly reduce the number of segments to be processed. This strategy is especially useful to achieve high throughput and low latency for use-cases that filter on an ID field.
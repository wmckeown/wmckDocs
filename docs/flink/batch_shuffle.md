# Batch Shuffle

*Adapted from: https://nightlies.apache.org/flink/flink-docs-master/docs/ops/batch/batch_shuffle/*

Flink supports a batch execution mode in both the DataStream API and Table / SQL for jobs executing across bounded input. In batch execution mode, Flink offers two modes for network exchanges: Blocking Shuffle and Hybrid Shuffle.

* Blocking Shuffle is the default data exchange mode for batch executions. It persists all immediate data, and can be consumed only after fully produced
* Hybrid Shuffle is the next generation data exchange mode for batch executions. It persists data more smartly, and allows consuming while being produced. This feature is still experimental.

## Blocking Shuffle
Unlike pipeline shuffle used for streaming applications, blocking exchanges persists data to some storage. Downstream tasks then fetch these values via the network. Such an exchange reduces the resources required to execute the ob as it does not need the  upstream and downstream tasks to run simultaneously.

As a whole, Flink provides two different types of blocking shuffles: `Hash Shuffle` and `Sort Shuffle`. As our Flink jobs only use the `Sort Shuffle`, we will focus on that.

### Sort Shuffle
`Sort Shuffle` is a blocking shuffle implementation introduced in Flink v1.13 and has since become the default blocking shuffle implementation as of Flink v1.15. Different from the `Hash Shuffle`, `Sort Shuffle` writes only one file for each result partition. When the result partition is read by multiple downstream tasks concurrently, the data file is opened only once and shared by all readers. As a result, the cluster uses fewer resources like inode^^ and file descriptors, which improves stability. Furthermore, by writing fewer files and making a best effort to read data sequentially, `Sort Shuffle` can achieve better performance than `Hash Shuffle` especially on SSDs. Additionally, `Sort Shuffle` uses extra managed memory as a data reading buffer and does not rely on `sendFile` or `mmap` mechanism, therefore it also works well with SSL.

Here are some config options that might need adjustment when using sort blocking shuffle:

* `taskmanager.network.sort-shuffle.min.buffers`: Config option to control data writing buffer size. For large scale jobs, you may need to increase this value, usually several hundreds of megabytes (mb) of memory is enough. Because this memory is allocated from network memory, to increase this value, you may need to increase the total network memory by adjusting `taskmanager.memory.network.fraction`, `taskmanager.memory.network.min` & `taskmanager.memory.network.max` to avoid a potential "insufficient number of network buffers error".
* `taskmanager.memory.framework.off-heap.batch-shuffle.size`: Config option to control data reading buffer size. For large-scale jobs, you may need to increase this value too and again, several hundreds of mb may be sufficient (We have this set to `256mb` - the default is `128mb`) Because this memory is cut from the framework off-heap memory, to increase this value, you also need to increase the total framework off-heap memory by adjusting `taskmanager.memory.framework.off-heap.size` to avoid a potential direct memory OOM Error.

## Performance Tuning Considerations
These guidelines may be helpful when looking to squeeze extra performance, especially for large-scale batch jobs:

### Increase the total size of network memory
Currently, the default network memory size is pretty modest. For large scale jobs, it is suggested to increase the `taskmanager.memory.network.fraction` to at least `0.2` to achieve better performance. At the same time, you may need to adjust the `taskmanager.memory.network.min` & `taskmanager.memory.network.max`.

### Increase the memory size for shuffle data write
As mentioned above, for large scale jobs it's suggested to increase the `taskmanager.network.sort-shuffle.min-buffers`^ (default 512) to at least 2 * parallelism if you have enough memory.

!!! Note
        You may need to increase the total size of network memory (See above) to avoid the "Insufficient number of network buffers" error after you increase tis config value.

### Increase the memory size for shuffle data read
As mentioned above, for large-scale jobs, it's suggested to increase the `taskmanager.memory.framework.off-heap.batch-shuffle.size` to a larger value. (eg. 256mb or 512mb - default is 128mb). Because this memory is cut from the framework off-heap memory, you must increase `taskmanager.memory.framework.off-heap.size` ***by the same size*** to avoid the direct memory OOM error.

## More Relevant TroubleShooting

| Exceptions                             | Potential Solutions                                                                                                              |
| -------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| Insufficient number of network buffers | This means the amount of network memory is not enough to run the target job and that you need to increase the target memory size |
| Too many open files   | This means that the file descriptors is not enough. Consider increasing the system limit for file descriptors and check if our code is consuming too many file descriptors |
| Connection reset by peer | Usually means the network is unstable or under heavy burden. SSL Handshake Timeout errors could also cause this. Increasing `taskmanager.network.netty.server.backlog` may help (Not sure tho ;P) | 
| Read buffer request timeout | This can happen only when you are using Sort Shuffle and it means a fierce contention of the shuffle read memory. To solve the issue, you can increase `taskmanager.memory.framework.off-heap.batch-shuffle.size` together with `taskmanager.memory.framework.off-heap.size` |
| Out of memory error | Consider increasing the corresponding memory size. For heap memory, you can increase `taskmanager.memory.task.heap.size` and for direct memory, you can increase `taskmanager.memory.task.off-heap.size` |









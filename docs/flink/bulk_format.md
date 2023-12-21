# Flink BulkFormat

## Format Types

* A `BulkFormat` reads batches of records from a file at a time, it is the most "low level" format to implement but offers teh greatest flexibility to optimise the implementation.

### Bulk Format
The BulkFormat reads and decodes batches of records at a time. Examples of bulk formats are formats like ORC and Parquet. The outer `BulkFormat` class acts mainly as a configuration holder and factory for the reader. The actual reading is done by the `BulkFormat.Reader` which is created in the `BulkFormat#createReader(Configuration, FileSourceSplit)` method. If a bulk reader is created based on a checkpoint during checkpointed streaming execution, then the reader is re-created in the `BulkFormat#restoreReader(Configuration, FileSourceSplit)` method.


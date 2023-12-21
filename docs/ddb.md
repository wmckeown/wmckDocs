# Amazon DynamoDB

## Overview
DynamoDB is a fully-managed NoSQL database that supports key-value and document data structures. It is a multi-region, highly durable database service that can provide single-digit millisecond (ms) performance at any scale. DynamoDB has a highly predictable performance; it can handle more than 10 trillion requests per day and support peaks of more than 20 million requests per second.

DynamoDB supports the following data types:

* **Scalar** - Number | String | Binary | Boolean | Null
* **Multi-Valued** - String Set | Number Set | Binary Set
* **Document** - List | Map

## Primary Keys
When creating a table, you must specify a partition (or hash) key. Dynamo will use the value of this key as an input to an internal hash function. The output of this hash function determines the physical storage partition in which the items will be stored. If a table has only one partition key, then no two items can have the same partition key value.

In addition, you can specify a sort (or range) key for your table. Items in a partition will be ordered by the sort key value. In a table with a sort key and a partition, multiple items may have the same partition key, but they **must have different sort keys**. Having both a partition and a sort key gives you more flexibility when querying data. Partition and sort keys must be scalar and are limited to Number | String | Binary data types.

The easist way to think of a DynamoDB table is to treat it like an incredibly fast, giant hash map. You can only query and request items from a DynamoDB instance based off of the table key, meaning it's important to consider what information you'll be using the most to request items from your table. For example, if the majority of your requests will be pointers for a given _orgId_ and _assetId_, then those values would be sensible partition and sort keys.

## Capacity Considerations
Although DynamoDB can technically scale to any volume, there are still considerations that need to be made regarding capacity allocations for DynamoDB tables.

### Capacity Modes
Dynamo defines capacity in two ways:

* **Read Capacity**: One Read Capacity Unit (RCU) is equivalent to one strongly-consistent read, or two eventually-consistent reads for a data block of up to 4KB
* **Write Capacity**: One Write Capacity Unit (WCU) is one write request of up to 1KB

DyanamoDB offers two types of capacity allocation, _on-demand_ and _provisioned_. 

**On-demand Capacity** sets pricing based on the amount of read and write request units the application consumes throughout the month. Dynamo can immediately serve all incoming read and write requests as long as the traffic does not exceed double the highest recorded level. If the request volume exceeds this limit, capacity will eventually be allocated but this process could take up to 30 minutes.

**Provisioned Capacity** allows developers to allocate a read and write capacity and pay for this allocated capacity regardless of whether they consume all of this capacity or not. If the application exceeds the provisioned capacity, AWS will throttle the requests.

#### So why not just use on-demand capacity everywhere?
On-demand is a good option for applications that have unpredicatble or sudden spikes in traffic, since read and write capacity is automatically provisioned based on demand. Provisoned capacity suits applications with more predictable usage. In practice, things are a little more complicated.

Whilst on-demand capacity allocation delivers the best-fit for scalability, the cost is approximately **7x higher** than provsioned capacity. In addition, provisioned capacity offers the option to purchase reserved capacity, which can save 40%-80% compared to non-reserved provisoned caapcity allocation. To use on-demand capacity allocation in the same scenario could cost **15-20x more** than reserved provisioned capacity allocation. For smaller applications, the flexibility of on-demand capacity allocation may be worth the additonal cost, but for larger applications the extra costs could run into the hundreds of thousands per month.

### Burst Capacity
DynamoDB provides some flexibility in per-partition throughput provisioning by providing Burst Capacity. If a partition's provisioned capacity is not fully used, Dynamo will store a portion of that unused capacity for later bursts of throughput to handle usage spikes.

DynamoDB currently retains up to 5 minutes of unused read and write capacity. During an occassional burst of read and write activity, these additional capacity units can be consumed quickly --sometimes even faster than a table's provisioned capacity.

### Adaptive Capacity
Adapative Capacity is a function that enables DynamoDB to run imbalanced workloads indefinitely. It minimises throttling due to throughput exceptions. It also helps you reduce costs by enabling you to provision only the throughput capacity that you need.

DynamoDB distributes the data across partitions and the throughput caapcity is distributed equally across these partitions. However, when data access is imbalanced, a "hot" partition can recieve a higher volume of read and write traffic compared to other partitions, leading to throttling errors on that partition.

DynammoDB Adapative Capacity enables the application to continue reading a writing to hot partitions without being throttled, provided that the traffic does not exceed the table's total provisioned capacity or the partition maximum capacity. Adaptive Capacity is enabled automatically for every DynamoDB table, at no additional cost.

## Read Consistency
DynamoDB techinically supports eventually-consistent and strongly-consistent reads. 

When you use an eventually-consistent read to retrieve data from a DynamoDB table, the response may not reflect the results of a recently completed write operation and may contain stale data. Dynamo is usually consistent across all availability zones within 1 second for an eventually-consistent read request.

Compared with eventually-consistent read requests, strongly-consistent read requests cost 2x much, may have a higher latency and are not supported on Global Secondary Indexes (GSI). In theory, when you request a strongly-consistent read, DynamoDB returns a response with the most up-to-date data, reflecting the updates from all prior write operations that were successful across all availability zones. In practice, we have found that strongly-consistent rates are not suffciently reliable and sometimes still return stale data or a null-pointer if requesting a newly-inserted item.

## Indexes
There are two types of indexes in DynamoDB, a Local Secondary Index (LSI) and a Global Secondary Index (GSI). Indexes can contain some or all of the additional attributes from the main table, you can choose which attributes to project from the base table when creating the index. DynamoDB automatically keeps all indexes synchronised with their respective base tables.

For all indexes, it's important to note that like the main table, the value of the primary key attributes cannot be updated during a write operation. This means if attributes acting as either the partition key or the sort key of the index are updated in the main table, it will require two write operations to update the item in the index, one to delete the item and another to re-insert it with the updated key value.

When creating an index for larger tables, it can be useful to temporarily increase the provisioned capacity of the table and index to reduce the time it takes to create the index. For example, to create the GSI for a table with ~100,000,000 items with a provisioned R/W capacity of 1100 units each would take ~1 day, 1 hour and 15 minutes. Increasing the provisioned capcity to 9000 units for table creation would drop this time to ~4 hours and 41 minutes. However, it's important that you remember to put your provisioned capacity for your tables back down to their original values after the index has been created.

### Local Secondary Indexes
Local Secondary Indexes (LSIs) maintain the same partition key as the main table, but use an alternative sort key which provides extra flexibility when querying terms. There's a limit of 5 LSIs per table and it's important to note that the read and write capacity throughput for a LSI will be included in the main table's workload and will consume the main table's provisioned capacity.

### Global Secondary Indexes
Global Secondary Indexes (GSIs) use non-key attributes from the main table as their partition and sort key values. Some applications might need to perform many kinds of queries using a variety of different attributes as query criteria. GSIs provide a means of using these different query criteria without scanning the main table an operation that gets progressively slower and more expensive as the table gets larger. The partition key and sort key of the table are always projected into the GSI. You can also project other attribute's to support your application's query requirements. When querying a GSI, DynamoDB can access any attribute in the projection as if those attributes were in their own table.

DynamoDB automatically synchronises each GSI with its base table. When an application writes or deletes items in a table, any GSIs on that table are updated asynchronously, using an eventually-consistent model. As a result, GSIs do not support strongly consistent reads. Applications never write directly to an index.

GSIs inherit the R/W capacity from the base table.

## Batching
DynamoDB supports batching multiple read and write operations into a single operation using `BatchWriteItem` and `BatchGetItem` requests. BatchWrites and BatchGets have different limitations and considerations.

### BatchWriteItem
The `BatchWriteItem` operation puts or deletes multiple items in one or more tables. A single `BatchWriteItem` can write up to 16MB of data, comprising of up to 25 `put` or `delete` requests. The individual `PutItem` and `DeleteItem` operations specified in `BatchWriteItem` are atomic; however `BatchWriteItem` as a whole is not. If any requested operations fail because the table's provisioned throughput is exceeded or an internal processing failure occurs, the failed operations are returned in the `UnprocessedItems` response parameter. You can investigate and optionally resend the requests.

If one or more of the following is true, DynamoDB rejects the entire batch write operation:

* One or more tables specified in the `BatchWriteItem` request does not exist
* Primary key attributes specified on an item in the request do not match the corresponding table's primary key schema
* You try to perform multiple operations on the same item in the `BatchWriteItem` request. For example, you cannot `put` and `delete` the same item in the same `BatchWriteItem` request
* Your request contains at least two items with identical hash and range keys (which is essntially two put operations)
* There are more than 25 requests in the batch
* Any individual item in a batch exceeds 400KB
* The total request size exceeds 16MB

It's important to note that the `put` request in a `BatchWriteItem` is exclusively an insert operation. DynamoDB does not support batch-updating items. This means that existing items that require updating must be updated using an `UpdateItem` request, one-at-a-time.

### BatchGetItem
The BatchGetItem operation returns the atributes of one or more items from one or more tables. Requested items are identified by the table's primary key

A single operation can retrieve up to 16MB of data, which can contain as many as 100 items. `BatchGetItem` returns a partial result when the 16MB request size-limit is exceeded, the table's provisioned throughput is exceeded ot an internal processing failure occurs. If a partial result is returned, the operation returns a value for `UnprocessedKeys` You can use this value to retry the operation starting with the next item to get.

By default, `BatchGetItem` performs eventually-consistent reads on every table inthe request. If you require strongly-consistent reads, you can set `Consistent=true` for any or all tables (see [Read Consistency] section for more information on this). In order to minimise response latency, BatchGetItem retrieves items in parallel.

When designing your application, keep in mind that DynamoDB does not return items in any particular order. To help parse the response by item, include the primary key values for the items in your request in the `Projection Expression` parameter. If a requested item does not exist, it is not returned in the result.

## Time to Live (TTL)
DynamoDB Time-to-Live (TTL) allows you to specifiy an expiration timestamp attribute in your table. Shortly after this expiration date and time has elapsed, DynamoDB will delete the item from your table without consuming any write capacity. TTL is provided at no extra cost. The item to be deleted must contain the specified attribute when TTL was enabled on the table and not every item in your table needs to contain a value for the expiration field. The TTL attribute's value must be a Number data type, in UNIX Epoch Seconds and with an expiration of no more than five years in the past.

It can take up to 48 hours after expiration for an item to be deleted from the table and associated indexes. During this time, the item can be read and updated as normal. If the expiration value is updated to a time in the future, then the item will not be deleted.

## DynamoDB Streams
A DynamoDB stream is an ordered flow of information about changes to items in a DynamoDB table. When you enable a stream on a table, DynamoDB captures information about every modification to data items in the table. Every time an item is inserted, updated or deleted in the table, DynamoDB Streams writes a stream record with the primary key attributes of the items that were modified.

A Stream Record contains information about a data modification to a single item in a DynamoDB table. You can configure the stream so that the stream records capture additional information such as the `before` and `after` images of modified items. For each item that is modified in a DynamoDB table, the stream records appear in the same sequence as the actual modifications to the item and these records are written in near-real-time.

## Using DynamoDB Locally
### DynamoDB SDK Overview
AWS provides DynamoDB SDKs for Java, JavaScript, Node.js, .NET, PHP, Python and Ruby. Your chosen SDK will provide a programmatic interface for working with DynamoDB. The SDK will construct and send HTTP(S) requests to the DynamoDB API, process the relevant HTTP(S) response and propagate it back to your application.

Each of the AWS SDKs provides important services to your application:
* Formatting HTTP(S) requests and realising request parameters
* Generating a cryptographic signature for each request
* Forwarding requests to a DynamoDB endpoint and receiving responses from DynamoDB
* Extracting the results from those responses
* Implementing basic retry logic in case of errors

## Local Test Setup
When writing an application that will interact with a DynamoDB table in production, you may wish to write tests that interact with a local table to test the functionality of your application. This can be done using test containers, using a `amazon/dynamo-local` Docker image and exposing and specifying a Dynamo port. Below is Java example of how to set up a test container and `DynamoDbClient` for a test environment.

```java
private static final int DYNAMO_PORT = 8000;
@Container
public static final GenericContainer dynamoContainer = new GenericContainer("amazon/dynamodb-local").withExposedPorts(DYNAMO_PORT);

static {
    dynamoContainer.start();
}

private final DynamoDbClient dynamo = DynamoDbClient.builder()
    .endpointOverride(URI.create(format("http://%s:%s",
        dynamoContainer.getContainerIpAddress(),
        dynamoContainer.getMappedPort(DYNAMO_PORT))))
    .region(US_EAST_1)
    .credentialsProvider(
        StaticCredentialsProvider.create(AwsSessionCredentials.create("accessKey", "secretKey", "sessionToken")))
```

After you've set up your test container and DynamoDB client, you need to set up your table using a `CreateTableRequest` which includes:

* Name - The name of your table
* The Key Schema - a list of `KeySchemaElements`, each of which should include the name of the element and whether it is a Hash Key (Partition Key) or Range Key (Sort Key)
* Attribute Definitions - a list of any non-key `AttributeDefinitions`, each of which should include the name and datatype of the attribute
* Provisioned Throughput - The provisioned read and write capacity values. (NOTE: It's not possible to throttle a DynamoDB table localy. So although you must define your provisioned throughput values, they have no impact on the performance of your local table)
* Any Local or Global Secondary Indexes you wish to add to your table

## Expressions
In DynamoDB you can use expressions to denote the attributes you want to read from an item, when writing to an item - to indicate any conditions that must be met and to indicate how the attributes are to be updated.

### Expression Attribute Names & Values
An *Expression Attribute Name* is a placeholder that you use in a DynamoDB expression as an alternative to an actual attribute name if required. An *Expression Attribute Name* must begin with a pound symbol '#' and be followed by one-or-more alphanumeric characters e.g `#attributeName`

If you need to compare an attribute with a value, you can define an *Expression Attribute Value* as a placeholder. *Expression Attribute Values* in DynamoDB are substitiutes for actual values that you want to use for comparisons and may not be known until runtime. An Expression Attribute Value must begin with a colon ':' and be followed by one or more alphanumeric characters e.g. `:attributeValue`

### Projection Expressions
To read data from a table, you can use `Get`, `Query` or `Scan` operations and Dynamo will return all attributes by default. To only get the necessary attribute, use a Projection Expression. A Projection Expression is a string that identifies the attributes that you want. To retrieve a single attribute, specify its name. For multiple attributes, the names must be comma-separated. You can use any attribute name in a projection expression, provided the that first character is `A-Za-z` and the second character (if present) is `A-Za-Z0-9`. If an attribute name does not meet this requirement, you must define this attribute name as a placeholder.

### Condition Expressions
When performing conditional writes or querying a DynamoDB table, you must use a condition expression. An operand in a condition expression can be a top-level attribute name or a document path that references a nested attribute.

#### Valid condition-expression syntax

* *operand* comparator *operand*
* *operand* BETWEEN *operand* AND *operand*
* *operand* IN (*operand*(','*operand*(,...)))
* function
* *condition* AND *condition*
* *condition* OR *condition*
* NOT *condition*
* (*condition*)

#### Valid comparitors

| Comprarator       | Description                                                                                             |
| ----------------- | ------------------------------------------------------------------------------------------------------- |
| a = b             | True if *a* is equal to *b*                                                                             |
| a <> b            | True if *a* is not equal to *b*                                                                         |
| a < b             | True if *a* is less than *b*                                                                            |
| a <= b            | True if *a* is less than or equal to *b*                                                                |
| a > b             | True if *a* is greater than *b*                                                                         |
| a >=b             | True if *a* is greater than or equal to *b*                                                             |
| a BETWEEN b AND c | True if *a* is greater than or equal to *b* but less than *c*                                           |
| a IN (b,c,d)      | True if a is equal to any value in the list. The list can contain up to 100 values, separated by commas |

#### Functions
You can use the following functions to determine whether an attribute existsin an item or to evaluate the value of an attribute. Function names are case-sensitive. For a nested attribute, you must provide its full document path.

| Function                      | Description                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| ----------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `attribute_exists (path)`     | True if the item contains the attribute specified by `<path>`                                                                                                                                                                                                                                                                                                                                                                                                                               |
| `attribute_not_exists (path)` | True if the attribute specified by `<path>` does not exist in the item                                                                                                                                                                                                                                                                                                                                                                                                                      |
| `attribute_type (path, type)` | True if the attribute at the specified path is of a particular data type. The type parameter must be one of the following: <ul><li>**S** - String</li><li>**SS** - String Set</li><li>**N** - Number</li><li>**NS** - Number Set</li><li>**B** - Binary</li><li>**BS** - Binary Set</li><li>**BOOL** - Boolean</li><li>**NULL** - Null</li><li>**L** - List</li><li>**M** - Map </li></ul> You must use an expression attribute for the `type` parameter                                    |
| `begins_with (path, substr)`  | True if the attribute specified by `path` begins with a particular substring                                                                                                                                                                                                                                                                                                                                                                                                                |
| `contains (path, operand)`    | True if the attribute specified by `path` is one of the following <ul><li>A String that contains a particlar substring</li><li>A Set that contains a particulat element within the set</li></ul> The *operand* must be a String if the attribute specified by `path` is a String. If the attribute specified by `path` is a Set, the operand must be the set's element type. The path and the operand must be distinct; that is, `contains(a,a)` returns an error                           |
| `size (path)`                 | Returns a number representing the attribute's size. The following data types can be used with size: <ul><li>If the attribute is of type String, `size` returns the length of the string</li><li>If the attribute is of type Binary, `size` returns the number of bytes in the attribute value</li><li>If the attribute is a Set data type, `size` returns the number of elements in the set</li><li>If the element is of type List or Map, `size` returns the number of child elements</li> |

#### Logical Evaluations
Use the `AND`, `OR` and `NOT` keywords to perform logical evaluations:

| Comparator | Description                       |
| ---------- | --------------------------------- |
| a *AND* b  | True if both *a* and *b* are true |
| a *OR* b   | True if either *a* or *b* is true |
| NOT *a*    | True if *a* is false              |

#### Precedence in Conditions
DynamoDB evaluates conditions from left to right using the following precedence rules:

1. = | <> | < | <= | > | >=
2. IN
3. BETWEEN
4. attribute_exists | attribute_not_exists | begins_with | contains
5. Parenthesis
6. NOT
7. AND
8. OR

## DynamoDB and Terraform
A DynamoDB table and its associated indexes can be easily added as resource to a terraform module.

### Main Argument Reference
Within this terraform module, you can define the following main arguments:

| Argument Name            | Required /<br>Optional                      | Description                                                                                                                                                                                         |
| ------------------------ | ------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `name`                   | R                                           | The name of a table. Mustbe unique within a region                                                                                                                                                  |
| `billing_mode`           | O                                           | Controls how you are charged for read and write throughput. Valid values are `PROVISIONED` and `PAY_PER_REQUEST`                                                                                    |
| `write_capacity`         | R only if <br> `billing_mode = PROVISIONED` | The provisioned write capacity units for the table                                                                                                                                                  |
| `read_capacity`          | R only if <br> `billing_mode = PROVISIONED` | The provisioned read capacity units for the table                                                                                                                                                   |
| `hash_key`               | R                                           | The name of the attribute used as your partition key - must be defined in attributes                                                                                                                |
| `range_key`              | O                                           | The name of the attribute used as your sort key - must be defined in attributes                                                                                                                     |
| `attribute`              | R                                           | A list of attribute definitions of any attributes used as partition or sort keys for the table or any associated indexes, and no other attributes                                                   |
| `ttl`                    | O                                           | Defines TTL and can only be specified once per table                                                                                                                                                |
| `local_secondary_index`  | O                                           | Describes a LSI on the table; these can only be allocated at creation. Once created, they are immutable                                                                                             |
| `global_secondary_index` | O                                           | Describes a GSI for the table; subject to the normal limits on the number of GSIs, projected attributes etc.                                                                                        |
| `stream_enabled`         | O                                           | Indicated whether streams are to be enabled (`true`) or disabled (`false`)                                                                                                                          |
| `stream_view_type`       | O                                           | When an item in the table is modified, `StreamViewType` determines what information is written to the table's stream. Valid values are KEYS_ONLY, NEW_IMAGE, OLD_IMAGE, NEW_AND_OLD_IMAGES          |
| `server_side_encryption` | O                                           | Encryption a rest options. DynamoDB tables are automatically encrypted at rest with an AWS owned Customer Master Key if this argument isn't specified.                                              |
| `tags`                   | O                                           | A map of tags to populate on the created table. If configured with a provider default_tags configuration block present, tags with matching keys will overwrite those defined at the provider_level. |
| `point_in_time_recovery` | O                                           | Point in time recovery options                                                                                                                                                                      |

### Nested Argument Reference
Some of the above arguments have the following nested fields:

| Argument Name                | Required /<br>Optional                           | Description                                                                                                                                                                                                                                                             |
| ---------------------------- | ------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| ***attribute***              |
| `name`                       | R                                                | The name of the attribute                                                                                                                                                                                                                                               |
| `type`                       | R                                                | Attribute type, which must be scalar data type: **S** - String, **N** - Number, **B** - Binary                                                                                                                                                                          |
| ***local_secondary_index***  |
| `name`                       | R                                                | The name of the index                                                                                                                                                                                                                                                   |
| `range_key`                  | R                                                | The name of the attribute used as your sort key. Must be defined in attributes.                                                                                                                                                                                         |
| `projection_type`            | R                                                | One of `ALL`, `INCLUDE`, `KEYS_ONLY` Where: <ul><li>`All` projects every attribute into the index</li><li>`KEYS_ONLY` projects just the hash and range key into the index</li><li>`INCLUDE` projects only the keys specified in the `non_key_attributes` parameter</li> |
| `non_key_attributes`         | O                                                | Only required with `INCLUDE` as a projection type; a list of attributes to project into the index. These do not need to be defined as attributes on the table.                                                                                                          |
| ***global_secondary_index*** |
| `name`                       | R                                                | The name of the index                                                                                                                                                                                                                                                   |
| `write_capacity`             | Required only if<br>`billing_mode = PROVISIONED` | The provisioned write capacity units for this index                                                                                                                                                                                                                     |
| `read_capacity`              | Required only if<br>`billing_mode = PROVISIONED` | The provisioned read capacity units for this index                                                                                                                                                                                                                      |
| `hash_key`                   | R                                                | The name of the attribute used as your partition key - must be defined in attributes                                                                                                                                                                                    |
| `range_key`                  | O                                                | The name of the attribute used as your sort key - must be defined in attributes                                                                                                                                                                                         |
| `projection_type`            | R                                                | One of `ALL`, `INCLUDE`, `KEYS_ONLY` Where: <ul><li>`All` projects every attribute into the index</li><li>`KEYS_ONLY` projects just the hash and range key into the index</li><li>`INCLUDE` projects only the keys specified in the `non_key_attributes` parameter</li> |
| `non_key_attributes`         | O                                                | Only required with INCLUDE as a projection type; a list of attributes to project into the index. These do not need to be defined as attributes on the table.                                                                                                            |
| ***server_side_encryption*** |
| `enabled`                    | R                                                | Whether or not to enable encryption at rest using an AWS managed KMS Customer Master Key (CMK).                                                                                                                                                                         |
| `kms_key_arn`                | O                                                | The ARN of the CMK that should be used for the AWS KMS encryption. This attribute should only be specified if the key is different to the default DynamoDB CMK, `alias/aws/dynamodb`                                                                                    |
| ***point_in_time_recovery*** |
| `enabled`                    | R                                                | Whether to enable point-in-time recovery. Note - it can take up to 10 minutes to enable for new tables. If the `point_in_time_recovery` block is not provided, then this defaults to false.                                                                             |

### Lifecycle Section
You can also define a lifecycle section in your DynamoDB table resource in Terraform, and within this section you can specify certain arguments to ignore changes for. It's recommended to use the `ignore_changes` list for `read_capacity` and `write_capacity` if there's an autoscaling policy attached to the table.

It's important to note that you should include the `global_secondary_index` argument in the `ignore_changes` list, as changes via Terraform to the GSIs will tear down all the GSIs associated with your table and re-create them. This is especially dangerous for larger, heavily used GSIs as it can take hours to re-create an index. This means that if you want to add a GSI to an existing table or change the read or write capacity for an existing GSI, you should do so manually in the AWS console for each region. You should keep your GSI definition up to date in Terraform for the purposes of setting up existing resources in new regions or recovering infrastructure settings. Ultimately, the Terraform definition of a GSI will have no impact on the resource in production.

For more information and DynamoDB docs, refer to the [DynamoDB Developer Guide here](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Introduction.html)

## Glossary
### Projection 
Represents attributes that are copied (projected) from the table into an index. These are in addition to the primary key attributes and index key attributes, which are automatically projected.

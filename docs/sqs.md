# Amazon SQS dead-letter queues

Amazon SQS supports _dead-letter queues_ (DLQ) which other queues (source queues) can target for messages that can't be processed (consumed) successfully. Dead-letter queues are useful for debugging your application or messaging system because they let you isolate unconsumed messages to determine why their processing doens't succeed. 

## How do dead-letter queues work?
Sometimes, messages can't be processed because of a variety of possible issues, such as erroneous conditions within the producer or consumer application or an unexpected state change that causes an issue with your application code. For example, if a user places a web order with a particular product ID, but the product ID is deleted, the web store's code fails and displays an error. The message with the order is then sent to the dead-letter queue.

Occasionally, producers and consumers might fail to interpret aspects of the protocol that they use to communicate, causing message corruption or loss. Also, the consumer's hardware errors might corrupt the message payload.

The _redrive policy_ specifies the _source queue_, the _dead-letter queue_ and the conditions under which Amazon SQS moves messages from the former to the latter if the consumer of the source queue fails to process a message a specified number of times. The `maxReceiveCount` is the number of times the consumer tries receiving a message from a queue without deleting it before moving it to the dead-letter queue. Setting the `maxReceiveCount` to a low value such as 1 would mean that any failure to receive a message would cause that message to be moved to the dead-letter queue. Such failures include network errors and client dependency errors. To ensure that your system is resilient against errors, set the `maxReceiveAccount` high enough to allow for sufficient retries.

The _redrive allow policy_ specifies which source queues can access the dead-letter queue. This policy applies to a potential dead-letter queue. You can choose whether to allow all source queues, allow specific source queues or deny all source queues. The default is to allow all source queues to use the dead-letter queue. If you choose to allow specific queues (using the `byQueue` option), you can specify up to 10 source queues using the source queue Amazon Resource Name (ARN). If you specify `denyAll`, the queue cannot be used as a dead-letter queue.

To specify a dead-letter queue, you can use the console or the AWS SDK for Java, You mst do this for each queue that sends messages to a dead-letter queue. Multiple queues of the same type can target a single dead-letter queue.

!!! info

    The dead-letter queue of a FIFO (First In, First Out) queue must also be a FIFO queue. Similarly, the dead-letter queue of a standard queue must also be a standard queue.

    You must use the same AWS account to create the dead-letter queue and the other queues that send messages to the dead-letter queue. Also, dead-letter queues must reside in the same AWS region as other queues that use the dead-letter queue. For example, if you create a queue in the us-east-1 (Ohio) region and you want to use a dead-letter queue with that queue, then the second queue must also be in the us-east-1 (Ohio) region.

    The expiration of a messaage is always based on its original enqueue timestamp. When a message is moved to a dead-letter queue, the enqueue timestamp is unchanged. The `ApproximateAgeOfOldestMessage` metric indiciates when the message moved to the dead-letter queue, _not_ when the message was originally sent. For example, assume that a message spends 1 day in the original queue before it is moved to a dead-letter queue. If the dead-letter queue's retention period is 4 days, the message is deleted fom the dead-letter queue after 3 days and the `ApproximateAgeOfOldestMessage` is 3 days. Thus it is best practice to set the retention period of a dead-letter queue to be longer than the retention period of the original queue.

## What are the benefits of dead-letter queues?
The main task of a dead-letter queue is to handle the lifecycle of unconsumed messages. A dead-letter queue lets you set aside and isolate messages that can't be processed correctly to determine why their processing didn't succeed. Setting up a dead-letter queue allows you to:

* Configure an alarm for any messages move to a dead-letter queue
* Examine logs for exceptions that might have caused messages to be moved to a dead-letter queue
* Analyse the contents of messages moved to a dead-letter queue to diagnose software or the producer or consumer's hardware issues
* Determine whether you have given your consumer sufficient time to process messages

## How do different queue types handle message failure?

### Standard Queues
Standard queues keep processing messages until the expiration of the retention period. This continuous processing of messages minimises the chances of having your queue blocked by messages that can't be processed. Continuous message processing also provides faster recovery for your queue.

In a system that processes thousands of messages, having a large number of messages that the consumer repeatedly fails to acknowledge and delete might increase costs and place extra load on the hardware. Instead of trying to process failing messages until they expire, it is better to move them to a dead-letter queue after a few processing attempts.

!!! note

    Standard queues allow a high number of in-flight messages. If the majority of your messages can't be consumed and aren't sent to a dead-letter queue, your rate of processing valid messages can slow down. Thus, to maintain the efficiency of your queue, make sure that your application correctly manages message processing.

### FIFO Queues

FIFO queues provide exactly-once processing by consuming messages in sequence from a message group. Thus, although the consumer can continue to retrieve ordered messages from another message group, the first message group remains unavailable until the message blocking the queue is processed successfully.

!!! note

    FIFO queues allow for a lower number of in-flight messages. Thus to keep your FIFO queue from getting blocked by a message, make sure that your application correctly handles message processing.

## When should I use a dead-letter queue?

:white_check_mark: Do use dead-letter queues with standard queues. You should always take advantage of dead-letter-queues when your applications don't depend on ordering. Dead-letter queues can help you troubleshoot incorrect message transmission operations.

!!! note

    Even when you use dead-letter queues, you should continue to monitor your queues and retry sending messages that fail for 

:white_check_mark: Do use dead-letter queues to decrease the number of messages and to reduce the possibilty of exposing your system to poison-pill messages (messages that can be recieved but cannot be processed)

:x: Don't use a dead-letter queue with standard queues when you want to be able to keep retrying the transmission of a message indefinately. For example, don't use a dead-letter queue if your program must wait for a dependent process to become active or available

:x: Don't use a dead-letter queue with a FIFO queue if you don't want to break the exact order of messages or operations. For example, don't use a dead-letter queue with instructions in an Edit Decision List (EDL) for a video editing suite, where changing the order of edits changes the context of subsequent edits

## Moving message out of a dead-letter queue

You can use a _dead-letter queue redrive_ to manage the lifecycle of unconsumed messages. After you have investigated the attributes and related metadata available for standard unconsumed messages in a dead-letter queue, you can redrive the messages back to their source queues. Dead-letter queue redrive reduces API call billing by batching the messages while moving them.

The redrive task uses Amazon SQS's 






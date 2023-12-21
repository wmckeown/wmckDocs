# Work with SNS
With Amazon Simple Notification Service, you can easily push real-time notification messages from your applications to subscribers over multiple communication channels. The page covers some of the basic functions of Amazon SNS.

## Create a Topic
A **topic** is a logical grouping of communication channels that defines which systems to send a message to, for example, fanning out a message to a AWS Lambda and a HTTP webhook. You send messages to Amazon SNS, then they're distributed to the channels defined in the topic. This makes the messages available to the subscribers.

To create a topic, first build a CreateTopicRequest object, with the name of the topic set using the `name()` method in the builder. Then, send the request object to Amazon SNS by using the `createTopic()` method of the SnsClient. YOu can capture the result of this request as a `CreateTopicResponse` object as demonstrated in the code snippet below:

### Imports

```java
import software.amazon.awssdk.regions.Region;
import software.amazon.awssdk.services.sns.SnsClient;
import software.amazon.awssdk.services.sns.model.CreateTopicRequest;
import software.amazon.awssdk.services.sns.model.CreateTopicResponse;
import software.amazon.awssdk.services.sns.model.SnsException;
```

### Code

```java
public static String createSNSTopic(SnsClient snsClient, String topicName)  {
    CreateTopicResponse result = null;

    try {
        CreateTopicRequest request = CreateTopicRequest.builder()
            .name(topicName)
            .build();

        result = snsClient.createTopic(request);
        return result.topicArn();
    } catch (SnsException e)    {
        System.err.println(e.awsErrorDetails().errorMessage());
        System.exit(1);
    }
    return "";
}
```

## List your Amazon SNS Topics
To retrieve a list of your AMazon SNS topics, build a ListTopicsRequest object. Then, send the request object to Amazon SNS by using the `listTopics()` method of the `SnsClient`. You can capture the result of this request as a `ListTopicsResponse` object.

The following code snippet prints out the HTTP status code of the request and a list of the Amazon Resource Names (ARNs) of your AMazon SNS Topics

### Imports
```java
import software.amazon.awssdk.regions.Region;
import software.amazon.awssdk.services.sns.SnsClient;
import software.amazon.awssdk.services.sns.model.ListTopicsRequest;
import software.amazon.awssdk.services.sns.model.ListTopicsResponse;
import software.amazon.awssdk.services.sns.model.SnsException;
```

### Code
```java
public static void listSNSTopics(SnsClient snsClient)   {
    try {
        ListTopicsRequest request = ListTopicsRequest.builder()
            .build();

        ListTopicsResponse result = snsClient.listTopics(request);
        
    }
}
```


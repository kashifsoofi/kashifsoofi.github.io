---
title:  "Amazon Kinesis with LocalStack and .NET Core"
date:   2019-11-09
categories:
  - aspnetcore, aws, LocalStack
tags:
  - C#
  - aspnetcore
  - aws
  - LocalStack
---
[Amazon Kinesis](https://aws.amazon.com/kinesis/) is a data stream processing service from Amazon that makes it possible to ingest and process high volumes of data in realtime.  

[LocalStack](https://github.com/localstack/localstack) is a fully functional local AWS cloud stack, that provides an easy-to-use test/mocking framework for developing Cloud applications. This is in no way a replacement for actual cloud, in my opinion it can facilitate in development, testing, automated testing with incurring any costs to developer/company.  

This article and code used is hugely derived from [AWS : Kinesis](https://sachabarbs.wordpress.com/2018/09/17/aws-kinesis/) excellently written by [Sacha Barber](https://github.com/sachabarber). How this differs from that article is that I am going to use LocalStack for testing, .NET Core 3 for development and Docker for running LocalStack and applications.

## LocalStack Setup
LocalStack can be installed on your computer as well as can be run in Docker. My preferred method is to run in Docker using docker-compose. I also use another Docker image to create resources, application is dependent on. I use [Docker awscli](https://github.com/xueshanf/docker-awscli) as base image to run the commands to create resources. For this sample we will need to create a stream, I am naming it `demo-stream`.  

## Message Producer
Create a .NET Core 3 Console application named `Demo.Producer` and setup a timed background service to create and send messages to `demo-stream`. Add `AWSSDK.Kinesis` package reference to project. Initialise `AmazonKinesisClient` in producer service. In `OnTimedEvent` method, create a new `DataMessage`, create `PutRecordRequest` and call `PutRecordAsync` method to put the data message on Kinesis stream.  

Following code is to setup kinesis client to connect to `localstack`. You can change the server name to `localhost` if you are running `localstack` on your machine or running application on your machine.  
```
var serverName = "localstack";
_kinesisClient = new AmazonKinesisClient(
    "DUMMY_KEY",
    "DUMMY_KEY",
    new AmazonKinesisConfig
    {
        ServiceURL = $"http://{serverName}:4568",
    });
```

Following is the code to send message.  
```
var dataMessage = new DataMessage
{
    Id = Guid.NewGuid(),
    CreatedOn = e.SignalTime,
};

var messageBytes = Encoding.UTF8.GetBytes(JsonSerializer.Serialize<DataMessage>(dataMessage));
var putRecordRequest = new PutRecordRequest
{
    StreamName = _streamName,
    Data = new MemoryStream(messageBytes),
    PartitionKey = "demo-partition",
};

var putRecordResponse = _kinesisClient.PutRecordAsync(putRecordRequest).GetAwaiter().GetResult();
```

If you want to wait for stream to be available before starting to send messages, please have a look at [Amazon Kinesis Client Library for .NET](https://github.com/awslabs/amazon-kinesis-client-net)'s SampleProducer code.  

## Message Consumer using Kinesis Data Streams API
Create a .NET Core Console application named `Demo.Consumer`. Add a message consumer service class inheriting from `BackgroundService`, override `ExecuteAsync` method to read data from `demo-stream` we used to put data messages using our producer application.  

This method of consumer is using the same 'AWSSDK.Kinesis' nuget package. It makes a call to get all the shards of stream, then it gets an iterator on each shard to get all the records and print a console message. It is quite simple but consumer application has to pull the messages and also keep track of the data it has already processed. Our sample application would get all the messages on each run until the messages are available.  


Following code is to setup kinesis client to connect to `localstack`. You can change the server name to `localhost` if you are running `localstack` on your machine or running application on your machine.  
```
var serverName = "localstack";
_kinesisClient = new AmazonKinesisClient(
    "DUMMY_KEY",
    "DUMMY_KEY",
    new AmazonKinesisConfig
    {
        ServiceURL = $"http://{serverName}:4568",
    });
```

Following is the full code to read the data from the stream.  
```
private async Task ReadFromStream()
{
    var describeRequest = new DescribeStreamRequest
    {
        StreamName = _streamName,
    };

    var describeStreamResponse = await _kinesisClient.DescribeStreamAsync(describeRequest);
    var shards = describeStreamResponse.StreamDescription.Shards;
    foreach (var shard in shards)
    {
        var getShardIteratorRequest = new GetShardIteratorRequest
        {
            StreamName = _streamName,
            ShardId = shard.ShardId,
            ShardIteratorType = ShardIteratorType.TRIM_HORIZON,
        };

        var getShardIteratorResponse = await _kinesisClient.GetShardIteratorAsync(getShardIteratorRequest);
        var shardIterator = getShardIteratorResponse.ShardIterator;
        while (!string.IsNullOrEmpty(shardIterator))
        {
            var getRecordsRequest = new GetRecordsRequest
            {
                Limit = 100,
                ShardIterator = shardIterator,
            };

            var getRecordsResponse = await _kinesisClient.GetRecordsAsync(getRecordsRequest);
            var nextIterator = getRecordsResponse.NextShardIterator;
            var records = getRecordsResponse.Records;

            if (records.Count > 0)
            {
                Console.WriteLine($"Received {records.Count} records.");
                foreach (var record in records)
                {
                    var dataMessage = await JsonSerializer.DeserializeAsync<DataMessage>(record.Data);
                    Console.WriteLine($"DataMessage Id={dataMessage.Id}, CreatedOn={dataMessage.CreatedOn.ToString("yyyy-MM-dd HH:mm")}");
                }
            }
            shardIterator = nextIterator;
        }
    }
}
```

## Message Consumer using Kinesis Client Library (KCL)
We can also use Amazon Kinesis Client Library (KCL) for .NET to write message processor. This allows us to focus on processing records, as many of the tasks e.g. load balancing, checkpointing are handled by KCL.

KCL for .NET is an interface to MultiLangDaemon which is provided as a part of [Amazon KCL for JAVA](https://github.com/awslabs/amazon-kinesis-client) and .NET KCL communicats with JAVA daemon.  

You can follow [Getting started](https://github.com/awslabs/amazon-kinesis-client-net#getting-started) on [Amazon Kinesis Client Library for .NET](https://github.com/awslabs/amazon-kinesis-client-net) to setup the project. To summarize perform following steps  
1. Download the [sources](https://github.com/awslabs/amazon-kinesis-client-net)
2. Copy `Bootstrap` and `ClientLibrary` projects to your solution
3. Copy `SampleConsumer.cs` to your consumer project as a starting point
4. Set executeableName with fullpath in kcl.properties file
5. Set streamName in kcl.properites file

In the sample project I have added `DataMessageProcessor` implementing `IShardRecordProcessor` to process records. Messages are processed in `ProcessRecords` method.  

### Dockerize KCL Consumer application
Follow standard process of dockerizing .NET Core application. Now remember KCL depends on MutilLangDaemon which is a JAVA application, to run that we have to install JAVA to our final image. I am using `dotnet/core/runtime:3.0-alpine` as base image, I am using following to install OpenJDK. It is not optimised, so there is room for improvement.  
```
RUN apk --no-cache add openjdk11 --repository=http://dl-cdn.alpinelinux.org/alpine/edge/community
```
I also execute following to download all the required packages to make it as part of the image.  
```
RUN dotnet Bootstrap.dll --properties /app/consumer/kcl.properties
```
Finally I set the entry point of the image as follows as advised on KCL for .NET README.  
```
ENTRYPOINT ["dotnet", "Bootstrap.dll", "--properties", "/app/consumer/kcl.properties", "--execute"]
```
  
Message processing cannot be tested with `localstack` by changing the client url (at least I don't know how to change those url in java daemon). However this can be achieved with dns hijacking and redirecting traffic to aws from kcl container to localstack, but that would the discussion of another post.  

## Source
Source code for the producer, consumer and kcl consumer demo application, and docker-compose files are hosted on GitHub in [Kinesis](https://github.com/kashifsoofi/aws/tree/master/Messaging/Kinesis) repository.  

## References
In no particular order  
[Amazon Kinesis](https://aws.amazon.com/kinesis/)  
[LocalStack](https://github.com/localstack/localstack)  
[Docker awscli](https://github.com/xueshanf/docker-awscli)  
[Google](https://www.google.com)  
[create_kinesis_stream.sh](https://gist.github.com/etspaceman/137f7f540af32bf106873813c830a699)  
[https://sachabarbs.wordpress.com/2018/09/17/aws-kinesis/](https://sachabarbs.wordpress.com/2018/09/17/aws-kinesis/)  
[https://stackoverflow.com/questions/53375613/why-is-the-java-11-base-docker-image-so-large-openjdk11-jre-slim/53383373#53383373](https://stackoverflow.com/questions/53375613/why-is-the-java-11-base-docker-image-so-large-openjdk11-jre-slim/53383373#53383373)  
And many more
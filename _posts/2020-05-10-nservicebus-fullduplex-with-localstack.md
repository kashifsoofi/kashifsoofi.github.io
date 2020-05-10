---
title:  "NServiceBus Full Duplex With LocalStack SQS and InMemory Persistence"
date:   2020-05-10
categories:
  - aspnetcore, aws, LocalStack, NServiceBus
tags:
  - aspnetcore
  - aws
  - LocalStack
  - NServiceBus
---
This is a repurposed [NServiceBus Full Duplex](https://docs.particular.net/samples/fullduplex/) using SQS transport and InMemory persistence.

## NServiceBus
[NServiceBus](https://particular.net/nservicebus) is a service bus implementation for .NET. As a developer, it helps you create decoupled applications that are easier to maintain, extend and scale horizontally.

## LocalStack
[LocalStack](https://github.com/localstack/localstack) is a fully funcational local AWS cloud stack, it provides an easy to use test duoble for developing cloud applications targeting AWS.

## Components
The application consists of following components

### Shared.Core
Common library with common message types that are exchanged between Client and Server. It also contains a POCO to load configuration from appsettings.

### Api.Core
ASP.NET Core 3.1 WebApi application, this acts as NServiceBus client endpoint and sends messages to server. The code to send messages is in `MessagesController.cs`. It also implements a MessageHandler for `DataResponseMessage` to receive reply from server.

### Server.Core
ASP.NET Core 3.1 console application, this acts as NServiceBus server endpoint. It implements message handler for `RequestDataMessage`, that handler is invoked by NServiceBus dispatcher when a message arrives in this endpoint's queue.

## Run Sample With LocalStack

### Configuration to run with LocalStack
The magic happens in `NServiceBusService.cs` file in both `Api.Core` and `Server.Core` projects in `ConfigureEndpoint` method. We create an instance of `AmazonSQSConfig` and set its `ServiceURL` property if it is set in application configuration.
```
var amazonSqsConfig = new AmazonSQSConfig();
if (!string.IsNullOrEmpty(this.nServiceBusOptions.SqsServiceUrlOverride))
{
    amazonSqsConfig.ServiceURL = this.nServiceBusOptions.SqsServiceUrlOverride;
}
```
We use `ClientFactory` to use our configuration.
```
var transport = endpointConfiguration.UseTransport<SqsTransport>();
transport.ClientFactory(() => new AmazonSQSClient(
    new AnonymousAWSCredentials(),
    amazonSqsConfig));
```

For S3 storage for large messages, we create an instance of `AmazonS3Config` by setting `ForcePathStyle` property to true. This setting is needed as by default the client expects to append the bucket name to the domain name in order to access the bucket, and that would fail to reach our local S3 service running in LocalStack.
```
var amazonS3Config = new AmazonS3Config
{
    ForcePathStyle = true,
};
if (!string.IsNullOrEmpty(this.nServiceBusOptions.S3ServiceUrlOverride))
{
    amazonS3Config.ServiceURL = this.nServiceBusOptions.S3ServiceUrlOverride;
}
```
We use `ClientFactory` to use our configuration.
```
var s3Configuration = transport.S3("bucketname", "Samples-FullDuplex-Client");
s3Configuration.ClientFactory(() => new AmazonS3Client(
    new AnonymousAWSCredentials(),
    amazonS3Config));
```

### Running the sample
Execute following on a powershell terminal to start up `LocalStack` and setup sqs queues required to run the sampel
```
./dev-env.ps1 start
```
Now you can run both `Api.Core` and `Server.Core` by following 1 of the below
* In `Visual Studio` by starting multiple projects
* In `Visual Studio` by starting `docker-compose.dcproj`
* From terminal, by using docker-compose up command

In all 3 cases you should have both applications running and you can open a browser, navigate to `Api.Core` url and can send messages to `Server.Core`.

## TL;DR
Complete code for the sample with all the changes to run it against LocalStack container can be found in  [GitHub Repository Branch](https://github.com/kashifsoofi/nservicebus-with-localstack/tree/FullDuplex-With-SQS-And-InMemoryPersistence).

## References
In no particular order  
[NServiceBus](https://particular.net/nservicebus)  
[LocalStack](https://github.com/localstack/localstack)  
[NServiceBus Full Duplex Sample](https://docs.particular.net/samples/fullduplex/)  
And many more

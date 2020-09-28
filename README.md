# azure_functions

## Performance and scale in Durable Functions (Azure Functions)
To optimize performance and scalability, it's important to understand the unique scaling characteristics of Durable Functions.

To understand the scale behavior, you have to understand some of the details of the underlying Azure Storage provider.

## History table
The History table is an Azure Storage table that contains the history events for all orchestration instances within a task hub. The name of this table is in the form TaskHubNameHistory. As instances run, new rows are added to this table. The partition key of this table is derived from the instance ID of the orchestration. An instance ID is random in most cases, which ensures optimal distribution of internal partitions in Azure Storage.

When an orchestration instance needs to run, the appropriate rows of the History table are loaded into memory. These history events are then replayed into the orchestrator function code to get it back into its previously checkpointed state. The use of execution history to rebuild state in this way is influenced by the Event Sourcing pattern.

## Instances table
The Instances table is another Azure Storage table that contains the statuses of all orchestration and entity instances within a task hub. As instances are created, new rows are added to this table. The partition key of this table is the orchestration instance ID or entity key and the row key is a fixed constant. There is one row per orchestration or entity instance.

This table is used to satisfy instance query requests from the GetStatusAsync (.NET) and getStatus (JavaScript) APIs as well as the status query HTTP API. It is kept eventually consistent with the contents of the History table mentioned previously. The use of a separate Azure Storage table to efficiently satisfy instance query operations in this way is influenced by the Command and Query Responsibility Segregation (CQRS) pattern.

## Internal queue triggers
Orchestrator functions and activity functions are both triggered by internal queues in the function app's task hub. Using queues in this way provides reliable "at-least-once" message delivery guarantees. There are two types of queues in Durable Functions: the control queue and the work-item queue.

## The work-item queue
There is one work-item queue per task hub in Durable Functions. It is a basic queue and behaves similarly to any other queueTrigger queue in Azure Functions. This queue is used to trigger stateless activity functions by dequeueing a single message at a time. Each of these messages contains activity function inputs and additional metadata, such as which function to execute. When a Durable Functions application scales out to multiple VMs, these VMs all compete to acquire work from the work-item queue.

## Control queue(s)
There are multiple control queues per task hub in Durable Functions. A control queue is more sophisticated than the simpler work-item queue. Control queues are used to trigger the stateful orchestrator and entity functions. Because the orchestrator and entity function instances are stateful singletons, it's not possible to use a competing consumer model to distribute load across VMs. Instead, orchestrator and entity messages are load-balanced across the control queues. More details on this behavior can be found in subsequent sections.

Control queues contain a variety of orchestration lifecycle message types. Examples include orchestrator control messages, activity function response messages, and timer messages. As many as 32 messages will be dequeued from a control queue in a single poll. These messages contain payload data as well as metadata including which orchestration instance it is intended for. If multiple dequeued messages are intended for the same orchestration instance, they will be processed as a batch.

## Queue polling
The durable task extension implements a random exponential back-off algorithm to reduce the effect of idle-queue polling on storage transaction costs. When a message is found, the runtime immediately checks for another message; when no message is found, it waits for a period of time before trying again. After subsequent failed attempts to get a queue message, the wait time continues to increase until it reaches the maximum wait time, which defaults to 30 seconds.

The maximum polling delay is configurable via the maxQueuePollingInterval property in the host.json file. Setting this property to a higher value could result in higher message processing latencies. Higher latencies would be expected only after periods of inactivity. Setting this property to a lower value could result in higher storage costs due to increased storage transactions.

##  Note
`
When running in the Azure Functions Consumption and Premium plans, the Azure Functions Scale Controller will poll each control and work-item queue once every 10 seconds. This additional polling is necessary to determine when to activate function app instances and to make scale decisions. At the time of writing, this 10 second interval is constant and cannot be configured.
`
## Orchestration start delays
Orchestrations instances are started by putting an ExecutionStarted message in one of the task hub's control queues. Under certain conditions, you may observe multi-second delays between when an orchestration is scheduled to run and when it actually starts running. During this time interval, the orchestration instance remains in the Pending state. There are two potential causes of this delay:

Backlogged control queues: If the control queue for this instance contains a large number of messages, it may take time before the ExecutionStarted message is received and processed by the runtime. Message backlogs can happen when orchestrations are processing lots of events concurrently. Events that go into the control queue include orchestration start events, activity completions, durable timers, termination, and external events. If this delay happens under normal circumstances, consider creating a new task hub with a larger number of partitions. Configuring more partitions will cause the runtime to create more control queues for load distribution.

Back off polling delays: Another common cause of orchestration delays is the previously described back-off polling behavior for control queues. However, this delay is only expected when an app is scaled out to two or more instances. If there is only one app instance or if the app instance that starts the orchestration is also the same instance that is polling the target control queue, then there will not be a queue polling delay. Back off polling delays can be reduced by updating the host.json settings, as described previously.

## Storage account selection
The queues, tables, and blobs used by Durable Functions are created in a configured Azure Storage account. The account to use can be specified using the durableTask/storageProvider/connectionStringName setting (or durableTask/azureStorageConnectionStringName setting in Durable Functions 1.x) in the host.json file.

## Durable Functions 2.x
### JSON
`
{
  "extensions": {
    "durableTask": {
      "storageProvider": {
        "connectionStringName": "MyStorageAccountAppSetting"
      }
    }
  }
}
`
## Durable Functions 1.x
### JSON
`{
  "extensions": {
    "durableTask": {
      "azureStorageConnectionStringName": "MyStorageAccountAppSetting"
    }
  }
}
`
If not specified, the default AzureWebJobsStorage storage account is used. For performance-sensitive workloads, however, configuring a non-default storage account is recommended. Durable Functions uses Azure Storage heavily, and using a dedicated storage account isolates Durable Functions storage usage from the internal usage by the Azure Functions host.

## Orchestrator scale-out
Activity functions are stateless and scaled out automatically by adding VMs. Orchestrator functions and entities, on the other hand, are partitioned across one or more control queues. The number of control queues is defined in the host.json file. The following example host.json snippet sets the durableTask/storageProvider/partitionCount property (or durableTask/partitionCount in Durable Functions 1.x) to 3.

## Durable Functions 2.x
### JSON
`
{
  "extensions": {
    "durableTask": {
      "storageProvider": {
          "partitionCount": 3
      }
    }
  }
}
`
## Durable Functions 1.x
### JSON
`
{
  "extensions": {
    "durableTask": {
      "partitionCount": 3
    }
  }
}
`
A task hub can be configured with between 1 and 16 partitions. If not specified, the default partition count is 4.

When scaling out to multiple function host instances (typically on different VMs), each instance acquires a lock on one of the control queues. These locks are internally implemented as blob storage leases and ensure that an orchestration instance or entity only runs on a single host instance at a time. If a task hub is configured with three control queues, orchestration instances and entities can be load-balanced across as many as three VMs. Additional VMs can be added to increase capacity for activity function execution.

The following diagram illustrates how the Azure Functions host interacts with the storage entities in a scaled out environment.

<img align="left" width="100" height="100" src="https://docs.microsoft.com/en-us/azure/azure-functions/durable/media/durable-functions-perf-and-scale/scale-diagram.png">

https://docs.microsoft.com/en-us/azure/azure-functions/durable/durable-functions-perf-and-scale

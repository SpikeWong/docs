---
type: docs
title: Workflow patterns
linkTitle: Workflow patterns
weight: 3000
description: "Write different types of workflow patterns"
---

Dapr Workflows simplify complex, stateful coordination requirements in microservice architectures. The following sections describe several application patterns that can benefit from Dapr Workflows:

## Task chaining

In the task chaining pattern, multiple steps in a workflow are run in succession, and the output of one step may be passed as the input to the next step. Task chaining workflows typically involve creating a sequence of operations that need to be performed on some data, such as filtering, transforming, and reducing.

In some cases, the steps of the workflow may need to be orchestrated across multiple microservices. For increased reliability and scalability, you're also likely to use queues to trigger the various steps.

<img src="/images/workflow-overview/workflows-chaining.png" width=800 alt="Diagram showing how the task chaining workflow pattern works">

While the pattern is simple, there are many complexities hidden in the implementation. For example:

- What happens if one of the microservices are unavailable for an extended period of time?
- Can failed steps be automatically retried?
- If not, how do you facilitate the rollback of previously completed steps, if applicable?
- Implementation details aside, is there a way to visualize the workflow so that other engineers can understand what it does and how it works?

Dapr Workflow solves these complexities by allowing you to implement the task chaining pattern concisely as a simple function in the programming language of your choice, as shown in the following example.

{{< tabs ".NET" >}}

{{% codetab %}}

```csharp
// Expotential backoff retry policy that survives long outages
var retryPolicy = TaskOptions.FromRetryPolicy(new RetryPolicy(
    maxNumberOfAttempts: 10,
    firstRetryInterval: TimeSpan.FromMinutes(1),
    backoffCoefficient: 2.0,
    maxRetryInterval: TimeSpan.FromHours(1)));

// Task failures are surfaced as ordinary .NET exceptions
try
{
    var result1 = await context.CallActivityAsync<string>("Step1", wfInput, retryPolicy);
    var result2 = await context.CallActivityAsync<byte[]>("Step2", result1, retryPolicy);
    var result3 = await context.CallActivityAsync<long[]>("Step3", result2, retryPolicy);
    var result4 = await context.CallActivityAsync<Guid[]>("Step4", result3, retryPolicy);
    return string.Join(", ", result4);
}
catch (TaskFailedException)
{
    // Retries expired - apply custom compensation logic
    await context.CallActivityAsync<long[]>("MyCompensation", options: retryPolicy);
    throw;
}
```

{{% /codetab %}}

{{< /tabs >}}

{{% alert title="Note" color="primary" %}}
In the example above, `"Step1"`, `"Step2"`, `"MyCompensation"`, etc. represent workflow activities, which are functions in your code that actually implement the steps of the workflow. For brevity, these activity implementations are left out of this example.
{{% /alert %}}

As you can see, the workflow is expressed as a simple series of statements in the programming language of your choice. This allows any engineer in the organization to quickly understand the end-to-end flow without necessarily needing to understand the end-to-end system architecture.

Behind the scenes, the Dapr Workflow runtime:

- Takes care of executing the workflow and ensuring that it runs to completion.
- Saves progress automatically.
- Automatically resumes the workflow from the last completed step if the workflow process itself fails for any reason.
- Enables error handling to be expressed naturally in your target programming language, allowing you to implement compensation logic easily.
- Provides built-in retry configuration primitives to simplify the process of configuring complex retry policies for individual steps in the workflow.

## Fan-out/fan-in

In the fan-out/fan-in design pattern, you execute multiple tasks simultaneously across potentially multiple workers, wait for them to finish, and perform some aggregation on the result.

<img src="/images/workflow-overview/workflows-fanin-fanout.png" width=800 alt="Diagram showing how the fan-out/fan-in workflow pattern works">

In addition to the challenges mentioned in [the previous pattern]({{< ref "workflow-overview.md#task-chaining" >}}), there are several important questions to consider when implementing the fan-out/fan-in pattern manually:

- How do you control the degree of parallelism?
- How do you know when to trigger subsequent aggregation steps?
- What if the number of parallel steps is dynamic?

Dapr Workflows provides a way to express the fan-out/fan-in pattern as a simple function, as shown in the following example:

{{< tabs ".NET" >}}

{{% codetab %}}

```csharp
// Get a list of N work items to process in parallel.
object[] workBatch = await context.CallActivityAsync<object[]>("GetWorkBatch", null);

// Schedule the parallel tasks, but don't wait for them to complete yet.
var parallelTasks = new List<Task<int>>(workBatch.Length);
for (int i = 0; i < workBatch.Length; i++)
{
    Task<int> task = context.CallActivityAsync<int>("ProcessWorkItem", workBatch[i]);
    parallelTasks.Add(task);
}

// Everything is scheduled. Wait here until all parallel tasks have completed.
await Task.WhenAll(parallelTasks);

// Aggregate all N outputs and publish the result.
int sum = parallelTasks.Sum(t => t.Result);
await context.CallActivityAsync("PostResults", sum);
```

{{% /codetab %}}

{{< /tabs >}}

The key takeaways from this example are:

- The fan-out/fan-in pattern can be expressed as a simple function using ordinary programming constructs
- The number of parallel tasks can be static or dynamic
- The workflow itself is capable of aggregating the results of parallel executions

While not shown in the example, it's possible to go further and limit the degree of concurrency using simple, language-specific constructs. Furthermore, the execution of the workflow is durable. If a workflow starts 100 parallel task executions and only40 complete before the process crashes, the workflow will restart itself automatically and schedule only the remaining 60 tasks.

## Async HTTP APIs

Asynchronous HTTP APIs are typically implemented using the [Asynchronous Request-Reply pattern](https://learn.microsoft.com/azure/architecture/patterns/async-request-reply). Implementing this pattern traditionally involves the following:

1. A client sends a request to an HTTP API endpoint (the _start API_)
1. The _start API_ writes a message to a backend queue, which triggers the start of a long-running operation
1. Immediately after scheduling the backend operation, the _start API_ returns an HTTP 202 response to the client with a `Location` header for the operation's _status API_
1. The _status API_ queries a database that contains the status of the long-running operation
1. The client repeatedly polls the _status API_ either until some timeout expires or it receives a "completion" response

The end-to-end flow is illustrated in the following diagram.

<img src="/images/workflow-overview/workflow-async-request-response.png" width=800 alt="Diagram showing how the async request response pattern works"/>

The challenge with implementing the asynchronous request-reply pattern is that it involves the use of multiple APIs and state stores. It also involves implementing the protocol correctly so that the client knows how to automatically poll for status and know when the operation is complete.

The Dapr workflow HTTP API supports the asynchronous request-reply pattern out-of-the box, without requiring you to write any code or do any state management. The following `curl` commands illustrate how the workflow APIs support this pattern.

```bash
curl -i -X TODO
```
```http
HTTP/1.1 202 Accepted
Content-Type: application/json

{TODO}
```

The HTTP client can then poll the endpoint in the `Location` response header repeatedly until it sees the "COMPLETE", "FAILURE", or "TERMINATED" status in the payload.

```bash
curl -i -X TODO
```
```http
HTTP/1.1 200 OK
Content-Type: application/json

{TODO}
```

## Monitor

The monitor pattern is a flexible, recurring process in a workflow that coordinates the actions of multiple threads by controlling access to shared resources. Typically:

1. The thread must first acquire the monitor. 
1. Once the thread has acquired the monitor, it can access the shared resource.
1. The thread then releases the monitor. 

This ensures that only one thread can access the shared resource at a time, preventing synchronization issues.

TODO: DIAGRAM?

In a few lines of code, you can create multiple monitors that observe arbitrary endpoints. The following code implements a basic monitor:

TODO: CODE EXAMPLE

## Next steps

{{< button text="Workflow architecture >>" page="workflow-architecture.md" >}}

## Related links

- [Try out Dapr Workflows using the quickstart](todo)
- [Workflow overview]({{< ref workflow-overview.md >}})
- [Workflow API reference]({{< ref workflow_api.md >}})
- Learn more about [how to manage workflows with the .NET SDK](todo) and try out [the .NET example](https://github.com/dapr/dotnet-sdk/tree/master/examples/Workflow)

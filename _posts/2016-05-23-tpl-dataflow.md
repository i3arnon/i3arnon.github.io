---
layout: post
title: TPL Dataflow Is The Best Library You're Not Using
description: TPL Dataflow is a concurrent actor library on top of the TPL. It's simple and efficient and not enough people are using it.
tags:
    - tpl-dataflow
    - tpl
    - async-await
---

[TPL Dataflow](https://msdn.microsoft.com/en-us/library/hh228603(v=vs.110).aspx) is an in-process actor library on top of the Task Parallel Library enabling more robust concurrent programming. It was first introduced in the [async-ctp](https://blogs.msdn.microsoft.com/csharpfaq/2010/10/28/whats-next-in-c-get-ready-for-async/) (the preview for async-await) but was eventually released as a standalone [nuget package](https://www.nuget.org/packages/Microsoft.Tpl.Dataflow). It abstracts away most of the hard work needed when building asynchronous and/or parallel processing code but I feel most people who might benefit from it aren't aware of it.
<!--more-->

A basic building block in the Dataflow library is the `ActionBlock`. You simply create it, tell it what to do, start posting items into it and wait untill it's done. It will buffer the items, start a task to process them sequentially (by default) and end the task when the buffer is empty. Here's a simple example:

```csharp
// Create a block with a synchronous action
var block = new ActionBlock<Hamster>(_ => _.Feed());

// Add items to the block and start processing
foreach (Hamster hamster in hamsters)
{
    block.Post(hamster);
}

block.Complete(); // Tell the block to complete and stop accepting new items
await block.Completion; // Asynchronously wait until all items completed processing
```

However, production-ready code would need to support async-await, concurrency, cancellation and capping the internal buffer to avoid uncontrolled memory growth. `ActionBlock` (and the rest of TPL Dataflow) support all that right out of the box with an options property-bag:

```csharp
// Create a block with an asynchronous action
var block = new ActionBlock<Hamster>(
    async _ => await _.FeedAsync(),
    new ExecutionDataflowBlockOptions
    {
        BoundedCapacity = 10000, // Cap the item count
        CancellationToken = cancellationToken, // Enable cancellation
        MaxDegreeOfParallelism = Environment.ProcessorCount, // Parallelize on all cores
    });

// Add items to the block and asynchronously wait if BoundedCapacity is reached
foreach (Hamster hamster in hamsters)
{
    await block.SendAsync(hamster);
}

block.Complete();
await block.Completion;
```

Notice that we now use `SendAsync` as it allows to asynchronously wait when there's no more room in the block for new items as opposed to `Post` which tries to add an item synchronously and returns `false` if the block is full. This allows throttling producers when the consumers can't handle the load. Implementing all these features yourself would take many lines of code, too much time and probably result in a few bugs.

Now, while `ActionBlock` is the most useful block and alone cover the vast majority of use-cases, there are other blocks like `BufferBlock`, `TransformBlock`, `BatchBlock`, etc. that can be joined together like LEGOs to create processing pipelines (i.e. data flows):

```csharp
// Create a block that receives IngredientBatches and generates Pizzas
var pizzaMakerBlock = new TransformBlock<IngredientBatch, Pizza>(
    _ => Pizza.Make(_.Dough, _.Sauce, _.Toppings));

// Create a block to concurrently and asynchronously deliver the Pizzas
var deliveryBlock = new ActionBlock<Pizza>(
    async _ => await _.DeliverAsync(),
    new ExecutionDataflowBlockOptions
    {
        MaxDegreeOfParallelism = 20, // delivery guys
    });

// Link the blocks together to build a pipeline
pizzaMakerBlock.LinkTo(
    deliveryBlock, 
    new DataflowLinkOptions { PropagateCompletion = true });

// Add items to the first block and let them flow through the pipeline
foreach (IngredientBatch batch in ingredientBatches)
{
    pizzaMakerBlock.Post(batch);
}

pizzaMakerBlock.Complete(); // Completion will propagate to the next block in the pipe
await deliveryBlock.Completion; // It's only necessary to wait for the last block
```

Dataflow's simplicity and robustness however don't seem to lead to popularity and usage. It only has [269 questions on Stack Overflow](http://stackoverflow.com/questions/tagged/tpl-dataflow) (>20% of them I answered) and it's [shadowed by Reactive Extensions](https://www.google.co.uk/trends/explore#q=tpl%20dataflow%2C%20reactive%20extensions&cmpt=q&tz=Etc%2FGMT-3) (which can cover some of the same use-cases). My assumption is that being one of the first parts of .NET to be released as a separate nuget package limited the exposure it would have otherwise received (at least it led the way for present-day .NET Core where all libraries are separate nuget packages). But in my experience when people discover Dataflow and see how it solves their issues [they](http://stackoverflow.com/a/24966167/885318) [are](http://stackoverflow.com/a/35686494/885318) [happy](http://stackoverflow.com/a/34843290/885318) [to](http://stackoverflow.com/a/34361999/885318) [embrace](http://stackoverflow.com/a/27842076/885318) [it](http://stackoverflow.com/a/26009467/885318).

So, this is me acting as a self-appointed TPL Dataflow evangelist. My entire product ([Microsoft ATA](https://www.microsoft.com/en-us/server-cloud/products/advanced-threat-analytics/overview.aspx)) runs inside various `ActionBlock`s' delegates and I encourage you to consider how it fits your code and architecture as well. 

If you already use TPL Dataflow I would be happy to hear about how and in what codebase in the comments.
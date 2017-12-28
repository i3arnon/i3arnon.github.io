---
layout: post
title: Introducing AsyncUtilities
description: A collection of somewhat useful utilities and extension methods for async programming
tags:
    - async-await
    - value-task
---

I've recently (~6 months ago) started collecting various utility classes and extension methods related to asynchronous programming into a library: [AsyncUtilities](https://github.com/i3arnon/AsyncUtilities). Since there already many useful tools in the BCL and in existing libraries out there these are quite on the fringe:

# Utilities

## `ValueTask`

When writing async methods that usually complete synchronously `ValueTask<TResult>` can be used to avoid allocating a `Task<TResult>` instance in the synchronous case. There isn't a non-generic version of `ValueTask<TResult>` in the BCL. An explanation can be found in the [comments for `ValueTask<TResult>` in the corefx repository](https://github.com/dotnet/corefx/blob/master/src/System.Threading.Tasks.Extensions/src/System/Threading/Tasks/ValueTask.cs#L46):

> There is no non-generic version of `ValueTask<TResult>` as the `Task.CompletedTask` property may be used to hand back a successfully completed singleton in the case where a `Task`-returning method completes synchronously and successfully.

However, when you have a tree of async methods without results and they in turn only call methods that return `ValueTask<TResult>` you need to either allocate `Task` instances, even if the operations complete synchronously, or you need to remove the `async` keyword and custom-implement the async mechanism with `Task.ContinueWith`.

Because I dislike both of these options I made a non-generic `ValueTask`. Here's how it can be used:

```csharp
async ValueTask DrawAsync(string name)
{
    var pen = await GetItemAsync<Pen>("pen");
    var apple = await GetItemAsync<Apple>("apple");

    var applePen = pen.JamIn(apple);
    applePen.Draw();
}

async ValueTask<T> GetItemAsync<T>(string name)
{
    var item = GetFromCache<T>(name);
    if (item != null)
    {
        return item;
    }

    return await GetFromDbAsync<T>(name);
}
```

# Extension Methods

## `Task.ContinueWithSynchronously`

When implementing low-level async constructs, it's common to add a small continuation using `Task.ContinueWith` instead of using an async method (which adds the state machine overhead). To do that efficiently and safely you need the `TaskContinuationOptions.ExecuteSynchronously` and make sure it runs on the `ThreadPool`:

```csharp
Task.Delay(1000).ContinueWith(
    _ => Console.WriteLine("Done"),
    CancellationToken.None,
    TaskContinuationOptions.ExecuteSynchronously,
    TaskScheduler.Default);
```

`Task.ContinueWithSynchronously` encapsulates that for you (with all the possible overloads):

```csharp
Task.Delay(1000).ContinueWithSynchronously(_ => Console.WriteLine("Done"));
```
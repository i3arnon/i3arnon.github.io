---
layout: post
title: Not All Beginnings Must Have An End
description: In APM you need to match every BeginXXX with an EndXXX. In this post I'll try to convince you that if a class implements IDisposable correctly you can forgo calling EndXXX if Dispose was already invoked
tags:
    - apm
    - idisposable
    - async-await
---

Those of you who are familiar with the [Asynchronous Programming Model (APM)](https://msdn.microsoft.com/en-us/library/ms228963(v=vs.110).aspx) know that you need to match every `BeginXXX` with an `EndXXX`. In this post I'll try my best to convince you that if a class implements `IDisposable` correctly you can forgo calling `EndXXX` if `Dispose` was already invoked. [I've tried doing that before on Stack Overflow](http://stackoverflow.com/q/30222269/885318) without much success and was faced with a lot of pushback.
<!--more-->

# Asynchronous Programming Model

First, a short explanation on what APM even is. APM is a pattern for writing callback-based asynchronous code. Almost all asynchronous code before .NET 4.5 and async/await was based on APM. It's a pattern where for each asynchronous operation there's a `BeginXXX` method and an `EndXXX` method (e.g. `BeginSend` and `EndSend`). `BeginXXX` accepts the operation's parameters and a callback to be invoked when the operation completes and returns an `IAsyncResult` representing the asynchronous operation. `EndXXX` (which is usually called inside the callback) accepts the same `IAsyncResult` returned from `BeginXXX` and returns the operation's result (if the operation hadn't completed yet the call will block until it does). Here's how it looks in code (I'll be using `UdpClient` for examples but it's the same for `TcpClient`, `Socket`, etc.):

```csharp
byte[] data = // ...
UdpClient udpClient = // ...
udpClient.BeginSend(
    data,
    data.Length,
    asyncResult =>
    {
        try
        {
            udpClient.EndSend(asyncResult); // Observe exceptions
            Console.WriteLine("Sent successfully");
        }
        catch (SocketException exception)
        {
            // Handle...
        }
    },
    null);
```

It's important to call `EndXXX` even when the operation doesn't return a result to observe exceptions.

Nowadays asynchronous code is usually written with async/await and .NET offers task-returning methods next to the APM method for most BCL types. However, under the hood most of these methods are still APM methods covered up with a task-returning overload using `Task.Factory.FromAsync`. You can see, for example, in [`UdpClient.ReceiveAsync`](http://referencesource.microsoft.com/#System/net/System/Net/Sockets/UDPClient.cs,57ff640dfcb1cbf0) the calls to `BeginReceive` and `EndReceive`:

```csharp
public Task<UdpReceiveResult> ReceiveAsync()
{
    return Task<UdpReceiveResult>.Factory.FromAsync(
        (callback, state) => BeginReceive(callback, state),
        (ar) =>
        {
            IPEndPoint remoteEP = null;
            Byte[] buffer = EndReceive(ar, ref remoteEP);
            return new UdpReceiveResult(buffer, remoteEP);
        },
        null);
}
```

# Cancellation

These async methods (which are actually APM methods) usually don't accept a `CancellationToken` because it was only added in .NET 4.0 (if they do it may only have an effect [before the operation started](http://referencesource.microsoft.com/#mscorlib/system/io/stream.cs,716)) and rarely accept a timeout so they are [non-cancellable](http://blogs.msdn.com/b/pfxteam/archive/2012/10/05/how-do-i-cancel-non-cancelable-async-operations.aspx). What you usually do if a task gets "stuck" is just *pretend* as if they **were** cancellable by attaching a task to it that you can cancel and abandoning the original task on cancellation, for example:

```csharp
static Task<UdpReceiveResult> ReceiveAsync(
    this UdpClient udpClient,
    CancellationToken cancellationToken)
{
    // Start the original operation
    Task<UdpReceiveResult> receiveTask = udpClient.ReceiveAsync();

    // Add support for cancellation
    return receiveTask.WithCancellation(cancellationToken);
}

static async Task<T> WithCancellation<T>(
    this Task<T> task,
    CancellationToken cancellationToken)
{
    // Create a self-cancelling TaskCompletionSource 
    var tcs = new TaskCompletionSourceWithCancellation<T>(cancellationToken);

    // Wait for completion or cancellation
    Task<T> completedTask = await Task.WhenAny(task, tcs.Task);
    return await completedTask;
}
```

`WithCancellation` uses `TaskCompletionSourceWithCancellation` which is a useful utility for having a self cancelling `TaskCompletionSource`. It accepts a `CancellationToken` and adds a registration to complete the `TaskCompletionSource` when the token is cancelled.

```csharp
class TaskCompletionSourceWithCancellation<TResult> : TaskCompletionSource<TResult>
{
    public TaskCompletionSourceWithCancellation(CancellationToken cancellationToken)
    {
        CancellationTokenRegistration registration =
            cancellationToken.Register(() => TrySetResult(default(TResult)));
        
        // Remove the registration after the task completes
        Task.ContinueWith(_ => registration.Dispose()); 
    }
}
```

That's all well and good, but that original task (the one returned from `ReceiveAsync` in this case) is still out there holding on to some state and waiting to be completed. The `WithCancellation` extension only allows you as the consumer to move forward on cancellation but it can't actually abort the original task. The only way to get rid of it is to dispose of the instance itself (if it's `IDisposable`). Calling `Dispose` on the instance will go over all the pending operations and complete them by invoking all the callbacks. Since these callbacks almost always call `EndXXX` (or `EndReceive` in this case) and the instance was already disposed these tasks will end up with an `ObjectDisposedException` (which will eventually lead to an `UnobservedTaskException`).

So, for each of these "stuck" tasks you can either let it linger forever (AKA memory leak) or clean it up with `Dispose` that results in an exception. Avoiding a memory leak is preferable but too much of these exceptions (>50 per second) could easily hurt your performance.

# IDisposable Trumps APM

Now, since calling `Dispose` on an `IDisposable` instance cleans all resources up and invoking `EndXXX` on that instance after `Dispose` has no effect as it throws an `ObjectDisposedException` immediately there's no reason to call `EndXXX` after cancellation to begin with, especially if these exceptions become a performance issue. You can't control the implementation of the task-returning wrapper, but you can create one of your own that does accept a `CancellationToken` but doesn't call `EndXXX` if the token is cancelled. That way you can abandon the non-cancellable "stuck" operation, dispose of the instance to clear resources and avoid the unnecessary `ObjectDisposedException`:

```csharp
static Task<UdpReceiveResult> UnsafeReceiveAsync(
    this UdpClient udpClient,
    CancellationToken cancellationToken)
{
    return UnsafeFromAsync(
        udpClient.BeginReceive,
        asyncResult =>
        {
            IPEndPoint remoteEP = null;
            byte[] buffer = udpClient.EndReceive(asyncResult, ref remoteEP);
            return new UdpReceiveResult(buffer, remoteEP);
        },
        cancellationToken);
}
```

This also requires having your own version of `Task.Factory.FromAsync` because this method is the one that actually invokes `EndXXX`

```csharp
static Task<TResult> UnsafeFromAsync<TResult>(
    Func<AsyncCallback, object, IAsyncResult> beginMethod,
    Func<IAsyncResult, TResult> endMethod,
    CancellationToken cancellationToken)
{
    // Create a self-cancelling TaskCompletionSource 
    var taskCompletionSource =
        new TaskCompletionSourceWithCancellation<TResult>(cancellationToken);

    beginMethod(
        asyncResult =>
        {
            try
            {
                // Don't call endMethod if cancellation was requested
                if (!cancellationToken.IsCancellationRequested)
                {
                    taskCompletionSource.TrySetResult(endMethod(asyncResult));
                }
            }
            catch (OperationCanceledException operationCanceledException)
            {
                taskCompletionSource.TrySetCanceled(
                    operationCanceledException.CancellationToken);
            }
            catch (Exception exception)
            {
                taskCompletionSource.TrySetException(exception);
            }
        },
        null);

    return taskCompletionSource.Task;
}
```

Keep in mind that this pattern is only appropriate in **extreme cases** where the exception rate has been shown to be too high (hence the `Unsafe` prefix).

# In Summary

Some asynchronous operations can "block" indefinitely (imagine waiting for a response that will never come) and the only way to really cancel them is to dispose of their underlying instance. In that case invoking `EndXXX` is pointless as it only throws an exception without doing anything. These exceptions are usually not an issue, but when they becomes one, it's perfectly safe to just avoid calling `EndXXX` and to let `Dispose` clear all resources held by the instance.

*[APM]: Asynchronous Programming Model
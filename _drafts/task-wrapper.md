---
layout: post
title: The Issue With Scoped Async Synchronization Constructs
description: Async code requires async synchronization constructs .NET doesn't provide. Most implementations have an issue as they usually return a task.
tags:
    - async-await
    - bugs
    - idisposable
---

With async-await becoming more and more prevalent in modern code so has the need for async synchronization constructs. Unlike their synchronous counterparts (e.g. `Monitor`, `Mutex`, `ReaderWriterLock`, etc.) .NET doesn't offer almost any built-in asynchronous synchronization constructs, but it does contain the basic building blocks to build them on your own (mainly `Task`, `TaskCompletionSource<T>` and `SemaphoreSlim.WaitAsync`). [Stephen Toub](https://github.com/stephentoub) published a series of posts (over 5 years ago) on the [Parallel Framework team's blog](https://blogs.msdn.microsoft.com/pfxteam/) demonstrating that by building [`AsyncSemaphore`](https://blogs.msdn.microsoft.com/pfxteam/2012/02/12/building-async-coordination-primitives-part-5-asyncsemaphore/), [`AsyncLock`](https://blogs.msdn.microsoft.com/pfxteam/2012/02/12/building-async-coordination-primitives-part-6-asynclock/), [`AsyncReaderWriterLock`](https://blogs.msdn.microsoft.com/pfxteam/2012/02/12/building-async-coordination-primitives-part-7-asyncreaderwriterlock/) and more.

However, there's an issue with most implementations of the scoped async synchronization constructs: they usually return a task.
<!--more-->

Let's take `AsyncLock` built using a `SemaphoreSlim` as an example. Since a semaphore limits the concurrent access to a resource to a certain degree a semaphore of 1 is basically a lock. Here's a simple implementation (a more efficient one can be found [here](http://stackoverflow.com/a/21011273/885318)):

```csharp
class AsyncLock
{
    private readonly SemaphoreSlim _semaphore = new SemaphoreSlim(1, 1);

    public async Task<IDisposable> LockAsync()
    {
        await _semaphore.WaitAsync();
        return new Releaser(_semaphore);
    }

    private class Releaser : IDisposable
    {
        private readonly SemaphoreSlim _semaphore;

        public Releaser(SemaphoreSlim semaphore)
        {
            _semaphore = semaphore;
        }

        public void Dispose()
        {
            _semaphore.Release();
        }
    }
}
```

While we could expose an API of `AsyncLock.AcquireAsync` and `AsyncLock.Release` to be used with a `try-finally` block:

```csharp
await _lock.AcquirAsync();
try
{
    // critical section...
}
finally
{
    _lock.Release();
}
```

It's much simpler to use (as most implementation do) a scoped API that returns a `Task<IDisposable>`. This allows using a `using` block to automatically release the lock at the end of the critical section (even when an exception is thrown inside it):

```csharp
using (await _lock.LockAsync())
{
    // critical section...
}
```

The subtle issue with this approach is that most developers forget that tasks are by themselves `IDisposable`. That makes the following code compile perfectly well:

```csharp
using (_lock.LockAsync())
{
    // critical section...
}
```

The missing `await` means the code doesn't wait for the lock to be acquired and proceeds regardless so the lock doesn't lock anything and the critical section can be run concurrently. Usually when you don't await a `Task` inside an async method the compiler will warn you, but not in this case as the returned `Task` isn't abandoned and is being used by the using block.

It's trivially easy to include this bug in your code (I know I have) but quite difficult to realize that once you did, especially since this will only blow up in runtime if you dispose of the task at the exact time when the lock is contended.

My simple solution is not to return `Task<IDisposable>` from async synchronization constructs, but a new awaitable named `TaskWrapper` that doesn't implement `IDisposable`. All `TaskWrapper` needs to do to be an awaitable is to return a valid awaiter, and it does so by returning the `TaskAwaiter` for the original underlying task:

```csharp
struct TaskWrapper<T>
{
    private readonly Task<T> _task;

    public TaskWrapper(Task<T> task)
    {
        _task = task;
    }

    public TaskAwaiter<T> GetAwaiter() => _task.GetAwaiter();
}
```

Since you can't return something other than `Task`/`Task<T>`/`void` from async methods (at least until C# 7.0 is released supporting [arbitrary async returns](/2016/07/25/arbitrary-async-returns/)) it's simpler to split `LockAsync` into 2 separate methods, an internal async one that actually implements the locking logic and another that wraps the returned `Task<IDisposable>` with a `TaskWrapper<T>`:

```csharp
public TaskWrapper<IDisposable> LockAsync() =>
    new TaskWrapper<IDisposable>(LockInternalAsync());

private async Task<IDisposable> LockInternalAsync()
{
    await _semaphore.WaitAsync();

    return new Releaser(_semaphore);
}
```

Doing this prevents our offensive code from compiling as `TaskWrapper<IDisposable>` doesn't implement `IDisposable` and so can't be used in a using statement, but awaiting it results in an `IDisposable` (not to be confused with `Task<IDisposable>`) that can and should be used in that using statement. That way you don't need to worry whether you've missed an `await` somewhere since the compiler will do that for you.

Next you can find a more efficient (and more complicated) implementation of `AsyncLock` using `TaskWrapper`:

```csharp
class AsyncLock
{
    private readonly SemaphoreSlim _semaphore;
    private readonly IDisposable _releaser;
    private readonly Task<IDisposable> _releaserTask;

    public AsyncLock()
    {
        _semaphore = new SemaphoreSlim(1, 1);
        _releaser = new Releaser(_semaphore);
        _releaserTask = Task.FromResult(_releaser);
    }

    public TaskWrapper<IDisposable> LockAsync()
    {
        var waitTask = _semaphore.WaitAsync();
        return new TaskWrapper<IDisposable>(
            waitTask.IsCompleted
                ? _releaserTask
                : waitTask.ContinueWith(
                    (_, releaser) => (IDisposable)releaser,
                    _releaser,
                    CancellationToken.None,
                    TaskContinuationOptions.ExecuteSynchronously,
                    TaskScheduler.Default));
    }

    private class Releaser : IDisposable
    {
        private readonly SemaphoreSlim _semaphore;

        public Releaser(SemaphoreSlim semaphore)
        {
            _semaphore = semaphore;
        }

        public void Dispose()
        {
            _semaphore.Release();
        }
    }
}
```
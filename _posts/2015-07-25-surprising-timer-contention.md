---
layout: post
title: Surprising Contention In System.Threading.Timer
tags: [contention, lock, performance, timer]
---

While profiling our application's performance we stumbled upon a surprising contention point inside `System.Threading.Timer` (I say surprising as `System.Threading.Timer` is the more appropriate timer for multi-threaded environments out of the available timers)
<!--more-->
This can be demonstrated by executing the following piece of code:

```csharp
static void Main()
{
    for (var i = 0; Environment.ProcessorCount > i; i++)
    {
        Task.Factory.StartNew(() =>
        {
            while (true)
            {
                new Timer(_ => { }, null, TimeSpan.FromMilliseconds(100), Timeout.InfiniteTimeSpan);
            }
        }, TaskCreationOptions.LongRunning);
    }

    Console.ReadLine();
}
```

It starts a thread for each core on the machine that creates timers in a loop (as long as memory allows). If you open Performance Monitor while this runs you would see very high values for the "Contention Rate / sec" performance counter (more than 1000 per second on my 4 core machine).

[Looking at the code](http://referencesource.microsoft.com/#mscorlib/system/threading/timer.cs) you can see that when you create a new `Timer` (or change an existing timer) the internal `Timer.Change` method takes a lock on a singleton instance of `TimerQueue`. Since this is a simple lock (i.e. `Monitor`) on a singleton instance, every `Timer` you create in your entire `AppDomain` is contention waiting to happen:

```csharp
internal bool Change(uint dueTime, uint period)
{
    bool success;

    lock (TimerQueue.Instance) // Global lock.
    {
        if (m_canceled)
            throw new ObjectDisposedException(null, Environment.GetResourceString("ObjectDisposed_Generic"));

        // prevent ThreadAbort while updating state
        try { }
        finally
        {
            m_period = period;

            if (dueTime == Timeout.UnsignedInfinite)
            {
                TimerQueue.Instance.DeleteTimer(this);
                success = true;
            }
            else
            {
                if (FrameworkEventSource.IsInitialized && FrameworkEventSource.Log.IsEnabled(EventLevel.Informational, FrameworkEventSource.Keywords.ThreadTransfer))
                    FrameworkEventSource.Log.ThreadTransferSendObj(this, 1, string.Empty, true);

                success = TimerQueue.Instance.UpdateTimer(this, dueTime, period);
            }
        }
    }

    return success;
}
```

The notes for that [`TimerQueue` class](http://referencesource.microsoft.com/#mscorlib/system/threading/timer.cs,208ff87939c84fe3) show that the fact that timers are created quite often was taken into consideration. Especially in this part:

> "Perf assumptions: We assume that timers are created and destroyed frequently, but rarely actually fire."

By looking at this other part:

> "`TimerQueue` maintains a list of active timers in this `AppDomain`. We use a single native timer, supplied by the VM to schedule all managed timers in the `AppDomain`."

I assume they could have created several of these native timers to reduce the contention (but maybe that's not the case).

Now, you might say that you don't really create many of these timers, but a modern .Net server using async-await extensively creates many timers hidden away from sight. For example, every single call to `Task.Delay` creates a timer internally which is disposed when the task completes. It's also common to create a `CancellationTokenSource` that automatically cancels itself after a certain interval using an internal `Timer`.

For us the main issue was `CancellationTokenSource`s created with a `TimeSpan` used to timeout our many (>100/s) I/O operations. Since the timeout for these was always the same I made a utility class that uses a single `Timer` and performs an action on all the items (`CancellationTokenSource`s in this case) that their timeout expired:

```csharp
public class CollectiveTimer<T>
{
    private readonly ConcurrentQueue<QueueItem> _queue;
    public CollectiveTimer(Action<T> action, TimeSpan timeout, CancellationToken cancellationToken)
    {
        _queue = new ConcurrentQueue<QueueItem>();
        Task.Run(async () =>
        {
            while (!cancellationToken.IsCancellationRequested)
            {
                QueueItem queueItem;
                var now = DateTime.UtcNow;
                while (_queue.TryPeek(out queueItem) && now - queueItem.Time >= timeout)
                {
                    _queue.TryDequeue(out queueItem);
                    action(queueItem.Item);
                }

                await Task.Delay(TimeSpan.FromMilliseconds(50));
            }
        });
    }
    public void Enqueue(T item)
    {
        _queue.Enqueue(new QueueItem(item, DateTime.UtcNow));
    }
    private sealed class QueueItem
    {
        public T Item { get; private set; }
        public DateTime Time { get; private set; }
        public QueueItem(T item, DateTime time)
        {
            Item = item;
            Time = time;
        }
    }
}
```

Usage:

```csharp
var collectiveTimer = new CollectiveTimer<CancellationTokenSource>(
    cts => cts.Cancel(), 
    TimeSpan.FromMilliseconds(200),
    cancellationToken);
```

This made our contention completely disappear (at least until the next one)
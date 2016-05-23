---
layout: post
title: LongRunning Is Useless For Task.Run With async-await
description: How do you pass TaskCreationOptions.LongRunning to Task.Run? You can't, and for async-await you shouldn't.
tags:
    - task-run
    - async-await
    - task-creation-options
    - task-factory-startnew]
---

Back in the olden days of .NET 4.0 we didn't have `Task.Run`. All we had to start a task was the complicated `Task.Factory.StartNew`. Among its parameters there's a `TaskCreationOptions` often used to specify `TaskCreationOptions.LongRunning`. That flag gives TPL a hint that the task you're about to execute will be longer than usual.

Nowadays with .NET 4.5 and above we mostly use the simpler and safer `Task.Run` but it isn't uncommon to wonder how do you pass `TaskCreationOptions.LongRunning` as a parameter like we used to do with `Task.Factory.StartNew`.
<!--more-->

The answer is that **you can't**. This of course isn't limited just to `TaskCreationOptions.LongRunning`. You can't pass any of the `TaskCreationOptions` values. However most of them (like `TaskCreationOptions.AttachedToParent`) are there for extremely esoteric cases while `TaskCreationOptions.LongRunning` is there for your run-of-the-mill long running task.

An often suggested workaround is to go back to `Task.Factory.StartNew`, which is perfectly fine for synchronous delegates (i.e. Action, `Func<T>`), however for asynchronous delegates (i.e. `Func<Task>`, `Func<Task<T>>`) there's the whole [`Task<Task>` confusion](http://stackoverflow.com/a/24777502/885318). Since `Task.Factory.StartNew` doesn't have specific overloads for async-await asynchronous delegates map to the `Func<T>` where T is a Task. That makes the return value a `Task<T>` where T is a Task, hence `Task<Task>`.

The .NET team anticipated this issue and it can be easily solved by using [`TaskExtensions.Unwrap`](https://msdn.microsoft.com/en-us/library/dd780917(v=vs.110).aspx) (which is the accepted answer on [the relevant Stack Overflow question](http://stackoverflow.com/q/26921191/885318) [^1]):

```csharp
Task<Task> task = Task.Factory.StartNew(async () =>
{
    while (IsEnabled)
    {
        await FooAsync();
        await Task.Delay(TimeSpan.FromSeconds(10));
    }
}, TaskCreationOptions.LongRunning);

Task actualTask = task.Unwrap();
```

However that hides the actual issue which is:

# Task.Run with TaskCreationOptions.LongRunning doesn't make sense for async-await.

The internal implementation [^2] creates a new dedicated thread when you use `TaskCreationOptions.LongRunning`. Here's the code for [`ThreadPoolTaskScheduler.QueueTask`](http://referencesource.microsoft.com/#mscorlib/system/threading/Tasks/ThreadPoolTaskScheduler.cs,55):

```csharp
protected internal override void QueueTask(Task task)
{
    if ((task.Options & TaskCreationOptions.LongRunning) != 0)
    {
        // Run LongRunning tasks on their own dedicated thread.
        Thread thread = new Thread(s_longRunningThreadWork);
        thread.IsBackground = true; // Keep this thread from blocking process shutdown
        thread.Start(task);
    }
    else
    {
        // Normal handling for non-LongRunning tasks.
        bool forceToGlobalQueue = (task.Options & TaskCreationOptions.PreferFairness) != 0;
        ThreadPool.UnsafeQueueCustomWorkItem(task, forceToGlobalQueue);
    }
}
```

But when an async method reaches an await for an uncompleted task the thread it's running on is released. When the task is completed the rest will be scheduled again, this time on a different `ThreadPool` thread. That means that we created a new thread needlessly. This doesn't only waste time for the developer but also hurts performance as creating new threads is costly (otherwise we wouldn't need the `ThreadPool` in the first place).

So, if you ask yourself how to use `Task.Run` with `TaskCreationOptions.LongRunning` when your delegate is asynchronous, save yourself and your application some time and keep using `Task.Run` as it is:

```csharp
Task task = Task.Run(async () =>
{
    while (IsEnabled)
    {
        await FooAsync();
        await Task.Delay(TimeSpan.FromSeconds(10));
    }
});
```

[^1]: Which is the top result when searching for *"Task.Run and LongRunning"*.
[^2]: Which could change in the future.

*[TPL]: Task Parallel Library
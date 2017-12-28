---
layout: post
title: Duck Typing And Async/Await
description: The C# compiler uses duck typing for async/await, which allows us to inject support for awaiting collections of tasks.
tags:
    - async-await
---

As some of you may know, many of the features in C# light up by "duck typing". Duck typing is where you accept an object that behaves in a certain way (i.e. has certain methods, properties, etc.) instead of a specific type, or as is usually explained: "If it walks like a duck, and talks like a duck, it's a duck".
<!--more-->

The `foreach` statement for example, doesn't look for types that implement `IEnumerable`/`IEnumerable<T>` but instead expects a `GetEnumerator` method that returns some enumerator type (can be either a `class` or a `struct`) that has `MoveNext` and `Current`. So while the following code clearly breaks in runtime the compiler has no issues with it:

```csharp
void Foo()
{
    foreach (var item in new FakeEnumerable())
    {
        Console.WriteLine(item);
    }
}

class FakeEnumerable
{
    public FakeEnumerator GetEnumerator() =>
        new FakeEnumerator();
}

struct FakeEnumerator
{
    public bool MoveNext() =>
        throw new NotImplementedException();

    public object Current =>
        throw new NotImplementedException();
}
```

One of these features is async/await (or the Task-based Asynchronous Pattern). The compiler doesn't expect `Task` or `Task<T>` specifically, it looks for a `GetAwaiter` method that returns an awaiter that in turn implements `INotifyCompletion` and has:

- `IsCompleted`: Enables optimizations when the operation completes synchronously.
- `OnCompleted`: Accepts the callback to invoke when the asynchronous operation completes.
- `GetResult`: Returns the result of the operation (if there is one) and rethrows the exception if one occurred.

`GetAwaiter` can also be an extension method so it's possible to turn existing types to awaitables with the right custom awaiter.

I recently made such an awaiter to allow awaiting a collection of tasks together by calling `Task.WhenAll` and wrapping the returned `Task`'s awaiter:

```csharp
struct TaskEnumerableAwaiter : INotifyCompletion
{
    private TaskAwaiter _awaiter;

    public bool IsCompleted => _awaiter.IsCompleted;

    internal TaskEnumerableAwaiter(IEnumerable<Task> tasks) =>
        _awaiter = Task.WhenAll(tasks).GetAwaiter();

    public void OnCompleted(Action continuation) =>
        _awaiter.OnCompleted(continuation);

    public void GetResult() =>
        _awaiter.GetResult();
}

static class EnumerableExtensions
{
    public static TaskEnumerableAwaiter GetAwaiter(this IEnumerable<Task> tasks) =>
        new TaskEnumerableAwaiter(tasks);
}
```

The `GetAwaiter` extension method for `IEnumerable<Task>` returning this `TaskEnumerableAwaiter` means the compiler can treat a collection of tasks as an awaitable and await it just like any other task:

```csharp
async Task DownloadAllAsync()
{
    var urls = new []
    {
        "http://www.google.com",
        "http://www.github.com",
        "http://www.twitter.com"
    };

    await urls.Select(DownloadAsync);
}

async Task DownloadAsync(string url)
{
    var httpClient = new HttpClient();
    var content = await httpClient.GetStringAsync(url);
    Console.WriteLine(content);
}
```

This may be confusing for developers who aren't familiar enough with async/await so I can't recommend this as a best-practice, but if you're still interested I added this support to my [AsyncUtilities](https://www.nuget.org/packages/AsyncUtilities/) library.

It also has support for awaiting an `IEnumerable<Task<TResult>>`, disabling `SynchronizationContext` capturing with `ConfigureAwait` and it utilizes C# 7.2 `readonly struct`. The complete code can be found [on github](https://github.com/i3arnon/AsyncUtilities/tree/master/src/AsyncUtilities/TaskEnumerableAwaiter).
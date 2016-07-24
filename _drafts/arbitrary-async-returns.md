---
layout: post
title: Return Any (Task-Like) Type From Async Methods
description: You can now, on top of void/Task/Task<T>, return any task-like type from async methods (e.g. ValueTask). 
tags:
    - value-task
    - async-await
    - roslyn
---

Since async-await was added to the C# language you could `await` any custom awaitable type as long as it follows a specific pattern: has a `GetAwaiter` method that returns an awaiter that has `IsCompleted`, `OnCompleted` and `GetResult` (more on it [here](http://stackoverflow.com/a/28236920/885318)). But the compiler is stricter when it comes to the return type of an async method. You can only return 3 types from an async method: `void`, `Task` and `Task<T>`.

`Task` is for async methods that don't have a result (i.e. procedures) while `Task<T>` is for async methods that return a result (i.e. functions). That leaves `async void` methods (or a-void as I like to call them) which mainly exist for backwards-compatibility with event handlers (the `void Button_Click(object sender, EventArgs e)` kind) and should be avoided elsewhere as unhandled exceptions inside them will crash the entire process.
<!--more-->

These compiler rules [were recently expanded](https://github.com/dotnet/roslyn/pull/12518) to also allow returning any custom type from an async method as long as it follows a specific pattern. The main reason for doing so is to ease the use of [`ValueTask<T>`](/2015/11/30/valuetask/) (which already [supports this feature](https://github.com/dotnet/corefx/pull/10201)) and allow to return it from an async method but the feature supports any custom `Task`-Like type that follows a specific pattern. For example if you have a custom `HardTask<T>` type for these extra-hard tasks, in order to return it from an async method you would need:

 - A public static method named `CreateAsyncMethodBuilder` on the custom type that returns the custom method builder:

```csharp
class HardTask<TResult>
{
    public static HardTaskMethodBuilder<TResult> CreateAsyncMethodBuilder() { ... }
}
```

 - A method builder that enables the compiler to actually build the `HardTask<T>`. It's very similar to [`AsyncTaskMethodBuilder<T>`](http://referencesource.microsoft.com/#mscorlib/system/runtime/compilerservices/AsyncMethodBuilder.cs,5916df9e324fc0a1,references) but the main difference is that the `Task` property returns the custom `HardTask<T>` instead of `Task<T>`: 

```csharp
struct HardTaskMethodBuilder<TResult>
{
    public void Start<TStateMachine>(ref TStateMachine stateMachine)
        where TStateMachine : IAsyncStateMachine { ... }
    public void SetStateMachine(IAsyncStateMachine stateMachine) { ... }
    public void SetResult(TResult result) { ... }
    public void SetException(Exception execption) { ... }
    // Returns HardTask<TResult>, not Task<TResult>
    public HardTask<TResult> Task { get; }
    public void AwaitOnCompleted<TAwaiter, TStateMachine>(
        ref TAwaiter awaiter, 
        ref TStateMachine stateMachine)
        where TAwaiter : INotifyCompletion
        where TStateMachine : IAsyncStateMachine { ... }
    public void AwaitUnsafeOnCompleted<TAwaiter, TStateMachine>(
        ref TAwaiter awaiter,
        ref TStateMachine stateMachine) 
        where TAwaiter : ICriticalNotifyCompletion 
        where TStateMachine : IAsyncStateMachine { ... }
}
```

The main driver behind this feature is performance. This will allow broader use of `ValueTask<T>` which is a discriminated union of `T` for synchronous cases and `Task<T>` for asynchronous ones. By being a `struct` it keeps the synchronous case allocation-free and reduces GC pressure. This is useful for operations that usually complete synchronously but occasionally run asynchronously like async streams or async collections but the best example in my opinion are caches.

When using a cache most results are returned synchronously from the cache but for the few cache-misses you usually need to do some I/O asynchronously. Until now you could either create a new `Task` in the synchronous case with `Task.FromResult` (which allocates a new instance the GC needs to collect) or complicate the code and cache the `Task` objects themselves. But now with this feature you can, for example, refactor this `HamsterProvider` that holds a cache of `Hamster` objects returned from MongoDB:

```csharp
class HamsterProvider
{
    ConcurrentDictionary<string, Hamster> _dictionary; // ...
    IMongoCollection<Hamster> _collection; // ...

    public ValueTask<Hamster> GetHamsterAsync(string name)
    {
        Hamster hamster;
        if (_dictionary.TryGetValue(name, out hamster))
        {
            return new ValueTask<Hamster>(hamster);
        }
        Task<Hamster> task = _collection.Find(_ => _.Name == name).SingleAsync();
        task.ContinueWith(_ => _dictionary.TryAdd(_.Result.Name, _.Result));
        return new ValueTask<Hamster>(task);
    }
}
```

Into this much more readable code using async-await instead of `Task.ContinueWith` without adding any allocations or degrading performance:

```csharp
class HamsterProvider
{
    ConcurrentDictionary<string, Hamster> _dictionary; // ...
    IMongoCollection<Hamster> _collection; // ...

    public async ValueTask<Hamster> GetHamsterAsync(string name)
    {
        Hamster hamster;
        if (_dictionary.TryGetValue(name, out hamster))
        {
            return hamster;
        }
        hamster = await _collection.Find(_ => _.Name == name).SingleAsync();
        _dictionary.TryAdd(hamster.Name, hamster);
        return hamster;
    }
}
```

While this feature is geared towards `ValueTask<T>` it's not limited to it. You can use it with any type you like as long as you implement the builder correctly. For example you can create a task that adds a log before each `await`, implement the Maybe monad as [Chad Kimes](https://github.com/ckimes89) did with [`NullableTaskLike<T>`](https://github.com/ckimes89/arbitrary-async-return-nullable#nullable-tasklike) or even support Win-RT's `IAsyncAction` when C# adds extension static methods.

Since this change was merged into the `dev15-preview-4` branch I assume this will show up in the next Visual Studio 15 preview (not to be confused with VS 2015) and end up in C# 7.0 when it's released. If you want to go deeper you can take a look at [Lucian Wischik](https://twitter.com/lwischik)'s [feature proposal](https://github.com/ljw1004/roslyn/blob/features/async-return/docs/specs/feature%20-%20arbitrary%20async%20returns.md) and [Charles Stoner](https://github.com/cston)'s [implementation PR](https://github.com/dotnet/roslyn/pull/12518).
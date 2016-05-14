---
layout: post
title: On The Efficiency Of ValueTask
description: ValueTask is a discriminated union of a T and a Task enabling allocation-free synchronous implementations of asynchronous operations.
tags: [async-await, value-task, corefx, channels]
---

The [corefxlab](https://github.com/dotnet/corefxlab ".NET Core Lab") repository contains library suggestions for [corefx](https://github.com/dotnet/corefx ".NET Core Libraries") which itself is a repo containing the .NET Core foundational libraries.  One of the gems hidden among these libraries is `ValueTask<T>` that was added by [Stephen Toub](https://github.com/stephentoub "stephentoub") as part of the [`System.Threading.Tasks.Channels`](https://github.com/dotnet/corefxlab/blob/master/src/System.Threading.Tasks.Channels/README.md) library but may be extremely useful on its own. The full implementation of `ValueTask<T>` can be found [here](//github.com/dotnet/corefx/blob/master/src/System.Threading.Tasks.Extensions/src/System/Threading/Tasks/ValueTask.cs), but this is an interesting subset of the API:

```csharp
public struct ValueTask<TResult>
{
    public ValueTask(TResult result);
    public ValueTask(Task<TResult> task);
    public static implicit operator ValueTask<TResult>(Task<TResult> task);
    public static implicit operator ValueTask<TResult>(TResult result);
    public Task<TResult> AsTask();
    public bool IsCompletedSuccessfully { get; }
    public TResult Result { get; }
    public ValueTaskAwaiter GetAwaiter();
    public ValueTaskAwaiter ConfigureAwait(bool continueOnCapturedContext);
    // ...
}
```
<!--more-->
I first noticed `ValueTask<T>` in the API documentation when reviewing the channels PR made to corefxlab. [I suggested adding a short explanation](https://github.com/dotnet/corefxlab/pull/335#issuecomment-149829696) which Stephen quickly provided:

> "`ValueTask<T>` is a discriminated union of a `T` and a `Task<T>`, making it allocation-free for `ReadAsync<T>` to synchronously return a `T` value it has available (in contrast to using `Task.FromResult<T>`, which needs to allocate a `Task<T>` instance). `ValueTask<T>` is awaitable, so most consumption of instances will be indistinguishable from with a `Task<T>`."

`ValueTask`, being a `struct`, enables writing async methods that do not allocate memory when they run synchronously without compromising API consistency. Imagine having an interface with a `Task` returning method. Each class implementing this interface must return a `Task` even if they happen to execute synchronously (hopefully using `Task.FromResult`). You can of course have 2 different methods on the interface, a synchronous one and an async one but this requires 2 different implementations to avoid ["sync over async"](http://blogs.msdn.com/b/pfxteam/archive/2012/04/13/10293638.aspx) and ["async over sync"](http://blogs.msdn.com/b/pfxteam/archive/2012/03/24/10287244.aspx).

`ValueTask<T>` has implicit conversions from both `T` and `Task<T>` and can be awaited by itself which makes it extremely simple to use. Consider this possibly async provider API returning a `ValueTask<T>`:

```csharp
interface IHamsterProvider
{
    ValueTask<Hamster> GetHamsterAsync(string name);
}
```

This provider interface can be implemented synchronously (for in-memory hamsters) by performing a lookup in a `Dictionary` and returning the `Hamster` instance (which is implicitly converted into a `ValueTask<Hamster>` without any additional allocations):

```csharp
class LocalHamsterProvider : IHamsterProvider
{
    readonly ConcurrentDictionary<string, Hamster> _dictionary; // ...
    public ValueTask<Hamster> GetHamsterAsync(string name)
    {
        Hamster hamster = _dictionary[name];
        return hamster;
    }
}
```

Or asynchronously (for hamsters stored in MongoDB) by performing an asynchronous query and returning the `Task<Hamster>` instance (which is implicitly converted into `ValueTask<Hamster>` as well):

```csharp
class MongoHamsterProvider : IHamsterProvider
{
    IMongoCollection<Hamster> _collection; // ...
    public ValueTask<Hamster> GetHamsterAsync(string name)
    {
        Task<Hamster> task = _collection.Find(_ => _.Name == name).SingleAsync();
        return task;
    }
}
```

The consumer of this API can await the `ValueTask<Hamster>` as if it was a `Task<Hamster>` without knowing whether it was performed asynchronously or not with the benefit of no added allocations: 

```csharp
Hamster hamster = await Locator.Get<IHamsterProvider>().GetHamsterAsync("bar");
```

While this example shows 2 different implementations, one synchronous and the other asynchronous, they could easily be combined. Imagine a provider using a local cache for hamsters and falling back to the DB when needed. 
The few truly asynchronous calls to `GetHamsterAsync` would indeed require allocating a `Task<Hamster>` (which will be implicitly converted to `ValueTask<Hamster>`) but the rest would complete synchronously and allocation-free:

```csharp
class HamsterProvider : IHamsterProvider
{
    readonly ConcurrentDictionary<string, Hamster> _dictionary; // ...
    IMongoCollection<Hamster> _collection; // ...
    public ValueTask<Hamster> GetHamsterAsync(string name)
    {
        Hamster hamster;
        if (_dictionary.TryGetValue(name, out hamster))
        {
            return hamster;
        }
        Task<Hamster> task = _collection.Find(_ => _.Name == name).SingleAsync();
        task.ContinueWith(_ => _dictionary.TryAdd(_.Result.Name, _.Result));
        return task;
    }
}
```

This kind of *"hybrid"* use of async-await is very common, since these outbound operations to a data store, remote API, etc. can usually benefit from some kind of caching. 

It's hard to quantify how much of an improvement widespread usage of `ValueTask` can bring as it greatly depends on the actual usage but avoiding allocations not only saves the allocation cost but also greatly reduces the garbage collection overhead. [That's why it was requested to be added to .NET Core regardless of the channels library](https://github.com/dotnet/corefx/issues/4708 "Add ValueTask to BCL").
---
layout: post
title: Evolution of An Async LINQ operator
description: From deferred execution to cancellation and ConfigureAwait(false), an async LINQ operator has some tricky parts to get correctly.
tags:
    - async-await
    - async-enumerable
    - interactive-extensions
---

LINQ (Language-Integrated Query) has been around for quite a while in the world of .NET (since .NET Framework 3.5 and C# 3.0) but recently async streams (i.e. [`IAsyncEnumerable<T>`](https://docs.microsoft.com/en-us/dotnet/api/system.collections.generic.iasyncenumerable-1)) were added to .NET and with them came async LINQ. It allows using the richness of LINQ while reaping the benefits of `async-await` by asynchronously accessing databases, microservices and remote APIs. 

[`System.Linq.Async`](https://www.nuget.org/packages/System.Linq.Async/) provides the standard LINQ operators we've all come to expect like `Where`, `Select`, `GroupBy`, etc. as extensions for `IAsyncEnumerable` but it's also common for developers to add their own custom ones. Doing that has some pitfalls in "old-school" (synchronous) LINQ but even more so when you throw `async-await` into the mix.
<!--more-->

For example, if we take `Where`, which receives an enumerable and a predicate to filter the items with, a *very* naive implementation may be something like (note the new `await foreach`):

```csharp
static async Task<List<T>> Where<T>(this IAsyncEnumerable<T> source, Func<T, bool> predicate)
{
    // Create a list to hold the filtered items
    var filteredItems = new List<T>();
    await foreach (var item in source)
    {
        if (predicate(item))
        {
            // Add the filtered item to the list
            filteredItems.Add(item);
        }
    }

    // Return the list containing the filtered items
    return filteredItems;
}
```

However, that implementation breaks one of the foundational properties of LINQ: Deferred Execution.

## Deferred Execution
Deferring the execution means the operation isn't executed at the point where the method is invoked, it only executes when the enumerable gets enumerated (e.g. with `await foreach`). The easiest way to fix that would be to utilize C# 8.0's async iterators:

```csharp
static async IAsyncEnumerable<T> Where<T>(this IAsyncEnumerable<T> source, Func<T, bool> predicate)
{
    await foreach (var item in source)
    {
        if (predicate(item))
        {
            // Return the filtered item and pause until the next MoveNextAsync call
            yield return item;
        }
    }
}
```

An async iterator method is a method that:

1. Returns an `IAsyncEnumerable<T>`.
1. Is declared with the `async` modifier.
1. Contains `yield return` (or `yield break`) statements.

The compiler, behind the scenes, turns this kind of methods into state machines. They allow to start returning items only when the consumer asks for them, which is done by enumerating the returned async enumerable.

So now when `Where` is called no execution has started yet, it will only start when the `await foreach` is reached:

```csharp
IAsyncEnumerable<Hamster> hamsters = // ..
IAsyncEnumerable<Hamster> filteredHamsters = hamsters.Where(hamster => hamster.Name == "Bar");
// No execution has happened yet
await foreach (Hamster hamster in filteredHamsters)
{
    Console.WriteLine(hamster.Age);
}
```

Using an async iterator also achieves another foundational property of LINQ: Streaming Execution.

## Streaming Execution
When the execution is streaming the items are returned one at a time as the consumer asks for them. The iterator doesn't need to consume the entire source enumerable in order to return the filtered items, it can return them one by one (in fact, the source enumerable can even be infinite).

Now, while we want the main execution of our method to be deferred there is a part we want to run eagerly right when the method is called: argument validation. If something is wrong with one of the arguments passed to our operator method we don't want to wait until the result is enumerated in order to notify the consumer about it. We want to "fail fast" and throw an exception quickly. Since an async iterator only starts running when the result is enumerated we'll have to break our method into two. 

In order to do that we can utilize another relatively new C# feature: local methods. That way the local method can be an async iterator while the outer method can eagerly validate the arguments:

```csharp
static IAsyncEnumerable<T> Where<T>(this IAsyncEnumerable<T> source, Func<T, bool> predicate)
{
    // Argument validation
    if (source is null) throw new ArgumentNullException(nameof(source));
    if (predicate is null) throw new ArgumentNullException(nameof(predicate));

    return Core();

    // Local async iterator method
    async IAsyncEnumerable<T> Core()
    {
        await foreach (var item in source)
        {
            if (predicate(item))
            {
                yield return item;
            }
        }
    }
}
```

Everything up to here is relevant for both async and synchronous LINQ, but implementing an async LINQ operator has some extra things you need to get right, starting with cancellation. 

## WithCancellation
Since the underlying operations being performed when enumerating async streams are potentially remote asynchronous operations that may take a while to complete it's beneficial to be able to cancel them when needed (imagine a long DB query for a web request that already timed out). In order to support cancellation the `IAsyncEnumerable.GetEnumerator` method accepts a `CancellationToken`, but since people don't usually call `GetEnumerator` by themselves (they call it implicitly via `await foreach`) `IAsyncEnumerable` has an extension method to "bake in" the `CancellationToken` into the enumerable:

```csharp
CancellationToken cancellationToken = //..
IAsyncEnumerable<Hamster> filteredHamsters = // ..
await foreach (var hamster in filteredHamsters.WithCancellation(cancellationToken))
{
    Console.WriteLine(hamster.Age);
}
```

Cancellation in .NET is cooperative, which means that the consumer may pass a `CancellationToken` but the code being executed needs to look at that token to observe the cancellation and act upon it. But how can our `Where` method, which was already invoked and returned an enumerable, access this token being passed afterwards while enumerating the same enumerable? With the C# compiler's help.

Async iterator methods can accept a `CancellationToken` annotated with the [`EnumeratorCancellation`](https://docs.microsoft.com/en-us/dotnet/api/system.runtime.compilerservices.enumeratorcancellationattribute) attribute. That tells the compiler to create a `CancellationToken` that gets cancelled whether the consumer passed a token using `WithCancellation` in the `await foreach` or whether they passed a token directly when invoking the async iterator method.

All we need it to do is add that `CancellationToken` parameter to our async iterator method and use it when enumerating the source enumerable:

```csharp
static IAsyncEnumerable<T> Where<T>(this IAsyncEnumerable<T> source, Func<T, bool> predicate)
{
    if (source is null) throw new ArgumentNullException(nameof(source));
    if (predicate is null) throw new ArgumentNullException(nameof(predicate));

    // Using the CancellationToken parameter's default value
    return Core();

    async IAsyncEnumerable<T> Core([EnumeratorCancellation] CancellationToken cancellationToken = default)
    {
        await foreach (var item in source.WithCancellation(cancellationToken))
        {
            if (predicate(item))
            {
                yield return item;
            }
        }
    }
}
```

We can only add that token to the local `Core` method since the outer method isn't an async iterator (no `async` keyword). However, we wouldn't want to do that even if we could. Imagine calling multiple LINQ operators one after another and passing the same token again and again (e.g. `Where(.., token).Select(.., token).OrderBy(.., token)`). We want to allow the consumer to pass the token once when enumerating, and it's on us to make sure the token "moves through the pipe" of all the operators and reach the actual asynchronous operation we want to cancel (e.g. the DB query).

## ConfigureAwait(false)
Another uniquely `async-await` thing it's important to get right is when to call `ConfigureAwait(false)`. When awaiting inside async methods, unless there's a good reason not to, you usually want to ignore the `SynchronizationContext` if there is one. The `SynchronizationContext` allows to return back to some context (usually the UI thread) after the asynchronous operation being `await`ed resumes. Using it in places where it's not needed slightly hurts performance and may lead to deadlocks (more on it [here](https://blog.stephencleary.com/2012/07/dont-block-on-async-code.html)).

So you can use `ConfigureAwait(false)` when enumerating an `IAsyncEnumerable` just like you can when awaiting a task:

```csharp
static IAsyncEnumerable<T> Where<T>(this IAsyncEnumerable<T> source, Func<T, bool> predicate)
{
    if (source is null) throw new ArgumentNullException(nameof(source));
    if (predicate is null) throw new ArgumentNullException(nameof(predicate));

    return Core();

    async IAsyncEnumerable<T> Core([EnumeratorCancellation] CancellationToken cancellationToken = default)
    {
        await foreach (var item in source.WithCancellation(cancellationToken).ConfigureAwait(false))
        {
            if (predicate(item))
            {
                yield return item;
            }
        }
    }
}
```

It's interesting to note that `await foreach` is pattern-based just like `foreach`, `await` and other capabilities in C#. Both the `WithCancellation` and `ConfigureAwait` methods return the [`ConfiguredCancelableAsyncEnumerable`](https://docs.microsoft.com/en-us/dotnet/api/system.runtime.compilerservices.configuredcancelableasyncenumerable-1) struct which doesn't implement the `IAsyncEnumerable` interface. But it does have a `GetAsyncEnumerator` method that returns an enumerator that has `MoveNextAsync`, `Current` & `DisposeAsync`. If follows the required pattern and therefore can be used in `await foreach` (I've explained more about these patterns in C# [in an earlier post](2018/01/02/task-enumerable-awaiter/)).

## Where, WhereAwait & WhereAwaitWithCancellation
Our `Where` implementation is now complete, but that method still has some limitations. It allows to filter an async enumerable using a predicate, but that predicate delegate is synchronous (it returns a `bool`, not a `Task<bool>`). What if for example, you can only filter the items by calling another microservice? Async LINQ covers these cases as well. 

The convention in async LINQ is to have three kinds of "overloads" for each operator:

1. An implementation that accepts a synchronous delegate (e.g. our `Where` method).
1. An implementation that accepts an async delegate (a delegate with an async return type). 
    - That method should have an `Await` suffix to signify that the delegates are being awaited (`WhereAwait` in our case).
1. An implementation that accepts a cancellable async delegate (an async delegate that accepts a `CancellationToken` on top of its other parameters).
    - That method should have an `AwaitWithCancellation` suffix (`WhereAwaitWithCancellation`).

Naming the versions differently helps the consumer to distinguish between the versions, but it also makes overload resolution easier for the compiler and helps it to target type the lambda expressions passed as arguments.

### WhereAwait
The signature for `WhereAwait` is similar to `Where`, only instead of accepting `Func<T, bool>` as the predicate it's now `Func<T, ValueTask<bool>>`. `ValueTask<T>` is an awaitable like `Task<T>` that allows to avoid `Task<T>` allocations in cases where the operations ran synchronously ([I've posted about it](/2015/11/30/valuetask/) when it was introduced). Using `ValueTask<T>` instead of `Task<T>` helps the consumer to generally avoid some unnecessary allocations which reduces GC pressure.

The implementation is also similar to `Where`. We just need to `await` the `ValueTask<bool>` returned from invoking the predicate, and to ignore the `SynchronizationContext` with `ConfigureAwait(false)`:

```csharp
static IAsyncEnumerable<T> WhereAwait<T>(this IAsyncEnumerable<T> source, Func<T, ValueTask<bool>> predicate)
{
    if (source is null) throw new ArgumentNullException(nameof(source));
    if (predicate is null) throw new ArgumentNullException(nameof(predicate));

    return Core();

    async IAsyncEnumerable<T> Core([EnumeratorCancellation] CancellationToken cancellationToken = default)
    {
        await foreach (var item in source.WithCancellation(cancellationToken).ConfigureAwait(false))
        {
            // Await the ValueTask<bool> returned from the predicate
            if (await predicate(item).ConfigureAwait(false))
            {
                yield return item;
            }
        }
    }
}
```


```csharp
IAsyncEnumerable<Hamster> hamsters = // ..
IAsyncEnumerable<Hamster> filteredHamsters = hamsters.WhereAwait(async hamster => await FindHamsterTypeAsync(hamster) == HamsterType.Dwarf);
```
### WhereAwaitWithCancellation
`WhereAwaitWithCancellation` allows the consumer to not only perform an asynchronous operation inside the predicate, but also to cancel that operation when the enumeration is cancelled. All we need for that to work is to accept the correct delegate and pass our `CancellationToken` when invoking it:

```csharp
static IAsyncEnumerable<T> WhereAwaitWithCancellation<T>(this IAsyncEnumerable<T> source, Func<T, CancellationToken, ValueTask<bool>> predicate)
{
    if (source is null) throw new ArgumentNullException(nameof(source));
    if (predicate is null) throw new ArgumentNullException(nameof(predicate));

    return Core();

    async IAsyncEnumerable<T> Core([EnumeratorCancellation] CancellationToken cancellationToken = default)
    {
        await foreach (var item in source.WithCancellation(cancellationToken).ConfigureAwait(false))
        {
            // Pass the CancellationToken to the predicate itself, on top of the enumerable
            if (await predicate(item, cancellationToken).ConfigureAwait(false))
            {
                yield return item;
            }
        }
    }
}
```

```csharp
IAsyncEnumerable<Hamster> hamsters = // ..
IAsyncEnumerable<Hamster> filteredHamsters = hamsters.WhereAwaitWithCancellation(async (hamster, token) => await FindHamsterTypeAsync(hamster, token) == HamsterType.Dwarf);
```

## Summary
It has been common, in my experience, for many projects to add their own custom extensions to LINQ and I expect the same would gradually happen with async LINQ as its adoption grows. With async LINQ (just as with `async-await` as a whole) it's a bit easier to shoot yourself in the foot. To avoid that you need to make sure the execution is deferred and streaming, cancellation is observed, `ConfigureAwait(false)` is used appropriately and the right "overloads" are implemented.

---
Note: This isn't how most operators are actually implemented in async LINQ. In order to utilize some optimizations the async iterators are custom implemented, instead of relying on the compiler generated ones. You can see the `Where` versions' implementations [here](https://github.com/dotnet/reactive/blob/f25e69141ea82f7fea11dd12901c1e6561ecaa22/Ix.NET/Source/System.Linq.Async/System/Linq/Operators/Where.cs).

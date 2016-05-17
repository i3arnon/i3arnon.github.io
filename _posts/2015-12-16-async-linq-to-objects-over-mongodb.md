---
layout: post
title: Async LINQ To Objects Over MongoDB
description: The MongoDB C# driver has async LINQ to MongoDB support. This shows how to add async LINQ to Objects support over MongoDB using Rx's Ix.
tags: [async-await, mongodb, reactive-extensions, interactive-extensions, async-enumerable]
---

I've been working with MongoDB for the past few years and lately the guys behind the [C# driver](https://github.com/mongodb/mongo-csharp-driver "mongo-csharp-driver") are working on some interesting things, especially around async-await support.

This year they released a complete rewrite of the driver with an async-only API. Now since `IEnumerator` doesn't support asynchronous enumeration (because `MoveNext` is synchronous) they used their own enumerator interface called `IAsyncCursor` which looks like this:
<!--more-->

```csharp
public interface IAsyncCursor<out TDocument> : IDisposable
{
    IEnumerable<TDocument> Current { get; }
    Task<bool> MoveNextAsync(
        CancellationToken cancellationToken = default(CancellationToken));
}
``` 

To enable asynchronous iteration without [having to implement it yourself](http://stackoverflow.com/a/29683170/885318 "How is an IAsyncCursor used for iteration with the mongodb c# driver?") they added a subset of the LINQ operators, but only those that materialize the query (e.g. `ForEachAsync`,  `ToListAsync`, etc.)

In the latest version of the driver ([v2.1](https://github.com/mongodb/mongo-csharp-driver/releases/tag/v2.1.0)) they added full "LINQ to MongoDB" support (i.e. `IQueryable`) that you can use to create your mongo queries. Just like with LINQ to SQL you can use `Where`, `Select` and so forth to build a complex expression-based query and have it sent to the server only when you start enumerating the `IQueryable`.

The problem arises when you need an expression to be executed on the client-side and not be sent to the server. This is mainly relevant for expressions that can't be translated into a mongo query. For example the following code comparing two of the queried document's fields (which is unsupported in MongoDB) will throw an exception saying: *"Unsupported filter: ([FirstName] == [LastName])."*

```csharp
async Task LinqToMongo()
{
    IMongoCollection<Hamster> collection = GetCollection();
    IMongoQueryable<Hamster> queryable = collection.AsQueryable();
    queryable = queryable.Where(_ => _.FirstName == _.LastName); // unsupported filter
    Console.WriteLine(await queryable.CountAsync());
}
```

Normally you would just cast the `IQueryable` into an `IEnumerable` (hopefully with `AsEnumerable`) and use LINQ to Objects which also supports deferred execution. However, since `IEnumerable` is synchronous doing that defeats the whole purpose of using async-await to begin with. You could also materialize the whole collection into memory and then use client-side filters but that can take too much memory and time.

A former coworker of mine ([Tsach Vayness](https://linkedin.com/in/tsachv "Tsach Vayness")) suggested finding an existing library with async LINQ to Objects support and plugging it into the MongoDB C# driver. That enables using all the LINQ to Objects operators over MongoDB. There are a few of these libraries and the best, in my opinion, is [Reactive Extensions' Interactive Extensions](https://github.com/Reactive-Extensions/Rx.NET#interactive-extensions "Rx.NET") ([Ix-Async on nuget.org](https://www.nuget.org/packages/Ix-Async/ "Ix-Async")).

All that's needed is an adapter from mongo's `IAsyncCursorSource`, `IAsyncCursor` to Interactive Extensions' `IAsyncEnumerable`, `IAsyncEnumerator` (which are already pretty similar) and then you can use all of Ix's operators on the MongoDB cursors. Here's the previous example comparing two of the queried document's fields fixed by moving the filter to the client-side:

```csharp
async Task LinqToObjects()
{
    IMongoCollection<Hamster> collection = GetCollection();
    IMongoQueryable<Hamster> queryable = collection.AsQueryable();
    IAsyncEnumerable<Hamster> asyncEnumerable = queryable.ToAsyncEnumerable();
    asyncEnumerable = asyncEnumerable.Where(_ => _.FirstName == _.LastName);
    Console.WriteLine(await asyncEnumerable.Count());
}
```

Most of the "magic" enabling this happens in `AsyncEnumeratorAdapter.MoveNext`. First, you create an `IAsyncCursor` out of the `IAsyncCursorSource` in an async fashion with `ToCursorAsync` (which is possible because `MoveNext` returns a `Task`). Then you call (and await) `MoveNext` on the created `_asyncCursor`. If it returned `true` then `_asyncCursor.Current` contains a batch of items you can enumerate and call `_asyncCursor.MoveNext` again when the batch is completed. You repeat that continually until the underlying `MoveNext` returns `false` meaning there are no more items to enumerate. 

```csharp
public async Task<bool> MoveNext(CancellationToken cancellationToken)
{
    if (_asyncCursor == null)
    {
        _asyncCursor = await _asyncCursorSource.ToCursorAsync(cancellationToken);
    }
    if (_batchEnumerator != null && _batchEnumerator.MoveNext())
    {
        return true;
    }
    if (_asyncCursor != null && await _asyncCursor.MoveNextAsync(cancellationToken))
    {
        _batchEnumerator?.Dispose();
        _batchEnumerator = _asyncCursor.Current.GetEnumerator();
        return _batchEnumerator.MoveNext();
    }
    return false;
}
```

Here is the full code for the adapters:

```csharp
public static class AsyncCursorSourceExtensions
{
    public static IAsyncEnumerable<T> ToAsyncEnumerable<T>(
        this IAsyncCursorSource<T> asyncCursorSource) => 
        new AsyncEnumerableAdapter<T>(asyncCursorSource);

    private class AsyncEnumerableAdapter<T> : IAsyncEnumerable<T>
    {
        private readonly IAsyncCursorSource<T> _asyncCursorSource;

        public AsyncEnumerableAdapter(IAsyncCursorSource<T> asyncCursorSource)
        {
            _asyncCursorSource = asyncCursorSource;
        }

        public IAsyncEnumerator<T> GetEnumerator() => 
            new AsyncEnumeratorAdapter<T>(_asyncCursorSource);
    }

    private class AsyncEnumeratorAdapter<T> : IAsyncEnumerator<T>
    {
        private readonly IAsyncCursorSource<T> _asyncCursorSource;
        private IAsyncCursor<T> _asyncCursor;
        private IEnumerator<T> _batchEnumerator;

        public T Current => _batchEnumerator.Current;

        public AsyncEnumeratorAdapter(IAsyncCursorSource<T> asyncCursorSource)
        {
            _asyncCursorSource = asyncCursorSource;
        }

        public async Task<bool> MoveNext(CancellationToken cancellationToken)
        {
            if (_asyncCursor == null)
            {
                _asyncCursor = await _asyncCursorSource.ToCursorAsync(cancellationToken);
            }
            
            if (_batchEnumerator != null &&
                _batchEnumerator.MoveNext())
            {
                return true;
            }
            
            if (_asyncCursor != null &&
                await _asyncCursor.MoveNextAsync(cancellationToken))
            {
                _batchEnumerator?.Dispose();
                _batchEnumerator = _asyncCursor.Current.GetEnumerator();
                return _batchEnumerator.MoveNext();
            }
            
            return false;
        }

        public void Dispose()
        {
            _asyncCursor?.Dispose();
            _asyncCursor = null;
        }
    }
}
```
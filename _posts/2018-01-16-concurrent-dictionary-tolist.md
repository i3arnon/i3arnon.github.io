---
layout: post
title: ConcurrentDictionary Is Not Always Thread-Safe
description: The ConcurrentDictionary members are thread safe, but not when used through one of the interfaces it implements.
tags:
    - bugs
---

`ConcurrentDictionary` is a thread-safe dictionary implementation but surprisingly (at least to me) not all of its members can be safely used by multiple threads concurrently. The Thread Safety section on the `ConcurrentDictionary` MSDN article has this to say:

> All public and protected members of `ConcurrentDictionary<TKey, TValue>` are thread-safe and may be used concurrently from multiple threads. However, members accessed through one of the interfaces the `ConcurrentDictionary<TKey, TValue>` implements, including extension methods, are not guaranteed to be thread safe and may need to be synchronized by the caller.

<!--more-->

It is guaranteed that invoking the `ConcurrentDictionary` methods and properties is thread safe, but since it implements several interfaces (e.g. `IDictionary`, `ICollection`) using one of their members (or extension methods for them) isn't. One of these examples is LINQ's `Enumerable.ToList`, which is an extension method for `IEnumerable`. Using it on a `ConcurrentDictionary` instance while it's being mutated by other threads may throw an `ArgumentException`. Here's a simple repro:

```csharp
var dictionary = new ConcurrentDictionary<int, int>();
Task.Run(() =>
{
    var random = new Random();
    while (true)
    {
        var value = random.Next(10000);
        dictionary[value] = value;
    }
});

while (true)
{
    dictionary.ToList();
}

```

This happens because the extension method calls the [`List` constructor](http://referencesource.microsoft.com/#mscorlib/system/collections/generic/list.cs,74) that accepts an `IEnumerable` and that constructor has an optimization for `ICollection` (which `ConcurrentDictionary` implements);

```csharp
public List(IEnumerable<T> collection) {
    ICollection<T> c = collection as ICollection<T>;
    if( c != null) {
        int count = c.Count;
        if (count == 0)
        {
            _items = _emptyArray;
        }
        else {
            _items = new T[count]; // Gets the dictionary size
            c.CopyTo(_items, 0); // Copy from the dictionary to the array
            _size = count;
        }
    }
    else {
        // ...
    }
}
```

It first gets the size of the dictionary by invoking `Count`, then initializes an array in that size and finally calls `CopyTo` to copy over all the `KeyValuePair` items from the dictionary to that array. Since the dictionary is mutated by multiple threads, the size can increase (or decrease) after `Count` is invoked but before `CopyTo` is. That will result in an `ArgumentException` when the `ConcurrentDictionary` tries to access the array outside its bounds.

If you just need a separate collection with the dictionary's items this exception can be avoided by calling the [`ConcurrentDictionary.ToArray` method](http://referencesource.microsoft.com/#mscorlib/system/Collections/Concurrent/ConcurrentDictionary.cs,692) instead, which operates in a similar way, but does so after acquiring all the dictionary's internal locks:

```csharp
public KeyValuePair<TKey, TValue>[] ToArray()
{
    int locksAcquired = 0;
    try
    {
        AcquireAllLocks(ref locksAcquired);
        int count = 0;
        checked
        {
            for (int i = 0; i < m_tables.m_locks.Length; i++)
            {
                count += m_tables.m_countPerLock[i];
            }
        }

        KeyValuePair<TKey, TValue>[] array = new KeyValuePair<TKey, TValue>[count];

        CopyToPairs(array, 0);
        return array;
    }
    finally
    {
        ReleaseLocks(0, locksAcquired);
    }
}
```

This method, however, shouldn't be confused with LINQ's [`Enumerable.ToArray` extension method](http://referencesource.microsoft.com/#System.Core/System/Linq/Enumerable.cs,942), that may throw an `ArgumentException` just like `Enumerable.ToList`. `ConcurrentDictionary.ToArray` wins the overload resolution over `Enumerable.ToArray` since the latter is an extension method but if the dictionary is used through one of its interfaces (that inherits from `IEnumerable`) the `Enumerable.ToArray` method will be called and the exception may be thrown. So the following piece of code doesn't throw:

```csharp
var dictionary = new ConcurrentDictionary<int, int>();
Task.Run(() =>
{
    var random = new Random();
    while (true)
    {
        var value = random.Next(10000);
        dictionary[value] = value;
    }
});

while (true)
{
    dictionary.ToArray();
}
```

But explicitly holding the dictionary in an `IDictionary` variable does:

```csharp
IDictionary<int, int> dictionary = new ConcurrentDictionary<int, int>();
Task.Run(() =>
{
    var random = new Random();
    while (true)
    {
        var value = random.Next(10000);
        dictionary[value] = value;
    }
});

while (true)
{
    dictionary.ToArray();
}
```
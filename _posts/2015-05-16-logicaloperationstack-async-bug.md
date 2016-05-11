---
layout: post
title: LogicalOperationStack Is Broken With async-await
description: LogicalOperationStack enables having nested logical operation identifiers. It's broken with async in .Net 4.5 and above.
tags: [async-await, bug]
---

`Trace.CorrelationManager.LogicalOperationStack` enables having nested logical operation identifiers where the most common case is logging (NDC). Evidently it doesn't work with `async-await`.
<!--more-->
# The Issue

Here's a simple example using `LogicalFlow`, my simple wrapper over the `LogicalOperationStack`:

```csharp
static void Main()
{
    OuterOperationAsync().Wait();
}

static async Task OuterOperationAsync()
{
    Console.WriteLine(LogicalFlow.CurrentOperationId);
    using (LogicalFlow.StartScope())
    {
        Console.WriteLine("\t" + LogicalFlow.CurrentOperationId);
        await InnerOperationAsync();
        Console.WriteLine("\t" + LogicalFlow.CurrentOperationId);
        await InnerOperationAsync();
        Console.WriteLine("\t" + LogicalFlow.CurrentOperationId);
    }
    Console.WriteLine(LogicalFlow.CurrentOperationId);
}

static async Task InnerOperationAsync()
{
    using (LogicalFlow.StartScope())
    {
        await Task.Yield();
    }
}
```

`LogicalFlow`:

```csharp
public static class LogicalFlow
{
    public static Guid CurrentOperationId
    {
        get
        {
            return Trace.CorrelationManager.LogicalOperationStack.Count > 0
                ? (Guid) Trace.CorrelationManager.LogicalOperationStack.Peek()
                : Guid.Empty;
        }
    }

    public static IDisposable StartScope()
    {
        Trace.CorrelationManager.StartLogicalOperation();
        return new Stopper();
    }

    private static void StopScope()
    {
        Trace.CorrelationManager.StopLogicalOperation();
    }

    private class Stopper : IDisposable
    {
        private bool _isDisposed;
        public void Dispose()
        {
            if (!_isDisposed)
            {
                StopScope();
            }
        }
    }
}
```

And the output is:

```
00000000-0000-0000-0000-000000000000
    49985135-1e39-404c-834a-9f12026d9b65
    54674452-e1c5-4b1b-91ed-6bd6ea725b98
    c6ec00fd-bff8-4bde-bf70-e073b6714ae5
54674452-e1c5-4b1b-91ed-6bd6ea725b98
```

The specific `Guid` values don't really matter. Both the outer lines should show `Guid.Empty` (i.e. `00000000-0000-0000-0000-000000000000`) and the inner lines should show **the same** `Guid` value.

# Multithreading

You might say that `LogicalOperationStack` is using a `Stack` which is not thread-safe and that's why the output is wrong. time** because every `async` operation is awaited when called and there's no use of combinators such as `Task.WhenAll`.

# The Root Cause

The root cause is that `LogicalOperationStack` is stored in the `CallContext` which has a copy-on-write behavior. That means that as long as you don't explicitly set something into the `CallContext` (and you don't with `StartLogicalOperation` as you just add to an existing stack) you're using the parent context and not your own.

This can be shown by simply setting **anything** into the `CallContext` before adding to the existing stack. For example if we changed `StartScope` to this:

```csharp
public static IDisposable StartScope()
{
    CallContext.LogicalSetData("Bar", "Arnon");
    Trace.CorrelationManager.StartLogicalOperation();
    return new Stopper();
}
```

The output is correct:

```
00000000-0000-0000-0000-000000000000
    fdc22318-53ef-4ae5-83ff-6c3e3864e37a
    fdc22318-53ef-4ae5-83ff-6c3e3864e37a
    fdc22318-53ef-4ae5-83ff-6c3e3864e37a
00000000-0000-0000-0000-000000000000
```

I've contacted the relevant developer at Microsoft and his response was this:

> "**I wasn't aware of this, but it does seem broken**. The copy-on-write logic is supposed to behave exactly as if we'd really created a copy of the `ExecutionContext` on entry into the method. However, copying the `ExecutionContext` would have created a deep copy of the `CorrelationManager` context, as it's special-cased in `CallContext.Clone()`. We don't take that into account in the copy-on-write logic."

# Solution

Use an `ImmutableStack` stored in the `CallContext` instead of the `LogicalOperationStack` as it's both thread-safe and immutable so when you call `Pop` you get back a new `ImmutableStack` that you then must set back into the `CallContext`.

```csharp
public static class LogicalFlow
{
    private static ImmutableStack<Guid> LogicalStack
    {
        get
        {
            return CallContext.LogicalGetData("LogicalFlow") as ImmutableStack<Guid> ?? ImmutableStack.Create<Guid>();
        }
        set
        {
            CallContext.LogicalSetData("LogicalFlow", value);
        }
    }

    public static Guid CurrentId
    {
        get
        {
            var logicalStack = LogicalStack;
            return logicalStack.IsEmpty ? Guid.Empty : logicalStack.Peek();
        }
    }
    
    public static IDisposable StartScope()
    {
        LogicalStack = LogicalStack.Push(Guid.NewGuid()); // Here's where the CallContext is copied using copy-on-write
        return new Stopper();
    }
    
    private static void StopScope()
    {
        LogicalStack = LogicalStack.Pop();
    }
}
```

Another options is to to store the stack in the new [`System.Threading.AsyncLocal`](https://msdn.microsoft.com/en-us/library/dn906268%28v=vs.110%29.aspx) class added in .Net 4.6 (or [Stephen Cleary](http://www.stephencleary.com)'s [`AsyncLocal`](https://github.com/StephenCleary/AsyncLocal)) instead which should handle that issue correctly.

Note: This came after I asked (and answered) a question on Stack Overflow: [Is `LogicalOperationStack` incompatible with async in .Net 4.5](http://stackoverflow.com/a/30130663/885318) which contains further discussion.

*[NDC]: Nested Diagnostic Context
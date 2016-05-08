---
layout: post
title: Protobuf-net Is Broken Around DateTime
description: Protobuf-net loses DateTime.Kind when deserializing which can mess with your DateTime values.
tags: [protobuf-net, bug]
---

[Protocol Buffers](https://github.com/google/protobuf) by Google are a great mechanism for serializing (and deserializing) structured data in a very fast and efficient way. [Protobuf-net](https://github.com/mgravell/protobuf-net) is [Marc Gravell](https://twitter.com/marcgravell)'s port of Protocol Buffers for the .Net ecosystem.

While [being very efficient](http://theburningmonk.com/2011/08/performance-test-binaryformatter-vs-protobuf-net/), protobuf-net has a big issue when deserializing .Net's `DateTime`s. Behind the scenes `DateTime`s are converted into [Unix-Time](https://en.wikipedia.org/wiki/Unix_time) which is a count (of ticks in this case) starting from the Unix Epoch (1970/01/01 UTC). When deserializing back to .Net protobuf-net adds that count to a `DateTime` representing the Epoch-Time resulting in the correct `DateTime` value. The issue with this process is that it loses the `DateTime`'s original `DateTimeKind`.
<!--more-->
[`DateTimeKind`](https://msdn.microsoft.com/en-us/library/shx7s921(v=vs.110).aspx) is an enum telling whether the `DateTime`'s value represents a local time, UTC time or unspecified. That value isn't serialized by protobuf-net so all `DateTime`s, be they local time or UTC, are deserialized as `DateTimeKind.Unspecified`.

`DateTimeKind.Unspecified` values have a behavior that I initially found surprising but later realized is the best possible option. Let's assume you're in Hawaii (because where else would you want to be?) and your time zone is UTC-10:00. If you have a `DateTime` value with `DateTimeKind.Unspecified` and you call `ToLocalTime` the method assumes the value is in UTC and "corrects" it, so 11:00 becomes 01:00. If however you call `ToUniversalTime` on that value the method now assumes it's in local time and "corrects" it so 11:00 becomes 21:00. So the same value is treated as local while adjusting to universal and universal when adjusting to local. Let's see that in code:

```csharp
static void Main()
{
    var dolly = new Sheep {DateOfBirth = new DateTime(1966, 07, 05, 11, 0, 0, DateTimeKind.Utc)};
    Console.WriteLine(dolly.DateOfBirth.ToString("HH:mm:ss K")); // "11:00:00 Z" (Z means UTC)

    dolly = Serializer.DeepClone(dolly); // Serialize and deserialize using protobuf-net
    Console.WriteLine(dolly.DateOfBirth.ToString("HH:mm:ss K")); // "11:00:00" (no Z means unspecified)
    Console.WriteLine(dolly.DateOfBirth.ToLocalTime().ToString("HH:mm:ss K")); // "01:00:00 -10:00" (Hawaii timezone)
    Console.WriteLine(dolly.DateOfBirth.ToUniversalTime().ToString("HH:mm:ss K")); // "21:00:00 Z"
}

[ProtoContract(ImplicitFields = ImplicitFields.AllPublic)]
class Sheep
{
    public DateTime DateOfBirth { get; set; }
}
```

This can get extremely problematic especially if you, like me, depend upon some library that uses `ToUniversalTime` or `ToLocalTime`. For me that library was the [.Net's MongoDB Driver](https://github.com/mongodb/mongo-csharp-driver) that stores all `DateTime`s in MongoDB in UTC. Using these 2 libraries together is impossible as `DateTime` values would keep changing value infinitely.

[I have posted on the protobuf-net repository](https://github.com/mgravell/protobuf-net/issues/44#issuecomment-105433396) on github explaining this and managed to convince him to fix this problem (which he did with [this commit](https://github.com/mgravell/protobuf-net/commit/e601b359c6ae56afc159754d29f5e7d0f05a01f5)). However, this fix was made 5 months prior to me writing this post and there's still isn't a new release including the fix (the latest stable release is from 2013/09/30).

But don't fear... I do have a workaround you can use for the meantime. Protobuf-net uses a [`DateTime` value representing the Unix Epoch](https://github.com/mgravell/protobuf-net/blob/15174a09ee3223c8805b3ef81c1288879c746dfa/protobuf-net/BclHelpers.cs#L48) (1970/01/01) to create all the `DateTime`s by adding the relevant delta to the epoch value. Since creating a new `DateTime` from an existing `DateTime` preserves the `DateTimeKind`, replacing the single epoch value with a UTC one will result with all protobuf-net `DateTime` values having `DateTimeKind.UTC`. We can do that by using reflection and replacing the epoch value with a UTC one:

```csharp
typeof (BclHelpers).
    GetField("EpochOrigin", BindingFlags.NonPublic | BindingFlags.Static).
    SetValue(null, new DateTime(1970, 1, 1, 0, 0, 0, 0, DateTimeKind.Utc));

var dolly = new Sheep {DateOfBirth = new DateTime(1966, 07, 05, 11, 0, 0, DateTimeKind.Utc)};
Console.WriteLine(dolly.DateOfBirth.ToString("HH:mm:ss K")); // "11:00:00 Z" (Z means UTC)

dolly = Serializer.DeepClone(dolly); // Serialize and deserialize using protobuf-net
Console.WriteLine(dolly.DateOfBirth.ToString("HH:mm:ss K")); // "11:00:00 Z"
```

I admit it's not pretty, but until Marc releases a new version, it's preferable to building your own protobuf-net.

*[UTC]: Coordinated Universal Time
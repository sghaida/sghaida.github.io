---
layout: post
title: "C# Object Generator"
date: 2014-12-17 09:00:00 +0000
categories: [csharp, programming]
tags: [csharp, object generation, programming]
---


# Random Object Generator for Testing CRUD Applications

## Introduction

Most of the time, when developing a CRUD application, the required data is not available early enough to support UI development. As a result, developers often resort to the *dirty approach*: manually inserting data directly into the database.

While manual insertion can work in some cases, it quickly becomes impractical. For example:

* When writing test cases to verify that inserts behave correctly
* When you need to provide large data sets (e.g., lists of objects) as input
* When you want repeatable and automated test data generation

There are many utilities available that solve some or all of these problems. However, the question remains: **how can we build such a solution ourselves?**

---

## Background

I was looking for a way to populate objects on the fly with random data so I could use them to verify the integrity of my functions. I found a few approaches on Stack Overflow, but I decided to build my own solution from scratch—mainly to understand the problem space better.

What I ended up with is not a fully completed or optimized solution, but it lays a solid foundation for anyone interested in this topic.

---

## Using the Code

The implementation is intentionally simple and consists of two main classes:

* **`Invoker`**
  Responsible for creating strongly typed property setters using expression trees.

* **`RandomObjectsGenerator<T>`**
  Generates random values based on property types and uses the `Invoker` to populate an object.

---

## The `Invoker` Class

The `Invoker` class exposes a `CreateSetter` method, which returns an `Action<T, object>`.
This delegate represents the actual setter for a given property on type `T`.

```csharp
public class Invoker
{
    public static Action<T, object> CreateSetter<T>(PropertyInfo propertyInfo)
    {
        var targetType = propertyInfo.DeclaringType;
        var info = propertyInfo.GetSetMethod();
        Type type = propertyInfo.PropertyType;

        var target = Expression.Parameter(targetType, "t");
        var value = Expression.Parameter(typeof(object), "st");

        var condition = Expression.Condition(
            Expression.Equal(value, Expression.Constant(DBNull.Value)),
            Expression.Default(type),
            Expression.Convert(value, type)
        );

        var body = Expression.Call(
            Expression.Convert(target, info.DeclaringType),
            info,
            condition
        );

        var lambda = Expression.Lambda<Action<T, object>>(body, target, value);
        return lambda.Compile();
    }
}
```

This approach avoids reflection-based invocation at runtime and provides better performance and type safety.

---

## The `RandomObjectsGenerator<T>` Class

This class is responsible for generating random values and assigning them to object properties using the setters produced by the `Invoker`.

```csharp
public class RandomObjectsGenerator<T> where T : class, new()
{
    private static Random rand = new Random();

    private static List<long> randomLongNumbersList = new();
    private static List<int> randomIntNumbersList = new();
    private static List<DateTime> randomDateTimeList = new();

    // Randomization bounds
    private int minLongRandBound = 1;
    private long maxLongRandBound = long.MaxValue;

    private int minIntRandBound = 1;
    private int maxIntRandBound = int.MaxValue;
```

### Random Value Generators

Each primitive type has its own generator method.

#### Integer

```csharp
private int GetIntNumber()
{
    byte[] buf = new byte[8];
    rand.NextBytes(buf);

    int intRand = BitConverter.ToInt32(buf, 0);
    int value = Math.Abs(intRand % (minIntRandBound - maxIntRandBound)) + minIntRandBound;

    if (!randomIntNumbersList.Contains(value))
        randomIntNumbersList.Add(value);
    else
        return GetIntNumber();

    return value;
}
```

#### Long

```csharp
private long GetLongNumber()
{
    byte[] buf = new byte[8];
    rand.NextBytes(buf);

    long longRand = BitConverter.ToInt64(buf, 0);
    long value = Math.Abs(longRand % (minLongRandBound - maxLongRandBound)) + minLongRandBound;

    if (!randomLongNumbersList.Contains(value))
        randomLongNumbersList.Add(value);
    else
        return GetLongNumber();

    return value;
}
```

#### Decimal

```csharp
private decimal GetDecimal()
{
    byte scale = (byte)rand.Next(29);
    bool sign = rand.Next(2) == 1;

    return new decimal(
        GetIntNumber(),
        GetIntNumber(),
        GetIntNumber(),
        sign,
        scale
    );
}
```

#### Boolean

```csharp
private bool GetBool()
{
    return rand.Next(100) <= 50;
}
```

#### String

```csharp
private string GetString()
{
    return Guid.NewGuid().ToString();
}
```

#### DateTime

```csharp
private DateTime GetDateTime()
{
    DateTime startingDate = DateTime.Now.AddYears(-2);
    int range = (DateTime.Today - startingDate).Days;

    DateTime value = startingDate
        .AddDays(rand.Next(range))
        .AddHours(rand.Next(0, 24))
        .AddMinutes(rand.Next(0, 60))
        .AddSeconds(rand.Next(0, 60))
        .AddMilliseconds(rand.Next(0, 999));

    if (!randomDateTimeList.Contains(value))
        randomDateTimeList.Add(value);
    else
        return GetDateTime();

    return value;
}
```

#### Byte

```csharp
private byte GetByte()
{
    byte[] buffer = new byte[10];
    rand.NextBytes(buffer);
    return buffer[rand.Next(0, 9)];
}
```

---

## Generating the Random Object

```csharp
public static T GenerateRandomObject()
{
    var randObjGen = new RandomObjectsGenerator<T>();
    var setters = new Dictionary<string, Action<T, object>>();

    const BindingFlags flags = BindingFlags.Public | BindingFlags.Instance;
    var properties = typeof(T).GetProperties(flags).ToList();

    foreach (var prop in properties)
        setters.Add(prop.Name, Invoker.CreateSetter<T>(prop));

    T obj = new T();

    var typedValueMap = new Dictionary<Type, Delegate>
    {
        { typeof(int), new Func<int>(() => randObjGen.GetIntNumber()) },
        { typeof(long), new Func<long>(() => randObjGen.GetLongNumber()) },
        { typeof(decimal), new Func<decimal>(() => randObjGen.GetDecimal()) },
        { typeof(bool), new Func<bool>(() => randObjGen.GetBool()) },
        { typeof(DateTime), new Func<DateTime>(() => randObjGen.GetDateTime()) },
        { typeof(string), new Func<string>(() => randObjGen.GetString()) },
        { typeof(byte), new Func<byte>(() => randObjGen.GetByte()) }
    };

    foreach (var setter in setters)
    {
        var type = properties
            .Where(p => p.Name == setter.Key)
            .Select(p => p.PropertyType)
            .FirstOrDefault();

        if (type != null && typedValueMap.ContainsKey(type))
            setter.Value(obj, typedValueMap[type].DynamicInvoke(null));
    }

    return obj;
}
```

---

## Design Notes

### Why Use a Delegate Dictionary?

.NET does not support switching directly on `Type` (e.g., `switch(type)`).
Since all random generators are parameterless and return typed values, a dictionary mapping `Type → Delegate` allows dynamic invocation based on property type.

As long as the type is known, the appropriate delegate can be invoked using:

```csharp
DynamicInvoke(null)
```

---

### Why Use the `Invoker`?

For each `PropertyInfo`, we need a fast and reusable setter. The `Invoker` builds these setters once using expression trees and stores them in a dictionary for later use. This avoids repeated reflection and improves performance.

---

## Notes & Limitations

This implementation is **not complete or fully optimized**, but it serves its intended purpose. Some important limitations include:

* **No support for nested objects**
  Sub-objects must be generated separately and manually assigned.

* **Forced uniqueness for `int`, `long`, and `DateTime`**
  This can cause issues with foreign key relationships when inserting generated data into a database.

* **No configuration or extensibility layer**
  All generation logic is hardcoded.

I may enhance this implementation in the future if the need arises. For now, it fits my requirements, and I hope it proves useful to others as well.

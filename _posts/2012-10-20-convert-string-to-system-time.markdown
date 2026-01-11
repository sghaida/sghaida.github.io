---
layout: post
title: "Convert string to SYSTEMTIME using C"
date: 2012-10-20 09:00:00 +0000
categories: [programming]
tags: [c/c++, programming]
---

# Converting a Date String to `SYSTEMTIME` (C and C#)

I just want to share a simple and straightforward solution with you. I’m not sure how helpful it will be for everyone, but for those who need it—it’s here 🙂

This example shows how to convert a date string into a `SYSTEMTIME` structure in **C**, and then how to work with the same structure from **C#**.

---

## The `SYSTEMTIME` Structure

`SYSTEMTIME` is a typedef struct defined as follows:

```c
typedef struct _SYSTEMTIME {
    WORD wYear;
    WORD wMonth;
    WORD wDayOfWeek;
    WORD wDay;
    WORD wHour;
    WORD wMinute;
    WORD wSecond;
    WORD wMilliseconds;
} SYSTEMTIME;
```

---

## Converting a Date String to `SYSTEMTIME` in C

The following function takes a date string and converts it into a `SYSTEMTIME` structure.

> **Note:**
> The expected date format is: `yyyy-MM-dd hh:mm`

```c
SYSTEMTIME convertStringToSystemTime(char *dateTimeString)
{
    SYSTEMTIME systime;

    memset(&systime, 0, sizeof(systime));

    sscanf_s(
        dateTimeString,
        "%d-%d-%d%d:%d:%d",
        &systime.wYear,
        &systime.wMonth,
        &systime.wDay,
        &systime.wHour,
        &systime.wMinute
    );

    return systime;
}
```

---

## Doing the Same from C#

A good question is: **how do we do this from C#?**

Below is an example that demonstrates how to call the Windows API function `SetSystemTime` from C# using P/Invoke.

---

## Importing `SetSystemTime`

```csharp
[DllImport("kernel32.dll", SetLastError = true)]
public static extern bool SetSystemTime([In] ref SYSTEMTIME st);
```

---

## Setting the System Time from C#

```csharp
private void SetSysTime(
    short stYear,
    short stMonth,
    short stDay,
    short stHour,
    short min)
{
    var sysTime = new SYSTEMTIME
    {
        wYear = stYear,
        wMonth = stMonth,
        wDay = stDay,
        wHour = stHour,
        wMinute = min
    };

    SetSystemTime(ref sysTime);
}
```

---

## Defining the `SYSTEMTIME` Struct in C#

We define a struct that mirrors the native `SYSTEMTIME` layout.
The layout is marked as **sequential** to ensure compatibility with the unmanaged structure.

```csharp
[StructLayout(LayoutKind.Sequential)]
public struct SYSTEMTIME
{
    public ushort wYear;
    public ushort wMonth;
    public ushort wDayOfWeek;
    public ushort wDay;
    public ushort wHour;
    public ushort wMinute;
    public ushort wSecond;
}
```

---

## Calling the Function

Here’s how you can call the function from C#:

```csharp
SetSysTime(2012, 2, 11, 23, 59);
```

---

By externalizing `SetSystemTime` and defining a matching structure in C#, you can seamlessly work with system time across native and managed code.

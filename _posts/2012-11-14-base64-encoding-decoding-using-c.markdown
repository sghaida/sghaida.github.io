---
layout: post
title: "Base64 Encoding/Decoding using C"
date: 2012-11-14 09:00:00 +0000
categories: [programming]
tags: [c/c++, programming]
---

# Base64 Encoding and Decoding on Windows Using Crypt32

Recently, I was struggling to understand how Base64 works on Windows. I had read the RFC and implemented a library based on it, but the result behaved like a crippled implementation. After that, I started comparing the size of a Base64-encoded file with the size of the same file after decoding using C#. What I found was that it did not seem to fully comply with the RFC specifications—especially when it comes to padding size.

---

## Why Base64?

I needed Base64 encoding because I was writing a DLL using **VC++** and wanted to consume this DLL from **.NET web services**. To do this, all files must be sent in Base64 format, decoded on my side for processing, then encoded again and sent back to the web service to be consumed by the client application.

---

## The Solution

The best approach I found in C was to use the Windows API function:

> **`CryptStringToBinary`**  
(from `Crypt32.lib`)

This function handles Base64 decoding correctly and adheres to Windows’ implementation standards.

---

## Setup Requirements

To use this API:

- Add **`Crypt32.lib`** to your VC++ linker  
  - *Project Properties → Linker → Input → Additional Dependencies*
- Include the appropriate headers

---

## Decoding Base64 in C (Two-Step Process)

You need to make **two calls**:

1. **Determine the required buffer size**
2. **Decode the Base64 data into the allocated buffer**

---

## Example: Base64 Decoding

```c
WCHAR *decodedDocFile = NULL;  // Will hold the decoded content
DWORD cbBinary = 0;            // Will hold the size of the decoded buffer
PWSTR pwsPlainFileBase64;      // Initialized elsewhere, contains BASE64 content

// Get the decoded buffer size
if (CryptStringToBinary(
        (const PWSTR)pwsPlainFileBase64,
        wcslen(pwsPlainFileBase64),
        CRYPT_STRING_BASE64,
        NULL,
        &cbBinary,
        NULL,
        NULL))
{
    decodedDocFile = (WCHAR*)malloc(cbBinary);

    // Decode the file
    if (CryptStringToBinary(
            (const PWSTR)pwsPlainFileBase64,
            wcslen(pwsPlainFileBase64),
            CRYPT_STRING_BASE64,
            (BYTE*)decodedDocFile,
            &cbBinary,
            NULL,
            NULL))
    {
        printf("File has been decoded successfully");
    }
}
```

---

## Base64 Encoding

For encoding binary data into Base64, you can use:

> **`CryptBinaryToString`**

### Function Signature

```c
BOOL WINAPI CryptBinaryToString(
    _In_       const BYTE *pbBinary,
    _In_       DWORD cbBinary,
    _In_       DWORD dwFlags,
    _Out_opt_  LPTSTR pszString,
    _Inout_    DWORD *pcchString
);
```

---

## References

For more information, check the official Microsoft documentation:

- [https://msdn.microsoft.com/en-us/library/windows/desktop/aa380285(v=vs.85).aspx](https://msdn.microsoft.com/en-us/library/windows/desktop/aa380285%28v=vs.85%29.aspx)
- [https://msdn.microsoft.com/en-us/library/windows/desktop/aa379887(v=vs.85).aspx](https://msdn.microsoft.com/en-us/library/windows/desktop/aa379887%28v=vs.85%29.aspx)

---

I hope this helps clarify how Base64 encoding and decoding work on Windows using native APIs.

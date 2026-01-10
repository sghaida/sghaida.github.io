---
layout: post
title: "Cryptographic Algorithms"
date: 2011-10-10 09:00:00 +0000
categories: [security]
tags: [cryptography]
---

# Cryptography and Hashing Algorithms Overview

Let’s discuss **cryptography algorithms** and **hashing algorithms**, what distinguishes them, and their respective strengths and weaknesses.

When it comes to cryptography, there are **two major schools**:

- **Symmetric Algorithms**
- **Asymmetric Algorithms**

There is also a **hybrid approach** that uses both at the same time.  
Which approach you choose depends entirely on your needs—as we’ll see later.

---

## Symmetric Algorithms

Symmetric algorithms use a **single key** to both encrypt and decrypt data. These algorithms are typically fast and well suited for encrypting large blocks of data.

A well-known example is the **DEA (Data Encryption Algorithm)**, specified in the **DES (Data Encryption Standard)**.  

- **Triple DES (3DES)** improves security over DES  
- **AES (Advanced Encryption Standard)** has become the modern government standard

---

## Chapter 1: AES

### Section 1: Rijndael

AES is based on a **Substitution-Permutation Network (SPN)** design principle. It is fast in both software and hardware and, unlike DES, does not use a Feistel network.

Key characteristics:

- Fixed block size: **128 bits**
- Key sizes: **128, 192, or 256 bits**
- Rijndael (the original algorithm) supports block and key sizes in multiples of 32 bits
- AES operates on a **4×4 byte matrix** called the *state*
- All calculations occur in a finite field

AES encryption consists of multiple transformation rounds. Each round applies several steps, including key-dependent transformations. Decryption applies the reverse operations using the same key.

---

### Section 2: Serpent

Serpent was a finalist in the AES competition, finishing second to Rijndael.

Features:

- Block size: **128 bits**
- Key sizes: **128, 192, or 256 bits**
- 32-round substitution-permutation network
- Uses eight 4-bit S-boxes in parallel
- Designed for maximum parallelism

Serpent is **unpatented** and placed entirely in the **public domain**, allowing unrestricted use in software and hardware implementations.

---

## Chapter 2: 3DES

Triple DES enhances DES by increasing the effective key length without introducing a new cipher.

Keying options:

- **Option 1**: Three independent keys (168-bit key, ~112-bit effective security)
- **Option 2**: Two keys (112-bit key, ~80-bit effective security)

While still considered secure for certain use cases, 3DES is slower and is being phased out in favor of modern algorithms.

---

## Chapter 3: Blowfish

Blowfish is a symmetric block cipher designed in 1993 by **Bruce Schneier**.

Key points:

- Fast in software
- Unpatented and in the public domain
- Key-dependent S-boxes
- Complex key schedule

Drawbacks:

- Slow key setup
- 4 KB memory footprint
- Not suitable for very small embedded systems

Blowfish inspired later designs and remains popular in cryptographic software.

---

## Chapter 4: Twofish

Twofish is a successor to Blowfish and an AES finalist.

Characteristics:

- Block size: **128 bits**
- Key size: up to **256 bits**
- Feistel network
- Key-dependent S-boxes
- Complex key schedule

Twofish is unpatented and included in the **OpenPGP (RFC 4880)** standard, but it has seen less adoption than Blowfish.

---

## Chapter 5: IDEA

**IDEA (International Data Encryption Algorithm)** is a symmetric block cipher designed to replace DES.

Features:

- Block size: **64 bits**
- Key size: **128 bits**
- Uses addition, multiplication, and XOR
- Used in early versions of **PGP**

IDEA is fast, secure, and has no known practical attacks, but its usage has declined over time.

---

## Chapter 6: Other Symmetric Algorithms

For a list of less common symmetric encryption algorithms, see:  
**Symmetric-key algorithm (Wikipedia)**

---

## Asymmetric Algorithms

Asymmetric algorithms use **two related keys**:

- A **public key** for encryption
- A **private key** for decryption

These algorithms are commonly known as **public-key cryptography**, first introduced in 1978 with RSA.

Advantages:

- Secure key exchange
- Proof of origin and integrity

Disadvantages:

- Much slower than symmetric algorithms
- Not suitable for encrypting large amounts of data

Asymmetric encryption is often used to encrypt symmetric keys rather than full messages.

---

## Chapter 7: RSA

RSA is significantly slower than symmetric algorithms like DES or AES.

In practice:

1. A symmetric key encrypts the data
2. RSA encrypts the symmetric key
3. Both are transmitted to the recipient

Security depends heavily on the quality of random number generation for the symmetric key.

---

## Chapter 8: Other Asymmetric Algorithms

For a list of less common asymmetric algorithms, see:  
**Public-key cryptography (Wikipedia)**

In practice, **RSA dominates** real-world usage.

---

## Hash Algorithms

Hash algorithms convert data of arbitrary size into a **fixed-length message digest**. They are **one-way functions**, making them ideal for:

- Integrity verification
- Password storage
- Detecting tampering

---

## Chapter 9: MD5

MD5 produces a **128-bit hash** and was widely used for file integrity and security applications.

However:

- MD5 is **not collision resistant**
- Practical collision attacks exist
- Unsuitable for digital signatures or SSL certificates

MD5 is now considered **cryptographically broken**.

---

## Chapter 10: SHA-2

SHA-2 is a family of hash functions:

- SHA-224
- SHA-256
- SHA-384
- SHA-512

Designed by the NSA and standardized by NIST, SHA-2 is widely used in:

- TLS / SSL
- SSH
- IPsec
- PGP

SHA-1 is being retired in favor of SHA-2, and SHA-3 is being developed as a future standard.

---

## Chapter 11: Other Hash Algorithms

For a list of less common hash algorithms, see:  
**Cryptographic hash function (Wikipedia)**

---

## Cipher Algorithms Comparison

| Algorithm | Key Size | Block Size | Structure | Rounds | Cracked? | Known Attacks |
|---------|---------|------------|-----------|--------|----------|--------------|
| Rijndael (AES) | 128 / 192 / 256 bits | 128 bits | SPN | 10 / 12 / 14 | No | Side-channel |
| Twofish | 128 / 192 / 256 bits | 128 bits | Feistel | 16 | No | Truncated differential |
| Blowfish | 32–448 bits | 64 bits | Feistel | 16 | No | Second-order differential |
| DES | 56 bits | 64 bits | Feistel | 16 | Yes | Brute force, differential |
| 3DES | 112 / 168 bits | 64 bits | Feistel | 48 | No | Theoretical |
| IDEA | 128 bits | 64 bits | Mixed ops | 8 | No | Theoretical |
| RSA | Unlimited | N/A | Public key | N/A | No | Timing, side-channel |
| MD5 | N/A | 512 bits | Modular ops | 4 | Yes | Collision, preimage |
| SHA-2 | N/A | 512 / 1024 bits | Logical ops | 64 / 80 | No | Preimage, birthday |

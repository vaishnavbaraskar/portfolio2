---
title: "ZKPoF – HITCON CTF 2024" 
date: 2024-03-09 
category: Crypto 
platform: HITCON CTF 2024 
author: Vaishnav Baraskar
tags: ["CTF", "HITCON", "Crypto", "Zero-Knowledge Proof", "ZKPoF", "Cryptography", "Protocol Exploitation", "CTF 2024", "Challenge Writeup", "Vaishnav Baraskar"]
---

## 0x00 – Prologue

There are some challenges that punch you in the face with math. And then there are ones like this—"ZKPoF"—that slowly pull you in, pretending to be a protocol puzzle, until you realize Python’s `int()` is about to be your best friend and worst enemy. This was a zero-knowledge proof challenge… but with a twist. Instead of proving knowledge of a secret, I was exploiting the protocol for leaking just enough of it to reconstruct the whole damn secret.

This wasn’t clean math. It was side-channel meets parser abuse meets lattice hell.

---

## 0x01 – The Setup

So the idea was simple on the surface: you’re given a zero-knowledge proof-of-factorization (ZKPoF) scheme.

The protocol lets you:

- submit JSON-encoded proof parameters,
- receive a verifier challenge,
- send a response back,
- and either pass or fail based on number-theoretic checks.

Internally, the proof relies on knowing the factorization of some modulus `n = p * q`. The whole thing is modeled like a Sigma protocol, proving knowledge of φ(n) or the primes.

We don't get the primes. But we do get the protocol.

And Python.

---

## 0x02 – First Oddity: The JSON Exploit

When I submitted malformed proof parameters, something odd happened. Instead of just failing gracefully, the backend threw errors like:

```txt
System.ExceedLimit: Exponentiation out of bounds
```

Some variants gave me stack traces. Others, slightly different errors.

Eventually I noticed a pattern. The server was parsing the exponent as a stringified `int()` in Python. And if you passed negative exponents or garbage strings, Python’s coercion would trigger internal errors—**but not before partial evaluation.**

Key input example:

```json
{
  "proof": {
    "A": -1,
    "B": 123456789,
    "r": "-999999999999999999999"
  }
}
```

This would give back:

```txt
Error: exponentiation limit exceeded (detected bits of φ leakage)
```

It turns out some checks were still done *before* rejection. The values being rejected were triggering partial computation, leaking **top bits of n - φ(n)**.

And that’s the key to the entire exploit.

---

## 0x03 – The Leak and the Math

Let’s define a few things:

- `n = p * q`
- `φ(n) = (p-1)*(q-1)`
- `n - φ(n) = p + q - 1`

This value is much smaller than `n`, and leaking its MSBs gives us a foothold.

After probing with different malformed `r`, I could get the backend to crash after computing with partial exponents that leaked the top 20–30 bits of `n - φ(n)` via timing or error side channels.

I repeated this several times, recorded the bit patterns, and assembled the upper portion of `n - φ(n)`.

---

## 0x04 – Coppersmith Time

Once you have partial bits of `p + q`, it’s game time.

I used a classical result:

```math
If you know upper bits of p + q, and n = p*q,
you can solve for p and q via Coppersmith’s method (small root lattice attack).
```

Basic SageMath outline:

```python
from sage.all import *

# Known values
n = <leaked modulus>
kbits = <MSBs of p+q>

PR.<x> = PolynomialRing(Zmod(n))
f = x + (1 << (bits - kbits))

# Search small roots
roots = f.small_roots(X=2**(bits//2), beta=0.4)
for root in roots:
    maybe_pq = root + (1 << (bits - kbits))
    test = maybe_pq ** 2 - 4 * n
    if is_square(test):
        p = (maybe_pq + sqrt(test)) // 2
        q = n // p
        break
```

After a few parameter tweaks… boom. I had `p` and `q`.

---

## 0x05 – Forging a Proof

Now with `p` and `q` in hand, I could compute φ(n), generate valid zero-knowledge proofs, and bypass every check.

Here’s a snippet of the forge step:

```python
from Crypto.Util.number import getRandomRange, inverse

n = p * q
phi = (p - 1) * (q - 1)
g = 2
r = getRandomRange(1, phi)
A = pow(g, r, n)

# Verifier sends challenge c
c = 42  # say
z = (r + c * secret) % phi

proof = { 'A': A, 'z': z }
```

Backend accepted this as a valid proof. Flag returned instantly.

---

## 0x06 – Flag

```
hitcon{the_error_is_leaking_some_knowledge}
```

---

## 0x07 – Takeaways

- Error messages matter. They leak. Always.
- Python’s int coercion + JSON parsing is a goldmine.
- Negative exponents aren’t always rejected up front.
- Bit-leaks + Coppersmith = Dead crypto
- You don’t need to fully understand the ZKP if you can bend the protocol to scream secrets

---

## 0x08 – Tools Used

- Python3 / SageMath
- Coppersmith lattice solver (Sage’s built-in)
- Burp Suite (to script JSON injections)
- jq / mitmproxy (for observing malformed inputs)

---


---
title: "Blindspot – SAS CTF 2025"
date: 2025-07-14
category: Crypto
platform: SAS CTF 2025
author: Vaishnav Baraskar
tags: ["CTF", "SAS CTF", "Crypto", "Cryptanalysis", "Blind Decryption", "SAS CTF 2025", "CTF Writeup", "Challenge Solving", "Vaishnav Baraskar"]
---


## 0x00 – Prologue

I saw "ECDSA nonce reuse" and knew we were about to crack something open real clean. When you’re dealing with ECDSA, one reused `k` is all it takes to rip the private key wide open. It’s not about breaking the algorithm—it’s about catching the dev slipping. And they slipped.

This was about turning one careless mistake into total compromise.

---

## 0x01 – Challenge Setup

The server signs messages using ECDSA, and if you send valid requests, it responds with the signature `(r, s)`.

Now here’s the twist: every once in a while, the server reuses the nonce `k`. That’s a death sentence.

ECDSA signature generation:

```
r = (k * G).x mod n
s = k⁻¹(m + d * r) mod n
```

If the same `k` is used in two different signatures:

```
s1 = k⁻¹(m1 + d * r) mod n
s2 = k⁻¹(m2 + d * r) mod n
```

We can isolate `d`:

```
d = ((s1 - s2) * inverse(m1 - m2, n)) mod n
```

---

## 0x02 – Getting the Signatures

So, I started spamming the server with different messages, logging each message and its associated signature:

```python
messages = []
signatures = []

for i in range(1000):
    msg = os.urandom(32).hex()
    r, s = get_signature_from_server(msg)
    messages.append(msg)
    signatures.append((r, s))
```

Once you collect enough `(msg, r, s)` triples, you just start scanning for reused `r` values—because reused `r` = reused `k`.

---

## 0x03 – Exploitation: Recovering the Private Key

Once I had two messages signed with the same `r`, all the math lined up.

```python
from hashlib import sha256
from Crypto.Util.number import inverse

def recover_private_key(m1, m2, s1, s2, r, n):
    z1 = int(sha256(m1).hexdigest(), 16)
    z2 = int(sha256(m2).hexdigest(), 16)
    s_diff = (s1 - s2) % n
    z_diff = (z1 - z2) % n
    d = (s_diff * inverse(z_diff, n)) % n
    return d
```

Once I had `d`, that was game over.

---

## 0x04 – Forging Signatures

With `d` in hand, I started crafting valid signatures manually.

```python
def sign_with_d(m, d, k, G, n):
    r = (k * G).x % n
    z = int(sha256(m).hexdigest(), 16)
    s = (inverse(k, n) * (z + d * r)) % n
    return (r, s)
```

Every forged signature passed validation. I quickly hit the `counter_sign` threshold the challenge enforced and unlocked the final endpoint.

---

## 0x05 – Flag

After blasting the server with my handcrafted sigs, it folded. The flag popped up in plaintext:

```
sasctf{{r3us3d_n0nc3s_br34k_cryp70}}
```

---

## 0x06 – My Take

ECDSA is solid—until a human tries to implement it without understanding `k`. Nonce reuse isn’t just a weakness, it’s a backdoor. The minute I saw those repeating `r`s, I knew I had a private key waiting for me.

---

## 0x07 – Tools Used

- Python + `ecdsa` library
- SageMath (for sanity-checking math)
- Hashlib + pycryptodome
- A notebook and some reckless server traffic

---

## 0x08 – Lessons Learned

- Don’t reuse `k`. Ever.
- Always double-check your randomness source.
- Logging `r` values during ECDSA can expose nonce reuse fast.
- Once you have `d`, you’re holding the keys to the kingdom.

---
title: "Encrypted Mail – DUCTF 2023" 
date: 2023-09-02 
category: Crypto & Protocol 
platform: DownUnderCTF 2023 
author: Vaishnav Baraskar
tags: ["CTF", "DUCTF", "DownUnderCTF", "Crypto", "Protocol Analysis", "Cryptography", "Secure Communication", "CTF 2023", "Vaishnav Baraskar"]
---

0x00 – Prologue

This one was different. Not just some base64 puzzle or random math CTF fluff. It had structure. It had depth. I knew from the first glance that "Encrypted Mail" was hiding something sophisticated. There was a Zero-Knowledge Proof involved — that alone made me crack my knuckles. That phrase isn’t tossed around unless it means business.

The challenge centered on a weird internal mail system where users could “prove” their identity without revealing secrets. But like any system, if the ZKP implementation’s off by even a byte, I’m slipping in.

And that’s exactly what happened.

---

## 0x01 – First Look: The App

So, I hit the challenge and it booted up a small web app where you could register users and send mail. There was a separate identity for `admin`, and a mysterious user called `flag_haver`. Of course, my eyes were on them.

After setting up my account, I noticed the app’s proof submission system.

Each login or action was gated behind a cryptographic “proof” — likely generated client-side.

Let’s break it down.

---

## 0x02 – The ZKP Protocol (What Was Meant to Happen)

Here’s the rough flow I reverse-engineered from the JS source and network traffic:

```txt
1. You choose a private key (secret).
2. You compute some public data from it (usually g^x mod p).
3. You use a ZKP to prove you know x without leaking it.
```

The proof looked like a simplified Schnorr-style interaction:

```python
# Prover side (client)
priv = random_secret()
pub = pow(g, priv, p)

r = random_nonce()
a = pow(g, r, p)

# Server gives you a challenge:
e = H(a || pub || message)

z = (r + e * priv) % q

# Proof: (a, z)
```

The server then checks:

```python
a' = (g^z * pub^-e) % p
H(a' || pub || message) == e
```

All looks good — until you peek at how `e` was calculated.

---

## 0x03 – The Flaw: Deterministic `e` = Same `z`

So I did this:

```bash
curl -X POST /prove -d '{"pub":..., "proof": [a, z]}'
```

But I noticed something funny — if I sent the same message and public key, the `e` challenge was always the same.

That’s... *not* good.

In proper ZKPs, the `e` challenge must come from the verifier and should be **fresh** and **random** each time. Here, the client was controlling it.

Meaning: replay attacks. But it gets better — if I saw someone else’s `pub` and `proof`, I could forge things myself.

---

## 0x04 – Getting Admin Privileges

So, I needed a proof that passed for the admin account.

Captured a valid `(a, z)` pair from a legit admin login. Now, even if I didn’t know their private key, I could replay the proof.

But to do that, I needed to find the admin’s `pub`. Guess what? It was public. Listed in the `/users` page.

Boom:

```python
admin_pub = ...  # from the listing
admin_proof = (a, z)  # sniffed or logged from traffic
```

Sent this to the server:

```bash
curl -X POST /prove -d '{"pub": admin_pub, "proof": [a, z]}'
```

I was now logged in as admin.

---

## 0x05 – Forging the Mail Command

Next up: contacting `flag_haver`.

I noticed from the source code and the UI that you could send “mail” with arbitrary subject and message.

The only trick was, you had to send it **from** an authenticated user.

Now that I was `admin`, I could send anything to anyone.

So I did:

```bash
curl -X POST /send -d '{"from": "admin", "to": "flag_haver", "subject": "give", "body": "drop it"}'
```

And surprise — this triggered an automated rule in `flag_haver`'s inbox to reply with the flag if the sender was `admin`.

A moment later, I checked the `admin` inbox:

```bash
curl /inbox/admin
```

---

## 0x06 – The Flag and Its Format

Inside that inbox, buried in a JSON response:

```json
{
  "subject": "Here",
  "body": "DUCTF{f4ulty_proofs_and_fake_admins_win_games}"
}
```

Game over.

---

## 0x07 – What Went Wrong (In Their Code)

So here’s a quick breakdown of their mistake:

- The client was generating the challenge (`e`) for the ZKP instead of the server.
- Since `e` was based on deterministic inputs, same input = same proof.
- No randomness = replay attacks possible.
- They allowed unauthenticated access to user public keys.
- They reused message IDs in a way that let you spoof message contexts.

All of it combined meant I could:

1. Forge login as `admin` using replayed proof.
2. Send message to `flag_haver` impersonating `admin`.
3. Trigger flag leak.

---

## 0x08 – Tools & Tactics

- `Burp Suite` and `mitmproxy` for intercepting login proofs
- Custom Python scripts to replay proofs and automate requests
- Manual inspection of JS crypto to reverse protocol logic
- `curl` for raw API manipulation

---

## 0x09 – Outro

This challenge was a beautiful blend of protocol design, crypto assumptions, and good old HTTP poking. It wasn’t about breaking AES or factoring primes — it was about knowing how these systems fail in the real world.

And once again, trusting the client turned out to be their biggest mistake.

I just helped them learn that lesson the hard way.

---

## 0x0A – Flag

```
DUCTF{f4ulty_proofs_and_fake_admins_win_games}
```


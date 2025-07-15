---
title: "Binary Badlands – HTB University CTF 2024"
date: 2024-04-21
category: Crypto
platform: HTB University CTF 2024
author: Vaishnav Baraskar
tags: ["CTF", "HTB", "HTB University", "Crypto", "Binary Exploitation", "CTF 2024", "Reverse Engineering", "Challenge Writeup", "Cryptanalysis"]
---


## 0x00 – Prologue

I knew from the moment I saw "MD5" in the challenge description, things were about to get weird. Anyone who's spent enough time around outdated crypto knows MD5 is a landmine. It’s fast, broken, and predictable in just the right (or wrong) ways. This challenge leaned all the way into that mess.

Binary Badlands didn’t hand you a key—it just gave you a broken door and dared you to walk through it.

---

## 0x01 – Challenge Overview

There’s a service that registers users using the **MD5 hash of the username** as an identifier.

That sounds like a terrible idea. That *is* a terrible idea. And that’s where the challenge lives.

- You send a username.
- The backend computes `md5(username)`.
- That hash gets stored as your unique identifier.

But MD5 is collision-prone, especially when you’re allowed to choose arbitrary strings. The idea?

**Generate two usernames with the same MD5 hash. Register one. Log in with the other.**

Why does that work? Because when the server checks `md5(username)` and uses that to verify you, it has no clue whether you used the *original* or the *collision*. Game on.

---

## 0x02 – Getting a Collision

To pull this off, I needed a pair of strings such that:

```python
md5(user1) == md5(user2) and user1 != user2
```

### Step 1: Pick a Collision Generator

I used [HashClash](https://github.com/cr-marcstevens/hashclash), the classic collision toolkit.

Install instructions are painful on modern systems, so I switched to an existing Docker image for it. Fast and works out of the box.

Here’s the process I followed:

```bash
# Clone HashClash
$ git clone https://github.com/cr-marcstevens/hashclash
$ cd hashclash
$ make -C src

# Run fastcoll
$ ./src/fastcoll -o msg1.bin msg2.bin
```

Now `msg1.bin` and `msg2.bin` contain two binary strings with identical MD5 hashes.

### Step 2: Alphanumeric-ify

Raw binary isn't accepted as a username. The service only took alphanumerics.

I had two options:

- Try to force fastcoll to generate only safe bytes (hard)
- **Base64 encode** the collision blocks and decode on server (if possible)
- Or, use **known alphanumeric MD5 collisions** from research papers.

I opted for the third approach and used the classic "alphanumeric MD5 collisions" technique published here: [https://marc-stevens.nl/research/md5-1.html](https://marc-stevens.nl/research/md5-1.html)

Using that, I had two distinct strings with identical MD5s.

```python
user1 = "pocAAAA..."
user2 = "pocBBBB..."

assert md5(user1) == md5(user2)
```

---

## 0x03 – Registration and Exploitation

Now with `user1` and `user2` in hand:

- Register `user1` with a known password (say `pass123`)
- Try logging in using `user2` and the same password

The backend, trusting only the MD5(username), thinks you're the same person.

### Example Interaction:

```http
POST /register
{
  "username": "pocAAAA...",
  "password": "hunter2"
}

POST /login
{
  "username": "pocBBBB...",
  "password": "hunter2"
}
```

And boom—access granted.

---

## 0x04 – The Flag

Once logged in, I got access to the admin panel—previously tied to the original user.

Inside: a single HTML page with the flag.

```
HTB{finding_alphanumeric_md5_collisions_for_fun_...}
```

Classic crypto mistake. Classic win.

---

## 0x05 – My Take

This wasn’t about hardcore math or obscure crypto. It was about reminding devs (and us) that trusting hash functions for identity is bad news.

My thought process throughout:

- MD5? Bad.
- Hashing usernames? Worse.
- Collision? Practically guaranteed.
- Alphanumeric? Slight detour.
- Flag? Mine.

---

## 0x06 – Tools Used

- `HashClash` (for MD5 collision generation)
- Python (hash verification)
- Burp Suite (to automate form abuse)
- Hashcat (optional for reverse testing)

---

## 0x07 – Final Words

A lesson in why even seemingly harmless cryptographic shortcuts can destroy systems. MD5 isn't just old—it's broken. Don't hash identifiers. And definitely don’t trust client-provided strings without validation.

But if they do?

You’ll find me in the Badlands.


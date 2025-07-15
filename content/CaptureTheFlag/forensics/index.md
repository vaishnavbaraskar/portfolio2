---
title: "Hard Forensics – BlackHat MEA Quals 2023"
date: 2023-11-04
category: Forensics
platform: BlackHat MEA CTF
author: Vaishnav Baraskar
tags: ["CTF", "BlackHat MEA", "Forensics", "File Analysis", "Disk Forensics", "Memory Dump", "CTF 2023", "Challenge Writeup", "Vaishnav Baraskar"]
---


Sometimes, you get a JPEG, and you just know it’s lying to you. It smiles at you innocently like any regular image, but as a hacker, you know better. So, I stared at the given JPEG for a moment — instinctively opened it in a hex editor. Why? Because standard images don’t end with a bunch of gibberish appended to them.

That’s how this challenge kicked off. Classic case of “look deeper.”

---

## 0x01 – Initial Recon: The JPEG

First things first:

```bash
file image.jpg
```

Yup, it was a normal JPEG on the surface. But once I cracked it open with `xxd`, I started scrolling through and saw something odd toward the end:

```bash
xxd image.jpg | less
```

There were plaintext strings that clearly didn’t belong in a raw image file.

One in particular stood out:

```txt
https://pastebin.com/raw/EXAMPLEURL
```

That was it. Breadcrumb #1. I knew I was following some chain.

---

## 0x02 – Chasing the Pastebin

Pulling the raw Pastebin:

```bash
curl https://pastebin.com/raw/EXAMPLEURL
```

Inside was a Mega link. That's when I knew it was going to get dirty.

```txt
https://mega.nz/file/EXAMPLE#KEYEXAMPLE
```

I used `megadl` to avoid the browser nonsense:

```bash
megadl 'https://mega.nz/file/EXAMPLE#KEYEXAMPLE'
```

Boom. It dropped a ZIP file. Unzipped it and saw what looked like a Chrome user data directory.

```bash
unzip dump.zip -d chrome_data
```

---

## 0x03 – Chrome User Profile Analysis

Alright, Chrome user folder? That means `Extensions/`, `Default/`, some cached junk — the usual suspects.

I knew the flag wouldn’t be obvious. Time to grep through extensions:

```bash
grep -r BHFLAG chrome_data/
```

Nothing.

Went into:

```bash
chrome_data/Default/Extensions/
```

There were several folders — some auto-generated IDs like `aabbccddeeff...`. Popped into one, found some `background.js` and `content.js` files.

At first glance, everything looked obfuscated. Variables were like `_0x2f1b`, code all mashed up into one-liners. This wasn’t just some noob JS dev — someone deliberately tried to hide something.

Time to deobfuscate.

---

## 0x04 – Digging Through Obfuscated JavaScript

I copied the script into a JS beautifier:

```js
const _0xabc = ["ZEdWemRBPT0=", "log"];
console[_0xabc[1]](atob(_0xabc[0]));
```

Once decoded, I got:

```js
console.log(atob("ZEdWemRBPT0="));
```

Which gives:

```txt
flaG==
```

But in our case, it wasn’t that simple. There was a huge string that looked like reversed base64.

Example:

```js
let secret = "NjYzYjMwYjAwMmVjMmVkNTQ0NTExMDc1NzExZGNjZWEwMjJiMzM2fQ==".split("").reverse().join("");
console.log(atob(secret));
```

Ran it in Node.js:

```bash
node
> let secret = "NjYzYjMwYjAwMmVjMmVkNTQ0NTExMDc1NzExZGNjZWEwMjJiMzM2fQ==".split("").reverse().join("");
> console.log(Buffer.from(secret, 'base64').toString())
```

Output:

```txt
BHFLAGY{6133b20aeccd11750114f4b45a2de5c822700b36}
```

Gotcha.

---

## 0x05 – Extra: Chrome Extension Fingerprinting

Curious to go a bit further — I fingerprinted the extension to see if it was custom-built for this CTF.

```bash
cat manifest.json
```

Saw a vague name like:

```json
{
  "name": "Chrome Toolbox",
  "version": "1.0",
  "background": {
    "scripts": ["background.js"]
  },
  "permissions": ["tabs"]
}
```

Not on the Web Store. Likely custom packed.

I even tried loading it manually in a test Chrome profile with `--load-extension=PATH`, but yeah — definitely just for embedding the payload.

---

## 0x06 – Thoughts

What I liked about this challenge:

- Clean breadcrumb trail (JPEG ➝ Pastebin ➝ Mega ➝ Chrome dump ➝ JS reverse)
- Not overly noisy or filler
- The obfuscation was basic but good enough to throw off a lazy glance

What I’d improve if I was designing it:

- Maybe throw a decoy extension or a rabbit hole folder to up the forensics tension

But all in all, solid challenge.

---

## 0x07 – Flag

```
BHFLAGY{6133b20aeccd11750114f4b45a2de5c822700b36}
```

---

## 0x08 – Tools Used

- `xxd` and `grep` (for static digging)
- `curl` and `megadl`
- `Node.js` for decoding
- JS beautifiers and `chrome://extensions/` for manual inspection

---

## 0x09 – Outro

It’s the kind of challenge that rewards persistence. Each layer gives you just enough to keep going — no wild guessing required. You follow the data, decode it, deobfuscate it, and get rewarded.

Felt like peeling back layers of a dirty onion. And I liked it.


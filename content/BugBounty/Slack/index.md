---
title: "The Phantom Subdomain: How I Found Slack’s Forgotten Backdoor"
description: "A journey through DNS shadows, S3 whispers, and OAuth ghosts that led to a live phishing vector"
date: '2024-09-14'
categories: ["Digital Forensics", "Bug Bounty Tales"]
tags: ["Subdomain", "Open Redirect", "Slack", "Phishing"]
header:
  image: "/images/slack-phantom-subdomain.jpg"
  caption: "A forgotten subdomain — quiet but deadly"
---

## Prologue — The Digital Graveyard

**Midnight. The glow of my monitor painted the walls a faint blue as I scrolled through Slack’s sprawling domain records.**

Most hackers love the thrill of breaking into fortified systems — I’ve always been drawn to the forgotten ones. The ones left behind.

That’s when I saw it: `legacy-auth.slack.com`.

A subdomain that shouldn’t exist. Or at least, shouldn’t respond.

I leaned forward, fingers hovering over the keyboard.

**"Let’s see if this ghost still breathes."**

## Chapter 1 — The First Whisper

A quick dig command confirmed it — the domain resolved. No errors. No warnings. Just a silent, open door.

**First test:**

```bash
curl -I https://legacy-auth.slack.com
```

The server responded with a **200 OK**. No content. No fanfare. Just… presence.

> "Why would Slack leave an auth-related subdomain dangling like this?"

My mind raced:

- Possibility 1: A deprecated system still lingering in the shadows.
- Possibility 2: A misconfiguration waiting to be weaponized.
- Possibility 3: A trap.

I needed to know more.

## Chapter 2 — The Bucket in the Dark

**The Server: `AmazonS3` header caught my eye.**

> "An S3 bucket? For an auth endpoint?"

That’s when it clicked — this wasn’t a live service. This was a corpse. A leftover artifact from an old authentication flow, now abandoned but still technically alive.

**Next move:** Check if the bucket was claimable.

- Attempted to fetch `/.env` → **404**
- Tried `%2e%2e` (directory traversal) → **403**
- Tested `/?debug=1` → **Nothing**

No luck. But then—

> "What if this thing still processes redirects?"

## Chapter 3 — The Silent Redirect

I recalled Slack’s OAuth flow. Old documentation mentioned `redirect_to` parameters.

**Crafted test URL:**

```plaintext
https://legacy-auth.slack.com?redirect_to=https://evil.com
```

**Sent it.**

**The response:**

```http
302 Found
Location: https://evil.com
```

**My breath caught.**

> "Oh. Oh no."

This wasn’t just a dead subdomain — it was a live phishing weapon.

### The Impact

- No warnings. Users would see `slack.com` before being silently redirected.
- Perfect for phishing.

A malicious link could look like:

```
"Hey, check this Slack message: https://legacy-auth.slack.com?redirect_to=fake-login.slack.xyz"
```

- No takeover needed.
- The vulnerability was live **right now**.

## Chapter 4 — The Report

I documented everything:

- Steps to reproduce
- Proof of concept
- Potential attack scenarios

**Sent it via HackerOne.**

**Slack’s response?**

- Acknowledged in < 24 hours
- Fixed within days (They nuked the subdomain)
- Bounty awarded

## Epilogue — Why It Mattered

This wasn’t a flashy RCE or a SQLi. It was a quiet bug — the kind most people overlook.

But in the right (wrong) hands, it could’ve tricked thousands.

**A ghost in the machine. A whisper in the DNS records. A backdoor, not through code — but through neglect.**

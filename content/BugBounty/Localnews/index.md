---
title: "LocalNews and the Whispering Header - SQLi in a Forgotten Log"
description: "How a harmless User-Agent turned into a SQL injection vector—and what the server logs whispered back"
date: '2024-02-17'
categories: ["Weekend Bug Bounties"]
tags: ["SQLi", "Log Poisoning", "User-Agent"]
header:
  image: "/images/localnews-sqli-banner.jpg"
  caption: "Visual metaphor of an SQL injection whispering through headers"
---

## Prologue — When Headers Speak

**10:47 PM — Rain tapped against the window while Burp Suite ran idle.** I was deep into recon on a small CMS platform called *LocalNews*. The payout was modest, the target obscure—but that’s the beauty of it. Quiet places often hide loud bugs.

I wasn’t expecting much. Just a few requests here and there to check how deep this rabbit hole went. But the moment I noticed the server logging behavior, my coffee got a refill.

This is the story of how a simple HTTP header broke into a database.

---

## Chapter 1 — The User-Agent with a Secret

Every request leaves a trail—sometimes in logs, sometimes in caches. I was curious:

*What if LocalNews logs the User-Agent header?*

Some internal admin panels do that for analytics or debugging. Harmless, until it isn’t.

My first nudge:

```bash
curl -A "HelloWorld" https://localnews-cms.com
```

No response on the frontend. But I had Burp Collaborator running in parallel and a gut feeling something got stored.

Time to push.

---

## Chapter 2 — The Delay That Said Everything

Next attempt:

```bash
curl -A "1' AND (SELECT sleep(5))--" https://localnews-cms.com
```

The server stalled for exactly 5 seconds.

I raised an eyebrow.

"That wasn’t lag. That was a whisper from the database."

It confirmed the User-Agent header was being interpolated directly into an SQL statement.

---

## Chapter 3 — What’s in the Logs, Stays in the Logs

This was classic log-based SQL injection. The devs likely had something like:

```python
cursor.execute(f"INSERT INTO logs (ua) VALUES ('{user_agent}')")
```

No escaping. No sanitizing. Just raw string shoving into a SQL query.

I imagined an ancient PHP logger, held together by duct tape and false hope.

I had time-based confirmation. Now, I wanted data.

---

## Chapter 4 — Credentials in Cleartext (Well, Almost)

I crafted the next payload like a love letter to `UNION SELECT`:

```bash
curl -A "1' UNION SELECT 1,concat(username,':',password),3 FROM users--" https://localnews-cms.com
```

I didn’t expect an immediate win. But when I checked the logs (visible via error traces), there it was:

```
admin:5f4dcc3b5aa765d61d8327deb882cf99
```

**MD5. Classic.**

I smiled. The hash? That’s `password` in disguise.

The site stored admin credentials in plaintext hashes. Bad practice, better bounty.

---

## Chapter 5 — My Thought Process

**Why User-Agent?** Because it’s usually overlooked. It’s meant to be informative, not dangerous. But when web servers blindly log anything without sanitization, you get injection vectors where you least expect them.

**Why time-based SQLi?** Because there was no output. A blind spot. Time becomes your oracle.

**Why MD5?** That’s a question for their devs.

---

## Chapter 6 — Reporting the Quiet Leak

This was never a flashy bug. No fancy payloads, no XSS sparks. Just patience and curiosity.

I submitted the report via HackerOne with:

- All curl payloads
- Timeline of delay-based confirmations
- MD5 leak proof
- A suggestion to migrate logging to prepared statements

Slack pinged me 12 hours later.

**They confirmed.**

**Patched the next day.**

**Bounty awarded: $25.**

Small, but it wasn’t about the money. It was about hearing a database whisper through headers.

---

## Chapter 7 — Root Cause Analysis

- Raw SQL string concatenation
- Lack of prepared statements for logging
- User-supplied input in logs (User-Agent, Referer, etc.)

---

## Chapter 8 — Mitigations

After the patch:

- Logging functions moved to parameterized queries
- Audit of other headers used in logs
- No more visible error traces exposing log output

---

## Epilogue — Why This Matters

Sometimes, you don’t need zero-days or deep fuzzing to find a vuln. All it takes is one header—and a little imagination.

*SQLi lives everywhere. Even in the quiet logs of forgotten CMS platforms.*

Let this be a reminder:

**Every input is an entry point. Every header a hypothesis.**

And when you're stuck? Just ask yourself:

*"What would the server log?"*

---
title: "Curiosity & file_id=187: My First Bug Bounty Journey with FileSharePro"
description: "How a boring file URL became my first bounty-worthy vulnerability—an IDOR that leaked CEO documents."
date: '2023-02-09'
categories: ["Bug Bounty", "First Bounty"]
tags: ["IDOR", "Authorization", "Bugcrowd", "File Disclosure"]
header:
  image: "/images/filesharepro-idor.jpg"
  caption: "When numbers unlock secrets — the unintended door of FileSharePro"
---

## Prologue — A New Hunter’s First Spark

**They always say your first bounty feels different.**  
For me, it started with a file URL. Not a secret admin panel or a vulnerable upload endpoint. Just a link:  
`https://filesharepro.com/download?file_id=123`

It was a lazy September evening. I had just made myself some chai and was half-heartedly poking around FileSharePro’s web interface. No recon tools. No Burp magic. Just the browser and my terminal.

I wasn’t even sure I was doing bug bounty *right*. But something about that `file_id=123` caught my attention. Why 123? What about 124?

> "It’s probably access-controlled," I told myself.  
> "But what if it isn’t?"

---

## Chapter 1 — The Curiosity of file_id

It began with observation.

```text
Logged-in user’s document:
https://filesharepro.com/download?file_id=123
```

A simple GET request. No auth tokens in the URL. No hashes. Just `file_id=123`.

I tried incrementing it manually:

```bash
curl -I "https://filesharepro.com/download?file_id=124"
```

**200 OK**

Huh?

```bash
curl -I "https://filesharepro.com/download?file_id=125"
```

**200 OK**

That’s when the realization sank in. These weren’t just IDs. These were **unguarded** file references.

---

## Chapter 2 — Pattern Recognition

As I continued testing, I saw a pattern:

- My file was `123`.
- `124`, `125`, `126` all returned 200s.
- `130` returned a PDF.
- `135` gave me someone’s invoice.
- `187` returned… **CEO_salary_report.pdf**

> “There’s *no way* I should be seeing this.”

I quickly downloaded it to confirm.  
It contained sensitive salary information — including the C-suite.

I sat back. Palms sweating.

**This wasn’t just a bug. It was a classic IDOR — and a serious one.**

---

## Chapter 3 — Bash as a Flashlight

I crafted a basic bash loop:

```bash
for i in {124..200}; do  
  curl -I "https://filesharepro.com/download?file_id=$i" | grep "200 OK"  
done
```

Hits came back fast. Dozens of valid files were exposed.

If I replaced `-I` with `-O`, I could download everything.

But I didn’t.

**Ethical hacking is hacking with restraint.**

---

## Chapter 4 — Thinking Like a Developer

Why was this possible?

Here’s my assumption of the backend logic:

```python
@app.route('/download')
def download_file():
    file_id = request.GET.get("file_id")
    file_path = db.query("SELECT path FROM files WHERE id = ?", [file_id])
    return send_file(file_path)
```

Seems harmless — but there’s no **ownership check**.

Any authenticated (or unauthenticated) user could enumerate file IDs and access arbitrary files.

---

## Chapter 5 — Reporting the Ghost Door

I wrote my report with care:

- Clear title: **IDOR via file_id leads to sensitive document exposure**
- Impact: Access to internal corporate files (CEO salary report)
- PoC: `curl -O https://filesharepro.com/download?file_id=187`
- Fix Recommendation: Use UUIDs + enforce ownership validation

Submitted to **Bugcrowd**.

Waited nervously. Kept refreshing.

---

## Chapter 6 — The Response

24 hours later:  
**"Triaged. Valid. Severity: Medium."**

48 hours later:  
**"Patched. Bounty awarded: $25"**

I smiled.

Not for the amount. For the validation.

For a new hunter, it wasn’t about the money.  
It was about proving to myself I could do it.

---

## Root Cause Summary

**What went wrong?**

- file_id was treated as a trusted parameter
- No backend validation for user-file ownership
- IDs were sequential and guessable

---

## Remediation

**Patched within 5 days**.  
Fixes:

- file_id replaced with non-sequential UUIDs
- Download handler now checks file ownership

---

## Reflections — The First Find

This was my first bounty.  
No exotic exploit chain. No fuzzing wizardry.

Just curiosity. Observation. Patience.

I learned that **even the smallest oversight can lead to serious exposure** — and that bug bounty is often about asking the simple question:

> “What happens if I change this number?”

---

## Timeline

- **September 10, 2023 – 9:12 PM**: Observed `file_id` behavior
- **September 10, 2023 – 9:42 PM**: Confirmed IDOR with sensitive file
- **September 11, 2023 – 1:15 AM**: Submitted report to Bugcrowd
- **September 11, 2023 – 6:30 PM**: Triaged and acknowledged
- **September 14, 2023 – 8:00 AM**: Fixed and bounty awarded

---

## Final Thoughts

This bug taught me the power of simplicity.

You don’t need recon scripts or fancy tools to find vulnerabilities. Sometimes, all it takes is an eye for detail and a hunch worth exploring.

**If you’re just starting in bug bounty, trust your curiosity.**  
Because that’s all I had when I found `file_id=187` — and it was enough.

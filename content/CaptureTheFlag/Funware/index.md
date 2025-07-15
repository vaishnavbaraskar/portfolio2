---
title: "Decrypting Shadows: Reversing Ransomware from CactusCon's FunWare"
description: "A deep-dive into a PyInstaller ransomware sample hidden in a disk image. From mounting raw images to decrypting XOR’d files — this is the story of unwinding FunWare."
date: '2022-02-07'
categories: ["Forensics", "Reverse Engineering", "CTF Writeups"]
tags: ["Ransomware", "PyInstaller", "Disk Image", "XOR Decryption", "Python RE"]
header:
  image: "/images/cactuscon-funware.jpg"
---

# **Prologue — Digging Through Bytes**

I remember that lazy afternoon. Coffee was cold. The air around me was still, except for the quiet hum of my laptop. I booted up the CactusCon 2022 CTF and saw something that immediately got my attention: **FunWare**. A name oddly whimsical for what turned out to be a ransomware reversing challenge.

---

# **Initial Recon — Mounting The Disk**

The first artifact was a `.img` file — a disk image. First instinct? Mount it and see what's hiding.

```bash
fdisk -l funware.img
mount -o loop funware.img /mnt/funware
ls /mnt/funware
```

Once mounted, I wandered through the filesystem. Nothing too odd at first — until I stumbled upon a directory filled with encrypted files and a single suspicious binary named `funware`.

My gut said PyInstaller. That hunch would prove right.

---

# **Binary Dissection — PyInstaller Unpacking**

Before diving in, I confirmed the binary type.

```bash
file funware
```

Output?

```
funware: ELF 64-bit LSB executable, dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2
```

Smelled like Python inside a wrapper. So I used the go-to combo: `binwalk` and `pyinstxtractor`.

```bash
binwalk -e funware
python3 pyinstxtractor.py funware
```

Inside the extracted directory, I was greeted with `.pyc` files and embedded Python modules. Now we were talking.

---

# **Code Analysis — Unraveling the XOR Logic**

One of the files caught my eye — something like `ransomcore.pyc`. I decompiled it with `uncompyle6`.

```bash
uncompyle6 ransomcore.pyc > ransomcore.py
```

The encryption logic looked something like this:

```python
def xor_encrypt(data, key):
    return bytes([b ^ key[i % len(key)] for i, b in enumerate(data)])
```

There it was — classic XOR with a hardcoded key.

The key?

```python
key = b'c4ctu5c0n'
```

I took a moment. Laughed. Of course they had to use "cactuscon" leetspeak.

---

# **Decryption — Breaking The Cipher**

I threw together a quick Python script to decrypt all files under the `/mnt/funware/encrypted` directory.

```python
import os

KEY = b'c4ctu5c0n'

def xor_decrypt(data, key):
    return bytes([b ^ key[i % len(key)] for i, b in enumerate(data)])

enc_dir = '/mnt/funware/encrypted'
out_dir = '/mnt/funware/decrypted'
os.makedirs(out_dir, exist_ok=True)

for fname in os.listdir(enc_dir):
    path = os.path.join(enc_dir, fname)
    with open(path, 'rb') as f:
        encrypted = f.read()
    decrypted = xor_decrypt(encrypted, KEY)
    with open(os.path.join(out_dir, fname), 'wb') as f:
        f.write(decrypted)
```

---

# **Verification — Known Header Analysis**

After decryption, I picked one file and opened it in a hex viewer. The header matched the PNG magic bytes:

```
89 50 4E 47 0D 0A 1A 0A
```

Clean. Confirmed.

---

# **Reflections — Skills Sharpened**

This wasn’t just about reversing some toy ransomware. It was about chaining multiple disciplines:

- Disk image forensics
- PyInstaller unpacking
- XOR key analysis
- Bulk decryption and verification

The process felt like solving a puzzle — one byte at a time.

---

# **Bonus — Alternate Decryption in CyberChef**

After identifying the XOR key, I also used [CyberChef](https://gchq.github.io/CyberChef/) to test out decryption on one file.

Steps:
- Load file as hex
- Use "XOR Brute Force" or manual XOR with `c4ctu5c0n`
- Output matched the Python result

That double validation added confidence.

---

# **Epilogue — Quiet Satisfaction**

There’s a weird peace in reversing malware. Watching obfuscation crumble, revealing intent beneath noise. FunWare wasn’t the hardest challenge, but it was methodical — and that’s what I loved.

I left the decrypted files intact, zipped the Python scripts, and closed the terminal.

Just another quiet day cracking binaries.

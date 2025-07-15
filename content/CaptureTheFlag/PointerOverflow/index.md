---
title: "PointerOverflow CTF 2024 – DF"
date: 2024-05-03
category: Forensics
platform: PointerOverflow CTF 2024
author: Vaishnav Baraskar
tags: ["CTF", "PointerOverflow", "Forensics", "Digital Forensics", "File Recovery", "Data Extraction", "CTF 2024", "Challenge Writeup", "Vaishnav Baraskar"]
---

## 0x00 – Prologue

Forensics challenges usually start out tame—bit of file carving, maybe some strings, or sleuthing around disk images. But sometimes, one of those USB dumps hits differently.

This was one of those.

"DF 100 – A Record of Events" came wrapped in a raw binary blob, straight from a USB device. My job? Extract the story—and the flag—buried somewhere inside.

---

## 0x01 – Initial Recon

The file extension screamed `raw`. So first instinct: plug it into some digital forensic tools and start digging.

```bash
$ file usb_dump.raw
usb_dump.raw: data

$ binwalk usb_dump.raw
DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
... (mostly entropy) ...
```

Nothing obvious from binwalk. So I loaded it into **FTK Imager** and mounted it to inspect file system artifacts.

If this was truly a USB image, I expected FAT32 or exFAT.

And sure enough:

```txt
Volume Name :  USB_VOLUME
File System :  FAT32
```

That's when things got more interesting.

---

## 0x02 – Files That Shouldn't Exist

Inside the volume, I spotted some deleted files. Most of them looked like junk: temp files, thumbnail caches.

But one stood out:

```
/Documents/_log.txt (deleted)
```

I exported and ran strings:

```bash
$ strings _log.txt | less
```

And there it was—a strange conversation log mixed with debug lines. Some parts were base64-encoded, some redacted, but something like this stood out:

```
U1RSSU5HOkZMQUdfcG9jdGZ7dXdzcF81N3I0bjYzcl8xbjRfNTdyNG4zXzE0bmR9
```

Which decoded to:

```bash
$ echo "U1RSSU5HOkZMQUdfcG9jdGZ7dXdzcF81N3I0bjYzcl8xbjRfNTdyNG4zXzE0bmR9" | base64 -d
STRING:FLAG_poctf{uwsp_57r4n63r_1n_4_57r4n63_14nd}
```

---

## 0x03 – Artifact Recovery

Just to be sure this wasn’t a red herring, I carved through the slack space of the disk.

Using `foremost`:

```bash
$ foremost -i usb_dump.raw -o output/
```

It recovered some PNGs, PDFs, and a partial .docx that confirmed the USB was used for documenting internal investigations.

The presence of `_log.txt` in the deletion records and encoded data in the disk confirmed that the flag wasn’t a fluke.

---

## 0x04 – Final Flag

```
poctf{uwsp_57r4n63r_1n_4_57r4n63_14nd}
```

Clean. Hidden. But not hidden enough.

---

## 0x05 – Thoughts

This challenge was less about exploit dev and more about being methodical. It was digital archeology. USBs tell stories—they carry logs, metadata, and flags for those who know where to chisel.

When the frontend gives you nothing, plug it in, mount it, and start looking where people forget to clean.

---

## 0x06 – Tools Used

- FTK Imager (for mounting and analyzing the USB)
- binwalk / strings / xxd
- base64 (obviously)
- foremost (slack space recovery)
- hexedit (to eyeball deleted data)

---
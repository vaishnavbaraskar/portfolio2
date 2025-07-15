---
title: "Strings Unleashed: Unsafe Length Handling in BookPro Reader"
description: "How careless metadata parsing and an unchecked `rep movsb` led to return address corruption and command execution."
date: '2024-06-22'
categories: ["Buffer Overflows", "Memory Exploits"]
tags: ["Windows x64", "Metadata Parsing", "Zip Exploitation", "BookPro Reader"]
header:
  image: "/images/bookpro-heap.png"
  caption: "Metadata parsing without bounds checks is a dangerous game"
---

## 3. Unsafe Length Handling in BookPro Reader (x64)

**Platform:** Windows 10 x64  
**Target:** BookPro Reader (EPUB/ZIP Parser)  
**Discovered:** May 2024  
**Status:** Unreported – no known disclosure policy  
**CVSS (Est.):** 8.1 (High) – Stack corruption via overlong metadata entry

---

### ✦ My Thought Process

I was exploring BookPro Reader to analyze how it handled EPUBs internally (since EPUBs are just ZIP files with XML/HTML inside). I figured the metadata parser would be a good starting point — especially since it's invoked early during file load.

Once I unpacked the binary and traced how it handled `metadata.xml`, I noticed something weird in `load_metadata`. The use of `rep movsb` wasn’t guarded by any sort of length check. Felt like the classic "assume ZIP metadata is small" mistake.

---

### ✦ Disassembly Breakdown

```asm
.text:0000000140012000 load_metadata:
    mov     rdx, [rsp+arg_8]     ; source: zip metadata string
    mov     rax, [rsp+arg_0]     ; destination: stack or heap buffer
    rep     movsb                ; copies string, blindly
```

There’s no validation of the input size, and I confirmed the destination was on the stack in some cases. So if you overfill the metadata entry in the zip file, you eventually blast through return addresses on the stack.

And yeah — that’s exactly what I did.

---

### ✦ Exploitation Strategy

It’s a classic case of buffer overrun through ZIP metadata parsing. Here’s how I pulled it off:

1. Created a malicious EPUB file (just a renamed ZIP).
2. Injected a long metadata string in `content.opf` (or `metadata.xml`).
3. Crafted the metadata so the string would overflow the target buffer.
4. Lined up shellcode after padding — made sure it aligned with stack execution path.

The end result? Controlled return address → jump to shellcode.

---

### ✦ Payload Details

I didn’t go crazy with the payload. Just used basic command execution to confirm exploitability.

```asm
mov     rcx, offset "cmd.exe"
call    WinExec
ret
```

This was injected right after the padding. The overwritten return address pointed to the payload directly on the stack.

Here's how I encoded it in the metadata:

```xml
<dc:title>AAAAAAAA...[padding]...[shellcode bytes]...</dc:title>
```

ZIP structure was valid — the parser just kept reading blindly.

---

### ✦ Debugging + Tuning

Used WinDbg and `sxe av` to catch the crash. Confirmed the exact offset needed to overwrite the return address via trial runs.

Breakpoints in `load_metadata` helped me step through the parsing process.

Also played with ZIP alignment and compression — made sure the shellcode wasn't mangled by decompression.

Bonus: the `WinExec` call worked even when DEP was enabled, since the stack was marked RWX (classic legacy config).

---

### ✦ What Went Wrong (For Them)

- No length checks on incoming strings  
- Blind `rep movsb` copy  
- Buffer on the stack — ripe for ROP/ret overwrite  
- ZIP metadata treated as "trusted"  

---

### ✦ Personal Reflection

It’s 2024 and we still have apps parsing metadata like it’s the ‘90s. No bounds checks, no sanity checks, just "parse and hope." Book readers are low-hanging fruit — nobody’s fuzzing them aggressively, and they deal with weird formats daily.

All I had to do was dig into the ZIP, extend a field, and watch the stack implode.

No disclosure this time — not my circus.

---
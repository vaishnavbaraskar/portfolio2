---
title: "Signed Once, Loaded Twice: Plugin Signature Bypass in CodeWorks IDE"
description: "A gap between validation and execution — how I bypassed CodeWorks' plugin signature system with a well-timed swap."
date: '2023-04-11'
categories: ["Binary Exploitation", "Plugin Security"]
tags: ["TOCTOU", "DLL Injection", "Windows x64", "Reverse Engineering"]
header:
  image: "/images/codeworks-plugin-bypass.jpg"
  caption: "When checks are separate from load — timing becomes everything"
---

# Signature Bypass in CodeWorks IDE Plugin Loader

## Prologue: Not All Checks Are Made Equal

It started with curiosity, like it usually does. I wasn’t even targeting CodeWorks specifically. I was just bouncing around dev tools I had lying around — checking how they loaded plugins, how they validated them, and if they did anything... out of order.

That’s how this one fell into my lap. CodeWorks IDE v3.8 — a fairly common IDE with support for third-party plugins. It looked polished. Fancy GUI, nice dark mode. But underneath that gloss? One misplaced assumption.

## First Clue: Signature Validation

I fired up x64dbg and dropped a breakpoint on CreateFileW, just to see how it was loading its plugin DLLs. What caught my eye was how neatly the plugin was being validated before loading.

```asm
.text:140012000 load_plugin proc near
.text:140012000     mov     rax, [rsp+arg_0] ; plugin path
.text:140012005     call    validate_sig
.text:14001200A     test    eax, eax
.text:14001200C     jz      short load_abort
.text:14001200E     call    load_pe_image
```

So it did validate the plugin. All good, right?

Not quite.

## The Gap – Time of Check vs Time of Use

Here’s what went through my head:

- Step 1: It reads the plugin from disk
- Step 2: It checks the signature
- Step 3: If valid, it loads the plugin again using load_pe_image

That’s two separate disk reads. Two file accesses. Two chances for me to sneak something in between.

This is the classic TOCTOU — Time of Check to Time of Use — in the flesh. The validation step looked at one version of the file. The load step could easily load a different one if I was fast enough.

## Exploitation Strategy – Old Trick, New Wrapper

I built two DLLs:

- One signed with a dev certificate (dummy plugin)
- One completely unsigned and malicious

The goal was to:

1. Let CodeWorks validate the signed one
2. Immediately swap in the unsigned one before it loads the plugin
3. Profit

This isn’t new. We’ve seen it with privilege escalation bugs. But here it was, hiding inside an IDE plugin loader.

I wrote a quick swapper in Python to monitor when CodeWorks opened the file and rapidly swap it before the second open.

## The Payload – Just Enough to Show It Worked

This was my injected assembly inside the malicious DLL's entry point:

```asm
    xor rax, rax
    mov rdi, rsp
    sub rsp, 0x100
    lea rbx, [rip+payload]
    call rbx
payload:
    ; write to memory, open reverse shell, etc.
```

This could have been anything — reverse shell, keylogger, memory patch. But I kept it simple to prove execution.

## Outcome: No Signature, No Problem

After the swap, CodeWorks loaded my unsigned plugin into its process. No complaints, no alerts. It trusted the result of validate_sig without revalidating the actual file it loaded.

I had arbitrary code execution inside the IDE.

## Impact

This kind of issue could:

- Allow unsigned plugins in otherwise locked-down enterprise environments
- Let attackers inject malicious payloads into the IDE
- Possibly pivot to other dev tools, depending on plugin privileges

## What Should Have Happened

The loader should have:

- Validated the *actual* in-memory file being loaded
- Used a single open handle from start to finish
- Avoided TOCTOU entirely by loading from memory buffer after validation

## Final Thoughts

Sometimes the biggest bugs aren’t low-level overflows or exotic kernel tricks. They’re just someone assuming that a file won’t change between two steps.

It only took a fraction of a second — and the right DLL was in the wrong place.

## Summary

- Vulnerability: Signature check bypass via TOCTOU
- Platform: Windows x64 (CodeWorks IDE 3.8)
- Technique: Swap plugin DLL after validation but before loading
- Impact: Unsigned arbitrary code execution in IDE
- Fix: Avoid TOCTOU, use same memory or file handle after validation


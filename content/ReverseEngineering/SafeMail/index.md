---
title: "Heap Drift: Misaligned Write in SafeMail’s Attachment Parser"
description: "How a filename parser, unaligned `rep movsb`, and heap chunk corruption gave me use-after-free control."
date: '2024-02-08'
categories: ["Heap Corruption", "Memory Exploits"]
tags: ["Windows x86", "Heap Metadata", "UAF", "Desktop Clients"]
header:
  image: "/images/safemail-heap.png"
  caption: "Legacy email clients and fragile heaps never mix well"
---

###  Prologue

Started out just poking at SafeMail’s desktop client because I was curious how they handled attachments. It’s always those small parsing subsystems where things fall apart. I loaded up the binary in IDA and watched the way filenames were processed when attachments were being saved.

Didn’t take long to see something sketchy in how they were copying the filename into a heap buffer.

I had a hunch. The code was using `rep movsb`, and I already knew alignment wasn’t being enforced. My first thought: “Alright, what happens when you feed it an unaligned filename with precise offsets?”

Turns out: corrupted heap metadata, and a clean use-after-free.

---

###  Root Cause Deep Dive

The vulnerability is in the attachment filename parser. The binary copied filenames using a byte-by-byte `rep movsb` approach, blindly trusting input length.

```asm
.text:00411000 parse_filename:
    mov     eax, [ebp+arg_0]     ; length of user input
    lea     edi, [heap_buffer]   ; destination (heap)
    lea     esi, [user_input]    ; attacker-controlled source
    rep     movsb                ; fast copy without alignment
```

There’s no length validation, no alignment padding, no null-termination check—just raw copying into heap memory.

So if I give it a carefully sized filename, I can make the next chunk misaligned. And because of how LFH (Low Fragmentation Heap) works, that corruption will fly under the radar... until it doesn’t.

---

###  Weaponizing the Bug

The corrupted heap chunk doesn’t immediately crash anything, but later during cleanup, that chunk gets freed, and the metadata is messed up. The allocator doesn’t complain... it just leaves a dangling pointer lying around.

Here’s how I exploited it:

1. Create a filename long enough to overrun into next chunk.
2. Free the corrupted chunk using a background cleanup process (SafeMail calls `free()` from a background thread for attachments).
3. Control where that freed chunk is reused. That pointer gets passed into `save_attachment()` and eventually `memcpy()` uses it.

Boom — use-after-free. Now I control the memory being referenced by a legit code path.

---

###  Payload Engineering

I didn’t need to get super fancy with the payload. I used basic shellcode that pops calc.exe — just to prove exploitability.

```asm
xor     eax, eax
mov     al, 0x50              ; syscall index for WinExec
lea     ebx, [esp+20]         ; pointer to "calc.exe"
call    WinExec
```

And the heap buffer that held it? Got in through the filename input:

```c
char* filename = "A...A\x90\x90\xEB\x04\xCC...calc.exe\x00";
```

`rep movsb` helpfully copied that directly into the heap for me, and the misaligned pointer just happened to get reused after the corrupted chunk was freed.

---

###  Debugging It All

The crash was flaky at first. Took me a bit to realize Windows 10 LFH was helping me — but not always.

I used:

- `gflags /p /enable safemail.exe /full`
- PageHeap + AppVerifier
- WinDbg with `!heap -p -a` and `!analyze -v`
- Logging the actual pointer values during `free()`

Eventually found a pattern: multiple attachment uploads caused predictable heap behavior. That gave me the same layout each time.

Also had to slow the process down using a custom delay in SafeMail’s background thread. Timing was crucial for consistent uaf.

---

###  Secondary Write Primitive

At one point, I realized I had not only a uaf read — but a write as well.

`memcpy(dst, src, len)` was hitting the freed chunk, and I had control over both `dst` and `len`. That meant I could potentially use it as an arbitrary write primitive.

Didn’t go full chain-RCE with that, but it’s a juicy detail worth keeping in mind.

---

###  Mitigations (For the Vendor, if they care)

- Avoid `rep movsb` unless you enforce alignment and verify length.
- Don’t trust user input for heap operations — sanitize and align.
- Enable safe unlinking and LFH metadata validation in release builds.
- Add a NULL terminator and use safer routines like `memcpy_s()`.

---

###  Final Thoughts

I wasn’t even trying to find a memory corruption bug. Just digging through email client internals. But I’ve learned — these old-style Win32 desktop clients? They leak vulnerabilities like a faucet.

Modern mitigations don’t mean much when the root logic is this sloppy.

Reverse engineering this was fun. Classic heap mess.  
Didn’t report — they don’t have a program. But it’s in the archive now.

---
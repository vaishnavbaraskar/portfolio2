---
title: "Signed to Compromise: Kernel Overflow in XLogDriver.sys"
description: "How a signed USB logger opened the gates to Ring-0 via unchecked IOCTL copy — and how I stole SYSTEM."
date: '2023-06-10'
categories: ["Exploit Development", "Kernel Exploitation"]
tags: ["Kernel Exploits", "Windows x64", "Signed Drivers", "Token Stealing"]
header:
  image: "/images/xlogdriver-kernel.jpg"
  caption: "Signed and dangerous — when drivers become doorways"
---


# Signed Driver Exploit in XLogDriver.sys (x64 Kernel)

## Prologue: A Signed Invitation to Ring-0

This one hit different. I was sifting through a bunch of random vendor drivers, most of them dusty utilities for things like USB logs and peripheral diagnostics. Nothing fancy. But one caught my eye: `XLogDriver.sys`.

It was signed. Legitimately. Which meant if it had a bug, it could be loaded on pretty much any modern Windows box without triggering Driver Signature Enforcement.

Game on.

## Discovery — The IOCTL That Trusted Too Much

Using IDA Pro and WinDbg, I reversed the main `HandleIoctl` function. It was small, compact, and terrifying.

Here’s the relevant disassembly:

```asm
XLogDriver!HandleIoctl:
    mov rax, [rcx+0x08] ; user-mode ptr
    mov rdx, [rcx+0x10] ; size
    mov rdi, kernel_buffer
    rep movsb           ; unsafe
```

Let me break that down:

- `rcx` points to a user-supplied structure from DeviceIoControl.
- It pulls a user pointer from `[rcx+8]` and a size from `[rcx+10]`.
- Then it performs a blind `rep movsb` into a kernel buffer.

So yeah, user-controlled data and size directly copied into a privileged memory region, with zero validation.

## Exploitation — Smashing Into Ring-0

I crafted a custom IOCTL request where:

- The source buffer was a big blob of junk with a carefully placed function pointer.
- The size field was larger than the kernel buffer it was copying into.

Once the overflow hit, it walked into adjacent kernel memory and landed on an IRP structure used by another part of the driver. That IRP contained a function pointer. Guess what I replaced it with?

Yup, my shellcode.

### Payload (Ring-0 Token Stealing)

Here’s the payload I injected to steal SYSTEM token:

```asm
; Token stealing payload for Windows x64
xor rax, rax
mov rax, [gs:188h]       ; Current _KTHREAD
mov rax, [rax+0xb8]      ; _EPROCESS structure
mov rcx, rax
mov rdx, [rcx+0x2f0]     ; SYSTEM token from another process
mov [rax+0x358], rdx     ; Replace current token
```

This gives me full SYSTEM privileges — instantly.

I used this access to map an unsigned kernel driver of my own, allowing persistent rootkit-like access while the legit signed driver covered my tracks.

## Extra Payload — Disabling ETW (Just for Fun)

After SYSTEM, I threw in another payload that stomped on the ETW registration table:

```asm
; Disable ETW logging to stay invisible
mov rcx, qword ptr [ETW_GLOBAL_TABLE]
xor rax, rax
mov [rcx], rax
```

This made sure no system logs caught what I was doing. Not necessary, but satisfying.

## Outcome

- Escalated to SYSTEM by exploiting a signed kernel driver
- Achieved arbitrary code execution in ring-0
- Bypassed Driver Signature Enforcement by piggybacking on the signed driver
- Disabled logging via ETW tampering

## Mitigation

This driver should never have been allowed to perform unchecked memory copies with user-controlled sizes. At a minimum:

- Validate buffer sizes before copy
- Use safer routines like `RtlCopyMemory` with explicit bounds
- Avoid trusting IOCTL inputs directly

And honestly? Maybe stop signing drivers like this without a proper security audit.

## Final Thoughts

There's something poetic about turning a legit signed driver into a SYSTEM-level payload delivery mechanism. Windows lets you load it with no questions asked — and all you need is one unchecked copy to flip the entire kernel.

In the end, the driver signed itself into my exploit chain.

## Summary

- Vulnerability: Unchecked memory copy via IOCTL in signed driver
- Platform: Windows 10 x64
- Technique: Heap overflow to IRP object + function pointer overwrite
- Payload: Kernel shellcode to steal SYSTEM token
- Bonus: ETW logging disabled post-exploitation
- Fix: Size checks, defensive coding in kernel drivers

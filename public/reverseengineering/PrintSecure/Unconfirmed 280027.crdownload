---
title: "Stacking Bytes: Heap Overflow in PrintSecure’s Spooler"
description: "How a metadata length field, a misplaced `rep movsb`, and a vtable pointer led to SYSTEM shell through the print spooler."
date: '2023-11-06'
categories: ["Heap Overflow Exploits"]
tags: ["Windows x64", "Heap Corruption", "RPC", "Privilege Escalation"]
header:
  image: "/images/printsecure-heap.png"
  caption: "PostScript metadata meets unchecked bytes"
---

# **Prologue — Midnight Layers & Metadata Games**

I’d been reverse engineering some internal RPC routines on a lightly documented print management service—**PrintSecure**, used across several enterprise Windows Server 2019 deployments. The kind of service that hums quietly in the background, doing menial job routing, completely overlooked. That's usually a good place to find something sharp.

After a couple hours inside Ghidra and x64dbg, stepping through weird PostScript packet handlers, I found what I was looking for: a tiny unchecked `rep movsb`. One of those legacy leftovers that still punch through memory when no one’s watching.

---

## **Platform**

- Windows Server 2019 x64  
- PrintSecure Enterprise Network Spooler (build 10.2.1479)

---

## **Overview**

The vulnerability lies in the print spooler’s handling of PostScript job metadata sent over its custom RPC channel.  

Specifically, the handler for this metadata chunk trusted a declared length field directly from the client and used it in a raw memory copy operation. There were no bounds checks, no size verification—just straight memory action. And that’s how we got here.

---

## **Vulnerability Analysis**

Here's what the disassembly looked like when I cracked open the responsible routine — `parse_metadata()` — in Ghidra and then confirmed with x64dbg:

```asm
.text:140010B00 parse_metadata proc
.text:140010B00     push    rbp
.text:140010B01     mov     rbp, rsp
.text:140010B04     sub     rsp, 100h
.text:140010B0B     mov     rax, [rcx+10h] ; get length from request
.text:140010B0F     mov     rsi, [rcx+18h] ; pointer to data
.text:140010B13     lea     rdi, [rbp-80h] ; target heap buffer
.text:140010B17     rep movsb             ; unchecked copy
```

So yeah, `rax` is fully attacker-controlled. That `rep movsb` operation doesn't verify the length — and it writes directly into a heap-allocated buffer.

If you send metadata with an inflated length, it just plows through memory. Perfect for classic heap chunk overlap.

---

## **Exploit Path**

With the ability to overflow adjacent heap memory, I focused on tampering with a virtual function pointer (vtable) stored just after the vulnerable buffer.

The flow went like this:

1. Send a malicious metadata blob with an oversized declared length.
2. Heap overflow corrupts an object’s vtable pointer nearby.
3. Trigger service logic that calls the corrupted virtual method.
4. Controlled pointer leads to arbitrary shellcode.

I placed a stub that pointed to a fake function table in memory. Then, when the service hit the call, it jumped into my payload.

---

## **Payload: Assembly Stub**

The injected payload for hijacking control looked like this:

```asm
_start:
    mov rax, 0x1122334455667788  ; dummy function pointer (replace with shellcode addr)
    mov [rbx], rax               ; overwrite object's vtable ptr
    call [rbx]                   ; trigger the overwritten pointer
```

In the real exploit, `0x1122334455667788` would be a pointer to shellcode residing in the heap, often something like reverse shell or token stealing logic, depending on the goal.

---

## **Outcome**

Once the hijacked pointer got triggered, I was executing within the **PrintSecure Spooler’s SYSTEM context**. From there:

- Spawned a remote SYSTEM shell
- Accessed printer configurations, logs, and registry keys
- Established persistent RDP access (using Print Spooler privileges)

No user interaction. No alerts fired.

---

## **Impact**

- **CVE Status:** Privately reported, patch in progress
- **Privilege Escalation:** Yes (Local SYSTEM)
- **Remote Trigger:** Yes (Authenticated RPC)
- **Affected Builds:** At least 10.2.1479 and earlier

---

## **Lessons**

Unchecked memory copies still exist, even in well-funded enterprise systems. This one came down to basic trust in client data and a legacy instruction doing what it does best—move bytes without asking questions.

No stack canaries. No heap cookie defense triggered. Just a clean overwrite and detour into shellcode.

Sometimes, it really is that simple.

---

# **Endgame**

I closed the debugger, left the PoC running in a loop, and watched SYSTEM access roll in like a warm breeze through an open port. All from a spooler that thought it was just printing.

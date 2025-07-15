---
title: "Chaining Control: ROP Exploitation in HealthDesk Report Viewer"
description: "Turning a forgotten CSV parser and a static binary into full shell access through a classic ROP chain."
date: '2023-09-14'
categories: ["ROP Exploits"]
tags: ["Windows x86", "ROP", "Buffer Overflow", "Legacy Systems"]
header:
  image: "/images/healthdesk-rop.png"
  caption: "ROP gadgets from an old CSV parser"
---

# **Prologue — CSVs, Gadgets & Shells**

It started with a curiosity hit—an old install of **HealthDesk Report Viewer**, still alive on a legacy Windows 7 x86 box. Binary hadn't been touched since 2010. And it was one of those static base address builds, no ASLR, no DEP, no nothing. Just waiting.

One rainy afternoon, I dug into the CSV import feature, and sure enough, the parser was playing loose with user input. Specifically, a `strcpy` on unchecked data.

Things escalated quickly.

---

## **Platform**

- Windows 7 x86  
- HealthDesk Report Viewer (build 2.4.8.0)

---

## **Overview**

The vulnerability lives inside the CSV import handler. The input data from the CSV file is passed to a routine that calls `strcpy` into a local buffer. That buffer? 0x400 bytes on the stack.

So, if we drop a long enough CSV string, we can overwrite the return address—and in this case, build a full ROP chain. No mitigations stopped us, thanks to the binary's static base and lack of stack protection.

---

## **Disassembly Insight**

Here's what the function looked like during reversing:

```asm
.text:00401350 csv_import proc
.text:00401350     push    ebp
.text:00401351     mov     ebp, esp
.text:00401353     sub     esp, 400h
.text:00401359     mov     eax, [ebp+0Ch] ; input string
.text:0040135C     lea     edi, [ebp-400h]
.text:00401362     call    _strcpy        ; unsafe!
```

Classic stack buffer overflow with zero checks. It’s just waiting for a CSV row longer than 1024 bytes.

---

## **Exploit Path**

My approach was straightforward:

1. Craft a malicious `.csv` line that exceeds 1024 bytes.
2. Overflow into the saved return address.
3. Inject a ROP chain that marks the stack executable using `VirtualProtect`.
4. Land into shellcode at the end of the buffer.

---

## **ROP Chain: Sample Gadget Flow**

This is a simplified version of the ROP chain used in the actual exploit:

```asm
    ; VirtualProtect to make stack executable
    push 0x40              ; PAGE_EXECUTE_READWRITE
    push 0x1000            ; size
    push 0x1000            ; MEM_COMMIT
    push esp               ; lpAddress
    call VirtualProtect

    ; Jump to shellcode
    jmp esp
```

With a static binary and known image base, every gadget address was hardcoded. No fuzzing needed—just clean gadget stitching.

---

## **Outcome**

Once the CSV payload was parsed by the report viewer, the shell popped instantly. Here's what followed:

- Reverse shell access as user running the GUI
- Pivoted into local network through old SMB shares
- Dropped persistence using scheduled tasks

The entire exploit was embedded into a legitimate-looking CSV file. Nothing on disk, no alerts, no AV hits.

---

## **Impact**

- **Privilege Escalation:** From user-space GUI to full shell
- **Exploit Method:** Classic stack overflow with ROP chain
- **Mitigations Bypassed:** ASLR (not present), DEP (disabled), Stack Cookies (not present)
- **User Interaction:** Yes (CSV import via GUI)

---

## **Lessons**

Old binaries still carry sharp edges. The kind of vulnerabilities that modern stacks forgot—raw `strcpy`, static base, no mitigations.

The fact that this service parsed attacker-supplied CSVs and ran it on a trusted backend gave it teeth. And when gadgets are predictable, all you need is patience.

---

# **Endgame**

Dropped the payload, clicked "Import CSV", and watched the shell connect back. 

Sometimes, exploitation is just about finding the right dusty corner of legacy code... and stringing a few gadgets together.


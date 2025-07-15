---
title: "Overflowing Authority: Stack Smash in LocalAdminTool.exe"
description: "A misused strcpy and a named pipe handshake led to full privilege escalation through stack overflow."
date: '2023-10-02'
categories: ["Stack Overflow", "Privilege Escalation"]
tags: ["Windows x64", "Named Pipe", "Shellcode", "Stack Exploit"]
header:
  image: "/images/localadmin-stack.png"
  caption: "A named pipe, a strcpy, and admin shells"
---

# **Prologue — Pipes, Stacks, and Hidden Elevation**

I was digging around Windows 10 utilities installed on a workstation used for internal admin scripting. One tool stood out — `LocalAdminTool.exe`. It used a named pipe to receive commands, and the binary hadn’t seen a patch in years.

Once I opened it up in IDA and started examining pipe input handling, the problem jumped out. A classic `strcpy` into a stack buffer. In x64. In 2023. Just sitting there.

---

## **Platform**

- Windows 10 x64  
- LocalAdminTool.exe (internal IT build, 1.9.3)

---

## **Overview**

The vulnerability lives in the pipe handling logic. When commands are sent to the named pipe, the service pulls them into a stack buffer using `strcpy`.

There’s no length validation. The attacker can write as much as they want. And with no stack cookie present, it’s possible to overwrite the return address and hijack control.

---

## **Function Disassembly**

Here's what the core function looks like:

```asm
.text:0000000140011000 handle_pipe_input:
    sub     rsp, 0x200
    lea     rdi, [rsp+20h]
    call    strcpy          ; unsafe copy from pipe data
```

So, data from the pipe is dumped into `[rsp+0x20]` without bounds checking.

---

## **Exploit Path**

The exploit path was straightforward:

1. Send a long string to the named pipe.
2. Overwrite the return address on the stack.
3. Point it to shellcode placed earlier in the same buffer.
4. Return into the shellcode and run as the elevated process.

Because stack cookies weren’t enabled and DEP wasn’t properly enforced, this exploit landed easily.

---

## **Payload: Shellcode (x64)**

Here’s the core shellcode used to launch a new elevated `cmd.exe` session:

```asm
    mov rcx, offset cmd_string
    call WinExec
    ret

cmd_string db "cmd.exe", 0
```

The WinExec address was resolved statically due to no ASLR in this build.

---

## **Extra: Named Pipe Interaction Example**

This was the basic PowerShell one-liner used to send payloads through the named pipe:

```powershell
$pipe = new-object System.IO.Pipes.NamedPipeClientStream(".", "adminpipe", "Out")
$pipe.Connect()
$writer = new-object System.IO.StreamWriter($pipe)
$writer.Write("A" * 600 + "[ROP or shellcode here]")
$writer.Flush()
$writer.Close()
$pipe.Close()
```

This overflows the buffer and delivers control.

---

## **Outcome**

The exploit gave me full **admin privileges** on the target machine:

- Spawned an elevated shell
- Modified local group policies
- Escalated into system service accounts

The kicker? It all happened without touching disk or triggering UAC. Just a pipe, a buffer, and some shellcode.

---

## **Impact**

- **Privilege Escalation:** Yes (Local to Admin)
- **Exploit Method:** Stack overflow via named pipe input
- **Mitigations Bypassed:** Stack cookie (absent), DEP (misconfigured)
- **User Interaction:** None (pipe-based delivery)

---

## **Lessons**

Even modern systems carry old mistakes. `strcpy` on stack data in 2023 is hard to justify. But it's still out there — usually in legacy tools made for internal use.

This one turned out to be a quiet privilege ladder, just waiting for someone to overflow the right buffer.

---

# **Endgame**

Pushed the payload, caught the elevated shell, and rewrote the rules from inside.

No alerts. No interruptions. Just the sound of a prompt blinking back, waiting for its next elevated command.

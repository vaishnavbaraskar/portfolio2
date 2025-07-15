---
title: "Copy, Paste, Exploit: Buffer Overflow in EduGrade Import Engine"
description: "An old-school `strcpy` stack smash lets us hijack execution and spawn shellcode from the stack."
date: '2024-09-18'
categories: ["Buffer Overflows", "Memory Exploits"]
tags: ["Windows x64", "strcpy", "Stack Overflow", "EduGrade"]
header:
  image: "/images/edugrade-buffer.png"
  caption: "Stack buffer plus strcpy equals game over"
---

## 4. Buffer Overflow in EduGrade Import Engine (x64)

**Platform:** Windows 10 x64  
**Target:** EduGrade Desktop (Import Engine Parser)  
**Discovered:** August 2024  
**Status:** Local Privilege Escalation (unpatched)  
**CVSS (Est.):** 7.6 – Local overflow leads to code execution

---

### ✦ My Thought Process

I got curious about how EduGrade was parsing import files — especially `.csv` and `.txt` files with student data. Thought maybe it’d use standard libraries, maybe be hardened a bit. Nah. It was as C-style as it gets.

I poked around in IDA and found the function that handled student name parsing. That’s when I saw this beauty: `strcpy` copying raw data into a fixed-size stack buffer.

---

### ✦ Disassembly Breakdown

```asm
.text:0000000140011000 import_student_record:
    lea     rdi, [rsp+80h]      ; local stack buffer
    mov     rsi, rdx            ; user-supplied name field
    call    strcpy              ; crash and burn
```

That’s it. No bounds checking. No `strncpy`. Just trust the user and pray.

The destination is a local stack buffer, and the source is whatever came from the import file. So it’s classic stack smashing territory. All I needed to do was control the name field in the file.

---

### ✦ Exploitation Plan

Here's how I lined everything up:

1. Crafted a `.csv` file with a super long name field (300+ bytes).
2. The payload had padding → shellcode → overwritten return address.
3. The return address pointed directly back to the shellcode on the stack.
4. The function returned → jumped straight into my code.

--- 

### ✦ Payload: Stack-Based Shellcode

I went with minimal shellcode — just enough to get a calc or command shell for proof-of-concept. In this case, `syscall_spawn` was an internal helper syscall I repurposed.

```asm
xor     rdi, rdi                ; null out argument
mov     rax, syscall_spawn      ; load syscall index
syscall                         ; execute process
```

The shellcode lived just after the padding inside the same student name buffer.

--- 

### ✦ Debugging & Testing

I used x64dbg and set breakpoints right before the `strcpy`. After letting it rip, I confirmed the overwrite worked — return pointer jumped directly to shellcode.

Used the following pattern to get exact offset:

```bash
./pattern_create.rb -l 400
./pattern_offset.rb -q [crash value]
```

After replacing the crash offset with a jmp address (or return to shellcode), it worked like a charm.

---

### ✦ What They Screwed Up

- `strcpy` on stack buffers in 2024  
- No bounds check on user input  
- Didn’t use safer alternatives like `strncpy` or length parsing  
- DEP was disabled and stack was executable  

---

### ✦ Final Thoughts

I didn’t expect to find such a textbook bug in a modern import engine. But here we are. `strcpy` lives on.

This was a fun one — no heap trickery, no bypasses — just raw stack overflow like it’s 2003. If you ever want to get into low-hanging local bugs, start with educational tools. They’re almost always full of legacy code.

Didn’t report this one. It’s local-only, and honestly, they’re not gonna patch it unless it hits a headline.

---
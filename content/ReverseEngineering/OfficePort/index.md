---
title: "Phantom Libraries: DLL Hijacking in OfficePort Scheduler"
description: "How a misconfigured DLL search order in a modern Windows 11 scheduler app let me plant persistent payloads with ease."
date: '2024-02-21'
categories: ["DLL Hijacking", "Persistence"]
tags: ["Windows 11", "DLL Injection", "OfficePort", "Local Execution"]
header:
  image: "/images/officeport-dllhijack.png"
  caption: "Hijacked DLLs and persistent shells on modern Windows"
---

# **Prologue — The Ghost in the Folder**

DLL hijacking never really died. It's just waiting for the right developer to forget a LoadLibrary call. That’s exactly what happened with OfficePort Scheduler, a scheduling utility built for enterprise task planning on Windows 11.

One quiet session, while inspecting the app startup behavior, I caught it trying to load `tasklib.dll` from its working directory. No digital signature checks. No manifest restrictions. Just a plain old `LoadLibrary()` from wherever the binary lives.

All I had to do was drop my own version of `tasklib.dll` next to it.

---

## **Platform**

- Windows 11 Pro (22H2)
- OfficePort Scheduler x64 — Build 10.1.7
- Application launched at login via Startup registry key

---

## **Overview**

The vulnerability stemmed from how OfficePort Scheduler searched for `tasklib.dll`. Rather than loading from `System32` or using signed DLLs, the binary issued a direct `LoadLibraryA("tasklib.dll")` from the application root.

Since that folder was writable by the user, I had a clean injection point.

---

## **Discovery**

Using tools like ProcMon and API Monitor, I watched the startup behavior:

```plaintext
LoadLibraryA("tasklib.dll") = C:\Users\User\AppData\Local\OfficePort\tasklib.dll
```

It searched the current working directory first — classic mistake.

No SafeDllSearchMode override. No SetDllDirectory protection. It defaulted to the local folder.

---

## **Exploit Path**

1. **Write a custom DLL** with the same exported symbols as `tasklib.dll`.
2. **Drop it into the app's directory**, typically:
   ```
   C:\Users\User\AppData\Local\OfficePort\
   ```
3. **Wait for OfficePort to launch** (auto-started on login).
4. **Code inside the malicious DLL executes immediately** under user context.

---

## **Payload Entry (DLL Stub)**

```cpp
BOOL WINAPI DllMain(HINSTANCE hinstDLL, DWORD fdwReason, LPVOID lpvReserved) {
    if (fdwReason == DLL_PROCESS_ATTACH) {
        open_reverse_shell(); // connect back to listener
    }
    return TRUE;
}
```

The function `open_reverse_shell()` establishes a callback to my listener on port 443.

---

## **Assembly View (Simplified)**

```asm
DllMain:
    push rbp
    mov rbp, rsp
    call open_reverse_shell
    pop rbp
    ret
```

---

## **Persistence Bonus**

Since OfficePort Scheduler starts up via `HKCU\Software\Microsoft\Windows\CurrentVersion\Run`, the payload executes on every login.

No extra steps needed — persistence was baked in.

---

## **Outcome**

- Malicious DLL was loaded successfully on app startup
- Reverse shell opened reliably every login
- No AV alerts due to unsigned DLL loading by trusted app
- Zero user interaction required after planting the payload

---

## **Impact**

- **Exploit Type:** DLL hijacking / search order hijack
- **Impact:** Local code execution and persistence
- **Required Access:** Low (user-writeable folder)
- **Detection Evasion:** High (runs under trusted app name)
- **Mitigations Bypassed:** None enforced (SafeDllSearchMode disabled)

---

## **Fix Recommendations**

- Use full path to load critical libraries
- Enable SafeDllSearchMode and SetDefaultDllDirectories
- Sign all dependent DLLs and verify digital signatures
- Lock down application directories from being user-writeable

---

## **Lessons**

DLL hijacking on Windows 11 still lives because the fundamentals never changed. As long as devs rely on lazy LoadLibrary calls and apps run from writeable directories, attackers will always have a foot in the door.

---

# **Endgame**

Dropped my DLL, zipped up the folder, and watched a SYSTEM-signed binary do all the work for me on every boot. That’s not hacking — that’s automation.

DLL hijacking might be old school, but it still hits like a freight train when paired with persistence.

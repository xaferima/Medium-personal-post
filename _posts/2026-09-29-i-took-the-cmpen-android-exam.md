---
layout: post
title: "I took the CMPen-Android exam — here's everything you need to know"
date: 2026-09-29
description: "A full walkthrough of the CMPen-Android exam by The SecOps Group — 4 hours, 15 flags, practical Android pentesting. Tools, experience, question breakdown, and honest review."
categories: [security, certifications, mobile]
image: /Medium-personal-post/assets/images/cover-cmpen-android.jpg
---

# I took the CMPen-Android exam — here's everything you need to know

**Certified Mobile Pentester – Android** by The SecOps Group. 4 hours. A real APK on your own device. Practical, no essays.

I went in confident after years of mobile pentesting. I came out with merit — and a reminder that the best certification exams are the ones that make you work for it.

This is not a sponsored post. I paid for the exam myself. I'm writing this because I wish someone had given me this overview before I started.

---

## The 30-second summary

|                     |                                                                        |
| ------------------- | ---------------------------------------------------------------------- |
| **Duration**        | 4 hours                                                                |
| **Format**          | APK download + live VPN + Capture The Flag                             |
| **Questions**       | 15 flags (some are multiple choice, not all require a flag submission) |
| **Pass**            | 60%                                                                    |
| **Pass with Merit** | 75%                                                                    |
| **Retake**          | 1 free attempt included                                                |
| **AI allowed?**     | **No** (internet searches OK)                                          |
| **Device**          | Physical Android device or emulator, rooted                            |

---

## What makes this exam different

Most security certifications test your ability to **remember** things. This one tests your ability to **break an Android app** — for real.

You get an APK. You install it on your own device. You connect to their VPN, where a backend is waiting. No simulated environment, no "click here" scenarios. Just you, jadx, Frida, and an app that behaves like the ones your clients ship to production.

The time pressure is honest. 4 hours feels generous until you're staring at a certificate pinning implementation that won't die and question #7 is still open. The exam doesn't pause. It doesn't save progress. You either capture the flags or you don't.

---

## What you'll actually face

The syllabus is public, and passers have shared the general territory. Here's what the exam actually covers — roughly split **50/50 between static analysis and dynamic/exploitation**.

### Static analysis (~50% of questions)

This is where most of the fast points live. I found the majority of my flags from code review alone, before even running the app:

- Hardcoded credentials, API keys, tokens, crypto keys
- Manifest misconfigurations: `debuggable`, `allowBackup`, exported components, cleartext traffic
- Weak cryptography: ECB, static IVs, MD5/SHA1, "Base64-as-encryption"
- Certificate and signing scheme inspection
- Obfuscated code and packed strings

### Dynamic analysis (~50% of questions)

The other half requires running the app, instrumenting it, and intercepting traffic:

- Insecure logging via logcat (yes — the vendor's own sample question is about creds leaking into logs)
- Root detection bypass
- SSL pinning bypass
- Runtime memory analysis for decrypted secrets

### Component attacks

- Exported activities launched directly
- Content provider abuse: unauthorized reads, SQLi, path traversal
- Broadcast receiver spoofing
- IPC & intent-based attacks

### App-level exploitation

- Insecure WebViews and JavaScript bridges
- Business logic flaws (client-side checks that shouldn't be client-side)
- Misconfigured database storage

### Backend

- API security testing against the app backend: auth bypass, BOLA/IDOR, injection
- Access logs and forgotten endpoints

> **Note on question format:** Not every question asks you to submit a flag string. Some are **multiple choice** — you need to identify the correct statement about a vulnerability. Read each question carefully before diving in.

**Marks distribution:** Bypass questions (root detection, SSL pinning) carry the highest weight (~30 marks each). Component and activity exploitation questions are mid-range (~20 marks). Static analysis findings like hardcoded keys are lighter (~15 marks). Know where the points are.

That's a broad surface for 4 hours. You won't have time to do everything twice. The key is knowing exactly where each bug class lives and moving on when something resists.

---

## Tools you'll need

This is not a "use our web terminal" exam. You bring your own lab:

**Device:**

- Physical Android device (7.0+), rooted with Magisk — or an emulator (Genymotion/Memu/Nox are vendor-tested)

**Essential:**

- **adb** — your lifeline: logs, components, providers, file pulls
- **jadx / jadx-gui** — decompilation (the single most important tool — most flags come from here)
- **apktool** — smali patching, rebuild & resign
- **MobSF** — fast full static triage
- **Burp Suite** — backend and traffic analysis (SSL pinning bypass makes this mandatory)
- **Frida + frida-server** — instrumentation: bypasses, hooks, memory
- **Objection or LSPosed** — one-liner bypasses. I used LSPosed for root and SSL pinning bypasses and it worked perfectly. Pick whichever you're faster with.

**Nice to have:**

- Drozer — component attack surface enumeration (legacy, but on the syllabus)
- pidcat — sane logcat filtering
- apksigner / keytool — certificate and signing inspection (comes with JDK)
- CyberChef — decode/decrypt quickies

---

## My honest experience

**Result: Merit — 100% score.**

The exam delivered exactly what it promised: a real APK, a live backend over VPN, and 15 flags that test whether you can actually break an Android app — not just talk about it.

**Preparation:** one intensive week, physical device (Pixel 8 Pro, Android 17, rooted with KernelSU), focusing on:

- **DIVA** and **Damn Vulnerable Bank** for static analysis and storage bugs
- **InjuredAndroid** — CTF-style flags, closest thing to the exam format I found
- **OWASP UnCrackable L1–L3** — root and pinning bypasses until they became muscle memory
- **Allsafe** for Frida practice
- Building and freezing my own cheat sheet: every adb command, every Frida snippet, tested before exam day

**Exam day:** The APK was a compact banking-style app (`org.android.cmpen`) with a backend at `ninja.secops.group`. 15 questions, varied time per question — some I solved in 5 minutes with a static grep, others took 20+ minutes of instrumentation.

What surprised me: **about half the exam is static analysis**. I found most of my flags from `jadx` decompiled code before I even touched the app. The other half is dynamic — bypasses, traffic interception, component exploitation. The bypass questions (root detection, SSL pinning) carry the most weight, so get those working early.

Another thing: **not every question asks for a flag string**. Some are multiple choice — you need to identify the correct statement about the app's security posture. Read every question carefully before diving in.

The time pressure is honest. 4 hours feels generous until you realize you need to decompile, bypass pinning, intercept traffic, exploit components, and analyze crypto — all on the same app. But if your methodology is solid and your tools are tested, it's absolutely doable.

**Exam-day rules that matter:** internet searching is allowed; AI is **not**. That means your cheat sheet and your methodology have to carry you — no assistant will. Which, honestly, is exactly how a real engagement works.

---

## What the exam actually asked me

I won't give away answers — that defeats the purpose of a certification. But I can tell you what the exam covered, how the questions were structured, and what kind of thinking each one demanded.

### Root Detection Bypass (30 marks — multiple choice)

The first question. It asked me to examine the app's anti-reversing checks and identify the correct statement about its root detection. This is a static analysis question — you read the code and understand what the app does when it detects a rooted device. The app implements standard detection methods: checking build tags, looking for `su` binaries in common paths, and attempting to execute `which su`. The bypass is straightforward if you know your way around runtime hooks. The key insight: the detection exists, it works, but it can be neutralized.

### SSL Pinning Bypass (30 marks)

The heaviest question alongside root detection. The app uses OkHttp3's `CertificatePinner` with specific SHA256 hashes pinned to the backend domain. You need to bypass this to intercept the HTTPS traffic and find the flag in the server response. The flag isn't in the APK — it comes from the backend, so you need Burp configured, the pinning bypass active, and then you exercise the app while watching the traffic. Multiple endpoints are involved. This question tests whether you can actually set up and use a mobile intercept proxy end-to-end.

### Insecure Activity (20 marks)

The app has an activity that's exported without any permission protection in the manifest. The question asks you to identify it and exploit the weakness. Launching it directly — either from another app or via `adb` — triggers a server call with a hardcoded header value that includes secrets from the code. The flag appears in the response. This is a pure static analysis question if you know to check the manifest for `android:exported="true"` without intent filters or permission guards.

### Hardcoded Keys (15 marks)

The app has several secrets baked into the source code: a DES encryption key in `strings.xml`, a Firebase database URL, hardcoded strings used to build authentication headers, and encrypted values with their keys sitting right next to them. The question asks you to identify the hardcoded information. This is the fastest question if you run a simple secrets grep over the decompiled output — but only if you know what patterns to look for.

### The rest of the exam

The remaining questions cover the other syllabus areas: insecure logging (the app logs sensitive data including keys and plaintext to logcat), misconfigurations (`debuggable=true`, `allowBackup=true`, cleartext traffic allowed), an API endpoint with no authentication at all, and more. Each one tests a different vulnerability class from the OWASP Mobile Top 10.

The pattern I noticed: **the static analysis questions are the fastest to solve**. A grep over `jadx` output, a look at the manifest, a check of `strings.xml` — you can answer most of them in under 10 minutes each. The dynamic questions (bypasses, traffic interception) take longer because you need your environment working. Do the static ones first.

---

## What I'd recommend

**Root your device before exam day.** frida-server running, `frida-ps -U` verified, Burp CA installed. The exam doesn't wait while you debug Magisk modules.

**Start with jadx static analysis.** I found most of my flags from code review before I even ran the app. A secrets grep, a manifest check, a look at `strings.xml` — that's where the fast points are. Let MobSF scan in the background while you work through the static findings.

**Master the bypasses cold.** Root detection and SSL pinning gate other flags. Have your bypass scripts ready and tested — whether that's Frida, Objection, or LSPosed. I used LSPosed and it worked flawlessly. The key is having a method that you've tested on practice apps before exam day.

**Read every question carefully.** Not all of them ask for a flag string. Some are multiple choice where you need to identify the correct statement. Misreading a question wastes precious minutes.

**Timebox ruthlessly.** 15 flags, 240 minutes. Anything stuck past 20–25 minutes gets parked. Come back with the remaining time.

**Submit as you go.** Submit every flag the moment you have it, and screenshot proof. The clock doesn't do warnings.

**Practice with InjuredAndroid + UnCrackable.** Together they cover most of what this exam throws at you.

---

## Should you take it?

If you work in mobile security — or want to prove you can — yes. It's one of the most practical Android certs at this price point, and it sits in the same territory as the eMAPT ($599) and GMOB ($999) for a fraction of the cost.

It won't make you a mobile security expert in a weekend. But it will force you to build a real methodology: static triage, dynamic instrumentation, component attacks, and backend testing — exactly the flow of a professional mobile assessment.

---

## Resources to prepare

**Practice targets (free):**

- [DIVA (Payatu)](https://github.com/payatu/diva-android)
- [Damn Vulnerable Bank](https://github.com/rewanthtammana/Damn-Vulnerable-Bank)
- [InjuredAndroid](https://github.com/B3nac/InjuredAndroid)
- [AndroGoat](https://github.com/satishpatnayak/AndroGoat)
- [OWASP UnCrackable Crackmes](https://github.com/OWASP/owasp-mastg)
- [Allsafe](https://github.com/t0thkr1s/allsafe)
- [OVAA](https://github.com/oversecured/ovaa)

**Reference:**

- [OWASP MAS / MASTG](https://mas.owasp.org) — the bible
- [Mobile Hacking Lab](https://www.mobilehackinglab.com/)
- [HackTheBox mobile challenges](https://app.hackthebox.com/)

---

## GitHub Repository

All my notes, cheat sheets, templates, and helper scripts are available here:

[**CMPen-Android Exam Notes & Toolkit**](https://github.com/xaferima/CMPEN-ANDROID-NOTES)

Included in the repo:

- Full study notes covering the entire syllabus
- Copy-paste ready cheat sheet (adb, jadx, Objection, Frida, LSPosed, Drozer)
- Systematic APK audit checklist
- Helper scripts (frida-server start, proxy toggle, static scan pipeline, log harvesting)
- Frida hook inventory (SSL pinning bypass, root bypass, crypto tracing, memory scan)
- 7-day intensive study plan + exam-day strategy

Feel free to fork, star, and use it as a foundation for your own preparation.

---

## Final thoughts

The CMPen-Android won't make headlines like OSCP. But for mobile security practitioners, it's a serious, practical certification that respects your time and budget. No essays. No theory dumps. Just you, an APK, and 15 flags that won't capture themselves.

If you decide to go for it — good luck, and may your target apps always have `android:debuggable="true"`

---
layout: post
title: "Beerus Framework: Running a Full Mobile Pentest Stack Without Magisk"
date: 2026-11-06
description: "How I installed and validated Beerus Framework on a rooted Pixel 8 Pro with ReSukiSU — Frida Auto-Inject, sandbox exfil, memory dumps, ADB over network and boot persistence, without a single Magisk install."
categories: [security, mobile, frida]
image: /Medium-personal-post/assets/images/cover-beerus-framework-pixel8pro-2026.jpg
---

# Beerus Framework: Running a Full Mobile Pentest Stack Without Magisk

**How I installed and validated Beerus Framework on a rooted Pixel 8 Pro with ReSukiSU — and why it matters for authorized mobile assessments**

Offensive Security Researcher

---

## Table of Contents

1. [Introduction](#introduction)
2. [Disclaimer](#disclaimer)
3. [What is Beerus Framework?](#what-is-beerus-framework)
3. [The Feature Set — What You Actually Get](#the-feature-set--what-you-actually-get)
4. [Installing Without Magisk: Beerus on ReSukiSU](#installing-without-magisk-beerus-on-resukisu)
5. [Hands-On: Frida Auto-Inject Against an OWASP MASTG Playground App](#hands-on-frida-auto-inject-against-an-owasp-mastg-playground-app)
6. [Honest Assessment — Pros and Cons](#honest-assessment--pros-and-cons)
7. [Verdict](#verdict)
8. [References](#references)

---

## Introduction

Most Android security toolkits assume Magisk. You flash a Magisk zip, manage modules in the Magisk app, and reboot into a system where `su` is available through MagiskSU. That assumption breaks the moment your device runs a KernelSU-family solution — which is exactly what happened to my 2026 assessment device.

Beerus Framework is an Android application by the Hakai team (the same group behind hakaisecurity.io) that replaces an entire mobile pentest stack with a single app: Frida server management, script injection, sandbox data exfiltration, memory dumps, decoded manifests, ADB over the network, proxy profiles, system property manipulation, and boot-time persistence. No Magisk app required — just a KernelSU-compatible root provider and the Beerus module.

This post documents my real deployment on a **Google Pixel 8 Pro running Android 17 with ReSukiSU v4.2.0-rc1**, without ever opening or modifying the ReSukiSU manager app. Every screenshot and command in this article came from my own device during an authorized assessment workflow.

---

## Disclaimer

> **This guide is for authorized security testing only.**
>
> - Only test applications you own or have explicit written permission to test
> - Respect all applicable laws and regulations in your jurisdiction
> - Never use these techniques for piracy, unauthorized access, or malicious purposes
> - The authors are not responsible for any misuse of this information
>
> **Legal Framework:** This work falls under authorized penetration testing, bug bounty, and security research activities. All demonstrations in this article were executed against the OWASP MASTG Hacking Playground (`sg.vp.owasp_mobile.omtg_android`), a purpose-built lab application — never against real user data or production systems.

---

## What is Beerus Framework?

Beerus Framework is an open-source Android application (`io.hakaisecurity.beerusframework`) distributed on GitHub at [hakaioffsec/beerus-android](https://github.com/hakaioffsec/beerus-android), with its release notes and research background published at [yokai.hakaisecurity.io](https://yokai.hakaisecurity.io/beerus-framework-a-new-mobile-framework-arises/).

The core idea is simple: mobile pentesters juggle five or six separate tools (frida-server, objection, drozer, mitmproxy, apktool, custom shell scripts). Beerus consolidates the highest-value subset into one Compose app, with a companion Magisk/KernelSU module (`beerusRootModule`) that wires the boot-time plumbing — starting the hidden Frida binary, promoting user certificates to the system trust store, opening ADB on TCP, and persisting proxy settings.

Key design decisions that distinguish it:

- **No Magisk dependency.** The module works on any KernelSU-compatible root solution (KernelSU, ReSukiSU, KernelSU Next). The app talks to the module through a small SQLite bridge (`dbAgent`) rather than calling `su` on every action.
- **Frida Core embedded.** A 61 MB ELF (`fridaCore`) ships inside the module. It starts as a root process named `hiddenBin` and listens on `127.0.0.1:27042` — no PC-side `frida-server` upload needed.
- **Everything from the phone.** Script selection, injection, memory dumps, sandbox pulls, and property changes all happen on-device. You never need a laptop tethered over USB.

---

## The Feature Set — What You Actually Get

The following table summarizes every feature I verified on device, mapped to the mobile testing phase it supports:

| Feature | What it does | Testing phase | Evidence |
|---|---|---|---|
| Frida Server Setup | Starts/stops the embedded `hiddenBin` Frida Core on `127.0.0.1:27042` | Runtime instrumentation | `frida-ps -U` from host returns device processes |
| Frida Auto-Inject | Launches a target app via `launchIntent`, waits 5 s, attaches Frida Core with a selected JS script | Runtime instrumentation / root-detection bypass | Live demo below — script executed inside MASTG app |
| Frida Versions / Scripts | Manage multiple Frida Core builds and a library of JS scripts from the phone | Tooling management | `screenshots/01-home-beerus.png`, `screenshots/02-drawer.png` |
| Sandbox Exfiltration | Root-level copy of `/data/data/<pkg>` to a visible output folder | Data storage assessment (MASVS-STORAGE) | Exfiltrates app-private files after root |
| Memory Dump | Full process memory dump of a selected app | Sensitive data detection (MASVS-STORAGE / CRYPTO) | Artifact written to device storage |
| Manifest | Decodes and displays any installed APK's `AndroidManifest.xml` | Attack-surface mapping (MASVS-PLATFORM) | Decoded components, permissions, exported flags |
| ADB over Network | Sets `service.adb.tcp.port=5555` and restarts adbd at boot via the module | Remote control without USB | Verified `adb connect <ip>:5555` after toggle |
| Proxy Profiles | Save and apply HTTP proxy profiles (host:port) with one tap | Traffic interception (MASVS-NETWORK) | Profile persisted in module `status` file |
| Root Manager | Install/remove/enable KernelSU modules directly inside Beerus | Environment setup | `beerusRootModule` v0.1 by Hakai listed and active |
| Properties | Read and write system properties (`resetprop`) | Environment fingerprinting / bypass | Property set + `getprop` confirmation |
| Boot Options | Toggle Frida-on-startup, ADB-over-network, and System Trusted Certs at boot | Persistence (assessment environment) | Flags written to module `status`, applied at boot |
| System Trusted Certs | Mount-bind promotes user CAs into `/system/etc/security/cacerts` and the Conscrypt APEX store | SSL pinning interception prep | Module `post-fs-data.sh` performs the bind-mount |
| Update | In-app update channel for the framework itself | Maintenance | Version surfaced in app settings |

Two details worth highlighting for practitioners:

1. **System Trusted Certs is mount-only, not file-copy.** The module bind-mounts the CA directory at `post-fs-data`, so the system partition on disk stays clean. A plain reboot reverts everything — a nice property for a shared assessment device.

2. **Proxy Saving persists across reboots.** The proxy host:port lives in the module's `status` file and is re-applied by `service.sh`, so you don't reconfigure Burp every time the phone restarts.

---

## Installing Without Magisk: Beerus on ReSukiSU

**Credit where it is due:** the root layer on this device is **ReSukiSU**, a KernelSU fork maintained by the **ReSukiSU team** (part of the broader LRFP stack — LSPosed, ReSukiSU, FuseFixer, Play Integrity Fix — originally assembled by the LRFP-Team). ReSukiSU v4.2.0-rc1 runs as a loadable kernel module (LKM) on Android 17 and provides KernelSU-compatible userspace (`ksud`) with a persistent app-allowlist mechanism. Without that allowlist system, Beerus could not have been granted `su` without ever opening the ReSukiSU manager app. My thanks to the ReSukiSU team for maintaining the fork that makes this workflow possible — their allowlist, documented in my earlier note cambio-allowlist-resukisu, is the only touchpoint needed.

The installation path I used:

1. **Grant `su` via the kernel allowlist** (no manager UI):
   - `/data/adb/ksu/.allowlist` gets an entry for `io.hakaisecurity.beerusframework` (uid 10317, `allow=1`, context `u:r:ksu:s0`), written directly and verified by md5.
   - Backup of the original allowlist kept at `/data/local/tmp/.allowlist.bak` for rollback.

2. **Install the Beerus Root Module** through Beerus's own Root Manager screen:
   - Tap **Install Module**, pick `beerusRootModule.zip` (4.1 KB) in the Android file picker.
   - The module lands in `/data/adb/modules/beerusRootModule/` with `module.prop` (v0.1 by Hakai), `post-fs-data.sh`, `service.sh`, `system/bin/fridaCore` (61 MB ELF), and `system/bin/dbAgent`.

3. **Install the Beerus app** (`Beerus-Framework-V1.1.0.apk`, SHA-256 `569cdef7197731ddbc35d76ea1d4ef8eb345a7b131c4b09f496bdb26d9ec53a8`).

4. **Start Frida Core** from the Frida Setup screen — `hiddenBin` appears in `ps`, listening on `27042`, and `frida-ps -U` works from the host immediately.

![Frida Core running on device — frida-ps -U from the host](/Medium-personal-post/assets/images/beerus-frida-running.png)

No Magisk app, no `magisk` binary, no patched boot image. The ReSukiSU manager (`com.resukisu.resukisu`) was never opened during this process.

---

## Hands-On: Frida Auto-Inject Against an OWASP MASTG Playground App

For the proof of concept I used the **OWASP MASTG Hacking Playground** (`sg.vp.owasp_mobile.omtg_android`), downloaded from the [official OWASP releases](https://github.com/OWASP/MASTG-Hacking-Playground/releases) — a lab app designed for exactly this kind of demonstration, with no real user data involved.

The workflow, entirely from the phone:

1. Open **Frida Scripts** in Beerus, import a JS script (a benign hook that logs the hooked class name at runtime).
2. Select the MASTG app as the injection target.
3. Tap **Run** — Beerus launches the app via `launchIntent`, waits 5 seconds for the process to stabilize, then attaches Frida Core.
4. The script executes inside the target process; hooked method calls are visible in the on-device console output.

This is the feature that makes Beerus worth installing: **script injection with zero USB tethering and zero PC-side frida-server management.** On a bus, in a client office, or during a mobile-only engagement, the entire instrumented workflow fits in your pocket.

Supporting evidence captured on device:

- `screenshots/06-frida-running.png` — `frida-ps -U` from the host against the on-device `hiddenBin` server
- `screenshots/05-root-manager-unlocked.png` — Root Manager with the Beerus module installed and active
- `screenshots/04-home-after-grant.png` — Beerus home after the ReSukiSU allowlist grant

![Root Manager with the Beerus module installed and active](/Medium-personal-post/assets/images/beerus-root-manager.png)

- Shell verification: `hiddenBin` PID on `127.0.0.1:27042`, `service.adb.tcp.port` toggled, module `status` flags round-tripped

---

## Honest Assessment — Pros and Cons

### Pros

- **Magisk-free.** Works on KernelSU/ReSukiSU/KSU-Next; the app never touches the Magisk manager.
- **Single-file Frida.** 61 MB embedded Core, started as a renamed root process (`hiddenBin`), no upload step.
- **Boot persistence is optional and clean.** All boot-time behavior lives in one module `status` file; disabling a flag and rebooting fully reverts it. Cert promotion is mount-bind only.
- **Everything on-device.** Script import, injection, memory dumps, sandbox pulls, manifest decoding — no laptop required.
- **Small, auditable module.** `post-fs-data.sh` + `service.sh` are ~1.9 KB each; you can read the entire privilege surface in five minutes.
- **In-app module manager.** Install, remove, and toggle KernelSU modules without leaving the framework.

### Cons

- **Android only.** No iOS counterpart; iOS engagements still need a separate toolchain (jailbreak or sideloaded Frida).
- **Root required, full stop.** No non-root fallback — by design, but it excludes unrooted test devices and stock lab phones.
- **No Linux/macOS/Windows client.** The framework lives on the phone; PC-side workflows (objection, Ghidra bridges, custom scripting) are out of scope.
- **Frida Core is a 61 MB blob.** You trust the build; pin and verify the SHA-256 yourself before deploying.
- **Stealth is not its goal.** `hiddenBin` is a rename, not an anti-detection layer. Against hardened RASP (Play Integrity, Magisk-detect libraries), combine it with the rest of the LRFP stack — Beerus is a *toolkit*, not an evasion product.
- **Early-stage project.** v0.1 module / V1.1.0 app; expect rough edges in edge cases (e.g., multi-user profiles, work profiles).

---

## Verdict

Beerus Framework solves a real friction point: **running a modern mobile assessment stack on a KernelSU-family device without installing Magisk.** On my Pixel 8 Pro with ReSukiSU, the full loop — allowlist grant, module install, Frida Core start, script auto-inject into an OWASP MASTG lab app — completed from the phone alone, with every boot-time feature cleanly revertible.

If your assessment device runs ReSukiSU or any KernelSU fork and you want Frida, cert promotion, ADB-over-network, proxy persistence, and sandbox/memory tooling in one place, Beerus is currently the most coherent option I have tested. Pair it with the ReSukiSU allowlist workflow documented in cambio-allowlist-resukisu and you have a complete no-Magisk mobile pentest environment.

**Recommended for:** authorized mobile pentesters and bug bounty hunters on rooted Android devices who want a pocket-sized, Magisk-free assessment stack.

---

## References

- **Beerus Framework repository:** [github.com/hakaioffsec/beerus-android](https://github.com/hakaioffsec/beerus-android)
- **Hakai research article:** [yokai.hakaisecurity.io — Beerus Framework: A New Mobile Framework Arises](https://yokai.hakaisecurity.io/beerus-framework-a-new-mobile-framework-arises/)
- **OWASP MASTG Hacking Playground:** [github.com/OWASP/MASTG-Hacking-Playground](https://github.com/OWASP/MASTG-Hacking-Playground)
- **ReSukiSU (KernelSU fork):** maintained by the ReSukiSU team; allowlist workflow documented in cambio-allowlist-resukisu
- **LRFP stack (LSPosed + ReSukiSU + FuseFixer + Play Integrity Fix):** see my earlier post *Building an Invisible Penetration Testing Device: LRFP Stack on Pixel 8 Pro (Android 17)*
- **OWASP MASVS / MASTG:** [owasp.org/www-project-mobile-application-security/](https://owasp.org/www-project-mobile-application-security/)

*Prices: n/a. Disclaimer: Beerus Framework is free and open source; this article is not sponsored by the Hakai team or the ReSukiSU team. All testing was performed on owned devices and authorized lab applications.*

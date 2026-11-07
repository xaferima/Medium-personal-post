---
layout: post
title: "Beerus Framework: Frida, Sandbox Dumps, ADB and Proxies — The Whole Stack for Android Hacking From Your Phone"
date: 2026-11-06
description: "How I installed and validated Beerus Framework on a rooted Pixel 8 Pro with ReSukiSU — Frida, sandbox dumps, ADB and proxies, the whole stack from your phone, no Magisk required."
categories: [security, mobile, frida]
image: /Medium-personal-post/assets/images/cover-beerus-framework-pixel8pro-2026.jpg
---

# Beerus Framework: Frida, Sandbox Dumps, ADB and Proxies — The Whole Stack for Android Hacking From Your Phone

**How I installed and validated Beerus Framework on a rooted Pixel 8 Pro with ReSukiSU — and why it matters for authorized mobile assessments**

![](/Medium-personal-post/assets/images/cover-beerus-framework-pixel8pro-2026.jpg)

Offensive Security Researcher

---

## Table of Contents

1. [Introduction](#introduction)
2. [Disclaimer](#disclaimer)
3. [What is Beerus Framework?](#what-is-beerus-framework)
4. [Requirements & Quick Start](#requirements--quick-start)
5. [The Feature Set — What You Actually Get](#the-feature-set--what-you-actually-get)
6. [Installing Without Magisk: Beerus on ReSukiSU](#installing-without-magisk-beerus-on-resukisu)
7. [Hands-On: Frida Auto-Inject Against an OWASP MASTG Playground App](#hands-on-frida-auto-inject-against-an-owasp-mastg-playground-app)
8. [Beerus vs. the Manual Stack](#beerus-vs-the-manual-stack)
9. [Honest Assessment — Pros and Cons](#honest-assessment--pros-and-cons)
10. [Verdict](#verdict)
11. [Evidence](#evidence)
12. [References](#references)

---

## Introduction

Beerus Framework replaces five or six mobile pentest tools with one app — and it doesn't care whether your root comes from **Magisk**, **KernelSU**, or a KernelSU fork like **ReSukiSU**. I deployed it on a Pixel 8 Pro running Android 17 with ReSukiSU v4.2.0-rc1, without ever opening the ReSukiSU manager, and ran an entire assessment loop from the phone: Frida Core as a renamed root process, sandbox pulls, memory dumps, decoded manifests, ADB over the network, and proxy profiles that survive reboots.

This post covers what the framework actually does, how I installed it through the ReSukiSU kernel allowlist, the hands-on proof against an OWASP MASTG lab app, and an honest list of where it still falls short.

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

The core idea is simple: mobile pentesters juggle five or six separate tools (frida-server, objection, drozer, mitmproxy, apktool, custom shell scripts). Beerus consolidates the highest-value subset into one Compose app, with a companion root module (`beerusRootModule`) that wires the boot-time plumbing — starting the hidden Frida binary, promoting user certificates to the system trust store, opening ADB on TCP, and persisting proxy settings.

Key design decisions that distinguish it:

- **Runs on Magisk, KernelSU, and KernelSU forks.** The companion module is a standard Magisk-format zip that KernelSU and its forks also install — I run it on ReSukiSU, and it works the same on plain Magisk or KernelSU Next. The app talks to the module through a small SQLite bridge (`dbAgent`) rather than calling `su` on every action.
- **Frida Core embedded.** A 61 MB ELF (`fridaCore`) ships inside the module. It starts as a root process named `hiddenBin` and listens on `127.0.0.1:27042` — no PC-side `frida-server` upload needed.
- **Everything from the phone.** Script selection, injection, memory dumps, sandbox pulls, and property changes all happen on-device. You never need a laptop tethered over USB.

---

## Requirements & Quick Start

**Requirements:**

- Rooted Android device — Magisk, KernelSU, or a KernelSU fork (ReSukiSU, KernelSU Next)
- Beerus app `Beerus-Framework-V1.1.0.apk` — SHA-256 `569cdef7197731ddbc35d76ea1d4ef8eb345a7b131c4b09f496bdb26d9ec53a8`
- `beerusRootModule.zip` (4.1 KB) from the [GitHub releases](https://github.com/hakaioffsec/beerus-android/releases)

**Quick Start — from zero to instrumented in about five minutes:**

1. Install the APK (sideload or `adb install`).
2. Open Beerus → **Root Manager** → **Install Module** → pick `beerusRootModule.zip`.
3. Grant `su` to `io.hakaisecurity.beerusframework` — on ReSukiSU that is a kernel-allowlist entry (see the install section below); on Magisk/KernelSU, approve the prompt.
4. **Frida Setup** → Start — the embedded Core listens on `127.0.0.1:27042`.

Verify from the host over USB:

```bash
adb shell su -c 'ps -A | grep hiddenBin'                              # Frida process alive
adb shell su -c 'cat /data/adb/modules/beerusRootModule/status'       # feature flags
frida-ps -U                                                           # device processes visible
```

---

## The Feature Set — What You Actually Get

The following table summarizes every feature I verified on device, mapped to the mobile testing phase it supports:

| Feature | What it does | Testing phase |
|---|---|---|
| Frida Server Setup | Starts/stops the embedded `hiddenBin` Frida Core on `127.0.0.1:27042` | Runtime instrumentation |
| Frida Auto-Inject | Launches a target app via `launchIntent`, waits 5 s, attaches Frida Core with a selected JS script | Runtime instrumentation / root-detection bypass |
| Frida Versions / Scripts | Manage multiple Frida Core builds and a library of JS scripts from the phone | Tooling management |
| Sandbox Exfiltration | Root-level copy of `/data/data/<pkg>` to a visible output folder | Data storage assessment (MASVS-STORAGE) |
| Memory Dump | Full process memory dump of a selected app | Sensitive data detection (MASVS-STORAGE / CRYPTO) |
| Manifest | Decodes and displays any installed APK's `AndroidManifest.xml` | Attack-surface mapping (MASVS-PLATFORM) |
| ADB over Network | Sets `service.adb.tcp.port=5555` and restarts adbd at boot via the module | Remote control without USB |
| Proxy Profiles | Save and apply HTTP proxy profiles (host:port) with one tap | Traffic interception (MASVS-NETWORK) |
| Root Manager | Install/remove/enable root modules directly inside Beerus | Environment setup |
| Properties | Read and write system properties (`resetprop`) | Environment fingerprinting / bypass |
| Boot Options | Toggle Frida-on-startup, ADB-over-network, and System Trusted Certs at boot | Persistence (assessment environment) |
| System Trusted Certs | Mount-bind promotes user CAs into `/system/etc/security/cacerts` and the Conscrypt APEX store | SSL pinning interception prep |
| Update | In-app update channel for the framework itself | Maintenance |

Screenshot evidence for every row is in the [Evidence](#evidence) section at the end of this post.

Two details worth highlighting for practitioners:

1. **System Trusted Certs is mount-only, not file-copy.** The module bind-mounts the CA directory at `post-fs-data`, so the system partition on disk stays clean. A plain reboot reverts everything — a nice property for a shared assessment device.

2. **Proxy Saving persists across reboots.** The proxy host:port lives in the module's `status` file and is re-applied by `service.sh`, so you don't reconfigure Burp every time the phone restarts.

---

## Installing Without Magisk: Beerus on ReSukiSU

**Credit where it is due:** the root layer on this device is **ReSukiSU**, a KernelSU fork maintained by the **ReSukiSU team**. ReSukiSU v4.2.0-rc1 runs as a loadable kernel module (LKM) on Android 17 and provides KernelSU-compatible userspace (`ksud`) with a persistent app-allowlist mechanism. Without that allowlist system, Beerus could not have been granted `su` without ever opening the ReSukiSU manager app — my thanks to the ReSukiSU team for making this workflow possible. The allowlist change itself is documented in my earlier note on the ReSukiSU allowlist.

The installation path I used:

1. **Grant `su` via the kernel allowlist** (no manager UI):
   - `/data/adb/ksu/.allowlist` gets an entry for `io.hakaisecurity.beerusframework` (uid 10317, `allow=1`, context `u:r:ksu:s0`), written directly and verified by md5.
   - Backup of the original allowlist kept at `/data/local/tmp/.allowlist.bak` for rollback.

2. **Install the Beerus Root Module** through Beerus's own Root Manager screen:
   - Tap **Install Module**, pick `beerusRootModule.zip` (4.1 KB) in the Android file picker.
   - The module lands in `/data/adb/modules/beerusRootModule/` with `module.prop` (v0.1 by Hakai), `post-fs-data.sh`, `service.sh`, `system/bin/fridaCore` (61 MB ELF), and `system/bin/dbAgent`.

3. **Install the Beerus app** (`Beerus-Framework-V1.1.0.apk`, SHA-256 `569cdef7197731ddbc35d76ea1d4ef8eb345a7b131c4b09f496bdb26d9ec53a8`).

4. **Start Frida Core** from the Frida Setup screen — `hiddenBin` appears in `ps`, listening on `27042`, and `frida-ps -U` works from the host immediately.

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

> [!tip] **Shell verification** — `hiddenBin` PID on `127.0.0.1:27042`, `service.adb.tcp.port` toggled, module `status` flags round-tripped.

---

## Beerus vs. the Manual Stack

For years the default workflow was a bag of PC-side tools pointed at a USB cable. Here is what changes:

| Task | Manual stack | Beerus |
|---|---|---|
| Run Frida | Push `frida-server`, `chmod`, launch as root, connect over USB | Embedded Core, started from the UI, on-device |
| Trust user certs | Copy into `/system` + reboot (often needs remount) | Bind-mount at `post-fs-data`; disk stays clean |
| ADB over network | `setprop` + stop/start `adbd` by hand | One Boot Options toggle, persisted in `status` |
| Sandbox pull / memory dump | `run-as`/root `cp`, or frida-dump scripts | Two taps from the phone |
| Manifest decoding | apktool/jadx on a laptop | Manifest view on-device |
| Proxy config | Per-tool manual settings, lost on reboot | Proxy Profiles persisted across reboots |
| Setup per device | ~20–40 minutes | ~5 minutes |

What you give up: full scripting flexibility and PC-side automation. Beerus covers the 80% you repeat on every engagement — for the rest you still want a laptop, a terminal, and your own scripts.

---

## Honest Assessment — Pros and Cons

### Pros

- **Runs on Magisk and KernelSU alike.** One module, three root ecosystems (Magisk, KernelSU, ReSukiSU/forks); the app never touches the Magisk manager when you don't want it to.
- **Single-file Frida.** 61 MB embedded Core, started as a renamed root process (`hiddenBin`), no upload step.
- **Boot persistence is optional and clean.** All boot-time behavior lives in one module `status` file; disabling a flag and rebooting fully reverts it. Cert promotion is mount-bind only.
- **Everything on-device.** Script import, injection, memory dumps, sandbox pulls, manifest decoding — no laptop required.
- **Small, auditable module.** `post-fs-data.sh` + `service.sh` are ~1.9 KB each; you can read the entire privilege surface in five minutes.
- **In-app module manager.** Install, remove, and toggle root modules without leaving the framework.

### Cons

- **Android only.** No iOS counterpart; iOS engagements still need a separate toolchain (jailbreak or sideloaded Frida).
- **Root required, full stop.** No non-root fallback — by design, but it excludes unrooted test devices and stock lab phones.
- **No Linux/macOS/Windows client.** The framework lives on the phone; PC-side workflows (objection, Ghidra bridges, custom scripting) are out of scope.
- **Frida Core is a 61 MB blob.** You trust the build; pin and verify the SHA-256 yourself before deploying.
- **Stealth is not its goal.** `hiddenBin` is a rename, not an anti-detection layer. Against hardened RASP (Play Integrity, Magisk-detect libraries), combine it with the rest of your bypass stack — Beerus is a *toolkit*, not an evasion product.
- **Early-stage project.** v0.1 module / V1.1.0 app; expect rough edges in edge cases (e.g., multi-user profiles, work profiles).

---

## Verdict

Beerus Framework solves a real friction point: **running a modern mobile assessment stack from the phone alone, whatever your root provider is.** On my Pixel 8 Pro with ReSukiSU, the full loop — allowlist grant, module install, Frida Core start, script auto-inject into an OWASP MASTG lab app — completed from the device, with every boot-time feature cleanly revertible.

If your assessment device runs Magisk, KernelSU, or a KernelSU fork like ReSukiSU, and you want Frida, cert promotion, ADB-over-network, proxy persistence, and sandbox/memory tooling in one place, Beerus is currently the most coherent option I have tested.

**Recommended for:** authorized mobile pentesters and bug bounty hunters on rooted Android devices who want a pocket-sized assessment stack.

---

## Evidence

Screenshots captured on device during this deployment:

**1. Drawer navigation** — the full feature menu, from Frida Setup to Boot Options:

![](/Medium-personal-post/assets/images/beerus-02-drawer.jpg)

**2. Frida Setup** — embedded Frida Core v17.26.1, Status: Running:

![](/Medium-personal-post/assets/images/beerus-07-frida-setup.jpg)

**3. Frida Scripts** — script library and Add Script, all on-device:

![](/Medium-personal-post/assets/images/beerus-08-frida-scripts.jpg)

**4. Sandbox Exf/** — `/data/data` target list, including the MASTG lab app ("Attack me if u can"):

![](/Medium-personal-post/assets/images/beerus-09-sandbox-exf.jpg)

**5. ADB O/ Network** — network ADB toggle:

![](/Medium-personal-post/assets/images/beerus-12-adb-network.jpg)

**6. Boot Options** — Trusted System Certificates, ADB Over Network, and Frida boot flags:

![](/Medium-personal-post/assets/images/beerus-15-boot-options.jpg)

**7. Root Manager** — the Beerus module v0.1 by Hakai installed and active:

![](/Medium-personal-post/assets/images/beerus-05-root-manager-unlocked.jpg)

**8. Frida running** — `frida-ps -U` from the host against the on-device `hiddenBin` server:

![](/Medium-personal-post/assets/images/beerus-06-frida-running.jpg)

---

## References

- **Beerus Framework repository:** [github.com/hakaioffsec/beerus-android](https://github.com/hakaioffsec/beerus-android)
- **Hakai research article:** [yokai.hakaisecurity.io — Beerus Framework: A New Mobile Framework Arises](https://yokai.hakaisecurity.io/beerus-framework-a-new-mobile-framework-arises/)
- **OWASP MASTG Hacking Playground:** [github.com/OWASP/MASTG-Hacking-Playground](https://github.com/OWASP/MASTG-Hacking-Playground)
- **ReSukiSU (KernelSU fork):** maintained by the ReSukiSU team; allowlist workflow documented in cambio-allowlist-resukisu
- **OWASP MASVS / MASTG:** [owasp.org/www-project-mobile-application-security/](https://owasp.org/www-project-mobile-application-security/)

*Prices: n/a. Disclaimer: Beerus Framework is free and open source; this article is not sponsored by the Hakai team or the ReSukiSU team. All testing was performed on owned devices and authorized lab applications.*

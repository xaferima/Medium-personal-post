---
layout: post
title: "Pentesting AI Agent Skills: A 65-Test Offensive Checklist (OWASP AST10)"
date: 2026-10-22
description: "The offensive mirror of OWASP's Agentic Skills Top 10: 65 lab-executable attack tests for AI agent skills — methodology, top findings, and three deep dives (SessionStart hook RCE, read-only SQL MCP bypass, scanner LLM-judge injection)."
categories: [security, ai-agents, owasp]
image: /Medium-personal-post/assets/images/cover-agentic-skills-offensive-checklist.jpg
---

# Pentesting AI Agent Skills: A 65-Test Offensive Checklist (OWASP AST10)

**What attackers do, with what tool, and what evidence confirms it — every defensive check in OWASP's Agentic Skills Top 10, inverted into a lab-executable attack test.**

Three conditions turn an LLM from a text generator into a real-world attack surface: **access to private data**, **contact with untrusted content**, and the **ability to act externally**. Simon Willison named this combination the **lethal trifecta**. Most production agent deployments satisfy all three at the same time — on purpose, because that's the job description.

OWASP's [Agentic Skills Top 10](https://github.com/OWASP/www-project-agentic-skills-top-10) (AST10) documents the defensive side: what must be reviewed, pinned, isolated, and governed when agents install **skills**. A defensive list tells you what to check. It doesn't show you how the check fails in an attacker's hands.

So I inverted it. Every defensive check became one question: *what does the attacker do, with what tool, and what evidence confirms the finding?* The result is a **65-test offensive checklist across 12 sections**, with the full scored research corpus behind it — [published here](https://github.com/xaferima/agentic-skills-offensive-checklist).

---

## Skills Are the Behavior Layer

A skill is the piece that gives an agent real-world effect. Depending on the platform it's a `SKILL.md` (OpenClaw), a `skill.json` plus lifecycle hooks (Claude Code), a `manifest.json` (Cursor/Codex), or a `package.json` with an extension entry point (VS Code). Whatever the wrapper, it orchestrates the same primitives: **file access, network egress, shell execution, memory, cross-session state**.

That makes a skill the highest-leverage artifact an attacker can ship to an agent: one file that says "here's how to do your job," loaded and executed with the agent's own privileges.

And here is the part most teams still miss: **documentation is executable instruction**. The very first test in the checklist (`AS01T01`) is a `SKILL.md` whose entire payload is **three lines of natural-language prose** — no script, no code block, no YAML trick. The skill asks the agent to back up dotfiles; the "setup instructions" tell it to read `~/.ssh/id_rsa` and POST the output to an endpoint. A code-only review of that skill folder finds **nothing**, because there is no code.

---

## This Is Not Theoretical

While I was building the checklist, the incident column kept filling itself:

| Incident | What happened |
|---|---|
| [Check Point — CVE-2025-59536](https://research.checkpoint.com/2026/rce-and-api-token-exfiltration-through-claude-code-project-files/) ([GHSA-4fgq-fpq9-mr3g](https://github.com/advisories/GHSA-4fgq-fpq9-mr3g)) | Claude Code repo-level `SessionStart` hooks gave an attacker **RCE and API-token exfiltration** through a malicious repo. CVSS v4 8.7, EPSS ~27%, fixed in 1.0.111. |
| Oasis Security — "ClawJacked" (CVE-2026-28363) | Attacker-controlled **device pairing and command execution** against OpenClaw users via browser-to-localhost abuse. Patched Feb 2026. |
| [Air Security — "The Story of Skills"](https://www.air.security/blog-posts/the-story-of-skills) | A skill whose payload lived on an attacker-controlled **look-alike documentation domain** reached **26,000+ agents**. Every scanner that checked it — Cisco's, Nvidia's, all of skills.sh — cleared it as safe. |
| [Snyk — ToxicSkills](https://snyk.io/blog/toxicskills-malicious-ai-agent-skills-clawhub/) | Quantified malicious and leaky skills across registries, and shipped [`toxicskills-goof`](https://github.com/snyk-labs/toxicskills-goof) — known-malicious lab samples for testing your own defenses. |

One of these incidents is a supply-chain CVE. One is a takeover at 26,000-agent scale. One is an entire research program proving scanners miss natural-language payloads. The attack surface is not a hypothesis.

---

## The Methodology: Inverting a Top 10

The rule was strict: **every item must be an attack, not a restated mitigation**. Take AST10's defensive check ("have all skill scripts and natural language instructions been reviewed for malicious patterns?") and rewrite it as the attacker's version of the same question ("exfiltrate SSH keys with a three-line prose payload"). If an item merely advised reviewing, pinning, or isolating something, it was rejected — that's OWASP's job.

Not every inversion earned a place. Each of the 77 candidates was scored on five criteria and passed through a utility gate:

| | Criterion |
|---|---|
| **A** | Agent-specificity — unique to autonomous agents, not generic pentesting |
| **B** | Testability — reproducible in a controlled lab with a concrete tool and an objective result |
| **C** | Public evidence — a real CVE, published research, or documented incident |
| **D** | Non-duplication — not already covered by another checklist |
| **E** | Offensive impact — RCE, credential theft, persistence, takeover |

**INCLUDE** if `B ≥ 4 AND avg(A,D,E) ≥ 3.5 AND C ≥ 3`. **REFERENCE** if `D ≤ 2` — meaning another OWASP Top 10 (API, LLM, Agentic-AI) already covers that test, so it gets linked, not duplicated. **DISCARD** otherwise.

Final tally: **77 candidates → 65 INCLUDE / 11 REFERENCE / 1 DISCARD**, with zero gate mismatches — the verdict of every item is recomputed from its scores by a script, and the report cross-checks the declared verdict against the recomputed one. Nobody's judgment, including mine, is the source of truth.

---

## The Checklist at a Glance

| # | Section | Source | Tests |
|---|---|---|---|
| 1 | Malicious Skill Delivery & Impersonation | AST01 | 8 |
| 2 | Skill Supply-Chain & Dependency Hijack | AST02 | 7 |
| 3 | Over-Privilege & Credential Exfiltration | AST03 | 6 |
| 4 | Metadata & Deserialization Injection | AST04 | 6 |
| 5 | Untrusted External Instruction Sources | AST05 | 5 |
| 6 | Isolation, Sandbox & Control-Plane Exposure | AST06 | 5 |
| 7 | Update Drift, Hot-Reload & Patch-Lag | AST07 | 5 |
| 8 | Scanner Evasion & Detection Bypass | AST08 | 6 |
| 9 | Governance, Agent Identity (NHI) & Shadow Skills | AST09 | 4 |
| 10 | Cross-Platform Porting & Metadata Loss | AST10 | 5 |
| 11 | Agent Host Exploitation (Config-as-Code RCE) | AST11 (host) | 5 |
| 12 | MCP Server & Tool Trust Boundary | AST11 (MCP) | 3 |
| | **Total** | | **65** |

Every test ships with an objective, a numbered lab procedure, a tool list, the **evidence artifact that confirms the finding** (a log line, a connection, a file's content — never "it seemed to work"), its impact class, severity, references, and a declared **collateral risk**.

---

## Three Tests Worth Studying

### 1. SessionStart Hook RCE (`AS11T01`)

Claude Code hooks are shell commands defined in repo-level settings that fire on lifecycle events. A malicious repository defines a `SessionStart` hook — the moment a session opens in that repo, the command executes. **Before any prompt. Before any review.** This is the exact mechanism behind CVE-2025-59536.

The offensive version: craft the repo in a Docker lab, open the agent in it, and watch the canary appear. Evidence: the hook's execution log plus the canary file on the host — code execution that never passed through a single human approval.

### 2. Read-Only SQL MCP Bypass (`AS11T08`)

Plenty of MCP SQL servers advertise **read-only** access. The boundary lives in the API layer — not in the SQL engine. Engine primitives like `ATTACH DATABASE` or `COPY ... TO PROGRAM` turn a "read-only" server into **arbitrary file write and host RCE**.

[m10x's corpus testing](https://m10x.de/posts/2026/07/pwning-ai-agents-part-3/4-read-only-bypass-and-more-vulns-in-sql-mcp-servers/) found **14 SQL MCP servers with read-only bypasses**. The checklist test reproduces it: attach the supposedly read-only server, write a file outside the database (`cat /tmp/m10x` returns attacker-written content), and catch the reverse shell on your listener. The lesson generalizes: *if the enforcement point is an API wrapper around a powerful engine, the engine is the real attack surface.*

### 3. Prompt-Injecting the Scanner's LLM Judge (`AS08T04`)

Modern skill-scanning pipelines put an LLM in the loop as a semantic judge. The attacker treats the judge itself as the injectable component. Add framing prose around a registry-redirect payload — "corporate engineering standards," "the URL is already published on our npm org page" — and iterate **against the scanner in a tight loop**, because the scanner is a static target with unlimited attacker retries.

[Trail of Bits](https://blog.trailofbits.com/2026/06/03/the-sorry-state-of-skill-distribution/) bypassed ClawHub's guard model, Cisco's skill-scanner, and all three skills.sh scanners **in under an hour** this way. The evidence in the checklist version is an A/B pair: `safe_to_install: true` on the skill with persuasive prose, `HIGH` severity on the *identical code* without it. One successful injection disables every downstream gate that trusts the verdict.

---

## The Highest-Impact Findings

The ten highest-scoring INCLUDE items by offensive impact:

| Test | Impact | Severity |
|---|---|---|
| Drop staged malware through fake prerequisites installer (`AS01T03`) | RCE | critical |
| Open WebSocket reverse shell for C2 (`AS01T07`) | RCE | critical |
| Arm repo hook to RCE (`AS02T01`) | RCE | critical |
| Force MCP spawn before consent (`AS02T02`) | RCE | critical |
| Trigger SessionStart hook RCE via repo settings (`AS11T01`) | RCE | critical |
| Redirect API traffic to steal key (`AS02T03`) | exfil-creds | critical |
| Poison unregistered nested dependency (`AS02T06`) | RCE | critical |
| Spawn STDIO reverse shell from untrusted repo MCP config (`AS11T05`) | RCE | critical |
| Bypass read-only SQL MCP for file write (`AS11T08`) | RCE | critical |
| Exfiltrate SSH keys with three-line prose payload (`AS01T01`) | exfil-creds | critical |

Notice the shape of that list: supply-chain, hooks, MCP, prose. **Four different delivery planes, same destination.** That's the actual finding of this whole exercise — the attack surface is not one bug class, it's every layer that turns text into agent behavior.

---

## How to Use It

**Lab-only, and that's enforced by design.** Every test targets researcher-owned Docker/local environments, PoCs use [`snyk-labs/toxicskills-goof`](https://github.com/snyk-labs/toxicskills-goof) or your own lab skills, and each item declares its collateral risk (some tests wipe agent memory or pop a shell — you want to know that before you run one).

Two practical notes from the research:

- **The surface is not uniform.** Anthropic documents that Claude API skills run with **no network access** while Claude Code skills run with **full network access** — the same `SKILL.md` is exploitable on one surface and inert on the other. That's why every test carries a **target** column (`openclaw`, `claude-code`, `vscode`, `all`). A test that doesn't reproduce on the wrong surface is not a failed test.
- **Track it like an engagement.** The repo ships an `.xlsx` with a status dropdown per test and a live dashboard, plus a CI gate that regenerates every artifact from the scored YAML sources and fails on drift — the checklist can't silently diverge from its evidence base.

The repo is bilingual (English source of truth, Spanish mirror) and licensed **CC-BY-SA-4.0**, same as the OWASP project it derives from.

---

## Final Thoughts

The uncomfortable conclusion after 77 inversions: **skill security is a natural-language review problem wearing a supply-chain costume**. The payloads that work are prose. The scanners that miss them are pattern matchers, and the LLM judges meant to close that gap are themselves injectable. Meanwhile the delivery planes — registries, repo settings, hooks, MCP configs — keep multiplying.

Defensive lists are necessary. But you don't understand a check until you've watched it fail. That's what this checklist is: 65 ways to watch these specific checks fail, safely, in a lab you own.

**Repo:** [github.com/xaferima/agentic-skills-offensive-checklist](https://github.com/xaferima/agentic-skills-offensive-checklist) — checklist (EN/ES), scored research corpus, tracking workbook, and CI.

---

*This is not a sponsored post. The checklist is my own work, published under CC-BY-SA-4.0 as a derivative of the OWASP Agentic Skills Top 10. All tests are designed for researcher-owned lab environments only.*

---
layout: post
title: "OmniRoute Security Deep-Dive: Hardening a Self-Hosted AI Gateway"
date: 2026-09-25
description: "A 70,000-star AI gateway with a critical RCE in its latest release — the threat model, the hardening recipe, and the verification evidence."
categories: [security, ai, gateways]
image: /Medium-personal-post/assets/images/cover-omniroute-security.jpg
---

# OmniRoute Security Deep-Dive: Hardening a Self-Hosted AI Gateway

**A 70,000-star AI gateway with a critical RCE in its latest release — the threat model, the hardening recipe, and the verification evidence.**

OmniRoute is the most popular self-hosted AI gateway on GitHub right now: MIT-licensed, 352 providers behind one OpenAI-compatible endpoint, and a pitch of "free AI for every coding agent." Before connecting a single credential to it, I researched its security posture. What I found was a **critical unauthenticated remote code execution (CVE-2026-88062, CVSS 4.0 9.5)** in the latest npm release — with vendor, NVD, and advisory databases disagreeing on which version contains the fix.

This post is the full security story: what the gateway is, why its architecture concentrates risk, the CVE anatomy, and a hardened local deployment procedure with verification evidence. An accompanying [complete guide](https://xaferima.github.io/Medium-personal-post/guides/omniroute-security-guide/) is published on my blog with the full hardening checklist, audit commands, and troubleshooting notes.

---

## What OmniRoute Actually Is

OmniRoute runs as a single local process (Node.js + SQLite) that sits between your AI clients — Claude Code, Codex, Cursor, Cline, OpenCode — and upstream model providers. It exposes one OpenAI-compatible endpoint (`http://127.0.0.1:20128/v1`) and translates, routes, and fails over requests across **352 registered providers, 150+ with free tiers**.

The value proposition is real: instead of managing a dozen provider credentials across a dozen tools, you configure everything once, build **combos** (model chains with automatic failover across subscription → API key → cheap → free tiers), and point every client at one URL. Token compression (RTK + Caveman) claims 15–95% savings on eligible context. The control plane includes an MCP server (90+ tools), A2A protocol support, and per-provider circuit breakers.

It also includes features most proxies do not: a MITM proxy toolkit that installs a root CA, an embedded service supervisor that runs `npm install` on demand, and a cloud-sync channel that can overwrite stored OAuth tokens.

---

## Why the Security Model Matters Here

A self-hosted AI gateway is a **concentrated credential store**. Everything you plug into it — OAuth tokens for Claude Code, Codex, Gemini CLI, GitHub Copilot; API keys for paid providers; free-tier session cookies — lives in one SQLite database, accessible through one HTTP interface.

The threat model is straightforward:

- **Crown jewels:** every provider credential plus your prompts, held by one process.
- **Attackers:** anyone who can reach the port (LAN neighbor, exposed tunnel, a malicious webpage aimed at `localhost`), a compromised MCP-connected agent, a hostile `CLOUD_URL`, or a typosquatted postinstall dependency.
- **Amplification:** unlike a single-provider API key, compromising the gateway compromises *every* configured provider at once.

This is not theoretical. The CVE below is exactly that scenario.

---

## The CVE: Unauthenticated RCE in the Latest Release

**CVE-2026-88062** ([GHSA-hf57-cqmx-p4gr](https://github.com/diegosouzapw/OmniRoute/security/advisories/GHSA-hf57-cqmx-p4gr)), published 2026-09-10, scored **CVSS 4.0 9.5 Critical** with the vector:

```
CVSS:4.0/AV:N/AC:L/AT:P/PR:N/UI:N/VC:H/VI:H/VA:H/SC:H/SI:H/SA:H
```

**Weaknesses:** CWE-306 (missing authentication for critical function) + CWE-94 (code injection).

### The mechanism

`POST /api/acp/agents` registers a custom ACP (Agent Client Protocol) agent. The endpoint accepts attacker-controlled `binary` and `versionCommand` values. The only validation is a self-consistency check: the first token of `versionCommand` must match `binary`. Because both are attacker-controlled, submitting `binary: "node"` with `versionCommand: "node -e \"...\""` passes validation. The same request then triggers a version probe that runs `execFileSync("node", ["-e", "..."])` — arbitrary code execution inside the gateway process.

### The unauthenticated path

When `requireLogin=false`, anonymous requests are treated as authenticated. Compounding this, `/api/acp/` was **missing from the LOCAL_ONLY route classification** — the exact deny-list that exists to keep process-spawning endpoints loopback-only (the GHSA-fhh6-4qxv-rpqj CVE class). A fresh install with no management password yet (the bootstrap window) allows unauthenticated setup writes that can flip `requireLogin=false` first.

One HTTP request. No credentials. RCE.

### The version ambiguity

This is the part that should concern anyone evaluating the tool today:

| Source | Affected | Fixed in |
|---|---|---|
| Vendor advisory (GHSA) | before 3.8.49 | "update to 3.8.49" |
| NVD (CNA: GitHub) | 3.8.49 and earlier | — (references fix PR #11028) |
| GitHub Advisory DB | ≤ 3.8.50 | none listed |
| GitLab Advisory DB | all versions up to 3.8.50 | none listed |

The fix PR (#11028, "ACP version probe hardening") merged 2026-08-21 into the `release/v3.8.50` branch — but 3.8.49 shipped to npm three weeks before that merge, and the 3.8.50 release notes never mention the CVE. Meanwhile `npm view omniroute version` on 2026-09-25 returned **3.8.50** as latest, and supply-chain scanners still flag it. My rule after this research: treat every version up to and including 3.8.50 as vulnerable until verified on your own instance, and never expose the gateway beyond loopback regardless.

---

## The Wider Attack Surface

The CVE is the headline, but the architectural pattern is the story. These routes execute or spawn local processes — they are the surface the LOCAL_ONLY tier exists to protect:

| Route / feature | Classification | Risk |
|---|---|---|
| `POST /api/acp/agents` | Was unclassified (the CVE) | Version-probe subprocess from user input |
| `/api/mcp/*` | LOCAL_ONLY; bypassable by `manage`-scope Bearer | Spawns stdio bridges; agents can mutate routing and keys |
| `/api/cli-tools/runtime/*`, `/api/services/*`, `/api/vnc-session/*` | LOCAL_ONLY, never bypassable | `npm install`, node, Docker, Playwright spawns |
| `/api/db-backups/*` | Verify auth-gated (same fix PR closed an unauthenticated export) | Full SQLite including secrets |
| Cloud sync | Opt-in; pre-3.8.6 silently overwrote OAuth tokens | Hostile `CLOUD_URL` swaps credentials without detection |
| MITM toolkit, Zed keychain import | Opt-in privileged features | Root CA installation; keychain reads |

Two additional findings from the maintainer's own supply-chain review (Socket.dev flagged six behaviors on 3.8.5): cloud sync had **no HMAC or signature verification** before overwriting stored OAuth tokens — fixed in 3.8.6 with `X-Cloud-Sig` verification and an explicit opt-in; and the Zed credential import persisted keychain tokens in a single round-trip without per-account consent — fixed with a two-step discover/confirm flow. Both were real gaps that happened to be caught by a heuristic scanner.

---

## The Hardening Recipe

Every flag below exists because of a risk above. This is the exact environment I used for a verified local-only deployment (secrets generated per run with `openssl rand`, memory-only, never written to disk):

| Variable | Hardened value | Why |
|---|---|---|
| `OMNIROUTE_SERVER_HOST` | `127.0.0.1` | Default is `0.0.0.0` — binds every interface |
| `REQUIRE_API_KEY` | `true` | Default is `false` — `/v1/*` is anonymous by default |
| `JWT_SECRET` / `API_KEY_SECRET` | 48-byte base64 / 32-byte hex | Session signing and key encryption |
| `STORAGE_ENCRYPTION_KEY` | 32-byte hex | AES-256-GCM at rest; without it, secrets sit in plaintext SQLite |
| `INITIAL_PASSWORD` | 20+ chars, set **before first boot** | Closes the bootstrap-window takeover |
| `ALLOW_API_KEY_REVEAL` | `false` | No full keys rendered in the dashboard UI |
| `CORS_ORIGIN` | `http://localhost:20128` | Never `*` |
| `DATA_DIR` | explicit isolated path | Avoids the auto-created `~/.omniroute/` fallback store |
| `CALL_LOG_RETENTION_DAYS` | `7` | Prompt logs age out |
| `OMNIROUTE_MCP_ENFORCE_SCOPES` / `OMNIROUTE_MCP_SCOPES` | `true` / read-only allowlist | MCP agents get health/quota/usage reads — no `write:*`, no `admin` |

The MCP remote bypass kill-switch (`localOnlyManageScopeBypassEnabled`) stays **off**, keeping `/api/mcp/*` strict-loopback. The MITM toolkit, provider import, and cloud sync are left unused and unset (`CLOUD_URL` never configured).

---

## Verification Evidence

I don't trust documented security — I verify it. Seven checks against the hardened deployment (OmniRoute 3.8.50, npm, 2026-09-25):

| # | Check | Expected | Result |
|---|---|---|---|
| V-1 | Socket bind (`ss -tlnp`) | `127.0.0.1:20128` only | Pass — no `0.0.0.0` |
| V-2 | `GET /v1/models` without key | 401 | 401 |
| V-3 | `GET /` dashboard | Auth gate | 307 redirect to login |
| V-4 | `0.0.0.0` no-auth banner warning | Absent when hardened | Absent |
| V-5 | Environment vs bundled `.env` precedence | Environment wins | Confirmed in server log |
| V-6 | `GET /api/a2a/status` anonymous | Denied | `AUTH_001 Authentication required` |
| V-7 | `/api/mcp/stream` with spoofed external `Host`, no Bearer | 403 | 401 — guard evaluates the real connection IP (loopback), not the `Host` header; anonymous access still denied |

### Three findings that surprised me

1. **No fail-fast on missing secrets.** The SECURITY.md documentation says the server fails fast on missing or weak secrets. In practice, with no secrets set, the server auto-generated `STORAGE_ENCRYPTION_KEY` into `~/.omniroute/.env` and started normally. The fail-fast applies to *explicitly weak* values, not absent ones. If you don't bring your own secrets, it brings its own — and now there's a second credential store you didn't plan for.
2. **The insecure default is real.** Stock start binds `0.0.0.0` with no API-key requirement. The server prints its own warning banner about this, but the default stands. Hardening flags here are not optional polish — they are the difference between a local tool and a network-reachable credential store.
3. **`enc:v1` is pending until first credential.** A virgin `storage.sqlite` showed no encryption marker — expected, since nothing secret had been stored yet, but it means the encryption-at-rest claim only becomes verifiable after you connect your first provider. Re-check it at that point.

---

## Detection and Audit Commands

If you're running (or auditing) an OmniRoute instance, these read-only checks belong in your baseline. Full command set in the [guide](https://xaferima.github.io/Medium-personal-post/guides/omniroute-security-guide/#37-detection-and-audit-commands-read-only-placeholders-only):

```bash
# Pinned vs latest version (upgrade signal)
npm view omniroute version

# Rogue custom ACP agents (the CVE's persistence mechanism)
curl -s -H "Authorization: Bearer <MGMT_KEY>" http://127.0.0.1:20128/api/acp/agents

# Hunt unauthenticated ACP registration attempts in gateway logs
grep -a -E 'POST /api/acp/agents' "$DATA_DIR/logs/"* 2>/dev/null
```

If your instance was ever reachable beyond loopback with `requireLogin=false` (or before a management password was set), treat every provider credential it held as compromised and rotate them — assume the backups were readable too; the same fix PR closed an unauthenticated database-backup export.

---

## The Verdict

OmniRoute solves a real problem, and after hardening it is usable as a local-only tool: loopback bind, API-key enforcement, secrets in memory, MCP on a read-only allowlist. The free-tier pooling and combo failover are genuinely useful for solo developers minimizing cost.

But the honest security assessment is this: **a gateway that holds every credential you own should ship with secure defaults**, and this one ships listening on all interfaces with authentication disabled, auto-generates secrets when you don't supply them, and carried an unauthenticated RCE in its latest published release with no consensus on the fix version. The single-maintainer factor amplifies this — the security posture is improving fast (the 3.8.6 and 3.8.49 fixes are real), but there is no dedicated security team and no SOC 2 report to lean on.

My recommendation, in order of preference:

1. **For regulated or sensitive data:** use a governed alternative (LiteLLM self-hosted with RBAC, or OpenRouter Enterprise for managed compliance).
2. **For cost-optimized local use:** OmniRoute works — pinned version, the hardening environment above, loopback only, and re-verify against advisories after every upgrade.
3. **Never:** expose it to a LAN, tunnel, or shared network without `requireLogin=true` and the LOCAL_ONLY tier intact.

The full hardening checklist, residual-risk matrix, and audit commands are in the [complete guide](https://xaferima.github.io/Medium-personal-post/guides/omniroute-security-guide/). The source is available [on GitHub](https://github.com/xaferima/Medium-personal-post/blob/main/guides/omniroute-security-guide.md).

---

*This is not a sponsored post. All testing was performed locally on my own machine in September 2026. CVE information verified as of 2026-09-25 against [GHSA-hf57-cqmx-p4gr](https://github.com/diegosouzapw/OmniRoute/security/advisories/GHSA-hf57-cqmx-p4gr), [NVD](https://nvd.nist.gov/vuln/detail/CVE-2026-88062), and the vendor's [SECURITY.md](https://github.com/diegosouzapw/OmniRoute/blob/HEAD/SECURITY.md). Version and fix status for CVE-2026-88062 was ambiguous across sources at time of writing — re-verify before relying on any specific version.*

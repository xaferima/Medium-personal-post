---
layout: page
title: "OmniRoute — Complete Guide"
permalink: /guides/omniroute-security-guide/
---

# OmniRoute — Complete Guide

> Provenance: everything below comes from internet research plus a local hardening test executed on 2026-09-25 (OmniRoute 3.8.50, npm install, loopback-only).

## 1. What it is and what it is for

**OmniRoute** ([github.com/diegosouzapw/OmniRoute](https://github.com/diegosouzapw/OmniRoute), MIT license, TypeScript/Next.js + SQLite) is a **self-hosted local AI gateway**. It sits between your AI clients and upstream providers and exposes **one OpenAI-compatible endpoint** (`http://localhost:20128/v1`).

```text
Claude Code / Codex / Cursor / Cline / OpenCode
  → OmniRoute on localhost (:20128/v1)
    → format translation + compression + circuit breakers
      → Tier 1 subscriptions (OAuth) → Tier 2 API keys
        → Tier 3 cheap models → Tier 4 free providers
```

What it is for in practice:

- Point **every coding agent** at a single endpoint instead of configuring each provider separately.
- Pool what you already pay for: OAuth subscriptions (Claude Code, Codex, Gemini CLI, Qoder, Kiro, Cursor, GitHub Copilot…) plus API keys plus free tiers (~352 registered providers, 150+ with a free tier).
- Survive quota exhaustion: **combos** (model chains with 18+ routing strategies) fail over automatically in milliseconds.
- Spend fewer tokens: RTK + Caveman compression saves 15–95% on eligible tokens.
- Optional agent control plane: MCP server (90+ tools), A2A protocol (JSON-RPC skills), FTS5+vector memory, guardrails (PII masker, prompt-injection detector, vision-bridge SSRF guard).

## 2. OmniRoute vs OpenRouter

|  | OmniRoute | OpenRouter |
|---|---|---|
| Model | Self-hosted proxy you operate (Node.js + SQLite) | Managed SaaS on Cloudflare edge, zero ops |
| Setup | `npm i -g omniroute`, connect OAuth/keys, build combos | Account + credits + API key, ~2 minutes |
| Catalog | ~352 providers, 150+ free, 1200+ model IDs | 500+ models across 80+ providers, 25+ `:free` |
| Cost | Software $0; you pay only upstream usage ($0 possible on free tiers) | Provider passthrough, no token markup, but **5.5% fee on credit purchases**; BYOK allowance then 5% |
| Routing | Combos: priority, weighted, cost-optimized, context-relay, fusion, auto/chaos… quota-aware, multi-account round-robin | `model` + `provider` object; default = cheapest stable (inverse-square price); `:floor` (cheap), `:nitro` (throughput), `openrouter/auto` |
| Privacy | Keys/prompts stay on your machine; encryption at rest **optional**; guardrails **fail-open** | SOC 2 Type 2, GDPR, opt-in ZDR + EU routing; prompts transit third-party infra; per-model policies vary |
| Security maturity | Emerging (single maintainer, recent hardening fixes) | Mature managed team |
| MCP / A2A / memory | Built in | Not applicable (inference only) |
| Ops burden | Medium (updates, backups, credential rotation; SPOF for all clients) | None |
| Choose when | Solo dev minimizing cost, local-first, aggressive free-tier pooling, auditable single-maintainer project | Fast multi-model access, one invoice, no infrastructure, managed compliance artifacts |

Rule of thumb: prototype on OpenRouter, harden committed workloads locally with OmniRoute (or LiteLLM for team governance). Never send regulated data through OmniRoute without full hardening — and even then, prefer a governed alternative.

## 3. Security deep-dive — read before installing

> This section is a prerequisite for Section 4 (hardening). Every flag in Section 4 exists because of a risk documented here. Researched 2026-09-25 from the vendor advisory, NVD, GitHub/GitLab advisory DBs, the maintainer's Socket.dev attestation, and the authorization-architecture docs. No exploit code is included — only detection and audit commands.

### 3.1 Threat model

- **Crown jewels:** every provider API key and OAuth token you configure (OpenAI, Anthropic, Google, free-tier providers) plus your prompts — all held by one local process backed by one SQLite file.
- **Actors:** anyone/anything that can reach the port (LAN neighbor, exposed tunnel, malicious web page pointed at localhost), a compromised MCP-connected agent with broad scopes, a misconfigured or hostile `CLOUD_URL`, a typosquatted postinstall dependency.
- **Out of scope:** upstream provider breaches, model-output safety, endpoint-client (IDE) compromise — this guide covers the gateway itself.

### 3.2 Known CVEs and gaps

| ID | Severity (vendor vector, verbatim) | Weakness | Status for our tested 3.8.50 |
|---|---|---|---|
| **CVE-2026-88062** ([GHSA-hf57-cqmx-p4gr](https://github.com/diegosouzapw/OmniRoute/security/advisories/GHSA-hf57-cqmx-p4gr)) — ACP custom-agent RCE via `POST /api/acp/agents` (`binary` + `versionCommand` attacker-controlled → `execFileSync("node", ["-e", …])`) | CVSS 4.0 **9.5** `CVSS:4.0/AV:N/AC:L/AT:P/PR:N/UI:N/VC:H/VI:H/VA:H/SC:H/SI:H/SA:H` | CWE-306 (missing auth) + CWE-94 (code injection) | **Ambiguous — treat as vulnerable until you verify.** Vendor page says fixed in 3.8.49; NVD says 3.8.49 and earlier affected; advisory DBs list ≤3.8.50 with no patched version; fix PR #11028 merged into the 3.8.50 release branch but release notes don't mention it and scanners still flag 3.8.50. Upgrade to the newest release and verify (see 3.7). |
| **GHSA-fhh6-4qxv-rpqj class** — RCE-via-tunnel on spawn-capable routes (a leaked JWT over Cloudflare/ngrok reaching process-spawning endpoints) | n/a (design class) | CWE-306 pattern | Mitigated by design **if** you keep the LOCAL_ONLY tier intact: route-guard gate + CI check `check:route-guard-membership` + `SPAWN_CAPABLE_PREFIXES` deny-list. CVE-2026-88062 happened precisely because `/api/acp/` was missing from that classification. |
| **Socket.dev 6 findings on 3.8.5** (maintainer-attested, [SOCKET_DEV_FINDINGS.md](https://github.com/diegosouzapw/OmniRoute/blob/main/docs/security/SOCKET_DEV_FINDINGS.md)) | supply-chain heuristics | mixed | Two were real gaps, fixed in **3.8.6**: (a) cloud-sync silently overwrote OAuth tokens from any newer `CLOUD_URL` response — now HMAC `X-Cloud-Sig` verified and credential sync opt-in (`OMNIROUTE_CLOUD_SYNC_SECRETS`, default OFF); (b) Zed keychain import persisted without per-account consent — now 2-step discover→confirm with fingerprint match. Cosmetic: PowerShell `-EncodedCommand` → `-File` temp `.ps1` (0600); MITM/root-CA installer is opt-in loopback-only documented behavior; minified-standalone false positives. `OMNIROUTE_BUILD_PROFILE=minimal` amputates the 4 privileged modules (HTTP 503 stubs). |
| **MCP remote scope gap** (issue #7895) | privilege escalation by design | excessive authority | A remote MCP credential needed the broad `manage` scope just to pass the loopback gate — usable against the whole management API. Fixed via narrow `mcp:connect` scope + per-key scope binding (v3.8.49). Residual: stdio transport has no per-caller identity (global allowlist governs). |
| **Historical: hardcoded JWT secret; fail-open guardrails; optional encryption** | hardening debt | CWE-798 pattern / fail-open | Partially addressed (fail-fast on weak values, `STORAGE_ENCRYPTION_KEY` supported). Guardrails remain fail-open with per-request opt-out header — see 3.5. |

### 3.3 Attack surface map

| Route / feature | Tier | Why it matters |
|---|---|---|
| `POST /api/acp/agents` | Was **unclassified** (the CVE) — verify it is LOCAL_ONLY + auth-gated on your version | Spawns version-probe subprocess from user input |
| `/api/mcp/*` | LOCAL_ONLY, bypassable only by `manage`/`mcp:connect` Bearer (kill-switch available) | Spawns stdio bridges; agents can mutate routing/keys |
| `/api/cli-tools/runtime/*`, `/api/services/*`, `/api/tools/agent-bridge/*`, `/api/local/*`, `/api/vnc-session/*`, `POST /api/providers/{id}/login` | LOCAL_ONLY, **never bypassable** (zod + runtime veto) | `npm install`, node, MITM server, Docker, Playwright spawns |
| `/api/db-backups/*` | Verify auth-gated (same fix PR closed an unauthenticated export) | Full SQLite incl. secrets |
| OAuth callbacks (`/api/oauth/*/callback`) | Must stay remote-reachable | Token-bearing redirects — keep `CORS_ORIGIN` tight |
| MITM toolkit, Zed import, cloud sync | Opt-in privileged features | Root-CA install, keychain reads, remote token overwrite — disable all three if unused (`OMNIROUTE_DISABLE_MITM=1`, `OMNIROUTE_DISABLE_PROVIDER_IMPORT=1`, `CLOUD_URL` unset) |
| Dashboard `/` + `/api/settings/*` | Management auth; bootstrap window with no password = anonymous setup writes | Set `INITIAL_PASSWORD` before first boot on any non-loopback host |

### 3.4 Auth and defaults that hurt (our test evidence)

- Stock start binds **`0.0.0.0` with no API-key requirement** (own banner warns). Findings F-1/F-2 in Section 5.
- `requireLogin=false` disables auth **globally** (anonymous `POST /api/acp/agents` = the CVE's unauthenticated path).
- Fresh-install **bootstrap window** (no management password yet) allows unauthenticated setup writes that can flip `requireLogin=false` first.
- Server auto-creates `~/.omniroute/` with generated secrets when yours are missing — convenient, but a second credential store you must know about and clean.
- Good news (verified): unclassifiable routes fall back to MANAGEMENT (fail-closed), internal `x-omniroute-auth-*` headers are stripped, key comparison is timing-safe, DB errors fail closed on the client API.

### 3.5 Data-flow and privacy notes

- Prompts traverse: client → gateway (logs per `CALL_LOG_RETENTION_DAYS`, `noLog` opt-out per key) → upstream provider (each with its own retention policy). The gateway cannot override provider-side logging.
- Encryption at rest covers stored credentials only when `STORAGE_ENCRYPTION_KEY` is set; verify the `enc:v1` marker after storing your first provider credential (F-3).
- Compression (RTK/Caveman) drops "low-relevance" context — never the sole view for security audits of large logs; keep a raw-output path.
- Dependency footprint at 3.8.50: 1172 npm packages, ~3.3 GB, postinstall phase downloads platform binaries — run `npm audit`, pin versions, re-test after every upgrade.

### 3.6 Residual risks → mitigations (maps to Section 4)

| Residual risk | Mitigation (Section 4 flag/check) |
|---|---|
| CVE-2026-88062 on ≤ 3.8.50 | Upgrade to newest release; verify fix on-instance (3.8); `requireLogin=true` + strong password; loopback-only |
| RCE-via-tunnel class | Never bypass LOCAL_ONLY for spawn-capable prefixes; kill-switch OFF for remote MCP; no tunnels in local-only posture |
| Bootstrap-window takeover | Set `INITIAL_PASSWORD` in environment **before** first boot |
| Credential-store sprawl | Explicit `DATA_DIR`; know about `~/.omniroute/` fallback; encrypted backups; clean test artifacts |
| MCP agent overreach | `OMNIROUTE_MCP_ENFORCE_SCOPES=true` + read-only allowlist; per-client keys; toggles OFF when idle |
| Silent token overwrite via cloud sync | Leave `CLOUD_URL` unset; never enable `OMNIROUTE_CLOUD_SYNC_SECRETS` unless required |
| Prompt retention | `CALL_LOG_RETENTION_DAYS` 7–14; `noLog` keys for sensitive clients |
| Supply-chain drift | Pinned version + `npm audit` + re-run Section 5 checklist per upgrade |

### 3.7 Detection and audit commands (read-only, placeholders only)

Run against your instance; replace `<BASE>` with `http://127.0.0.1:20128` and `<MGMT_KEY>` with a management-scoped key (never paste real values into shared notes):

```bash
# Pinned vs latest version (upgrade signal)
npm view omniroute version
node -e "console.log(require('./node_modules/omniroute/package.json').version)"

# Rogue custom ACP agents (the CVE's persistence mechanism)
curl -s -H "Authorization: Bearer <MGMT_KEY>" <BASE>/api/acp/agents

# requireLogin posture + authz inventory
curl -s -H "Authorization: Bearer <MGMT_KEY>" <BASE>/api/settings/authz-inventory | head -c 2000

# Hunt unauthenticated ACP registration attempts in gateway logs
grep -a -E 'POST /api/acp/agents' "$DATA_DIR/logs/"* 2>/dev/null | head -20

# Unexpected files in the data dir (dropper artifacts)
ls -lat "$DATA_DIR" | head -20

# Read-only audit of admin actions in SQLite (python3, no sqlite3 CLI needed)
python3 -c "import sqlite3;db=sqlite3.connect('file:$DATA_DIR/storage.sqlite?mode=ro',uri=True);print([r for r in db.execute(\"SELECT name FROM sqlite_master WHERE type='table'\")])"
```

### 3.8 If exposed checklist

1. Upgrade to the latest release and re-verify with the 3.7 commands on your instance.
2. Set `requireLogin=true` with a real management password; confirm anonymous `POST /api/acp/agents` is rejected.
3. **Rotate every provider key the gateway ever held** (OAuth + API keys) at each provider — assume exfiltration.
4. Audit `/api/acp/agents` listing, running processes, `audit_log` table, and `DATA_DIR` for unexpected files; assume `db-backups` were readable pre-fix.
5. Re-run the full Section 5 verification checklist before returning the instance to service.

## 4. Hardened install (npm, local-only) — verified procedure

Tested 2026-09-25 with `omniroute@3.8.50` pinned (1172 packages, ~8 min install, ~3.3 GB). Isolated temp dir (`~/.cache/omniroute-test/app`).

### 4.1 Secrets (memory only, never on disk or git)

Generate per deployment; Redacted lengths shown, never values:

- `JWT_SECRET` — base64, 48 bytes (min 32 chars). Signs dashboard sessions.
- `API_KEY_SECRET` — hex, 32 bytes (min 16). Encrypts API key values.
- `STORAGE_ENCRYPTION_KEY` — hex, 32 bytes. Enables AES-256-GCM at rest (`enc:v1` format). **Without it, SQLite stores secrets in plaintext.**
- `INITIAL_PASSWORD` — strong random, 20+ chars. Replace the `123456` default on first login via Dashboard → Settings → Security.

The server rejects explicitly weak values (`changeme`, `secret`, `password`). Missing values do **not** fail fast — see finding F-1.

### 4.2 Hardening environment

| Variable | Hardened value | Purpose |
|---|---|---|
| `OMNIROUTE_SERVER_HOST` | `127.0.0.1` | Loopback bind only (default is `0.0.0.0`) |
| `PORT` | `20128` | Fixed local port |
| `REQUIRE_API_KEY` | `true` | Every `/v1/*` call needs a Bearer key (default `false`) |
| `ALLOW_API_KEY_REVEAL` | `false` | Never show full keys in dashboard UI |
| `AUTH_COOKIE_SECURE` | `false` locally (`true` only behind HTTPS) | Secure flag on session cookies |
| `CORS_ORIGIN` | `http://localhost:20128` (never `*`) | Cross-origin allowlist |
| `MAX_BODY_SIZE_BYTES` | `5242880` | 5 MB request cap |
| `MACHINE_ID_SALT` | unique per deployment | Fingerprint diversification |
| `DATA_DIR` | explicit isolated path | SQLite + logs + backups location |
| `CALL_LOG_RETENTION_DAYS` | `7`–`14` | Short prompt retention; `noLog` per key for sensitive clients |
| `OMNIROUTE_MCP_ENFORCE_SCOPES` | `true` | Deny MCP tools missing scopes |
| `OMNIROUTE_MCP_SCOPES` | `read:health,read:combos,read:quota,read:usage,read:models` | Least-privilege allowlist (no `write:*`, no `admin`) |

Launch: `omniroute serve --port 20128 --no-open` with the above exported. Precedence verified: **environment > `~/.omniroute/.env` > bundled package `.env`**.

### 4.3 Post-install steps

1. Change the initial password immediately.
2. Create **one API key per client** (Codex, OpenCode, …) with minimum scopes and model allowlist.
3. Keep MCP/A2A toggles OFF unless actively used; keep the `localOnlyManageScopeBypassEnabled` kill-switch OFF so `/api/mcp/*` stays strict-loopback. Never expose `/api/cli-tools/runtime/*` or `/api/services/*` remotely (arbitrary subprocess execution — the CVE class the LOCAL_ONLY tier exists to block).
4. Leave guardrails ON (`pii-masker`, `prompt-injection`, `vision-bridge`) but treat them as defense-in-depth: they are **fail-open**.
5. Encrypted backup of `DATA_DIR` before every upgrade; clean shutdown so SQLite WAL checkpoints (avoid `kill -9`).

## 5. Verification checklist (evidence, 2026-09-25 test)

| # | Check | Expected | Result |
|---|---|---|---|
| V-1 | `ss -tlnp` bind | `127.0.0.1:20128` only | pass |
| V-2 | `GET /v1/models` without key | 401 | 401 |
| V-3 | `GET /` dashboard | auth gate | 307 redirect to login |
| V-4 | `0.0.0.0` no-auth warning | absent when hardened | absent |
| V-5 | Env precedence over bundled `.env` | env wins | logged `ignored, the environment set it first` |
| V-6 | `GET /api/a2a/status` anonymous | denied | `AUTH_001 Authentication required` |
| V-7 | `/api/mcp/stream` spoofed external `Host`, no Bearer | 403 | 401 — guard evaluates connection IP (real loopback), not `Host`; anonymous access still denied, but true 403 needs a non-loopback connection test |

### Findings

- **F-1 (fail-fast gap):** with no secrets set, the server did **not** fail fast — it auto-generated `STORAGE_ENCRYPTION_KEY` into `~/.omniroute/.env` and started. Always export your own secrets; the auto-generated artifacts from this test were deleted.
- **F-2 (unsafe default confirmed):** stock start binds `0.0.0.0` with no API-key requirement (its own banner warns about it). Hardening flags are mandatory, not optional.
- **F-3 (pending):** no `enc:v1` marker in a virgin `storage.sqlite` — expected, nothing secret stored yet. Re-verify after connecting the first provider.
- **F-4 (observation):** a second loopback listener on `:20131` from the same process (likely internal WS bridge). Both loopback, no external exposure.

## 6. Secure daily use

- **Do:** per-client keys, loopback-only, short log retention, encrypted `DATA_DIR` backups, pinned versions + re-test after upgrades, `npm audit` on the pinned tag, monitor cost/failure-rate/latency and abnormal logins.
- **Never:** bind `0.0.0.0` / expose `:20128` to LAN or tunnels, enable `ALLOW_API_KEY_REVEAL` on shared instances, commit secrets or `.env` files, paste keys/tokens into screenshots/logs/issues, use MCP `write:*`/`admin` scopes for routine agents, rely solely on compression output for security audits of large logs (keep raw-output path).
- **Rotation:** on any suspected leak — rotate `JWT_SECRET`, `API_KEY_SECRET`, provider OAuth/API credentials, and dashboard password; purge logs from the exposure window.

## 7. Troubleshooting

- **Install is slow (~8 min):** long postinstall phase downloads platform binaries (`node ./script/install`); 2.5 GB+ `node_modules` is normal.
- **Server unreachable after `AUTH_COOKIE_SECURE=true` on plain HTTP:** expected — that flag requires HTTPS; keep `false` for pure-localhost.
- **Need remote MCP later:** deliberate decision — requires a `manage`-scoped Bearer via the `/api/mcp/` carve-out (fixed in v3.8.49 with narrow `mcp:connect`), per-key scope binding, TLS reverse proxy, and IP allowlist. Out of scope for the local-only posture in this guide.
- **Stale `~/.omniroute/` interfering:** the server reads it as fallback config; for isolated tests always set `DATA_DIR` explicitly and clean default artifacts.

## 8. References

- Source: [github.com/diegosouzapw/OmniRoute](https://github.com/diegosouzapw/OmniRoute) · Site: [omniroute.online](https://www.omniroute.online/es/) · Package: [npmjs.com/package/omniroute](https://www.npmjs.com/package/omniroute)
- Security model: `SECURITY.md` · Architecture: `docs/ARCHITECTURE.md` · Auth pipeline: `docs/architecture/AUTHZ_GUIDE.md` · Route guards: `docs/security/ROUTE_GUARD_TIERS.md` · MCP scopes: `docs/frameworks/MCP-SERVER.md`
- Comparatives: [OpenRouter docs](https://openrouter.ai/docs/faq) · [OpenRouter pricing](https://openrouter.ai/pricing) · [OpenRouter routing guide](https://openrouter.ai/blog/insights/model-routing/)
- Security advisories: [GHSA-hf57-cqmx-p4gr (CVE-2026-88062)](https://github.com/diegosouzapw/OmniRoute/security/advisories/GHSA-hf57-cqmx-p4gr) · [NVD CVE-2026-88062](https://nvd.nist.gov/vuln/detail/CVE-2026-88062) · [Socket.dev attestation](https://github.com/diegosouzapw/OmniRoute/blob/main/docs/security/SOCKET_DEV_FINDINGS.md) · [Route Guard Tiers](https://github.com/diegosouzapw/OmniRoute/wiki/Route-Guard-Tiers)

# ClawHost evaluation

> Phase 0, artifact 1. Decides whether `bfzli/clawhost` is a starting point
> for our deployment platform. Read alongside §6 of `docs/project_brief.md`.

## TL;DR

**Recommendation: cherry-pick patterns, build fresh** (path 2 of §6.3).

ClawHost is a real, working OpenClaw deployment dashboard with thoughtful
schema and a usable Hetzner+Cloudflare provisioning pipeline. But three of its
core architectural assumptions don't fit ours, and rewriting them inside
ClawHost is more work than greenfield against Vercel+Neon:

1. **Bun-as-long-lived-process** for the SSH terminal WebSocket and the
   in-memory auth cache. Vercel functions cannot host either.
2. **Polar.sh deeply integrated** into the provisioning pipeline — the VPS
   only spins up after a `subscription.active` webhook fires. Removing it
   touches schema, controllers, web components, and the state machine.
3. **Cloud-init applies essentially no hardening** — `ssh_pwauth: true`,
   passwordless sudo for the agent user, no fail2ban, no sysctl, no AIDE,
   no AppArmor, no SOPS, no `unattended-upgrades`. Adapting this to our §5
   checklist is ~80% of a rewrite.

What's worth stealing is concrete and tracked in
[Patterns worth stealing](#patterns-worth-stealing) below.

Estimated greenfield against our stack: **2–3 weeks**. Estimated fork-and-adapt: **4–6 weeks** with worse end-state.

## Methodology

- Code review only, no full-stack run. Approved scope cap.
- Repo cloned at `/tmp/spike/clawhost` (commit at time of survey: latest of `bfzli/clawhost` `main`).
- Survey delegated to a research agent with a structured prompt; findings cross-checked against the README + `CLAUDE.md` shipped in the repo.
- File and line references throughout cite the surveyed code; line numbers may drift if upstream moves.

## What ClawHost is

- **Stack:** Bun + Turborepo monorepo. Hono.js API on Bun runtime, Drizzle ORM against Neon Postgres, Firebase Admin SDK for sessions. React 18 + Vite + shadcn/ui frontend. Expo mobile app (3 screens, parallel UI). Electron desktop app (`apps/clawhostgo`) that runs OpenClaw locally — different product, not a thin client.
- **Inventory model:** the `agents` table (`apps/api/src/db/schema/agents.ts`) is the source of truth for VPSes — `userId`, `providerServerId`, `agentType`, `status`, `ip`, `subdomain`, encrypted `gatewayToken`, `hostKeyFingerprint`, plus four Polar columns and `deletionScheduledAt` for soft-delete. A separate `pendingAgents` table holds purchase intents with TTL.
- **Provisioning flow:** Polar checkout → `pendingAgents` row → `subscription.active` webhook → `provisionAgent.ts` calls Hetzner's API → cloud-init bootstraps the VPS → Cloudflare DNS record created → certbot via cloud-init → status reconciled by `syncAgentServers.ts` when the user opens the dashboard.
- **Auth:** Email OTP via Resend with hashed codes + per-IP-and-email rate limiting. Google and GitHub OAuth ride Firebase's client SDK directly. Server only ever sees Firebase JWTs.
- **SSH terminal:** `apps/api/src/services/terminalSocket.ts` — Bun-native WebSocket, `ssh2` to the VPS as `root` with the encrypted password from the DB, host-key TOFU stored in the `agents` table, xterm.js on the frontend, resize protocol via JSON control messages.
- **Roughly 12.4k LOC of API, 33.8k LOC of web.** Quality is high for a one-engineer-ish project: keyboard support, loading skeletons, theme/i18n, locale-aware formatting.

## Per-feature evaluation

For each feature: (a) does it exist, (b) implementation quality, (c) tag for
our project (KEEP / ADAPT / REPLACE / REMOVE / MISSING), (d) hours to adapt
vs hours to rebuild fresh against Vercel+Neon. "Run it" is skipped per the
approved scope cap; quality assessed from code.

| Feature | Exists? | Quality | Tag | Adapt (h) | Rebuild (h) |
|---|---|---|---|---|---|
| Hetzner one-click VPS provisioning | Yes — `controllers/agents/provisionAgent.ts` + per-verb wrappers in `services/hetzner/` | Solid; full pipeline w/ DNS + cloud-init + status reconciliation | **ADAPT** | 4–6 | 8–12 |
| Cloudflare DNS subdomain | Yes — `services/cloudflare.ts` (official SDK + 30s cache + inflight dedupe) | Solid; domain `clawhost.cloud` hardcoded at `cloudflare.ts:90` | **ADAPT** | 1–2 | 3–4 |
| Let's Encrypt SSL | Yes — embedded in cloud-init at `generateCloudInit.ts:280-290` | Naive (sequential `host` poll then certbot); no DNS-01 fallback | **ADAPT** | 1 | 2–3 |
| Browser SSH terminal | Yes — `services/terminalSocket.ts` + xterm.js | Solid (resize protocol, ping keepalive, host-key TOFU); root-password auth is the smell | **ADAPT** to key-based, **and host on our small VPS not Vercel** | 4–6 | 12–16 |
| Diagnostics, logs, files | Yes — `controllers/agents/{getAgentDiagnostics,getAgentLogs,listAgentFiles,readAgentFile,updateAgentFile}.ts` | Naive — fresh SSH connection per call, runs shell commands; works, doesn't scale | **REPLACE** with NanoClaw-side endpoints | 3–4 | 12–20 |
| Version mgmt + upgrade | Yes — `installAgentVersion.ts` (`systemctl stop` → `npm i -g` → restart over SSH) | No canary, no health gate, no rollback, no fleet view | **REPLACE** | 6–8 (per-host adapt) | 30–50 (proper canary/rollback orchestrator per our brief) |
| SSH key mgmt | Yes — `controllers/ssh-keys/*` + Hetzner SSH key endpoints + `sshKeys` table | Solid | **ADAPT** | 2–3 | 6–8 |
| Persistent storage volume | Yes — Hetzner volume primitives + `volumes` table; created at provision time only | Solid | **ADAPT** | 2–3 | 6–10 |
| Multi-auth (email OTP / Google / GitHub) | Yes via Firebase | OTP flow is solid (hashed, rate-limited, attempt counter); Google/GitHub ride Firebase | **REPLACE** — Firebase Admin doesn't fit Vercel+Neon idiomatically | 6–8 (drag Firebase along) | 12–20 (Auth.js / Clerk / Supabase Auth) |
| Polar.sh billing | Yes, deeply | Solid integration; provisioning pipeline assumes purchase-first | **REMOVE** | n/a | Removal: 6–10 |
| Cloud-init templating | Yes — `controllers/agents/helpers/generateCloudInit.ts` (string-templated YAML) | Brittle; one bad interpolation bricks provisioning | **REPLACE** with our hardened template (§5) | 8–12 to splice NanoClaw in | 30–50 to build a proper §5-compliant template |
| Inventory model (`agents` + `pendingAgents` schema split) | Yes | Solid; indexed; soft-delete via `deletionScheduledAt`; clean intent-vs-real separation | **KEEP/ADAPT** the schema shape | 2–4 | n/a |
| Update orchestration (canary, rollback) | **No** — per-agent only, no fleet logic | Aspirational | **MISSING** | n/a | 40–80 (matches our Phase 2 brief) |
| Health checks / heartbeats | **Partial** — Hetzner status polling only on dashboard open; no VPS-side heartbeat | Naive | **MISSING** | n/a | 16–24 |
| Audit log | **No** — only `console.error` strings | n/a | **MISSING** | n/a | 8–12 |

### Notes that don't fit the table

- **The SSH terminal is the highest-value pattern that ports cleanly.** It's
  ~600 LOC of Bun WebSocket + ssh2 + host-key TOFU. The reason it can't ride
  on Vercel is purely runtime — Vercel functions cap at 10–60 s and don't do
  long-lived bidirectional sockets. Hosting the WebSocket on our control-plane
  VPS is the natural place for it (the small persistent VPS already exists,
  has Tailscale, and is the only thing that can SSH to user VPSes without
  trombing through the operator's laptop).
- **Multi-auth REPLACE is consequential** — Phase 1 needs an auth choice for
  the operator dashboard. Auth.js (formerly NextAuth) with Neon as the adapter
  is the boring pick and cleanly handles email OTP + Google + GitHub. Will
  flag in `architecture.md`. Not blocking for this artifact.
- **The `pendingAgents` "intent record" pattern translates without Polar.**
  We have an analogous moment: operator clicks "add user" → Slack install
  link is generated → on first install completion the VPS provisions. The
  intent row models the "install link sent but not yet completed" state,
  with TTL cleanup if the user never installs.

## Cloud-init: the most consequential finding

ClawHost's cloud-init is at `apps/api/src/controllers/agents/helpers/generateCloudInit.ts`. Read carefully against our §5 hardening checklist, the gap is wider than I expected. Concrete failures:

- **SSH:** `ssh_pwauth: true` (line 188), root password auth allowed, no `PermitRootLogin prohibit-password`, no `PasswordAuthentication no`, no key-only enforcement, no `MaxAuthTries`, no modern cipher restriction. Our §5.2 turns essentially every one of these into the opposite.
- **The agent user is created with passwordless sudo:**
  `echo 'openclaw ALL=(ALL) NOPASSWD:ALL' > /etc/sudoers.d/openclaw` (line 13). Borderline-negligent for a security-positioned product. We need a non-root sudo user *with password / key-required sudo*, per §5.2.
- **No fail2ban, no AppArmor, no AIDE, no auditd, no SOPS, no `unattended-upgrades`.** Each is its own §5 subsection (5.4, 5.6, 5.7, 5.11, 5.1). All must be added.
- **No sysctl hardening at all.** `kernel.kptr_restrict`, `tcp_syncookies`, `rp_filter`, etc. — none. Our §5.5 has 25+ knobs to turn.
- **Hetzner Cloud Firewall is not configured** — UFW alone is enabled and opens 22, 80, 443 to the public internet. We need the deny-all-but-Tailscale model from §5.3.
- **The OpenClaw gateway is configured `dangerouslyDisableDeviceAuth: true` with `allowedOrigins: ['*']`** (lines 155–158). Internet-reachable on port 443 behind only TLS + a `?token=` URL parameter. The opposite of our defense-in-depth posture (Tailscale-only + HMAC + SOPS).
- **No `noexec` mounts. No journald hardening. No swap-disable-core-dumps. No reboot-on-kernel-update.**

The cloud-init is a string-templated YAML pipeline (`generateCloudInit.ts:186-294`) — fine for a 100-line bootstrap but not for the multi-section hardened template we need. We'd be rewriting it almost entirely to match §5, and once that's done there's nothing left of the original. This is the single biggest reason fork-and-adapt loses to greenfield.

## Patterns worth stealing

Concrete things to study before writing the equivalent in our stack. File references are to the cloned ClawHost repo.

- **`agents` / `pendingAgents` schema split** — clean way to model "intent
  initiated but real resource not yet created." Translates to "Slack install
  link sent, VPS not yet provisioned" in our model.
  `apps/api/src/db/schema/agents.ts`, `pendingAgents.ts`.
- **`CloudProvider` interface with TTL caching** — provider abstraction that
  could let us add a non-Hetzner cloud someday without code changes elsewhere.
  `apps/api/src/services/provider/getProvider.ts`.
- **TOFU host-key store with `crypto.timingSafeEqual`** —
  `apps/api/src/services/hostKeyStore.ts:36-49`. Pattern, not the code.
- **AES-256-GCM encryption helper** for tokens at rest in DB —
  `apps/api/src/lib/encryption/encrypt.ts`. Useful for the bot-token /
  refresh-token columns in our `slack_installs` table.
- **OTP flow with hashed codes + rate-limited attempts** —
  `apps/api/src/controllers/auth/sendOtp.ts`, `verifyOtp.ts`. We may not
  need email OTP at all if the operator dashboard uses Google/GitHub only,
  but the rate-limit-per-IP-and-identifier pattern is reusable.
- **xterm.js + Bun WebSocket + ssh2 protocol shape** for the browser SSH
  terminal, ported to our control-plane VPS rather than Vercel.
  `apps/api/src/services/terminalSocket.ts` + `apps/web/src/components/dashboard/AgentTerminalContent.tsx`.
- **Drizzle migration discipline** — 34 migrations, never edited after
  shipping. Worth adopting verbatim.
- **`syncAgentServers.ts` reconciliation pattern** — read provider state on
  dashboard open and update DB. Lazy heartbeat alternative; useful for the
  operator dashboard's "what's actually running" view.

## Smells & footguns found

- **Provisioning swallows DNS errors silently** — `provisionAgent.ts:139`.
  Server gets created even if Cloudflare fails. Don't repeat this.
- **`controlUi.dangerouslyDisableDeviceAuth: true` + `allowedOrigins: ['*']`**
  in cloud-init line 155 — already noted above. The biggest single security
  smell in the codebase.
- **Passwordless sudo for the agent user** (`generateCloudInit.ts:13`) —
  do not copy.
- **Domain `clawhost.cloud` hardcoded** at `cloudflare.ts:90` and various
  `externalUrls`. A fork would have to grep-and-replace.
- **`bun version:patch` runs on every commit** — auto-bumps `apps/api/package.json`
  patch version. Hence v0.0.203/206 numbers. Noisy.
- **Host-key fallback** — `terminalSocket.ts:112` calls `verify(true)` if the
  host-key store fails to read; effectively disables TOFU under failure.
  Should fail closed, not open.
- **Per-call SSH connections** for diagnostics/logs/files — opens a fresh
  ssh2 client per HTTP request. Works for one user, doesn't scale.
- **No cloud-init output validation beyond a unit test** — string interpolation
  bugs would brick provisioning silently.
- **The Electron `clawhostgo` app is a different product entirely** — local
  OpenClaw runner, not a remote dashboard. Surprising in a "self-hosted
  deployment" repo. Not relevant to us; don't get distracted by it.

## What we still don't know

Things the code review can't answer that may matter at Phase 1:

- **Does the provisioning pipeline actually complete end-to-end on a real
  Hetzner project?** We took the scope cap of code-review-only. If the
  cherry-pick decision sticks (which I expect), we'll never need to know.
  If `architecture.md` revisits the decision, we'd have to spend ~1 day to run it.
- **How does `installAgentVersion.ts` behave when the SSH command times out
  mid-upgrade?** The code path looks like it leaves the VPS in `updating`
  state forever. Phase 2's update orchestration must handle this; our
  GitHub-Actions-based design (per the brief) sidesteps the issue.
- **What's the actual VPS-side memory footprint of OpenClaw + nginx + certbot
  + the gateway?** Not directly relevant since we're using NanoClaw, but
  ClawHost's choice of CPX11 (3 €/mo, 2 vCPU, 2 GB) vs our CAX11 (3.29 €/mo,
  2 vCPU ARM, 4 GB) suggests they found 2 GB sufficient. Helps confirm
  CAX11 is comfortable for our load. Validate properly in artifact 2.

## Decision rationale (per §6.3)

The three §6.3 paths, scored against this survey:

| Path | Verdict | Reasoning |
|---|---|---|
| **Fork-and-adapt** | Reject | KEEP+ADAPT covers ~50% of needed features by hour count, *and* the load-bearing pieces (cloud-init, auth, runtime model) are all REPLACE. The §6.2 "fork if KEEP+ADAPT ≥ 60%" threshold isn't met, and the qualitative architecture mismatch (Bun-long-lived vs Vercel-serverless) is a hard blocker that no amount of porting fixes cleanly. |
| **Cherry-pick patterns, build fresh** | **Choose this** | Steal the schema, the encryption helper, the TOFU store, the OTP rate-limit pattern, the WebSocket terminal protocol shape (host on small VPS), the Drizzle discipline. Build everything else against Vercel+Neon+small-VPS+GitHub Actions per the brief. Estimated 2–3 weeks of greenfield. |
| **Use-as-is** | Reject (already ruled out by the brief, confirmed here) | Wrong agent (OpenClaw vs NanoClaw), wrong stack (Hono.js vs Vercel+Neon), zero hardening, Polar in the critical path. |

**Going with cherry-pick, build fresh.**

## Implications for `architecture.md`

Things that flow from this artifact into the eventual `architecture.md` (artifact 6) — not decisions yet, just things that will land in that doc:

- **Auth choice for the operator dashboard.** Auth.js + Neon adapter is the
  boring pick; covers Google + GitHub + magic-link OTP. Confirm in artifact 6.
- **The browser SSH terminal lives on the control-plane VPS, not Vercel.**
  Tailscale-bound on the VPS side; Vercel renders the xterm.js frontend and
  connects to the control-plane VPS WebSocket via an authenticated endpoint
  on the operator's tailnet. Implies the operator's laptop is on Tailscale —
  which it is per the brief.
- **The schema stub** — borrow `agents` and `pendingAgents` shape verbatim,
  rename to `vps_hosts` and `pending_provisions`. Add `slack_installs`,
  `integrations`, `audit_log`, `health_pings` per the brief's §4.1.
- **Cloud-init is a from-scratch artifact.** ClawHost's
  `generateCloudInit.ts` is a structural template only — the content is
  going to be the §5 hardening checklist verbatim, not a port.

## Status

- [x] ClawHost cloned and surveyed.
- [x] Per-feature evaluation produced.
- [x] Path decision proposed (cherry-pick, build fresh).
- [ ] Operator approval of this path before artifact 6 (`architecture.md`)
      locks it in. If you disagree with the decision, raise it now — it
      affects how much of the next four artifacts focus on greenfield design
      versus integration with existing code.

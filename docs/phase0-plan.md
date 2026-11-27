# Phase 0 plan and status

> Operational companion to `project_brief.md`. Captures the Phase 0 plan
> agreed with the operator, resolutions to brief ambiguities, decisions
> made beyond the brief, and current status. Lets a future session resume
> without this chat history.

## Status snapshot (as of 2026-11-27)

| # | Artifact | Status | Doc |
|---|---|---|---|
| 1 | ClawHost evaluation | ✅ done, merged | `docs/clawhost-evaluation.md` |
| 2 | NanoClaw spike + §5 hardening baseline | ⏸ waiting on operator credentials | `docs/nanoclaw-baseline.md` (not started) |
| 3 | Kimi K2.6 via Moonshot | pending (after artifact 2) | `docs/kimi-k26-via-moonshot.md` (not started) |
| 4 | Slack multi-install validation | pending (also needs sandbox workspace from operator) | `docs/slack-multi-install-validation.md` (not started) |
| 5 | Hetzner Firewall + Tailscale | pending (after artifact 2) | `docs/hetzner-tailscale-firewall.md` (not started) |
| 6 | Architecture proposal | pending (synthesis of the above) | `docs/architecture.md` (not started) |

Captured during Phase 0 kickoff but **separate from Phase 0 scope**:

- `docs/wiki-design.md` — Phase 3+ wiki design draft (✅ merged).

## Six-artifact plan (from the Phase 0 kickoff)

| # | Artifact | Scope | Estimate |
|---|---|---|---|
| 1 | `clawhost-evaluation.md` | Code-review-only survey against brief §6 | 1 day |
| 2 | `nanoclaw-baseline.md` | Provision 1× CAX11, apply §5 hardening manually, install NanoClaw + 3-4 MCP sidecars, document breakage per §5.14, measure mem/CPU headroom | 2-3 days + 7 days AIDE observation in parallel |
| 3 | `kimi-k26-via-moonshot.md` | Verify K2.6 endpoint + auth, latency from Hetzner EU/US/SG, per-key spending caps, data residency, OpenAI-compat status | 0.5-1 day |
| 4 | `slack-multi-install-validation.md` | Build minimal distributable app, install twice in sandbox workspace, validate per-user bot identities + token rotation + `apps.manifest.update` + slash commands | 1 day |
| 5 | `hetzner-tailscale-firewall.md` | Deny-all-but-Tailscale firewall + measure NAT-traversal vs relay fallback | 0.5 day |
| 6 | `architecture.md` | Synthesize into tech choices + concrete Phase 1 scope | 0.5-1 day |

Total estimate: **~6-8 working days** (plus 7 days of AIDE observation overlapping with later artifacts).

## Resolutions to brief ambiguities

From the Phase 0 kickoff conversation:

1. **Control-plane VPS state access.** §4.2's "no state of its own" + "no private keys other than its own SSH and Tailscale" means the control-plane VPS holds its own SSH/Tailscale keys, NOT the Neon password. Neon credentials live in Vercel env vars only. The control-plane VPS fetches inventory from an authenticated Vercel internal endpoint. Codify in `architecture.md`.
2. **Slack tier.** §2's "Tier 4" treated as shorthand for "per-user OAuth installs with refresh tokens." Validate the capability in artifact 4, not the tier label.
3. **`apps.manifest.update` per-install.** Flagged uncertain in the brief (§2 and §9.2). Plan: ship the cosmetic `username`/`icon_url` layer unconditionally; validate the identity layer in artifact 4 and commit to it only if it works on standard-tier workspaces.
4. **CAX11 (ARM, 4 GB) sufficiency.** Hard pass/fail validation in artifact 2. If NanoClaw + 3-4 MCP sidecars + eventual wiki MCP exceeds 4 GB headroom, cost model changes (CAX21 ~€6/mo per user).
5. **Granola integration mechanism.** Listed under both the OAuth proxy and webhook-based delivery in the brief — both likely true. Validation deferred to Phase 3 prep (out of Phase 0 scope).
6. **Doc path formatting in the brief.** The brief renders doc references like `docs/[name.md](http://name.md)` — a Notion export artifact. Produce new docs at the clean paths (e.g. `docs/nanoclaw-baseline.md`); ignore the broken links in the brief.

## Decisions made during Phase 0 that extend the brief

These sharpen or supplement §2/§3 of the brief:

- **Scope cap on artifact 1:** code-review-only. Don't run ClawHost's stack. Saved ~1 day.
- **NanoClaw spike approach for artifact 2:** manual-first, then capture working steps as cloud-init/Ansible. Isolates breakage per §5.14 and produces battle-tested automation.
- **Review cadence:** one-by-one. Each artifact is its own PR; operator reviews before the next artifact starts.
- **Operator dashboard auth:** **Clerk + Google SSO only.** No email/OTP, no GitHub, no MFA setup to manage (Google's own 2FA satisfies the 2FA requirement). No Firebase, no Auth.js.
- **Stay on Vercel.** The split architecture (Vercel public surfaces + control-plane VPS for Tailscale-bound/long-lived work) is correct. The WebSocket terminal and live log streaming live on the control-plane VPS; operator's browser reaches them over Tailscale for live dashboard features. Read-only dashboard views work without Tailscale.
- **Tailscale free tier.** 100-device limit covers our scale; ACLs work on free tier.
- **NanoClaw source:** fork `qwibitai/nanoclaw` (per upstream's expected deployment pattern).
- **Slack sandbox workspace for artifact 4:** operator creates when artifact 4 starts. Claude cannot create one (needs real email + verification).
- **Browser SSH terminal (ClawHost's most borrowable pattern):** deferred to Phase 3+ polish. Operator is on Tailscale and can `ssh agent@<tailscale-ip>` from their laptop. The WebSocket **pattern** remains valuable for Phase 2's live log tailing and rollout-progress views.

## Related designs captured in their own docs

- **Wiki (Phase 3+):** see `docs/wiki-design.md`. Headlines: markdown canonical, graph index derived later; all UX inside Slack via slash commands + App Home + agent DM; Granola transcripts as primary v0 ingest with auto-ingest on by default; separate repo (suggested `alexbalonperin/llm-wiki`).

## Pending credentials setup (for whoever resumes artifact 2)

Before artifact 2 can start, the operator needs to provide:

**Hetzner (dedicated-project pattern for blast-radius containment):**

1. Hetzner Cloud Console → New Project → name it `nanoclaw-spike` (or similar). Keep it dedicated — nothing else inside.
2. Inside the project: **Security → SSH Keys → Add SSH Key.** Add the session-generated spike key (the resuming session will regenerate a fresh keypair and print the new public key). Also add the operator's own laptop public key.
3. **Security → API Tokens → Generate API Token**, scope **Read & Write**, name `phase0-spike`. Paste the token in chat (it only displays once). Revoke after Phase 0.
4. Note: Hetzner does NOT offer action-level scoping (no "write without delete"). Project isolation is the only containment. Blast radius = whatever's in the project.

**Tailscale:**

5. Sign up at https://login.tailscale.com (Google SSO is fine; free tier, no credit card).
6. **Settings → Keys → Generate auth key:** reusable **off**, ephemeral **on** (so the spike VPS auto-cleans when destroyed), pre-approved **on**, tags empty for now, expiry 7 days. Paste in chat.

**Slack (needed before artifact 4, not artifact 2):**

7. Create a sandbox Slack workspace at https://slack.com/get-started (free; throwaway email or `+nanoclaw` alias).
8. Invite a second human/account so multi-install testing has ≥2 installs to validate.

## Resume guidance

For a future session (me or a fresh one) picking up Phase 0:

1. Read `docs/project_brief.md` end-to-end.
2. Read this doc (`docs/phase0-plan.md`).
3. Read the completed artifacts in the status snapshot above (and anything else merged since).
4. **If artifact 2 is next:**
   - Ask the operator for the Hetzner credentials above.
   - **Regenerate the SSH keypair** — session sandboxes don't persist filesystems across sessions. Any public key from older chat history is invalid.
   - Proceed with manual-first spike per §5.14 of the brief.
5. **If a later artifact is next:** merged docs show where we are; pick up from the first `(not started)` in the status snapshot.

## Open items not yet decided

Things we touched but haven't locked:

- **Auto-ingest default for Granola → wiki.** My lean: auto-on, Slack notification per ingest ("added today's meeting with X"). Operator to confirm when Phase 3 approaches.
- **Wiki schema seed, lint cadence, graph DB choice, multi-user wiki linkage** — all deferred to Phase 3 prep per `wiki-design.md`.
- **PR template.** Nothing in `.github/` yet. May want a standard PR template once code starts landing in Phase 1.

## Session artifacts noted but not recoverable

- An SSH keypair was generated in April 2026 for the aborted first attempt at artifact 2 (public key `ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIBHdTPDF3iDg2dE6QZDtKR4dQMv3dfcJaFck5IGxtPMn nanoclaw-spike-phase0`). The private key lived only in the session sandbox and is **gone**. Do not add that public key to Hetzner — it's unusable without its private key. Regenerate a fresh keypair when artifact 2 resumes.

---

End of Phase 0 plan. Update as artifacts land.

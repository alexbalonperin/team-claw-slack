# Project Brief — Team NanoClaw Deployment Platform

> Hand this document to Claude Code as the foundational context for the project.
> It defines what we're building, why, and how to start. Claude Code should
> read this fully before writing any code, and should produce a written plan
> for Phase 1 before touching any files.

## 1. What we're building, in one paragraph

A self-hosted deployment platform for personal AI assistants for an internal team of 10–15 people. Each team member gets their own NanoClaw instance running on their own dedicated VPS (Hetzner), with strong security isolation, per-user token isolation for third-party integrations (Gmail, Granola, Calendar, etc.), Kimi K2.6 (direct from Moonshot AI) as the underlying LLM provider, and Slack as the primary user-facing interface. The operator (one person, the user giving you this brief) provisions, updates, and monitors the fleet from a Vercel-hosted web UI backed by a Neon Postgres database, with a small persistent Hetzner control-plane VPS handling the operations that need a Tailscale-connected long-running process. End users (the team members) interact with their assistant entirely through Slack — they never SSH, never edit configs, never see the underlying VPS. All integration management happens inside Slack via the App Home tab. Each user names and personalizes their own assistant during onboarding.

## 2. Hard constraints — non-negotiable, validate before deviating

These are the architectural decisions already made. Do not relitigate without strong evidence. If something here turns out to be impossible, raise it explicitly with a written tradeoff analysis before changing course.

### Isolation and topology
- **One VPS per user.** Per-user isolation at the kernel level. No multi-tenant on one host. This is the supported NanoClaw security model and the blast-radius boundary we've committed to.
- **Hetzner Cloud as the VPS provider** for cost and API quality. CAX11 (ARM, ~€3.29/mo) as the default per-user size. Adapt later if specific users need more.
- **Each VPS sits behind a Hetzner Cloud Firewall** that allows only:
  - Tailscale's UDP port (41641 by default) inbound from anywhere.
  - SSH (port 22) inbound only from the operator's Tailscale IP — *not* from the public internet.
  - Outbound: unrestricted (the VPS needs to reach Moonshot, OAuth providers, etc.).
  Each VPS still has a public IPv4/IPv6 for Tailscale's NAT traversal but is otherwise unreachable from the internet. Verify this works — if Tailscale's NAT traversal fails consistently behind the firewall, fall back to a relay-only configuration.
- **Tailscale for all operator and inter-component connectivity.** The only public surfaces in the entire system are: (a) the Slack webhook receiver on Vercel, (b) the OAuth callback endpoints on Vercel, (c) the management web UI on Vercel. Nothing else is internet-reachable.

### LLM provider
- **Kimi K2.6 via Moonshot AI direct API**, not via OpenRouter. K2.6 is a significant upgrade over K2.5 and direct integration gives us better pricing and direct billing.
- LLM provider is **configurable per VPS**, not hardcoded. The current default is Moonshot; the architecture must allow per-user override (e.g. someone wants to test Claude or a local model) without code changes.
- **Per-user Moonshot API keys with per-user monthly spending caps** set at the Moonshot dashboard. The operator manages issuance.

### Agent runtime
- **NanoClaw, not OpenClaw.** Smaller codebase, container-isolated agents by design, native Anthropic Claude Agent SDK with provider skills for routing to non-Anthropic models.
- NanoClaw will use a custom provider skill (or `/add-opencode` configured to point at Moonshot) to route to Kimi K2.6. Phase 0 must verify this works end-to-end before locking it in.

### Secrets and tokens
- **Per-user secrets stay on the user's VPS, encrypted at rest with SOPS + age keys.** The control plane is a transit point for OAuth flows, never a vault. Plaintext tokens never persist outside the destination VPS.
- **Tokens never enter the agent's context window.** MCP servers hold tokens in their own process space and expose narrow, purpose-built tools to the agent.

### Slack
- **One Slack app, distributable, per-user installs (Tier 4).** Each user installs the app individually during onboarding, picks their assistant's name, and gets their own bot identity, token, and OAuth flow. Use `apps.manifest.update` per-install to set the per-user display name if technically feasible.
- **Slack App Home is the primary user-facing UI** for integration management. Connect, disconnect, reauth, see status, see spend — all happens inside Slack, rendered as Block Kit. End users have no other UI.

### Containerization
- **Docker is for component isolation within a VPS** (agent ↔ MCP servers ↔ secrets), not for user-vs-user isolation (that's the VPS boundary's job).

### Control plane
- **Hybrid control plane: Vercel + Neon Postgres + one persistent Hetzner control-plane VPS.** The pieces:
  - **Vercel-hosted Next.js app** for the operator UI, the Slack webhook receiver, the Slack OAuth callbacks, the third-party OAuth callbacks. Public surfaces only.
  - **Neon Postgres** for the inventory: users, VPS hosts, integration records, Slack install records, agent versions, audit log.
  - **Hetzner Cloud API called directly from Vercel** for provision/destroy operations (these are fast, atomic, idempotent).
  - **One persistent Hetzner control-plane VPS** (a CAX11, ~€3.29/mo, hardened identically to the per-user VPSes) running a small Node service. This is the only component on Tailscale that can reach the per-user VPSes. It exposes an authenticated HTTPS API to Vercel and handles:
    - Token shipping to per-user VPSes during OAuth flows (Vercel posts encrypted ciphertext + target VPS Tailscale IP; this VPS forwards over Tailscale).
    - Health-check polling and heartbeat aggregation.
    - Long-running fleet operations dispatched from Vercel.
    - The Slack-event-to-VPS routing path (Vercel webhook → control-plane VPS → target VPS).
  - **GitHub Actions as the heavy fleet operations worker** for things that benefit from full Ansible (multi-host updates, security audits, log pulls). Vercel dispatches `workflow_dispatch` events; the runner has SSH keys + Tailscale auth key in encrypted secrets, runs Ansible against the inventory, posts back via webhook.

The control-plane VPS is the smallest possible service — under 1000 lines of TypeScript, a few endpoints, no UI, no state of its own (state lives in Neon). Treat it as boring infrastructure that lives a long time.

### Operator UX
- The operator dislikes Claude Code scheduled tasks. Use cron, GitHub Actions Scheduled Workflows, or Neon's native scheduled jobs — not Claude Code's scheduler.

## 3. Soft constraints — preferences, override only with reason

- The operator's existing stack: Vercel, Supabase (now also Neon), Next.js, Claude Code, TypeScript. Lean into these.
- The team is in Tokyo / global. Hetzner EU is the default; verify Hetzner US-East or Singapore for Tokyo users if EU latency to Moonshot proves painful. Note that Moonshot's primary inference is in mainland China, so the relevant latency is `VPS region → Moonshot endpoint`, not `VPS region → user`.
- Security > cleverness. Boring, well-trodden patterns over novel ones.
- Operationally lazy in the right way: automate update rollouts, automate health checks, automate spending alerts, so the operator's ongoing burden is minutes/week not hours/week.

## 4. The four logical components

### 4.1 Vercel + Neon (public-facing control plane)

**Vercel app responsibilities:**
- Operator dashboard (provision, update, monitor, destroy users).
- Slack webhook receiver — validates signatures, looks up install record by `team_id` + `bot_user_id`, forwards events to the control-plane VPS for routing to the right user's per-user VPS.
- Slack OAuth flow — install URL, callback handler, install record persistence, refresh token handling, scope-update reauth handling.
- Third-party OAuth proxy — Google (Gmail + Calendar + Drive), Granola, Notion, Linear, Asana, etc. Receives callbacks, exchanges codes for tokens, encrypts to the user's VPS age public key, posts ciphertext to the control-plane VPS for delivery, drops plaintext from memory.
- App Home renderer — when Slack delivers `app_home_opened`, queries the control-plane VPS for current integration status (which the control-plane VPS has cached or fetches from the user's VPS) and renders Block Kit response.

**Neon database holds:**
- Users (operator-managed roster).
- VPS hosts (Hetzner instance ID, Tailscale IP, age public key, status, region, version).
- Slack installs (team ID, bot user ID, encrypted bot token, encrypted refresh token, scope set, install timestamp, user display name choice, avatar URL).
- Integration records (user ID, service, connection status, last refresh time, scopes granted, target VPS).
- Audit log (every operator action, every OAuth flow, every fleet update — append-only).
- Health pings (last heartbeat per VPS, last spend report, last audit pass).

**Vercel must hold long-running secrets:** Hetzner API token, Slack signing secret, Slack app credentials, third-party OAuth client IDs/secrets, the API token used to authenticate to the control-plane VPS. Use Vercel encrypted environment variables. Rotate quarterly.

**Vercel must NOT persist:** any user's plaintext bot tokens, integration tokens, or LLM API keys. These exist in memory only during OAuth flows, then are encrypted to the destination VPS and discarded.

### 4.2 Control-plane VPS (one persistent Hetzner CAX11, on Tailscale)

A small Node/Bun service, hardened identically to per-user VPSes (see §4.3), exposing an authenticated HTTPS API to Vercel. Responsibilities:

- **Token delivery:** Vercel posts `{target_tailscale_ip, encrypted_blob, integration_name}`; control-plane VPS POSTs to `https://<target>:port/secrets/<integration>` over Tailscale.
- **Slack event routing:** Vercel forwards Slack events with the target Tailscale IP; control-plane VPS POSTs to `https://<target>:port/slack/event`. Replies flow back through the same path.
- **Health polling:** every 5 minutes, hits `/health` on each registered VPS, writes results to Neon.
- **Heartbeat ingestion:** receives outbound heartbeats from each per-user VPS, writes to Neon.
- **Fleet command execution** for short operations (restart, status, log tail). Long-running multi-host operations (updates, audits) are still dispatched to GitHub Actions, but the control-plane VPS is the natural place for short single-host commands.

The control-plane VPS holds:
- A Tailscale node identity (tagged `tag:control-plane`).
- An API token used to authenticate Vercel's calls to it (rotated quarterly).
- The Hetzner API token (so it can update firewall rules, etc., when needed without a Vercel round-trip).
- No user secrets. No plaintext tokens. No private keys other than its own SSH and Tailscale.

Tailscale ACLs ensure that `tag:control-plane` can reach `tag:agent-vps` on specific ports, but `tag:agent-vps` nodes cannot reach each other. This is enforced at the Tailscale layer.

### 4.3 Per-user VPS (Hetzner CAX11, behind Hetzner Cloud Firewall, Tailscale-only access)

**Provisioned via Hetzner Cloud API + cloud-init.** First-boot cloud-init must complete the entire hardening checklist (§5) before the VPS is considered ready and accepts integration tokens.

**NanoClaw setup (after hardening completes):**
- Clone NanoClaw into `/opt/nanoclaw`.
- Generate a per-VPS gateway token (256-bit random, stored encrypted in SOPS).
- Render the compose file from a template, including:
  - NanoClaw container with `read_only: true`, `cap_drop: [ALL]`, `no-new-privileges:true`, non-root user.
  - Per-integration MCP server containers (initially none — added on integration connect).
  - Internal Docker network for agent ↔ MCP communication, isolated from other networks.
- Configure NanoClaw with the LLM provider (default Moonshot K2.6, configurable per VPS).
- Start NanoClaw via systemd unit that runs `docker compose up -d`.

**Tailscale-bound HTTP API for control plane:**
- `/health` — version, uptime, last LLM call timestamp, connected integrations, last 24h spend, last audit pass.
- `/secrets/<integration>` — receives encrypted token blobs, writes to SOPS-managed file.
- `/slack/event` — receives forwarded Slack events, returns the agent's response.
- All endpoints require an HMAC signature using a per-VPS shared secret derived at provisioning, even though Tailscale already gates access. Defense in depth.

**Outbound heartbeat to control-plane VPS** every 24 hours with health data.

**Per-VPS scoped Moonshot API key** stored in SOPS, with a monthly spending cap configured at Moonshot.

### 4.4 Slack workspace (one app, many per-user installs)

- One distributable Slack app, owned by the operator, installable into the team's workspace.
- Each user installs the app during onboarding via OAuth; picks their assistant's name and avatar in a Vercel onboarding page.
- Each install creates a separate bot identity, bot token, refresh token, and event subscription record.
- All installs point to the same webhook URL on Vercel.
- The webhook router uses `team_id` + `bot_user_id` from each event to identify which install (and thus which user's VPS) the event belongs to, and forwards via the control-plane VPS.
- App Home tab is the in-Slack management UI for each user — connect/disconnect integrations, see status, see spend.
- All `chat.postMessage` calls use `username` + `icon_url` overrides from the user's chosen identity.
- If `apps.manifest.update` per-install can set the workspace display name (Phase 0 must verify), use it for full per-user identity.

## 5. VPS hardening checklist — applies to per-user VPSes AND the control-plane VPS

This is the full hardening checklist that cloud-init must complete on first boot. Both the per-user VPSes and the control-plane VPS use this same baseline. Items are grouped by category.

### 5.1 Patching and package management
- Update all installed packages to current security versions on first boot.
- Install and enable `unattended-upgrades` configured for security and important updates only (no full distribution upgrades unattended).
- Configure `unattended-upgrades` to email the operator (via SMTP relay or systemd-journal alerting) on failure and to auto-reboot at 04:00 UTC if a kernel update requires it. Enable `Unattended-Upgrade::Automatic-Reboot-WithUsers "false"` so the reboot is delayed if a user session is active.
- Pin Docker, Tailscale, and any other added APT repositories with explicit GPG key verification. Refuse unsigned repos.
- Schedule a weekly `apt autoremove` to keep the package set tight.

### 5.2 User and SSH hardening
- Create a non-root sudo user (`agent` for per-user VPSes, `cp-admin` for the control-plane VPS).
- Lock the root account password (`passwd -l root`). Root login is impossible except via su from sudo.
- SSH config (`/etc/ssh/sshd_config.d/99-hardening.conf`):
  - `PasswordAuthentication no`
  - `PermitRootLogin no`
  - `PubkeyAuthentication yes`
  - `AuthenticationMethods publickey`
  - `PermitEmptyPasswords no`
  - `ChallengeResponseAuthentication no`
  - `KbdInteractiveAuthentication no`
  - `UsePAM yes` (still needed for session management)
  - `X11Forwarding no`
  - `AllowAgentForwarding no`
  - `AllowTcpForwarding no` (override if you genuinely need port forwarding)
  - `MaxAuthTries 3`
  - `LoginGraceTime 20`
  - `ClientAliveInterval 300`
  - `ClientAliveCountMax 2`
  - `Protocol 2`
  - `AllowUsers agent` (or `cp-admin` for control plane) — explicit allowlist.
  - Restrict to modern key exchange, ciphers, and MACs — disable all SHA1, 3DES, CBC ciphers, weak DH.
- Operator SSH key is deployed via cloud-init from a single source of truth (the Hetzner project's SSH keys section, or fetched from Vercel during provisioning).
- Run `sshd -T` after configuration to validate; fail provisioning if the config is invalid.

### 5.3 Firewall (defense in depth: Hetzner Cloud Firewall + UFW on host)
- Hetzner Cloud Firewall as the outer layer (configured via Hetzner API at provisioning time):
  - Inbound: Tailscale UDP port (41641) from anywhere; SSH (22) from operator's Tailscale IP only; everything else denied.
  - Outbound: unrestricted.
- UFW on the host as the inner layer:
  - `ufw default deny incoming`
  - `ufw default allow outgoing`
  - `ufw allow in on tailscale0` (allow all inbound on Tailscale interface)
  - `ufw allow 41641/udp` (Tailscale's NAT traversal)
  - `ufw enable`
- Verify with a deliberate test from an unrelated host that no port other than 41641/udp responds. Fail provisioning if anything else is reachable.

### 5.4 Brute-force and intrusion mitigation
- Install `fail2ban` with the `sshd` jail enabled.
  - `bantime = 24h` (aggressive; SSH should never be hit by anything legitimate after lockdown).
  - `findtime = 10m`
  - `maxretry = 3`
  - `bantime.increment = true`, `bantime.factor = 4` for repeat offenders.
- Configure fail2ban to log to systemd-journal so failures surface in normal log review.
- Add an additional jail for repeated 401/403 responses on the control-plane VPS's HTTPS API.

### 5.5 Kernel and sysctl hardening
Apply the following in `/etc/sysctl.d/99-hardening.conf` and reload:
- Network:
  - `net.ipv4.tcp_syncookies = 1`
  - `net.ipv4.conf.all.rp_filter = 1`
  - `net.ipv4.conf.default.rp_filter = 1`
  - `net.ipv4.conf.all.accept_source_route = 0`
  - `net.ipv4.conf.default.accept_source_route = 0`
  - `net.ipv4.conf.all.accept_redirects = 0`
  - `net.ipv4.conf.default.accept_redirects = 0`
  - `net.ipv4.conf.all.send_redirects = 0`
  - `net.ipv4.conf.default.send_redirects = 0`
  - `net.ipv4.conf.all.log_martians = 1`
  - `net.ipv4.icmp_echo_ignore_broadcasts = 1`
  - `net.ipv4.icmp_ignore_bogus_error_responses = 1`
  - `net.ipv4.tcp_rfc1337 = 1`
  - IPv6 equivalents for all of the above.
- Memory and process protection:
  - `kernel.randomize_va_space = 2` (full ASLR; default on modern Ubuntu but verify).
  - `kernel.kptr_restrict = 2` (hide kernel pointers).
  - `kernel.dmesg_restrict = 1` (only root can read dmesg).
  - `kernel.yama.ptrace_scope = 2` (only root can ptrace; required for many debuggers, but right tradeoff here).
  - `fs.protected_hardlinks = 1`
  - `fs.protected_symlinks = 1`
  - `fs.protected_fifos = 2`
  - `fs.protected_regular = 2`
  - `fs.suid_dumpable = 0` (don't dump SUID processes).
- Disable unused protocols at the module level via `/etc/modprobe.d/blacklist-rare-protocols.conf`: `dccp`, `sctp`, `rds`, `tipc` (not needed by anything we run).

### 5.6 AppArmor / mandatory access control
- Verify AppArmor is enabled and in enforce mode (`aa-status`).
- Ensure Docker's AppArmor profile is enforced for containers.
- For the agent and MCP server containers, write narrow AppArmor profiles that deny network access except to documented endpoints, deny filesystem access outside their workspace, and deny `ptrace` and other introspection capabilities. (Phase 1 ships permissive profiles; Phase 4 tightens.)

### 5.7 Auditing and integrity
- Install `auditd` with a baseline ruleset:
  - Identity changes (`/etc/passwd`, `/etc/shadow`, `/etc/group`, `/etc/sudoers`, `/etc/sudoers.d/`).
  - SSH config (`/etc/ssh/`).
  - Time changes (`adjtimex`, `settimeofday`, `clock_settime`).
  - Module loading (`init_module`, `delete_module`).
  - Privilege escalation (`setuid`, `setgid`, `chown`, `chmod` on sensitive paths).
  - Network config (`/etc/network/`, `/etc/hosts`, `/etc/resolv.conf`).
  - Cron and systemd unit changes.
  - All `execve` calls in Docker volume mount points.
- Configure `journald` to persist logs across reboots (`Storage=persistent`) with a sensible retention window (`SystemMaxUse=2G`, `MaxRetentionSec=90d`).

**AIDE configuration — must be tuned, not default.** AIDE's default ruleset on a Docker host produces hundreds of false positives daily because it monitors `/var/lib/docker/`, `/var/log/`, package databases, and other paths that legitimately change every minute. An untuned AIDE configuration is worse than no AIDE — operators learn to ignore the alerts and real intrusions get lost in the noise. The configuration below is the result of accepting that signal-to-noise matters more than raw coverage.

**Strict monitoring (every aspect: hash, size, perms, owner, mtime, inode):**
- `/boot/` — kernel and bootloader
- `/etc/` — system configuration (with the exclusions below)
- `/usr/bin/`, `/usr/sbin/`, `/bin/`, `/sbin/` — system binaries
- `/usr/lib/`, `/usr/lib64/`, `/lib/`, `/lib64/` — system libraries (excluding subdirs that legitimately churn)
- `/lib/modules/` — kernel modules
- `/etc/ssh/` — SSH configuration and host keys
- `/etc/sudoers`, `/etc/sudoers.d/` — sudo rules
- `/etc/passwd`, `/etc/shadow`, `/etc/group`, `/etc/gshadow` — identity files
- `/root/.ssh/` — root SSH keys (should not exist, but monitor in case it appears)
- `/home/*/.ssh/authorized_keys` — user SSH keys
- `/etc/sops/age/` — the SOPS age keypair itself. **Critical** — if the private key changes, immediate alert.
- `/etc/cron.d/`, `/etc/cron.daily/`, `/etc/cron.hourly/`, `/etc/cron.weekly/`, `/etc/cron.monthly/`, `/etc/crontab`
- `/etc/systemd/system/` — systemd unit files

**Permissions/ownership only (don't alert on contents changing, but alert if mode or owner changes):**
- `/var/log/` — logs change constantly, but if a log file's owner or permissions change, that's suspicious.

**Excluded entirely (expected churn, no security signal in monitoring them):**
- `/var/lib/docker/` — Docker daemon state, changes on every container action.
- `/var/lib/containerd/` — same.
- `/var/lib/nanoclaw/state/` — agent state, changes constantly.
- `/var/lib/apt/`, `/var/cache/apt/`, `/var/log/apt/` — package manager state, changes on every patch run.
- `/var/lib/dpkg/` — same, except `/var/lib/dpkg/status` and `/var/lib/dpkg/info/` which are useful for detecting unauthorized package installs (so monitor those specifically).
- `/var/lib/fail2ban/` — fail2ban database.
- `/var/lib/chrony/` — chrony drift file.
- `/var/log/journal/` — journald binary logs.
- `/tmp/`, `/var/tmp/` — by definition transient.
- `/proc/`, `/sys/`, `/dev/` — virtual filesystems.
- `/run/` — runtime state.
- The encrypted SOPS token blobs (`/var/lib/nanoclaw/secrets/*.enc`) — these legitimately change on every token refresh. The age private key that decrypts them is monitored strictly above; that's the security boundary that matters.

**Operational pattern:**
- Initialize the AIDE database **after** all of cloud-init completes (so the baseline includes the final hardened state, not the bootstrap state).
- Schedule a daily check at 04:00 UTC via cron.
- On mismatch, send a structured alert to the control-plane VPS (not email — email gets ignored). The control-plane VPS aggregates alerts and surfaces them in the Vercel dashboard with severity grading.
- After legitimate operator changes (e.g. installing a new MCP server, applying a NanoClaw update), update the AIDE baseline as part of the change procedure. Do not let the baseline drift.
- The Phase 4 healthcheck script verifies that the AIDE baseline timestamp is within an expected window for the current NanoClaw version — too old means baseline-update was skipped after an upgrade.

**Tuning expectation for Phase 0:** during the NanoClaw spike, run AIDE with this configuration for 7 days and count alerts. **The target is fewer than 1 alert per week under normal operation.** If the count is higher, identify the noisy path and add it to the exclusion list — but only after confirming it's genuinely expected churn, not real activity worth seeing. Document each exclusion decision in `docs/[aide-tuning.md](http://aide-tuning.md)`.

### 5.8 Time, NTP, and clock integrity
- Use `chrony` (preferred over `systemd-timesyncd` for accuracy and ACL support).
- Configure to sync with multiple Stratum-2 sources (`pool [2.pool.ntp.org](http://2.pool.ntp.org) iburst` and a Hetzner internal NTP if available).
- Reject NTP packets from unknown sources (`allow` directive scoped to [localhost](http://localhost) only; the VPS does not serve time).

### 5.9 Container runtime hardening
- Docker daemon config (`/etc/docker/daemon.json`):
  - `"icc": false` (disable inter-container communication on the default bridge; we use explicit user-defined networks).
  - `"userland-proxy": false`
  - `"no-new-privileges": true` as default.
  - `"live-restore": true` (containers survive daemon restarts).
  - `"log-driver": "journald"` (logs flow through systemd, retained per the journald config).
  - `"userns-remap": "default"` (user namespace remapping — significantly reduces container-escape impact). Test this with NanoClaw and MCP servers in Phase 0; some images break under userns. If something breaks, document and skip — do not disable as a workaround.
- All containers run with:
  - `read_only: true` filesystems
  - `cap_drop: [ALL]` and `cap_add` only the minimum needed (typically nothing)
  - `no-new-privileges: true`
  - Non-root `user:` directive
  - Explicit `mem_limit`, `cpus`, and `pids_limit` to prevent resource exhaustion.
  - tmpfs mounts for any directory the process needs to write to, with `noexec,nosuid` flags.

### 5.10 Filesystem and storage
- Separate partitions where possible: `/`, `/var`, `/var/log`, `/tmp`, `/home`. Mount `/tmp` and `/var/tmp` with `noexec,nosuid,nodev`. Mount `/home` with `nosuid,nodev` (we don't run setuid binaries from user homes).
- Set strict permissions on cloud-init artifacts: `/var/lib/cloud/instance/user-data.txt` should not be world-readable since it may briefly contain provisioning secrets.
- Encrypt the data partition where SOPS state and Docker volumes live with LUKS, with the key stored in TPM (if available on the Hetzner Cloud instance) or fetched from Hetzner's metadata service via cloud-init. For Phase 1 this is optional; for Phase 4 it should be standard. Document the tradeoff if skipped.

### 5.11 Application-layer secrets (SOPS + age)
- Install `age` and `sops` from official sources, pinned to specific versions.
- Generate the per-VPS age keypair on first boot. Private key in `/etc/sops/age/keys.txt`, mode `0400`, owned by the user that runs the MCP servers.
- Public key is exported to the control-plane VPS via Tailscale-authenticated POST during provisioning.
- Private key never leaves the VPS. Never logged. Never appears in heartbeats.
- The HMAC shared secret between control-plane VPS and this VPS (used to authenticate inbound API calls) is generated on first boot, returned to the control-plane VPS during provisioning registration, and stored in SOPS thereafter.

### 5.12 Logging and observability
- Forward structured logs to the control-plane VPS via a Tailscale-bound syslog endpoint (or to a small Loki instance if we add one in Phase 4). For Phase 1 local journald with persistent storage is acceptable.
- Log rotation via `logrotate` for any non-journald logs.
- A daily cron emits a summary of: number of failed SSH attempts (should be zero post-lockdown), number of fail2ban bans, number of auditd alerts, container restart count, AIDE integrity check result. This summary is sent to the control-plane VPS as part of the heartbeat.

### 5.13 Misc
- Disable unused services: CUPS, Avahi, ModemManager, snapd (we use APT, not Snap, for everything).
- Set timezone to UTC at the OS level (Tokyo time only at the application layer when displaying to users).
- Set `umask 027` system-wide so newly created files default to group-readable but not world-readable.
- Configure `motd` to display "Authorized access only. Activity is logged." for legal cover (this matters in some jurisdictions).
- Disable kernel core dumps for setuid programs (`fs.suid_dumpable = 0`, set above) and limit core dump size to zero in `/etc/security/limits.conf` for non-debug use.
- Set up a `/healthcheck` script that runs all of the above checks (firewall rules, SSH config, auditd rules, fail2ban status, sysctl values, AppArmor enforce, AIDE last-check freshness) and returns a single pass/fail. Run it nightly. Failures alert the operator.

### 5.14 Hardening items that may need adaptation — validate during Phase 0

The hardening checklist is the *target state*. Some items are known to occasionally interact badly with Docker, NanoClaw, or specific MCP servers in ways that aren't obvious until you try them. **Validate each of the items below during the Phase 0 spike. If an item demonstrably breaks something we need, document the failure mode, document the chosen workaround, and document the security cost. Do not silently disable, do not fight a working setup to enforce a checkbox.**

The "ship a working hardened-enough deployment" bar is higher than the "achieve every item on the checklist" bar.

Items most likely to need adaptation:

- **`userns-remap` in Docker daemon config (§5.9).** Some container images expect to run as root inside the container with UID 0 mapped to host UID 0. Remapping UIDs can break volume mounts, file ownership in bind mounts, and certain init systems. Test NanoClaw and each planned MCP server image. If userns-remap breaks an image we need, fall back to running without it but with `cap_drop: [ALL]` and `no-new-privileges` (which we have anyway). Document the tradeoff.
- **AppArmor enforce mode for containers (§5.6).** Default Docker AppArmor profile (`docker-default`) is fine for most workloads. Custom profiles are deferred to Phase 4 specifically because they often require tuning per image. Don't ship custom profiles in Phase 1.
- **Kernel module blacklist (§5.5).** DCCP, SCTP, RDS, TIPC are not used by anything we run, so this should be safe — but if Tailscale or NanoClaw surprise us by needing one of these, remove it from the blacklist rather than disabling Tailscale. (Vanishingly unlikely; mentioned for completeness.)
- **`AllowTcpForwarding no` in SSH config (§5.2).** If you ever need to SSH-tunnel to a per-VPS endpoint for debugging, this blocks it. The intended workflow is "use Tailscale for everything, never tunnel through SSH," so this should be fine — but if operator workflows discover a real need, override per-host rather than fleet-wide.
- **`ptrace_scope = 2` (§5.5).** Some debuggers and `strace`-based tools won't work even for root. Acceptable for production; if Phase 0 spike needs debugging tools, temporarily relax during spike, restore for fleet rollout.
- **Mount `noexec` on `/tmp` and `/var/tmp` (§5.10).** Some package installers and language runtimes (notably `pip`, occasionally `npm`) drop executable shims to `/tmp`. We're not running these in production paths, but if cloud-init or unattended-upgrades trips on `noexec /tmp`, document and consider remounting only the host's `/tmp` (containers have their own tmpfs anyway).
- **AIDE configuration (§5.7).** As discussed above. The whole point of §5.7 is to ship a tuned configuration that produces near-zero false positives. If Phase 0 finds the tuned config still produces too much noise, tune further before shipping — don't ship a noisy AIDE that operators will learn to ignore.

For each item that needs adaptation, the Phase 0 deliverable `docs/[nanoclaw-baseline.md](http://nanoclaw-baseline.md)` must document:
1. What broke and how it manifested.
2. What was tried before falling back.
3. The chosen workaround.
4. The security tradeoff in concrete terms ("we lose X, we keep Y").
5. Whether this is a per-VPS decision or fleet-wide.

### 5.15 What we explicitly skip (with reasoning)
- **SELinux:** Ubuntu uses AppArmor; switching to SELinux would be a significant rewrite for marginal gain.
- **Kernel hardening patches (grsecurity, etc.):** out of scope; we rely on upstream kernel hardening.
- **Disk encryption with manual passphrase entry on every boot:** would require interactive operator presence on every reboot; not viable for a fleet of 15.
- **HIDS like OSSEC or Wazuh:** auditd + AIDE + fail2ban + the daily healthcheck cover the same ground for our scale. Revisit if the fleet grows past 50.

## 6. Strong starting point — fork ClawHost, but verify first

There is an existing open-source project, **`bfzli/clawhost`** (https://github.com/bfzli/clawhost, MIT-licensed), that handles much of the management UI we need for OpenClaw. It is the closest published match to what we are building, though it targets OpenClaw rather than NanoClaw and assumes a persistent backend rather than Vercel + Neon + a small control-plane VPS. **Before writing any code, do a deep evaluation of ClawHost** against this brief.

### 6.1 What ClawHost reportedly does (verify each, do not trust)

- One-click VPS deployment to Hetzner.
- Automatic Cloudflare DNS subdomain creation.
- Automatic Let's Encrypt SSL.
- Browser-based SSH terminal via WebSocket.
- Diagnostics, logs, file management.
- Version management with upgrade buttons.
- SSH key management.
- Persistent storage volume attachment.
- Multi-auth (email OTP, Google, GitHub).
- [Polar.sh](http://Polar.sh)-integrated billing (probably not needed for internal team use).
- Web, mobile (iOS/Android), and desktop (macOS/Linux) clients.
- Cloud-init templating for VPS provisioning.
- TypeScript monorepo: Hono.js backend, React+Vite frontend, Bun + Turborepo, React Native mobile, Electron desktop.

### 6.2 The deep-dive task — do this before forking

Clone the repo. Read the code. Produce a written evaluation document, `docs/[clawhost-evaluation.md](http://clawhost-evaluation.md)`, with the following structure:

For each ClawHost feature, evaluate:

1. **Does it exist?** (Some features in marketing copy may be aspirational.)
2. **Does it work?** (Run it. Does the provisioning flow actually complete?)
3. **Does it map to our needs?** Tag each feature as one of:
   - **KEEP** — useful as-is, no changes.
   - **ADAPT** — useful but needs modification (e.g. swap OpenClaw for NanoClaw in the cloud-init).
   - **REPLACE** — concept is right but implementation doesn't fit (e.g. we want Vercel+Neon+small-VPS, not Hono.js with its own backend; or we want NanoClaw, not OpenClaw).
   - **REMOVE** — not needed for our use case (Polar billing, mobile/desktop clients).
   - **MISSING** — we need this and ClawHost doesn't have it.
4. **What's the cost of adapting vs rebuilding?** Be concrete (hours/days).

The features we *know* are missing in ClawHost and which we'll need to build:

- NanoClaw-specific cloud-init with the full hardening checklist from §5.
- Vercel+Neon control plane (ClawHost uses Hono.js with a self-hosted backend).
- The small Hetzner control-plane VPS service for token delivery and Slack event routing.
- Slack app — distributable, OAuth-based per-user installs, per-user-bot routing.
- Per-user integration management (Gmail, Calendar, Granola, etc.) with OAuth proxy and SOPS-encrypted token shipping.
- App Home tab as in-Slack management UI.
- Fleet update orchestration (canary first, then staggered rollout, with rollback) — implemented as GitHub Actions workflows.
- Token rotation handler for Slack tokens (Slack rotates them every ~12h with rotation enabled).
- Per-user Moonshot key provisioning and spend monitoring.
- Webhook receivers for push-based integrations (Granola transcripts, Calendar events).
- Heartbeat/health check ingestion from each VPS.
- Hetzner Cloud Firewall configuration as part of provisioning.

### 6.3 Decision after the deep dive

After producing `docs/[clawhost-evaluation.md](http://clawhost-evaluation.md)`, propose one of these paths:

- **Fork-and-adapt:** Fork ClawHost, modify cloud-init for NanoClaw + the §5 hardening + Tailscale + SOPS, port the backend from Hono.js to Vercel functions + Neon + the control-plane VPS, add the missing services. Best if KEEP+ADAPT is ≥60% of needed features after accounting for the backend port.
- **Cherry-pick patterns, build fresh:** Don't fork; build a new minimal Next.js app on Vercel modeled on ClawHost's good ideas (especially the cloud-init template structure and the inventory model). Likely best given the architecture mismatch.
- **Use-as-is:** Use vanilla ClawHost for OpenClaw, build only the Slack router and integrations as separate services. Ruled out because we want NanoClaw and Vercel+Neon.

Given the architecture mismatch, the realistic answer is probably "cherry-pick patterns, build fresh," but the deep dive should confirm this rather than assume it.

The operator should approve the path before any code is written.

## 7. Other ecosystem projects worth knowing about

Mention these in the evaluation doc and consider whether any patterns or code can be borrowed.

- **`jomafilms/openclaw-multitenant` (OCMT / OpenPaw)** — multi-tenant fork of OpenClaw with container isolation and encrypted vault. Architecturally the wrong direction for us (multi-tenant on one host) but the encrypted vault implementation may have useful patterns.
- **OpenClaw issue #17299 ("Agents Plane")** — the official thinking on enterprise multi-tenant deployment. Read it for design context.
- **OpenClaw issue #61123** — the family/small-team multi-tenant feature request. Read for the same reason.
- **Blink Claw** — commercial managed hosting that runs each agent on isolated [Fly.io](http://Fly.io) machines. Useful reference for "what does polished look like."
- **NanoClaw README at github.com/qwibitai/nanoclaw** — read fully. Pay attention to the `/add-opencode`, `/add-codex`, channel skill, and provider skill mechanisms.
- **Moonshot AI Kimi K2.6 docs** — verify the API, pricing, regional endpoint availability, billing options, and whether the provider exposes OpenAI-compatible endpoints (which would simplify NanoClaw integration via `/add-opencode` pointed at Moonshot's base URL).

## 8. Phased build plan

Each phase ends with a working, demoable artifact. Do not skip phases. Do not parallelize phases without explicit approval.

### Phase 0 — Evaluation and design (no code yet)

1. Read this brief in full.
2. Deep-dive ClawHost as described in §6.2. Produce `docs/[clawhost-evaluation.md](http://clawhost-evaluation.md)`.
3. Spike NanoClaw locally on a single Hetzner CAX11 — manually, end to end, with the full §5 hardening checklist applied, to understand the moving parts. Produce `docs/[nanoclaw-baseline.md](http://nanoclaw-baseline.md)` documenting what you actually had to do, what broke, what surprised you, and which §5 items needed adaptation. Particular focus: does the CAX11 (ARM, 4GB) handle NanoClaw + 3-4 MCP sidecars comfortably with all hardening enabled?
4. Verify Kimi K2.6 via Moonshot direct API works with NanoClaw. If Moonshot exposes an OpenAI-compatible endpoint, configure `/add-opencode` to point at it. If not, write a small custom provider skill. Produce `docs/[kimi-k26-via-moonshot.md](http://kimi-k26-via-moonshot.md)` with:
   - Confirmed endpoint URL and auth method.
   - Cost estimates per typical agentic interaction.
   - Latency measurements from Hetzner-EU and Hetzner-US (and Hetzner-Singapore if available) to Moonshot.
   - Whether per-user spending caps are configurable at Moonshot.
   - Data residency posture (where does inference happen, what's logged, what's the retention policy).
5. Verify Slack distributable app with per-user installs works as expected, including whether `apps.manifest.update` can set per-user display names. Produce `docs/[slack-multi-install-validation.md](http://slack-multi-install-validation.md)`.
6. Verify Hetzner Cloud Firewall + Tailscale interplay. Specifically: can a VPS with no public SSH still receive Tailscale connections? Does Tailscale's NAT traversal work through a deny-all-but-UDP-41641 firewall, or do we need a relay? Produce `docs/[hetzner-tailscale-firewall.md](http://hetzner-tailscale-firewall.md)`.
7. Propose architecture and tech choices in `docs/[architecture.md](http://architecture.md)`. Get operator approval before Phase 1.

### Phase 1 — Single-user end-to-end (the foundation)

Goal: One real teammate (or the operator themselves) using their assistant in Slack, with one integration (Calendar) connected, all the security patterns in place.

1. **Provisioning script:** Given a Hetzner API token and a Tailscale auth key, provision a CAX11, configure the Hetzner Cloud Firewall, run cloud-init with the full §5 hardening checklist, end up with NanoClaw running and reachable over Tailscale. Idempotent and rerunnable. The provisioning script must run the §5.13 healthcheck after first boot and refuse to mark the VPS as ready unless it passes.
2. **Control-plane VPS:** Provision the persistent Hetzner control-plane VPS using the same hardening as per-user VPSes. Deploy the small Node service handling token delivery, Slack event routing, and health polling.
3. **SOPS + age setup on per-user VPS:** Cloud-init generates an age keypair on first boot. Public key is exported back via the control-plane VPS to Vercel. Private key never leaves the VPS.
4. **Slack app:** One distributable app with OAuth install flow. The flow lands on a Vercel page, captures user-chosen assistant name and avatar, completes Slack OAuth, stores the install record in Neon (encrypted bot token + refresh token).
5. **Slack router (Vercel + control-plane VPS):** Vercel receives Slack events, identifies the install via `team_id` + `bot_user_id`, posts to control-plane VPS with target Tailscale IP, control-plane VPS forwards to the right NanoClaw, response flows back the same way. Vercel posts the reply to Slack using the install's bot token plus per-user `username`/`icon_url` overrides.
6. **OAuth proxy for Calendar:** Single-service first. User clicks "Connect Calendar" in App Home → Google OAuth → Vercel encrypts token to user's VPS age public key → posts ciphertext to control-plane VPS → control-plane VPS POSTs to user's VPS over Tailscale → Calendar MCP server picks it up.
7. **Calendar MCP sidecar:** Runs on the user's VPS, mounts the SOPS-encrypted secrets dir read-only, exposes Calendar tools to NanoClaw via the standard MCP protocol. Tokens never enter NanoClaw's context.
8. **Smoke tests:** Send a Slack DM to the assistant, ask it about today's schedule, verify it calls the Calendar tool and returns a sensible answer. Verify with a deliberate test that the agent cannot retrieve the Calendar token even when prompted to.

Phase 1 ends when the operator can demo: install the Slack app from a link, name their assistant, connect Calendar, message the bot, get a real answer that came from a Calendar API call, with the LLM running on Kimi K2.6 via Moonshot direct and the token stored encrypted on a hardened, firewalled, Tailscale-only Hetzner VPS.

### Phase 2 — Management UI and fleet operations

Goal: The operator can provision, update, and monitor multiple users from one Vercel dashboard.

1. Vercel + Neon management UI (per Phase 0 decision: forked-and-ported ClawHost or fresh build).
2. "Add user" flow: operator creates a user record in the dashboard → Slack install link is generated → on first install completion, a VPS is auto-provisioned with that user's age public key and Tailscale auth, registered in Neon, healthcheck passes before the user gets the "ready" signal.
3. "Update fleet" with canary, implemented as a GitHub Actions workflow:
   - Operator clicks "update" in the dashboard.
   - Vercel dispatches `workflow_dispatch` with target version.
   - Action joins Tailscale, runs Ansible against the canary VPS first.
   - Health checks run after canary update; on green, action proceeds to staggered rollout.
   - On red, action halts and posts back to Vercel with failure details.
   - All actions logged to Neon's audit log.
4. Health monitoring: each VPS posts a daily heartbeat to the control-plane VPS, which writes to Neon. Dashboard shows red/yellow/green per user based on heartbeat freshness, AIDE results, fail2ban activity, audit findings.
5. Spend monitoring: aggregate Moonshot usage per user (via Moonshot's API or via per-VPS heartbeats), alert if any user crosses 80% of their cap.

Phase 2 ends when the operator can onboard a new user end-to-end (including Slack install, VPS provision, Calendar connection) in under 10 minutes via the dashboard, and roll an update to all users in one click.

### Phase 3 — More integrations and polish

Goal: Real daily use across the team.

1. Add Gmail integration (highest demand, most-requested).
2. Add Granola integration (push-based webhook for new transcripts, routed by Granola account email).
3. App Home tab: full integration management UI inside Slack — connect, disconnect, reauth, see status, see spend per integration.
4. Per-user agent personalization: avatar, persona block in [SOUL.md](http://SOUL.md), optional per-user system prompt.
5. Reauth flow when Slack scopes are added: lazy reauth via "I gained new abilities" prompt on next interaction.
6. Onboarding doc / runbook for the operator — what to do when something breaks, how to rebuild a user from scratch, how to rotate keys.

### Phase 4 — Production hardening

Goal: This thing can run for a year without the operator losing sleep.

1. Backup strategy: nightly encrypted snapshots of each VPS's `/var/lib/nanoclaw/` to Hetzner Storage Box. Retention: 7 daily, 4 weekly, 6 monthly.
2. Disaster recovery runbook: rebuild a user's VPS from a clean image, restore state from snapshot, in under 30 minutes.
3. Tighten AppArmor profiles for agent and MCP server containers (§5.6 deferred items).
4. Enable LUKS encryption on data partitions if not done in Phase 1 (§5.10 deferred item).
5. Security audit: walk through the full token lifecycle and verify no plaintext leaks. Run the NanoClaw audit (or OpenClaw equivalent) across the fleet on a schedule via GitHub Actions.
6. Cost report: monthly summary per user (VPS + LLM + storage), surfaced in the dashboard.
7. Off-boarding flow: when a user leaves, one click to revoke OAuth grants, snapshot, archive, and destroy the VPS. Audit log retained.
8. Quarterly secret rotation: control plane secrets in Vercel, Hetzner API token on the control-plane VPS, all third-party OAuth client secrets, fleet SSH key, Tailscale auth keys.

## 9. Specific guidance on tricky bits

### 9.1 Slack token rotation

Slack rotates bot tokens every ~12 hours when rotation is enabled (which it should be). Build the refresh handler from Phase 1. Without it, the fleet works for a day, then half of it is dead. Refresh tokens are stored alongside bot tokens in the install record in Neon. When a token is within 1 hour of expiry, refresh it; if refresh fails, mark the install as "needs reauth" and prompt the user via App Home banner.

### 9.2 Per-user bot display name

Two layers, combined:
- **Cosmetic layer:** every `chat.postMessage` call from the router uses `username` and `icon_url` overrides set to the user's chosen name and avatar. Works in DMs and most channels. Cheap, always works.
- **Identity layer:** after OAuth completes, call `apps.manifest.update` to set the bot's display name in the workspace member list to the user's chosen name. May or may not work depending on Slack tier and current API rules — verify in Phase 0. If it works, use it. If not, accept that mention autocomplete shows a generic name; replies still appear personalized.

### 9.3 Token isolation from the agent

The MCP server holds the token. The agent calls narrow tools by name. Tool schemas never include credentials as parameters. Tool responses never include credentials. The agent's context window contains tool definitions and tool results, never tokens. Verify this with a deliberate test in Phase 1: prompt the agent to "tell me the value of the Gmail token" and confirm it cannot.

The MCP server should not expose generic tools like `gmail.raw_request(headers, body)` that would let a prompt-injected agent construct arbitrary calls. Every exposed tool should be purpose-built and bounded: `list_messages`, `get_message`, `send_message`, `archive`. The MCP server is a capability surface, not a credential proxy.

### 9.4 OAuth token shipping

End-to-end flow:
1. User clicks "Connect Gmail" in App Home.
2. Slack opens browser tab to the Vercel OAuth-start endpoint.
3. Vercel redirects to Google with scopes and a state parameter that encodes the user identifier and a CSRF token.
4. User consents at Google.
5. Google redirects to Vercel callback with auth code.
6. Vercel exchanges auth code for token (held in memory only).
7. Vercel fetches the user's age public key from Neon.
8. Vercel encrypts `{access_token, refresh_token, expires_at, scope}` with the age public key.
9. Vercel posts `{target_tailscale_ip, encrypted_blob, integration_name}` to the control-plane VPS authenticated endpoint.
10. Vercel drops the plaintext from memory.
11. Control-plane VPS POSTs the ciphertext to `https://<target>:port/secrets/gmail` over Tailscale, authenticated with the per-VPS HMAC shared secret.
12. The target VPS endpoint writes the ciphertext to a SOPS-managed file owned by the relevant MCP server's user.
13. The MCP server reads and decrypts at startup (or on next call), holds the token in memory, refreshes locally as needed.

Vercel must never log token values, never persist them to Neon, never include them in error messages. The control-plane VPS forwards bytes only — does not parse or log the encrypted blob. Test this with a deliberate audit in Phase 4.

### 9.5 Fleet updates via GitHub Actions

- Pin NanoClaw image versions in compose. Do not track `latest`.
- Update flow: operator clicks "update" in Vercel UI → Vercel calls GitHub API to dispatch `workflow_dispatch` with `target_version` and `target_hosts` parameters → GitHub runner spins up → joins Tailscale via auth key from secrets → runs Ansible against canary first → health check → on green, batched rollout to remaining hosts with N-minute delay between batches → posts results back to Vercel webhook → Vercel updates Neon and dashboard.
- Each VPS snapshots its `/var/lib/nanoclaw/state` to local backup before updating. On health check failure, automatic rollback: revert image tag, restore state snapshot, restart, alert operator.
- Notify users on the relevant Slack DMs at the start and end of the maintenance window. Automate this via the Slack router.

### 9.6 What we're not building

To prevent scope creep, the following are explicitly out of scope unless the operator says otherwise later:

- Multi-workspace Slack app distribution (we're a single-workspace internal tool).
- Mobile or desktop clients for the management UI (the Vercel web UI is sufficient for an operator with 15 users).
- Public-facing landing page or marketing site.
- Billing or payment integration (this is internal team tooling; if [Polar.sh](http://Polar.sh) comes via ClawHost, rip it out).
- Shared agent identities or team agents (each user has their own personal agent only).
- LLM cost optimization beyond per-user caps (we'll iterate on this if costs become an issue).
- Local LLM support (Ollama, etc.). Kimi K2.6 via Moonshot is the only provider for v1; the architecture allows per-VPS override but we won't ship config for other providers in v1.
- A separate end-user web UI. End users live entirely in Slack.

## 10. Operating principles for Claude Code on this project

- **Read before writing.** When the brief says "evaluate ClawHost," do the deep dive before proposing code changes.
- **Surface tradeoffs.** When two paths are reasonable, present both with honest pros and cons before committing. Don't silently pick.
- **Document as you go.** Every Phase produces a doc. The operator should be able to ramp a new contributor in a day by reading the docs.
- **Honor the security boundaries.** If something seems easier by relaxing a constraint (e.g. "let's just hold the tokens centrally for v1"), name it explicitly as a security concession and require explicit operator approval.
- **Prefer boring tools.** Ansible over a clever bespoke updater. SOPS over a homegrown vault. Standard NanoClaw skills over forking trunk. The novelty in this project is in the *combination* of pieces, not in any single piece.
- **Be honest about uncertainty.** If you don't know whether `apps.manifest.update` works for per-user display names in 2026, say so and validate it before relying on it. Don't assume.
- **Verify upstream claims.** ClawHost's feature list is from its README; not all of it may be implemented or working. Verify each by running it. Same for Moonshot's API claims.
- **Treat the §5 hardening checklist as a target, not a bludgeon.** §5.14 lists items that may need adaptation. If a hardening item demonstrably breaks something we need, document the failure mode, the chosen workaround, and the security cost — don't fight a working setup to satisfy a checkbox, and don't silently skip items either. The deliverable is a production-ready hardened deployment, not a perfect-but-broken one.
- **Tune before shipping.** AIDE in particular must be tuned to <1 alert per week of normal operation before going to fleet — a noisy integrity monitor is worse than none.
- **Stop and ask** when:
  - You hit a security tradeoff not covered in this brief.
  - A constraint in §2 turns out to be infeasible.
  - Phase scope expands materially.
  - You're about to write more than ~500 lines of speculative code without a clear validation path.

## 11. First action when you receive this brief

Do not write code. Do not start building. Your first response should be:

1. A confirmation that you've read this brief in full.
2. A list of any ambiguities or contradictions you found, with proposed resolutions.
3. A proposed plan for Phase 0 — specifically, what you'll evaluate about ClawHost, what you'll spike with NanoClaw + the §5 hardening checklist, what you'll validate about Kimi K2.6 via Moonshot, what you'll validate about Hetzner Firewall + Tailscale, what you'll validate about Slack multi-install — and what artifacts you'll produce, with rough time estimates.
4. Any clarifying questions for the operator before starting Phase 0.

Once Phase 0 plan is approved, begin the deep dive. Do not start Phase 1 work until Phase 0 artifacts are reviewed by the operator.

---

End of brief. Welcome to the project.

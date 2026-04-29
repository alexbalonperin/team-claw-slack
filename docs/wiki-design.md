# Per-user wiki — design draft

> **Status:** Phase 3+ design draft. Not in scope for Phase 0 / Phase 1 /
> Phase 2. Captures decisions agreed during Phase 0 evaluation; will be
> revisited and expanded as Phase 3 approaches.

## What this is

A per-user, AI-maintained personal knowledge base, modeled on Karpathy's
"LLM Wiki" pattern
([gist](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f)).

Rather than RAG over raw documents, an LLM **incrementally maintains** a
structured wiki — markdown files with cross-references — that compounds in
value over time. The agent ingests new sources (Granola transcripts,
emails, calendar events, ad-hoc notes), extracts entities and relationships,
and writes them into the wiki as durable, queryable pages.

The wiki lives **on each user's VPS**, ships as an optional Docker side-car
to NanoClaw, and is consumed by NanoClaw via MCP tools. No tokens to ship
from Vercel; all state is local; no external services beyond the LLM
(Moonshot K2.6).

## Why

- **The agent should remember.** Today's NanoClaw is stateless across
  conversations. With a wiki, "you mentioned X last week" becomes possible.
- **Personal context compounds.** Five years of meetings, conversations,
  notes — the agent can synthesize across all of them, surface forgotten
  connections, prep for upcoming meetings using past context.
- **Karpathy's bet:** LLMs are good at the bookkeeping humans abandon. They
  don't get bored of updating cross-references. Worth taking seriously.

## How it differs from token-based integrations

| | Calendar / Gmail / etc. | Wiki |
|---|---|---|
| External secrets | OAuth tokens | None — all local |
| State value | Replaceable (re-OAuth) | Years of personal context, irreplaceable |
| Agent posture | Read-mostly | Read **and** write — the agent co-authors |
| Disk growth | Bounded (cache only) | Grows monotonically |
| LLM cost | None on VPS | Ingest + lint flow through the per-user Moonshot budget |

## Storage model — markdown canonical, graph derived

**Decision:** ship markdown-only in v0; add a derived graph index in
Phase 4–5 once usage data shows where markdown hurts.

```
On the user's VPS:

  /var/lib/wiki/                  ← canonical; what gets backed up
    WIKI.md                         schema / config (the CLAUDE.md analog)
    index.md                        content catalog
    log.md                          append-only chronological record
    pages/*.md                      the wiki itself

  /var/lib/wiki-index/             ← derived; rebuilt from markdown
    embeddings.db                   vector index for semantic search
    graph.db                        graph index (Phase 4–5; KuzuDB or similar)
```

**Why markdown is canonical:**
- Karpathy's bet — the LLM is the relational engine. Loose markdown handles
  ~80% of "graph queries" via fuzzy retrieval.
- No DB lock-in. Markdown ports to git, Obsidian, or anything we choose
  later.
- Backup is `tar /var/lib/wiki/` — boring, robust.
- v1 reveals where markdown hurts; that real data informs the graph
  schema rather than us speculating up front.

**Why a derived graph index, later:**
- Multi-hop queries ("who do I work with most on what kinds of projects")
  are slow and imprecise via fuzzy markdown search alone.
- Entity dedup across Granola, Calendar, and Gmail benefits from explicit
  typed relations.
- Best of both: the agent picks the right tool per query type.

```
                ┌── wiki.search(query)        fuzzy, full-text + embedding
NanoClaw ──────┤
                └── wiki.graph_query(...)     structured, multi-hop  (Phase 4–5)
```

## User-facing surfaces — all inside Slack

Three input modes, no UI outside Slack.

### 1. Slash commands — muscle-memory mode

```
/wiki search <query>       Search and return top hits.
/wiki note <text>          Quick capture; agent files it.
/wiki recent               Pages changed in the last week.
/wiki ingest <url|text>    Pull in a source and update the wiki.
/wiki summarize <topic>    Synthesize across pages on a topic.
```

**Technical wrinkle:** Slack requires slash-command responses within
**3 seconds**. For slow operations (`/wiki ingest`, `/wiki summarize`) use
the deferred-response pattern: ack immediately ("working on it…"), then
post the real result via the `response_url` Slack hands you. Bake this in
from day one.

### 2. App Home — browse + edit mode

The Slack app's Home tab is the wiki UI:
- Search bar at top.
- "Recent pages" section.
- Click a page → modal with rendered markdown.
- "Edit" button → second modal with markdown text input.
- "Tag" / "Link" buttons for cross-references.
- File-attachment block for ingesting pasted text or PDFs.

Block Kit + modals give ~80% of an Obsidian-equivalent UX with zero install
for the user. Phone-friendly out of the box.

### 3. DM the agent — narrative mode

Natural-language interaction with NanoClaw, which calls the wiki MCP tools
internally:
- "What did I do last week?"
- "Remember that Alice prefers afternoon meetings."
- "Who have I worked with most on the migration project?"

This is the highest-value mode. The others exist for fast capture and rich
editing.

## MCP tool surface

Purpose-built tools, audit-logged. **No raw filesystem access.** Same
security pattern as the Gmail / Calendar MCPs.

**Read tools** (the agent uses freely):
- `wiki.search(query, top_k)` — semantic + keyword
- `wiki.read(page_id)`
- `wiki.list_recent(days, limit)`
- `wiki.list_by_tag(tag)`

**Write tools** (agent uses, all audit-logged with diff + reason):
- `wiki.append(page_id, content, reason)`
- `wiki.create(title, content, tags, reason)`
- `wiki.update(page_id, content, reason)` — rare; prefer `append`
- `wiki.link(from_page, to_page, reason)`
- `wiki.tag(page_id, tag)`

**Workflow tools** (orchestrate multiple read/write ops):
- `wiki.ingest(source)` — runs the full ingest workflow (touches up to
  ~15 pages per source per Karpathy's pattern)
- `wiki.lint()` — periodic health check (cron-callable)

**Explicitly absent:**
- `wiki.delete(page_id)` — destructive; user-only via App Home or git
  revert.
- `wiki.raw_query(...)` — agents shouldn't construct arbitrary access
  patterns.

## Ingestion sources

### v0 (Phase 3 ship target)

- **Granola transcripts and summaries — primary feed.** Granola webhook →
  Vercel → control-plane VPS → user's VPS → Granola MCP. Agent is notified
  ("new Granola transcript available"); agent calls
  `wiki.ingest(granola_transcript_id)`. The wiki MCP fetches the transcript
  from the Granola MCP, extracts entities, and updates pages. Auto-ingest
  on by default; user can disable globally in App Home or per-meeting in
  Granola.
- **Slash-command capture** — `/wiki note`, `/wiki ingest`.
- **Conversation memory** — agent decides when to capture from DM
  conversations ("remember that…", "I'll keep that in mind").

### v1 (Phase 4)

- **Gmail threads** — agent surfaces "this thread looks worth filing"
  prompts, or auto-ingests starred / specifically-labeled threads.
- **Calendar events** — auto-create timeline entries linking to relevant
  people / project pages.

### Explicitly out of scope

- **Web Clipper / browser extension** — would require user-installed
  software. Phase 5+ if ever.
- **Public-facing wiki URL** — no per-VPS browser surface.

## Privacy posture

The wiki is the most sensitive thing on the VPS — more sensitive than
tokens (replaceable) and more sensitive than email content (duplicated on
Google's servers anyway).

**Technical mitigations:**
- LUKS-encrypted data partition (§5.10 of the brief — promoted from
  optional to mandatory once the wiki ships).
- Backups encrypted with the user's age public key before leaving the VPS,
  so even Hetzner Storage Box cannot read them.
- Wiki state survives NanoClaw upgrades and VPS rebuilds; backed up on its
  own retention tier separate from agent state.

**Operator policy:**
- The operator does NOT read wiki contents without explicit user consent.
- Encryption-at-rest with operator's root access is the *practical* trust
  model — compromise of the operator account would expose the wiki. A
  per-user key derived from the user's Slack identity (rather than a
  fleet-wide key) closes this gap; deferred to Phase 5+ if the threat
  model demands it.

**User controls:**
- Disable auto-ingest globally or per-source in App Home.
- Revert agent changes via `git diff` (Phase 4 GitHub sync) or via an App
  Home "revert" button (v0 minimum).
- Mark specific Granola meetings as "do not ingest."

## Phasing

| Phase | Ship | Notes |
|---|---|---|
| Phase 1 | Nothing | Assistant launches stateless — sets correct expectation. |
| Phase 3 | Wiki MCP v0: markdown storage, App Home UI, slash commands, Granola auto-ingest | "Minimum that's actually useful" milestone. |
| Phase 4 | GitHub-private-repo sync; power users edit in Obsidian against a local clone | Same data, two UIs. |
| Phase 4–5 | Derived graph index; add `wiki.graph_query` tool | Decided based on Phase 3 usage data, not speculatively. |
| Phase 5+ | Per-user encryption keys (no operator read access) | Threat-model-dependent. |

## Resource budget

The wiki MCP runs as a Docker side-car on the user's CAX11. Estimated load:

- **Disk:** 100 MB–1 GB per heavy user over a year (text only). PDFs and
  attachments stored on Hetzner Storage Box, not the VPS.
- **RAM:** ~100–200 MB for the MCP server + ~200–500 MB if we run an
  embedding model locally. Could also call out to Moonshot for embeddings
  to keep RAM tight.
- **CPU:** bursty — only during ingest workflows. LLM calls are remote.

The CAX11 (4 GB RAM) currently budgets NanoClaw + 3–4 MCP sidecars +
system overhead. Adding the wiki MCP at ~300–700 MB needs validation
during artifact 2 (NanoClaw spike). If tight, options are: run embeddings
remotely, ship a smaller embedding model, or upgrade specific users to
CAX21.

## Repo and lifecycle

- **Separate repo.** Suggest `alexbalonperin/llm-wiki` (or an
  open-sourceable generic name — Karpathy's pattern is something the world
  wants, and a maintained MCP server implementation would have an
  audience). Built independently from `team-claw-slack`.
- **Side-car install model.** The provisioning script optionally adds the
  wiki service to a user's compose file based on a per-user flag. Same
  pattern as the integration MCPs.
- **Independent versioning.** Wiki and NanoClaw upgrade on different
  cadences. The MCP tool contract is the stable interface.

## Open questions

For the operator to weigh in on before Phase 3 starts:

1. **Auto-ingest default for Granola.** Auto-on with disable, auto-off
   with prompt, or per-meeting prompt? My lean: auto-on, surfaced via
   Slack notification ("I added today's meeting with X to the wiki —
   `/wiki recent`"). Easy to undo, demonstrates value immediately.
2. **Wiki schema first draft.** Which categories does a user have by
   default — people, projects, topics, meetings, decisions? `WIKI.md`
   needs a sane default before v0 ships.
3. **Lint cadence.** Weekly is cheap; daily is pricey. Per-user
   configurable in `WIKI.md`?
4. **Graph DB choice (Phase 4–5).** KuzuDB (embeddable, fast),
   Apache AGE on Postgres (single binary), Neo4j (mature but
   heavyweight). Defer the choice until Phase 4 prep.
5. **Multi-user / team wiki linkage.** v1 is strictly per-user. Could a
   future "team wiki" share entities across users with explicit consent?
   Out of scope for the foreseeable future, but worth tracking.

## Out of scope

- Wiki UI outside Slack. No public web surface. Power users get GitHub +
  Obsidian (Phase 4) and Tailscale + sshfs (Phase 5+); neither is the
  default UX.
- Multi-user team wiki.
- Real-time collaboration — single-user, single-agent.
- A separate mobile app — Slack on phone covers it.

---

End of draft. Iterate as we approach Phase 3.

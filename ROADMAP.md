# Vault Sync — v1 Product Roadmap

Open-source Obsidian vault sync on Cloudflare Workers, Durable Objects, R2, and Workflows.

---

## Product thesis

A self-hostable sync backend that a user deploys to their own Cloudflare account with `terraform apply` + `wrangler deploy`, paired with an open-source Obsidian plugin. The architecture replaces replication-style sync (LiveSync/CouchDB) with a **single linearization point**: one Durable Object per vault serializes all metadata writes, so conflicts are never ambiguous — they are detected synchronously, at write time, at exactly one place, with both versions in hand.

**Design stance:** serialization does not eliminate logical conflicts (two clients editing from the same base still collide); it converts conflict *resolution during replication* into *stale-write handling at the API boundary*, which is dramatically simpler to implement, test, and reason about.

---

## Architecture decisions (locked for v1)

| Concern | Decision | Rationale |
|---|---|---|
| Concurrency control | **Compare-and-swap (CAS)**: every write carries `base_version`; DO accepts iff it matches head, else returns 409 + current head | One detection point, synchronous, client has both versions at rejection time |
| Metadata store | Durable Object per vault (SQLite storage): file manifests, versions, tombstones, checkpoints | Transactional, single-threaded, no external DB |
| Content store | R2, content-addressed chunks: `v/{vault_id}/c/{sha256}` | Zero egress fees; dedup by construction |
| File identity | **Stable file ID** in metadata; path is a mutable attribute | Collapses rename edge cases into ordinary metadata updates instead of delete+create ambiguity |
| Deletes | **Soft delete via tombstones** with retention window (default 30 days); blobs GC'd after retention | Trash/restore for free; resurrection-bug rule: tombstone beats older versions, loses to newer edits |
| Background jobs | **Cloudflare Workflows** for GC, retention expiry, bulk import, integrity audits — never in the per-write hot path | DO is already durable/transactional; Workflows overhead belongs on batch jobs |
| Client events | DO **WebSocket hibernation** pushes change notifications to connected clients | Near-real-time propagation of agent/other-device writes |
| Encryption | **R2 built-in encryption at rest (AES-256-GCM) + TLS in transit. No application-layer crypto in v1.** Chunk envelope carries a version byte from day one so app-layer encryption can be added later without a format break | E2E judged too complicated for v1 and hostile to server-side features; explicitly documented as "Cloudflare holds the keys" |
| Auth | **Cloudflare Access managed OAuth with the zero-config OTP-email IdP**; Access provisioned entirely by Terraform | No IdP setup burden; Access is the OAuth server, email ownership is the identity |
| Infra as code | Terraform for all Access/Zero Trust resources; **Terraform state backend = R2** (S3-compatible backend) | Minimal dashboard usage is a hard requirement |
| Mobile | Research-informed but **not implemented** in v1; a **v2 deliverable** (part of the collaboration-surface interface work). Protocol designed for long-offline clients regardless | Plugins cannot background-sync (Capacitor/iOS); sync-on-open model deferred |

### Explicit non-goals for v1
- End-to-end encryption (revisit post-v1 as an opt-in mode; migration between modes is a designed-for Workflow).
- Automatic AI merging. AI-assisted resolution exists only as a **manual remediation option** on flagged conflicts.
- CouchDB-style replication, peer-to-peer sync, or multi-leaf revision trees.
- Mobile client support (protocol must not preclude it).
- Cross-user dedup (dedup is per-vault only).

---

## Milestone 0 — Foundations & infrastructure skeleton

**Goal:** a deployable empty system with the full toolchain and test rig in place before any sync logic exists.

- Monorepo layout: `core/` (pure sync engine, zero I/O), `worker/` (DO + routes), `plugin/` (Obsidian client), `terraform/` (Access + R2 + Zero Trust module), `workflows/`.
- Terraform module: R2 bucket(s), Zero Trust Access application for the Worker hostname, allow-policy on user email(s), OTP-email login method, **managed OAuth enabled** (`oauth_configuration.enabled = true`, loopback clients allowed). State stored in R2 via the S3-compatible backend. Documented residual manual steps: at most the one-time Zero Trust org/team-name creation.
- Test infrastructure stood up first-class: `@cloudflare/vitest-pool-workers` running tests inside **workerd** with real DO storage semantics and R2 bindings; `fast-check` wired for model-based tests; coverage gates in CI from day one.
- **Coverage policy (a v1 feature, not an afterthought):**
  - `core/`: **100% line + branch coverage, enforced in CI.** Achievable because core is a pure reducer: `(state, event) → (state', commands[])` with all I/O (vault FS, network, clock) behind injected interfaces.
  - Interface/adapter packages: 100% remains the goal, treated as **golden-master tests** — workerd makes real-API-shaped local execution cheap enough that this is realistic rather than mock theater. Where mocks would be pure ceremony, documented exemptions are allowed, but the default is full coverage.
  - Integration coverage metric: **edge coverage over declared state machines** (every transition of the client sync-session machine and every wizard machine appears in at least one scenario). This is the honest metric for concurrency logic; line coverage alone is not.

**Exit criteria:** `terraform apply` + `wrangler deploy` from a clean Cloudflare account yields an authenticated, empty, health-checked endpoint; CI enforces coverage gates.

---

## Milestone 1 — Core sync engine (CAS protocol, single client)

**Goal:** one desktop client syncing a vault correctly, including offline sessions. All conflict *detection* complete; resolution is minimal (reject + keep-both fallback).

### Protocol (the DO API contract)
- **Write**: `PUT file` with `{file_id, base_version, manifest, metadata}` → `200 {new_version}` or `409 {head_version, head_manifest}`. CAS is the only write path.
- **Delta listing**: `GET changes?since={checkpoint}` returns an ordered change feed (create/modify/rename/tombstone) — designed for days-stale clients, since long-offline divergence is the common case, not the edge case (this is also the mobile-readiness requirement).
- **Manifest lifecycle states**: `active | tombstoned | pending_approval`. The third state exists in the schema from M1 (used by the M5 agent approval gate): pending files are durable and versioned in the DO but excluded from client materialization until approved. Cheap to carry early; expensive to retrofit.
- **Chunk negotiation** (two-phase): client sends manifest hash list → DO replies with missing hashes → client uploads only those (presigned R2 URLs so blobs bypass the Worker) → client confirms → DO commits the manifest atomically.
- Timestamp handling: mtime comparisons truncate to **2000 ms resolution** to tolerate FAT-style filesystem jitter (lesson inherited from LiveSync).

### Client engine (in `core/`)
- Pure reducer sync-session state machine: `idle → scanning → negotiating → pushing/pulling → conflicted → resolving → idle`, plus offline/reconnect transitions. Explicitly declared in **XState** (locked) so machines are introspectable and integration tests can assert edge coverage mechanically from the machine definition.
- Vault scan + change detection against last-known manifests; stable file IDs assigned at first sight and persisted in plugin data.
- Tombstone semantics implemented and property-tested: tombstone wins over older inbound versions, loses to newer edits.

### Model-based testing (v1 feature)
- `fast-check` model runner: a simplified reference model of vault convergence; random operation sequences (edit / rename / delete / go-offline / reconnect) across N simulated clients executed against the real reducer.
- Invariants asserted: no lost acknowledged writes; convergence after quiescence; tombstone ordering; manifest→chunk reachability; rename never observed as silent data loss.
- Rename state machine has a dedicated integration suite covering: rename + concurrent edit of old path; offline client observing rename as delete+create; rename A→B vs A→C races; **case-only renames** (macOS/Windows case-insensitivity); rename + edit in one offline session.

**Exit criteria:** single client survives randomized model-based campaigns and multi-day offline simulation with zero invariant violations; `core/` at 100/100 coverage.

---

## Milestone 2 — Multi-client conflict handling & resolution UX

**Goal:** two+ concurrent clients (including simulated agents) with the full resolution ladder.

### Resolution ladder (in order)
1. **Identical-content cleanup**: differing versions with identical content resolve silently (adopted from LiveSync — eliminates a surprising share of "conflicts" for free).
2. **Silent auto-rebase**: on 409, fetch head, three-way merge (base → local edit vs head) using **diff-match-patch** (character-level semantic diffs; better for prose than line-based diff3), retry CAS. Non-overlapping hunks resolve invisibly — this covers the dominant human+agent collision ("agent appended while I edited another section").
3. **Deferred re-merge on overlap** (the typing-debounce optimization): keep local; while the user is still typing, hold the conflict in the background; **~5 s after typing stops**, re-attempt the merge against the (possibly newer) head — the conflict may have become mergeable. Implemented as a state in the session machine (cheap, testable); instrument how often it succeeds and drop it in v1.x if the metric says it's useless.
4. **Flag + remediation options**: status-bar badge count and Conflicts sidebar (never a blocking modal — with agents writing, conflicts arrive mid-thought). Per-conflict side-by-side diff with: keep mine / take theirs / merge editor / **AI resolve** (manual, opt-in; hunk + context sent, never whole files; both originals preserved as recoverable revisions before any AI merge is written).
5. Open-file protection: remote updates to the currently open note never clobber the editor buffer — clean merges apply in place; otherwise a subtle inline "updated remotely — review" banner (the one sanctioned inline interruption).

### Agent-friendly write semantics (conflict-free by construction)
- Extended write modes alongside CAS: `mode=append` (always transformable server-side against the new head → never conflicts) and `mode=patch` (DO applies the patch against head if it applies cleanly, else 409). Exposed in the API so agents/MCP tools use them natively.

### Realtime
- WebSocket change feed from the DO; connected clients pull deltas on notification. Obsidian's native live-reload handles closed files; the open-file path goes through the protection logic above.

**Exit criteria:** model-based campaigns with concurrent writers (human-profile + agent-profile operation mixes) converge; measured silent-resolution rate ≥ 95% of collisions in simulation; every resolution-ladder branch covered by integration scenarios.

---

## Milestone 3 — Chunking, dedup, and storage economics

**Goal:** attachments and large vaults handled efficiently; storage lifecycle closed.

### Chunking spec (v1, locked)
- Files ≤ **128 KiB** (≈ all markdown): single chunk, no CDC — CDC adds nothing for small text.
- Files > 128 KiB: **FastCDC**, min/avg/max = **256 KiB / 1 MiB / 4 MiB**. Average deliberately large: R2 bills per Class A op, so fewer-larger chunks beat dedup-optimal tiny ones on cost.
- Chunk identity: **SHA-256 of plaintext**, computed client-side; dedup decided at the DO by hash *before bytes move* (bandwidth is the win, and dedup remains per-vault).
- **Compress before store**: zstd (fallback gzip) per chunk client-side — markdown compresses 3–5×, a larger real-world saving than dedup for most vaults. Envelope: `[version_byte | codec | payload]`.
- Manifests: ordered `(chunk_hash, size)` lists + metadata in DO SQLite, keyed by file ID. Version history = retained old manifests.

### Garbage collection (v1, locked)
- **No live refcounting** (classic mid-flight-failure corruption source). **Mark-and-sweep** in a scheduled Workflow: enumerate hashes reachable from live + retention-window manifests, list the R2 prefix, delete unreferenced chunks older than a **48-hour safety window** (avoids racing in-flight uploads).
- Companion Workflows: retention expiry of tombstones; integrity audit (manifest↔chunk reachability, sampled hash verification); bulk initial-import for large vaults.

### Metrics to instrument from day one
- Dedup ratio, compression ratio, mean chunk size, Class A/B op counts per sync session, bytes-avoided-by-negotiation, GC sweep durations and reclaimed bytes, 409 rate, silent-rebase success rate, deferred-re-merge success rate.

**Exit criteria:** multi-GB vault with attachments syncs incrementally; GC provably never collects a reachable chunk (property-tested against the model); cost-per-sync tracked in CI benchmark.

---

## Milestone 4 — Auth flow & plugin onboarding

**Goal:** the full user journey from empty Cloudflare account to syncing vault, with the interaction-queue UX.

### Auth (managed OAuth, OTP-email)
- Plugin implements: RFC 8414 discovery from `/.well-known/oauth-authorization-server` on 401 → dynamic client registration → authorization-code + PKCE via system browser → **loopback redirect** (throwaway localhost listener; Electron permits this) → refresh-token storage in plugin data (never synced).
- Token cadence: **5–15 min access tokens, 1–2 week grant sessions** — silent background refresh, Access policies re-evaluated per refresh, human sees a login roughly biweekly. Client library must support RFC 8707 resource indicators.
- Worker validates the `Cf-Access-Jwt-Assertion` header — one code path for browser and OAuth traffic (managed OAuth tokens are opaque; Cloudflare resolves identity and forwards the signed assertion).
- Secondary mode kept in scope: plain service-token auth (`CF-Access-Client-Id/Secret` headers) for headless/agent deployments — also pure Terraform.

### Interaction queue & event-driven wizards (v1 feature, thoughtfully tested)
- A single serialized **interaction queue**: lifecycle events enqueue decision flows; exactly one modal renders at a time; non-urgent events degrade to the status-bar badge rather than interrupting.
- Wizard inventory: first-connect merge choice (**merge / remote-wins / local-wins**, with file-count/size preview and an offered local backup zip before first sync — the single most dangerous moment in the product); auth expired; empty-remote push confirmation; oversized-attachment policy; conflict-review entry point (badge-first, wizard only on user click).
- Each wizard is its own declared XState machine → included in the integration edge-coverage requirement. Queue discipline itself is property-tested (no stacking, no starvation, correct urgency degradation).

### Ongoing status surface
- Status bar: synced ✓ / syncing / offline / N conflicts. Panel: recent activity, pending conflicts (deep-link to resolver), trash (soft-deleted files with restore). **Pause-sync toggle** (non-negotiable table stakes for bulk vault reorganizations).

**Exit criteria:** a new user reaches "syncing" from a clean account using only README + `terraform apply` + `wrangler deploy` + plugin settings, touching the dashboard at most once (org creation); every wizard edge covered.

---

## Milestone 5 — Hardening & public beta

**Goal:** the trust-building layer that makes people willing to point a sync tool at years of notes.

- Version history UI: per-file revision browser backed by retained manifests; restore any revision.
- Config-file strategy for `.obsidian/`: default-exclude churn files (`workspace.json` et al.), last-write-wins for the rest in v1; JSON-aware precedence merge is the documented post-v1 upgrade (LiveSync-informed).
- Chaos/fault-injection suite: killed uploads mid-negotiation, DO eviction mid-session, clock skew, partial R2 failures — all runnable locally under workerd.
- Docs with an honest **threat-model section**: what R2-at-rest + Access protects against, what it doesn't (Cloudflare account compromise), and the post-v1 encryption-mode plan (per-vault mode set at creation; mode switch = designed re-encryption Workflow; envelope version byte already reserved).
- Naming/compliance: community-plugin directory requirements, open-source license, and no use of the trademarked "Obsidian Sync" name.
- Beta telemetry (opt-in, local-first): the Milestone 3 metrics + wizard funnel drop-off + conflict-ladder distribution, to decide v1.x priorities (e.g., keep or kill the 5 s deferred re-merge).

### Beta plan
- **Human cohort:** known Obsidian users (friends/colleagues) — real vaults, real habits, low-stakes trust relationship for early data-loss reports. Onboarding doubles as a test of the Milestone 4 exit criterion (README-only setup).
- **Agent experiment: entity-description notes.** An agent co-maintains a set of "entity description" notes in a shared vault. The purpose is **not** protocol validation — it's an exploration of what human–AI collaboration on a knowledge base feels like when the sync layer makes it safe: an agent working in the same vault as a person, continuously, without stepping on them.
  - **Updates to existing entities** flow freely — the human sees the agent's refinements appear (near-real-time via the change feed), can edit over them, and the ladder absorbs the interleaving invisibly. The question under study: does unmediated agent editing of shared notes feel like collaboration or contamination?
  - **New entities require human-in-the-loop approval.** Implementation: agent creates land as **proposed files** — committed to the DO (durable, versioned) but flagged `pending_approval` and not materialized into client vaults until approved. Approval surfaces through the interaction queue as a badge-first review item (title + preview → approve / edit-then-approve / reject); rejection tombstones the proposal. This reuses three existing v1 mechanisms (manifest states, interaction queue, tombstones) rather than adding a parallel review system, and the approval flow is itself an XState machine under edge coverage. The question under study: is create/update the right trust boundary, or does it want to be finer-grained (per-folder, per-tag, per-confidence)?
  - Protocol health metrics come along for free as instrumentation, not as the point: append/patch share of agent writes, proposal approval/rejection rates, agent-induced 409 rate, and whether agent writes ever reach the flagged-conflict rung (they shouldn't; if they do, that's a bug report, not an experiment finding).

**Exit criteria:** beta cohort runs ≥ 30 days with zero data-loss reports; the entity agent co-maintains a live shared vault continuously alongside its human (any agent-induced flagged conflict is triaged as a protocol bug); restore-from-history and restore-from-trash verified in the wild; plugin submitted to the community directory.

---

## Post-v1 backlog (explicitly deferred, design-compatible)

### v2 direction: multi-writer collaboration protocol with asymmetric trust
The v1 architecture, assembled, is more than a sync engine. The collaboration substrate emerged from the design decisions themselves, almost as a byproduct — look at what the architecture actually provides once the pieces are assembled:

- **A linearization point with intent-aware writes.** CAS plus `append`/`patch` modes means the system doesn't just move bytes — it understands *what kind of change* is being made, and can transform or reject changes based on that. Dumb sync tools reconcile states; this one mediates operations.
- **A staging state for contributions.** `pending_approval` means the vault has a notion of "proposed but not yet part of the record." That's not a sync concept at all — it's a *review* concept. Git has it (pull requests); Dropbox doesn't.
- **A human attention channel.** The interaction queue is a disciplined way to route decisions to a person without interrupting them — badge-first, one modal at a time, urgency-aware.
- **A real-time change feed** so participants see each other's work as it lands.

Together these form the core of a **multi-writer collaboration protocol with asymmetric trust** — some writers (the human) hold unrestricted commit rights; others (agents) hold scoped rights and a review path. The proposal/approval flow is functionally a lightweight pull-request system for markdown, designed for a reviewer who is a person going about their day and a contributor who is a tireless agent, rather than two engineers. v2 treats this substrate — not file movement — as the product surface. The M5 entity experiment is the design input: its findings on trust boundaries and the lived experience of agent co-authorship shape what v2 formalizes.

**Scope-pressure rule:** any v1 cut that damages these four primitives is cutting v2, not v1 polish.

- **Agent collaboration API** (first v2 deliverable): expose the proposal/approval machinery as a first-class API for agents — `pending_approval` manifest states, append/patch write modes, and the interaction-queue review flow packaged as a documented surface (likely an MCP server) so any agent can safely co-maintain a vault with a human. Trust boundaries configurable beyond create-vs-update (per-folder, per-tag, per-confidence).
- **Mobile client** (v2 deliverable): mobile is part of v2 rather than generic backlog because a large share of the interface experimentation is making the collaboration surface work on *both* desktop and mobile. Reviewing an agent's proposals is a natural phone activity (glance, approve, edit-then-approve — the same interaction grammar as notification triage), so the interaction queue and approval flow must be designed cross-device, not desktop-first-then-ported. Constraints already known: no background sync (plugin runs only while Obsidian is open → sync on open/focus/foreground-interval); days-stale clients are the normal case (the checkpoint delta feed already assumes this); the loopback OAuth redirect needs a mobile alternative; permissive CORS for `app://obsidian.md` / `capacitor://localhost` is shipped in v1 as a compatibility measure. XState machines are shared across platforms — only the rendering layer differs — so the v1 edge-coverage suite carries over to mobile intact.

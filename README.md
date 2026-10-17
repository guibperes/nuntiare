# nuntiare — Decentralized RSS backend for news-flash / eilmeldung

> **nuntiare** (Latin: "to announce"). Daemon binary: `nuntiared`. Each running instance is a **shard**.
> Status: design + implementation plan. Language: Rust.
> Last updated: 2026-10-17.

## 1. Goal

Keep RSS reading state (articles, read/starred, feeds, categories) consistent across
several personal devices **without a central server**:

- Every device runs a **shard**: a self-contained daemon that fetches feeds, stores
  everything in SQLite and serves an authenticated, single-user API to the local
  RSS client (eilmeldung, via the news-flash library).
- A shard works fully on its own (offline, away from home).
- When shards see each other, they exchange changes and **converge**. No
  master/slave, no leader election.
- Changes replicate at field level: new articles, read/unread, starred, feeds and
  categories added/renamed/moved/removed.

**v1** exposes a **Miniflux-compatible facade** so eilmeldung works unmodified
(news-flash already ships a Miniflux provider).
**v2** adds a native API and a provider in the news-flash core; the Miniflux facade
becomes optional. The architecture must make the facade a thin translation layer
from day one.

### Non-goals (v1)

- Multiple users, web UI, full Miniflux API coverage (only what news-flash uses).
- Third-party integrations (Wallabag, etc.), Fever / Google Reader APIs.

## 2. Validation against a real Miniflux

A Miniflux container (2.x, PostgreSQL) was run locally and eilmeldung 1.9.0 was
pointed at it with the Miniflux provider. Debug logs were analyzed to extract the
exact calls news-flash makes. Dataset: ~80 feeds, ~4,300 unread entries.

### 2.1 Test results

| #   | Change made on      | Action                                           | Result                                                                                                                                                                                                                                 |
| --- | ------------------- | ------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| T1  | server              | mark entries read via API                        | client shows them read after its next sync                                                                                                                                                                                             |
| T2  | server              | mark an entry the client had read back to unread | server keeps the state (client does not overwrite it); client follows on sync                                                                                                                                                          |
| T3  | both                | star on client / star on server                  | client → server immediately; server → client after sync                                                                                                                                                                                |
| T4  | client              | read entries in the TUI                          | `PUT /v1/entries` sent immediately, no sync needed                                                                                                                                                                                     |
| T5  | server              | new category + feed in it                        | appears in client after sync                                                                                                                                                                                                           |
| T6  | server              | delete a feed                                    | removed from client after sync                                                                                                                                                                                                         |
| T7  | server              | change state of an old entry                     | same as T2                                                                                                                                                                                                                             |
| T8  | client              | add feed (`c f`)                                 | `POST /v1/feeds` → `GET /v1/feeds/{id}` → `GET /v1/feeds/{id}/entries` → full sync, automatically                                                                                                                                      |
| T9  | client              | create category (`c a`)                          | `POST /v1/categories`, immediate                                                                                                                                                                                                       |
| T10 | client              | move feed (`c y` / `c p`)                        | `PUT /v1/feeds/{id}` (`category_id`), immediate                                                                                                                                                                                        |
| T11 | client              | rename feed / category (`c w`)                   | `PUT /v1/feeds/{id}` / `PUT /v1/categories/{id}`, immediate                                                                                                                                                                            |
| T12 | client              | remove category (`c d`, "move children up")      | **only** `DELETE /v1/categories/{id}` is sent; Miniflux cascades and deletes the feeds inside; client is inconsistent until next sync, then the feed disappears. Removing a feed works (`DELETE /v1/feeds/{id}`), empty category stays |
| T13 | client              | star / unstar 165 entries                        | one `PUT /v1/entries/{id}/bookmark` per entry, each a **toggle** (330 requests total)                                                                                                                                                  |
| T14 | client, server down | read/unread/star while offline                   | TUI shows "Service is not reachable anymore, going offline"; marking unread offline was unreliable                                                                                                                                     |
| T15 | server back         | restart server, resume                           | first request after reconnect is the **queued** `PUT /v1/entries`, then a full sync. TUI froze client-side during reconnection (server answered in ~20 ms). No queued `bookmark` call was replayed                                     |

### 2.2 Findings

1. **Every sync is a full snapshot, never incremental.** Per sync, news-flash calls:
   `GET /v1/categories`, `GET /v1/feeds`, starred entries (`limit=100`), **all**
   unread entries paginated by 100 (43 pages for ~4,300 entries), and the 300 most
   recent read entries. It never uses `changed_after`. Consequence: the server is the
   source of truth; whatever is in the unread set is unread on the client, everything
   else becomes read. **The client needs no knowledge of replication** — it follows
   whatever its local shard has converged to.
2. **Writes are immediate and outside the sync cycle** (read state, star, all
   structural changes).
3. **Offline writes are queued and replayed**, but only partially: read state was
   replayed; unread was unreliable and no star toggle was replayed. A shard on
   `localhost` is never offline, which removes this class of problem entirely (and
   the freeze on reconnect).
4. **Star is a toggle**, one request per entry. Replaying a toggle twice silently
   inverts the state → the shard must resolve the toggle into a concrete value
   before it enters the replication log.
5. **Deleting a category deletes its feeds** with Miniflux semantics, contradicting
   what the user asked for in the TUI (`c d` = move children up). Upstream bug in the
   news-flash Miniflux integration (to report). The shard is free to implement the
   expected behavior.
6. **Authentication cost dominates sync time with Basic auth.** Server time per sync:
   ~8.5 s with Basic (bcrypt on each of ~47 requests) vs ~3.9 s with `X-Auth-Token`.
   eilmeldung works with the token; the shard must not hash per request.
7. **Feed creation must fetch synchronously**: the client asks for the new feed's
   entries right after `POST /v1/feeds`.
8. YouTube `UULV…` (live) playlists of channels that never streamed return 404
   (unrelated to the shard; fix in the OPML generator).

## 3. API contract (v1 Miniflux facade)

Only the endpoints below were observed from eilmeldung 1.9.0. Response shapes must
match Miniflux's JSON (field names, types, nullability). Golden fixtures captured
from the real Miniflux instance are the reference (task P0.1).

### 3.1 Authentication

- Header `X-Auth-Token: <token>`. Token stored hashed (SHA-256), compared in constant
  time. No bcrypt on the request path.
- Failure: `401` with `{"error_message": "..."}`.
- Basic auth: not required. If ever added, cache the verified credential.
- Bind to `127.0.0.1` by default; the facade is for the local client only.

### 3.2 Read endpoints

| Method | Path                     | When the client calls it          | Notes                                                           |
| ------ | ------------------------ | --------------------------------- | --------------------------------------------------------------- |
| GET    | `/v1/me`                 | login / reconnect                 | user object (`id`, `username`, `is_admin`, `timezone`, …)       |
| GET    | `/v1/categories`         | every sync                        | `[{id, user_id, title, hide_globally}]`                         |
| GET    | `/v1/feeds`              | every sync                        | full feed objects incl. nested `category` and `icon` (nullable) |
| GET    | `/v1/feeds/{id}`         | after feed creation               | single feed object                                              |
| GET    | `/v1/feeds/{id}/entries` | after feed creation (empty query) | `{total, entries}`; apply Miniflux defaults                     |
| GET    | `/v1/entries`            | every sync                        | `{total, entries}`, see query shapes below                      |

Observed `/v1/entries` query shapes (all `order=published_at&direction=desc`):

- `status=unread&limit=100[&offset=N]` — paginated until exhausted
- `starred=true&limit=100` — no offset observed (see open question Q1)
- `status=read&limit=300` (and once `limit=100` on first sync)

Requirements: `total` must be exact (the client may stop on it or on a short page);
pagination must be stable while paging; index on `(status, published_at DESC)` and
`(starred, published_at DESC)`.

### 3.3 Write endpoints

| Method | Path                        | Body                                                       | Shard semantics                                                                 |
| ------ | --------------------------- | ---------------------------------------------------------- | ------------------------------------------------------------------------------- |
| PUT    | `/v1/entries`               | `{entry_ids, status}` (and `starred` since Miniflux 2.3.2) | `SetRead` per id → `204`                                                        |
| PUT    | `/v1/entries/{id}/bookmark` | —                                                          | read current value, emit `SetStarred(!current)` (**value, not toggle**) → `204` |
| POST   | `/v1/categories`            | `{title}`                                                  | `CreateCategory` → `201` + category                                             |
| PUT    | `/v1/categories/{id}`       | `{title}`                                                  | `RenameCategory` → category                                                     |
| DELETE | `/v1/categories/{id}`       | —                                                          | **deviation**: move feeds to the default category, then delete → `204`          |
| POST   | `/v1/feeds`                 | `{feed_url, category_id}`                                  | `CreateFeed` + **synchronous** first fetch → `201` + `{feed_id}`                |
| PUT    | `/v1/feeds/{id}`            | `{title?, category_id?}`                                   | `RenameFeed` / `MoveFeed` → feed                                                |
| DELETE | `/v1/feeds/{id}`            | —                                                          | `DeleteFeed` → `204`                                                            |

A default category always exists (Miniflux has no uncategorized feeds) and cannot be
deleted.

## 4. Architecture

### 4.1 Principles

1. **Core is protocol-agnostic.** All domain logic lives in `nuntiare-core`. APIs (Miniflux
   facade now, native API in v2) translate requests into core `Command`s and core
   query results into wire DTOs. No business rule lives in a facade.
2. **Every write is an operation in a log, from day one.** Even with a single shard,
   writes become ops with an HLC timestamp and are applied through the merge engine.
   Phase 2 then only adds transport, not a rewrite.
3. **Merges are commutative and idempotent.** Applying the same op twice, or ops in
   any order, yields the same state (field-level LWW registers + grow-only entry set).
4. **Global, deterministic identifiers.** Two shards fetching the same feed produce
   the same entry IDs; no coordination is needed to avoid duplicates.
5. **Fetching is an optimization, not a coordination problem.** Duplicate fetches are
   harmless; there is no leader election.

### 4.2 Crates (Cargo workspace)

```
nuntiare/
├── Cargo.toml                    # workspace
├── crates/
│   ├── nuntiare-core/            # domain model, commands, ops, HLC, LWW merge, ShardService. No I/O.
│   ├── nuntiare-store/           # SQLite: op log + materialized state, migrations
│   ├── nuntiare-fetch/           # HTTP fetching, parsing, entry identity, scheduler
│   ├── nuntiare-replication/     # peer protocol, anti-entropy, membership (phase 2+)
│   ├── nuntiare-api-miniflux/    # v1 facade: axum routes, Miniflux DTOs, i64 id mapping
│   └── nuntiare-api-native/      # v2 native API (placeholder)
└── bins/
    └── nuntiared/                # wiring, config, CLI (serve, token, import-opml, join, status)
```

Dependency rule (enforced by crate boundaries):

```
nuntiare-api-miniflux ──┐
nuntiare-api-native   ──┼──► nuntiare-core ◄── nuntiare-store
nuntiare-replication  ──┘          ▲       ◄── nuntiare-fetch
                                   │
                         nuntiared (wires adapters into core ports)
```

APIs never touch `nuntiare-store` directly. `nuntiare-core` defines ports (traits) that
the adapters implement: `Store`, `Clock`, `FeedSource`, `PeerTransport`.

Illustrative core surface:

```rust
#[async_trait]
pub trait ShardService {
    async fn categories(&self) -> Result<Vec<Category>>;
    async fn feeds(&self) -> Result<Vec<Feed>>;
    async fn entries(&self, q: &EntryQuery) -> Result<Page<Entry>>;
    /// Incremental changes; unused by the Miniflux facade, needed by the v2 native API.
    async fn changes_since(&self, cursor: Cursor) -> Result<ChangeSet>;
    async fn execute(&self, cmd: Command) -> Result<Outcome>;
}

pub enum Command {
    SetRead { entry_ids: Vec<EntryId>, read: bool },
    SetStarred { entry_id: EntryId, starred: bool }, // never a toggle
    CreateCategory { title: String },
    RenameCategory { id: CategoryId, title: String },
    DeleteCategory { id: CategoryId },               // moves feeds to default
    CreateFeed { url: Url, category: CategoryId },
    RenameFeed { id: FeedId, title: String },
    MoveFeed { id: FeedId, category: CategoryId },
    DeleteFeed { id: FeedId },
}
```

### 4.3 Data model

**Identifiers**

| Entity   | Global ID                                                               | Notes                                                  |
| -------- | ----------------------------------------------------------------------- | ------------------------------------------------------ |
| Shard    | ULID, generated once, plus a human name                                 | used as LWW tie-breaker                                |
| Feed     | `blake3(normalized feed URL)`                                           | re-adding the same URL yields the same feed            |
| Category | ULID at creation                                                        | titles can change, so not hash-based                   |
| Entry    | `blake3(feed_id ‖ guid)`, fallback `link`, fallback `title ‖ published` | identity rule is critical: unstable GUIDs = duplicates |

The Miniflux facade needs `i64` IDs: derive them deterministically as the first 63
bits of the global ID hash (identical on every shard, so the client could switch
shards). Detect collisions on insert and fail loudly.

**Operations** (`ops` table: `origin_shard`, `seq`, `hlc`, `kind`, `payload`):

- `EntryUpsert` (content, title, url, published_at, …; LWW on content)
- `EntrySetRead`, `EntrySetStarred`
- `FeedCreate`, `FeedSetTitle`, `FeedSetCategory`, `FeedDelete`
- `CategoryCreate`, `CategorySetTitle`, `CategoryDelete`
- `FeedFetched { at }` (fetch coordination, see 4.5)
- Phase 3: `PeerJoin`, `PeerLeave`

**Merge rules**

- Each mutable field is an LWW register keyed by `(hlc, origin_shard)`; max wins.
- Entries form a grow-only set; their read/starred flags are LWW registers.
- Delete is a tombstone flag. Field updates never clear it; only an explicit
  `FeedCreate` / `CategoryCreate` with a later HLC resurrects.
- A feed whose category is deleted (e.g. concurrent move + delete on two shards) is
  shown in the default category: resolved at read time, deterministic on every shard.
- Entries of deleted feeds are hidden, then purged by retention.

**Retention**: purge content of read entries older than N days, but **keep a minimal
row (id + flags)** so a re-fetch does not resurrect them as unread. Deterministic rule,
so all shards converge.

**Local-only state** (never replicated): HTTP `ETag` / `Last-Modified`, token hash,
id-mapping cache, peer connection state.

### 4.4 Replication (phase 2+)

- Each shard numbers its own ops with a monotonic `seq`. A **version vector**
  (`origin → highest contiguous seq`) summarizes what a shard has.
- **Anti-entropy, pull-based**: A sends its vector to B; B returns the ops A lacks
  (batched, streamed); then the reverse. Ops are applied idempotently
  (`UNIQUE(origin_shard, seq)`).
- Triggers: periodic (e.g. every 30 s) plus a nudge right after a local write.
- Separate listener for peers (not the facade port); payload JSON first, binary later
  if needed.
- Clocks: Hybrid Logical Clocks, never wall-clock comparison.

### 4.5 Fetch coordination

- `FeedFetched { at }` is replicated as an LWW register per feed.
- A shard fetches feed F when `now - last_fetched(F) >= interval + delay(self)`:
  - `delay = 0` for the **preferred shard** (config `preferred = "desktop"`), when online;
  - otherwise `grace + random jitter`, so non-preferred shards only fetch when the
    preferred one has not.
- Offline shard: no peer has fetched recently → it fetches by itself.
- Bonus: entries a shard missed while off (YouTube feeds only list the last 15 items)
  arrive through replication.

### 4.6 Security

| Phase | Client ↔ shard                   | Shard ↔ shard                                                                     |
| ----- | -------------------------------- | --------------------------------------------------------------------------------- |
| 1     | `X-Auth-Token`, `127.0.0.1` only | —                                                                                 |
| 2     | same                             | pre-shared network secret (HMAC-signed requests), over LAN or Tailscale/WireGuard |
| 3     | same                             | TLS (rustls) with per-shard self-signed certs, fingerprints pinned at join        |

### 4.7 Path to v2

- `nuntiare-api-native`: exposes `changes_since(cursor)` (incremental instead of a full
  snapshot), set semantics only (no toggles), batched commands, optional push
  (SSE/WebSocket).
- news-flash provider implemented in the news-flash core against the native API.
- Wire types live in a small `nuntiare-proto` crate so the news-flash provider can depend
  on it without pulling the daemon.
- Miniflux facade stays as an optional compatibility adapter.

### 4.8 Suggested crates

`tokio`, `axum`, `rusqlite` (bundled) + `rusqlite_migration`, `reqwest` (rustls),
`feed-rs`, `serde`/`serde_json`, `blake3`, `ulid`, `uhlc` (or a small in-house HLC),
`clap`, `toml`, `tracing`, `mdns-sd` (phase 3), `proptest`, `insta`.
`cr-sqlite` was considered and rejected for v1: a hand-written op log keeps semantics
explicit (toggle resolution, category deletion) and avoids a native extension.

## 5. Implementation plan

Each task lists its acceptance criterion ("Done when"). Phases are cumulative.

### Phase 0 — Preparation

- [ ] **P0.1 Golden fixtures.** Script that captures the real Miniflux JSON for every
      endpoint in §3 into `fixtures/miniflux/`.
      _Done when_ every read and write endpoint has at least one recorded response.
- [ ] **P0.2 Close Q1 and Q2** (open questions, §7) on the Miniflux test instance.
      _Done when_ both have a recorded answer in this document.
- [ ] **P0.3 Report the T12 bug** (category delete cascades) to eilmeldung/news-flash.
- [ ] **P0.4 Repository bootstrap.** Check that `nuntiare` is free on crates.io and
      GitHub; Cargo workspace with the empty crates of §4.2, CI running
      `cargo fmt --check`, `cargo clippy -D warnings` and `cargo test`.

### Phase 1 — Single shard, consistent with the TUI

- [ ] **P1.1 Domain model** in `nuntiare-core`: IDs, `Feed`, `Category`, `Entry`,
      `Command`, `EntryQuery`, `Page`. _Done when_ it compiles with no I/O dependencies.
- [ ] **P1.2 HLC + op model.** Commands produce ops; merge engine applies ops to state.
      _Done when_ property tests prove apply is idempotent and order-independent.
- [ ] **P1.3 SQLite store.** Schema and migrations for ops + materialized tables,
      indexes from §3.2, WAL mode, single writer. _Done when_ restart preserves state
      and the op log replays to an identical state.
- [ ] **P1.4 Entry identity + fetcher.** `reqwest` + `feed-rs`, conditional GET,
      identity rule of §4.3, per-feed interval scheduler.
      _Done when_ refetching an unchanged feed creates no new entries.
- [ ] **P1.5 Synchronous fetch on feed creation** (finding 7).
- [ ] **P1.6 CLI.** `nuntiared token create|revoke`, `nuntiared import-opml`,
      `nuntiared export-opml`. _Done when_ the current OPML imports with all feeds and
      categories.
- [ ] **P1.7 Facade: auth + read endpoints** with deterministic `i64` mapping.
      _Done when_ snapshot tests (`insta`) match the shape of the P0.1 fixtures.
- [ ] **P1.8 Facade: write endpoints** with the semantics of §3.3 (toggle → value,
      category delete → move to default). _Done when_ integration tests cover every write
      endpoint.
- [ ] **P1.9 Retention** (§4.3) with a minimal row kept for purged entries.
- [ ] **P1.10 Daemon packaging.** TOML config in XDG dirs, `tracing` logs, graceful
      shutdown, systemd **user** unit.
- [ ] **P1.11 Acceptance with eilmeldung.** Run the T1–T15 table against the shard.
      _Done when_ all pass, T12 behaves as the user expects, and sync takes less time
      than with Miniflux + token (~3.9 s server time for ~4,300 unread).
- [ ] **P1.12 Dogfood on one machine.** Back up eilmeldung state, point it at the
      local shard, use it daily for a week.

**Milestone M1:** one machine reads exclusively through its shard with no regressions.

### Phase 2 — Two shards

- [ ] **P2.1 Shard identity.** ULID + name, per-origin `seq`.
- [ ] **P2.2 Version vectors** and the "ops missing for vector V" query.
- [ ] **P2.3 Peer protocol.** Separate listener; endpoints for hello (identity +
      vector) and op exchange in batches.
- [ ] **P2.4 Peer authentication** with a pre-shared secret (HMAC).
      _Done when_ unsigned or wrongly signed requests are rejected.
- [ ] **P2.5 Anti-entropy loop.** Periodic, plus a nudge after local writes; static
      peer list in config. _Done when_ a write on A appears on B within one cycle.
- [ ] **P2.6 Join / bootstrap.** `nuntiared join <peer>` copies the full op log
      (acceptable for two shards).
- [ ] **P2.7 Fetch coordination** (§4.5) with the preferred shard and jitter.
      _Done when_, with both online, only the preferred shard fetches under normal
      conditions.
- [ ] **P2.8 Convergence simulation.** Two in-process shards over an in-memory
      transport; `proptest` generates random commands, partitions and delivery orders.
      _Done when_ the state hashes are equal after quiescence, across thousands of cases.
- [ ] **P2.9 Scenario tests:**
  - notebook offline for days, reading and adding feeds, then reconnecting;
  - concurrent rename and delete of the same feed;
  - feed moved into a category deleted concurrently;
  - the same entry starred on A and unstarred on B.
- [ ] **P2.10 `nuntiared status`.** Peers, vectors, replication lag, last fetch per feed.
- [ ] **P2.11 Real-world acceptance.** Desktop + notebook on the LAN, then over
      Tailscale; repeat the offline-travel story with eilmeldung on both machines.

**Milestone M2:** desktop and notebook converge automatically; manual state copying is
retired.

### Phase 3 — More than two shards

- [ ] **P3.1 Membership** as a replicated set (`PeerJoin` / `PeerLeave`);
      `nuntiared peer remove` for dead shards.
- [ ] **P3.2 Peer selection.** Full mesh for small N; random-k gossip as an option.
- [ ] **P3.3 Discovery.** mDNS (`_nuntiare._tcp`) on the LAN, plus static addresses
      (Tailscale).
- [ ] **P3.4 TLS with pinned certificates** (§4.6).
- [ ] **P3.5 Op log compaction.** Truncate ops acknowledged by all members (minimum
      vector); new shards bootstrap from a state snapshot + vector.
- [ ] **P3.6 Tombstone GC** after all members acknowledge.
- [ ] **P3.7 Fetch coordination for N shards** (ordered preference list).
- [ ] **P3.8 Partition testing.** 3–5 shards in containers or network namespaces,
      random partitions and restarts. _Done when_ convergence is reached after healing.
- [ ] **P3.9 Observability.** Metrics for op-log size, lag per peer and fetch counts.

**Milestone M3:** a third device joins, is removed and rejoins without manual repair.

### Phase 4 — v2 (not scheduled)

- [ ] Specify the native API (`nuntiare-proto`), incremental via `changes_since`.
- [ ] news-flash provider in the core; submit upstream.
- [ ] Make the Miniflux facade an optional cargo feature.

## 6. Decision log

Decisions taken while designing (2026-10-17), with the reason for each.

| #   | Decision                                                               | Reason                                                                                                                                                                                                         |
| --- | ---------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| D1  | Peer shards, no master/slave, no central server                        | must work offline away from home and converge on return; replaces manual copying of eilmeldung's state directory between desktop and notebook                                                                  |
| D2  | No leader election for fetching                                        | deterministic entry IDs make duplicate fetches harmless; election under partitions is the hardest part of distributed systems and buys nothing here. A preferred shard is kept only as a fetch priority (§4.5) |
| D3  | Field-level merge (LWW + tombstones), not "newest shard wins"          | comparing whole-shard timestamps loses half of the changes when both sides changed while apart                                                                                                                 |
| D4  | v1 speaks the Miniflux API as a facade                                 | news-flash already ships a Miniflux provider; validated end-to-end with eilmeldung 1.9.0 (§2)                                                                                                                  |
| D5  | Core decoupled from any API; v2 native API in the news-flash core      | the Miniflux contract forces full-snapshot syncs and toggles; a native API can be incremental and set-based                                                                                                    |
| D6  | `X-Auth-Token`, never per-request password hashing                     | measured: Basic auth doubled sync time (~8.5 s vs ~3.9 s)                                                                                                                                                      |
| D7  | One shard per device on `localhost`                                    | the client never goes offline: no partial offline queue, no freeze on reconnect (T14/T15)                                                                                                                      |
| D8  | Hand-written op log over cr-sqlite                                     | explicit control of semantics (toggle resolution, category deletion) and no native extension                                                                                                                   |
| D9  | Rust                                                                   | user's choice; also matches news-flash and eilmeldung, which eases the v2 provider                                                                                                                             |
| D10 | Rejected: Syncthing on eilmeldung's state directory                    | syncing a live SQLite file on two machines risks conflicts and corruption                                                                                                                                      |
| D11 | Rejected for the long term: single Miniflux on the desktop + Tailscale | zero code, but no new articles and an offline client whenever the desktop is off                                                                                                                               |

## 7. Open questions

- **Q1 — More than 100 starred entries.** The starred query uses `limit=100` without
  an observed offset. Does news-flash paginate when `total > 100`? Test: star 110 via
  `PUT /v1/entries`, then sync with the TUI closed and count in the TUI (`3`). If not,
  the facade may return all starred entries in one page.
- **Q2 — Offline queue contents.** Which actions does news-flash queue offline (read,
  unread, star, structural)? Test: one of each while the server is down, then count
  replayed requests. Mostly informative: a local shard is never offline.
- **Q3 — Entry identity for problematic feeds** (no GUID or changing GUIDs): collect
  examples from the podcast/blog feeds before freezing the rule.
- **Q4 — Retention window** (days) and whether starred entries are exempt
  (proposal: exempt).

## 8. References

- Miniflux API: <https://miniflux.app/docs/api.html>
- Miniflux configuration: <https://miniflux.app/docs/configuration.html>
- eilmeldung: <https://github.com/christo-auer/eilmeldung>
- news-flash: <https://gitlab.com/news-flash/news_flash>

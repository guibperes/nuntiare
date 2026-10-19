# nuntiare — Decentralized RSS backend for news-flash / eilmeldung

> **nuntiare** (Latin: "to announce"). Daemon binary: `nuntiared`. Each running instance is a **shard**.
> Status: design + implementation plan. Language: Rust.
> Last updated: 2026-10-18.

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
3. **Offline writes are queued and replayed** (details in §2.3). In the test only the
   read state was observed being replayed. A shard on `localhost` is never offline,
   which removes this class of problem entirely (and the freeze on reconnect).
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

### 2.3 Source-code analysis

Read on 2026-10-18: `news-flash` 3.2.0 (Miniflux provider, offline queue),
`miniflux_api` 0.8.0 (HTTP client and serde models) and `eilmeldung` 1.9.0 — the
versions pinned in eilmeldung's `Cargo.lock`. The code explains the observed behavior
and reveals rules no capture shows; the contract consequences are in §3.

**Sync algorithm** (`Miniflux::sync`):

- Categories, feeds, unread, starred and recent read entries are fetched concurrently.
- Unread **and starred** entries are paginated with `limit=100` and an increasing
  `offset`; the loop stops when a page has **fewer than 100** items. `total` is never
  read.
- Recent read entries: one request with `limit = sync_amount` (a news-flash setting;
  300 observed). The first sync after login (`initial_sync`) uses `limit=100`.
- Reconciliation: every locally unread entry missing from the remote unread set is
  marked read locally; every locally starred entry missing from the remote starred set
  is unstarred locally.
- Local changes made **while** a sync is running are cached and re-applied on top of
  the sync result, so the client protects itself against its own races, but not
  against the server changing during pagination.

**Writes**:

- Read/unread: `PUT /v1/entries` with an explicit status. Marking a whole feed,
  category or everything as read sends the **client's own list of entry ids**, not
  the `mark-all-as-read` endpoints. If the request fails, the client reverts the local
  change.
- Star: `PUT /v1/entries/{id}/bookmark` per entry. The provider ignores the desired
  value and toggles, assuming only entries that need a change are passed.
- Add feed without a category: `POST /v1/categories` with title `"New Category"`; on
  error (e.g. duplicate title) it lists categories and looks the title up. An optional
  custom title triggers `PUT /v1/feeds/{id}` right after creation. Then
  `GET /v1/feeds/{id}` and `GET /v1/categories`.
- Fetch one feed: `GET /v1/feeds/{id}` + `GET /v1/feeds/{id}/entries` with no query.
- Unsupported by the provider: editing a feed URL, moving categories, tags.

**Offline queue** (news-flash core): actions `Read`, `Unread`, `Mark`, `Unmark`, `Tag`,
`Untag` are stored per article; structural changes are not queued. On reconnect an
`ActionCache` cancels opposite pairs and replays the net result — star changes are
replayed **as toggles**.

**Connectivity** (eilmeldung `connectivity.rs`): `GET /healthcheck` every 60 s and on
network-interface changes; up to 10 attempts, 1 s apart, 10 s timeout each, while
holding the news-flash lock (explains the freeze in T15). Only **network** errors mean
offline: an HTTP error from `/healthcheck` still counts as reachable.

**Upstream bugs** (for P0.3):

- `remove_category` ignores `remove_children` (a `FIXME` in the code admits the API
  behavior is unknown) → cause of T12.
- `import_opml` sends **GET** with a body (Miniflux expects POST) and always returns
  an error, even on success → OPML import through eilmeldung with the Miniflux
  provider is broken.

## 3. API contract (v1 Miniflux facade)

Endpoints observed from eilmeldung 1.9.0 and confirmed in the news-flash source (§2.3).
Response shapes must match Miniflux's JSON (field names, types, nullability). The
executable reference is the resterm collection of (`contract/`), run against the real Miniflux.

### 3.1 Authentication

- Header `X-Auth-Token: <token>`. Token stored hashed (SHA-256), compared in constant
  time. No bcrypt on the request path.
- Failure: `401` with `{"error_message": "..."}`.
- Basic auth: not required. If ever added, cache the verified credential.
- Bind to `127.0.0.1` by default; the facade is for the local client only.
- The client builds URLs with `Url::join("v1/...")`: a base URL with a sub-path must
  end with `/` (configuration note).

### 3.2 General rules

**Status codes are compared for equality.** Any other code is an error, and for
read/unread updates the client also reverts its local change.

| Request                                                            | Expected status     |
| ------------------------------------------------------------------ | ------------------- |
| any `GET` (including `/healthcheck`)                               | `200`               |
| `POST /v1/feeds`, `POST /v1/categories`                            | `201`               |
| `PUT /v1/feeds/{id}`, `PUT /v1/categories/{id}`                    | **`201`** (not 200) |
| `PUT /v1/entries`, `PUT /v1/entries/{id}/bookmark`, every `DELETE` | `204`               |

**Error bodies** must be JSON `{"error_message": "..."}`; anything else makes the
client fail while parsing the error itself.

**Required fields.** The client's serde models ignore unknown fields but reject
missing ones (non-`Option` fields). **A single malformed entry fails the whole sync.**
Strings must never be `null` (e.g. empty `author` is `""`).

| Model           | Required fields                                                                                                                                                                                                                                                 |
| --------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| user (`/v1/me`) | `id`, `username`, `language`, `timezone`, `theme`, `entry_sorting_direction`                                                                                                                                                                                    |
| category        | `id`, `user_id`, `title`                                                                                                                                                                                                                                        |
| feed            | `id`, `user_id`, `title`, `site_url`, `feed_url`, `rewrite_rules`, `scraper_rules`, `crawler`, `checked_at`, `etag_header`, `last_modified_header`, `parsing_error_count`, `parsing_error_message`, `category` (object). Optional: `icon`                       |
| entry           | `id`, `user_id`, `feed_id`, `title`, `url`, `author`, `content`, `hash`, `published_at`, `created_at`, `status`, `starred`, `feed` (**full feed object**, with `category`), `reading_time`. Optional: `comments_url`, `changed_at`, `enclosures` (null → empty) |
| enclosure       | `id`, `user_id`, `entry_id`, `url`, `mime_type`, `size`                                                                                                                                                                                                         |
| entry list      | `total`, `entries`                                                                                                                                                                                                                                              |

Fields the client actually uses: entry `id`, `feed_id`, `title`, `url`, `author`,
`content`, `published_at`, `status`, `starred`, `enclosures`; feed `id`, `title`,
`site_url`, `feed_url`, `parsing_error_count`, `parsing_error_message`, `category.id`;
category `id`, `title`. `published_at` must be RFC 3339, otherwise the client silently
uses the current time.

### 3.3 Read endpoints

| Method | Path                     | When the client calls it          | Notes                                            |
| ------ | ------------------------ | --------------------------------- | ------------------------------------------------ |
| GET    | `/healthcheck`           | every 60 s and on network changes | plain `OK`; decides online/offline (§2.3)        |
| GET    | `/v1/me`                 | login with token                  | user object                                      |
| GET    | `/v1/categories`         | every sync; after adding a feed   | category array                                   |
| GET    | `/v1/feeds`              | every sync                        | feed array                                       |
| GET    | `/v1/feeds/{id}`         | after adding a feed               | single feed                                      |
| GET    | `/v1/feeds/{id}/entries` | after adding a feed (empty query) | `{total, entries}`; Miniflux defaults apply (Q6) |
| GET    | `/v1/entries`            | every sync                        | `{total, entries}`, query shapes below           |

`/v1/entries` query shapes (all `order=published_at&direction=desc`):

- `status=unread&limit=100[&offset=N]` — paginated until a page has fewer than 100 items
- `starred=true&limit=100[&offset=N]` — same pagination
- `status=read&limit=<sync_amount>` — 300 observed; `limit=100` on the first sync

Requirements:

- An `offset` past the end returns `200` with `{"total": N, "entries": []}`, never an
  error (happens whenever the count is a multiple of 100).
- Deterministic order with a tie-breaker: `published_at DESC, id DESC`. Without it,
  entries with equal timestamps can be skipped at a page boundary, and the client then
  marks them read locally until the next sync.
- `total` is required by the model but not used by the client.
- Indexes on `(status, published_at DESC, id DESC)` and `(starred, published_at DESC, id DESC)`.

### 3.4 Write endpoints

| Method | Path                        | Body                      | Shard semantics                                                                                                                     |
| ------ | --------------------------- | ------------------------- | ----------------------------------------------------------------------------------------------------------------------------------- |
| PUT    | `/v1/entries`               | `{entry_ids, status}`     | `SetRead` per id → `204`. Bodies can carry thousands of ids (mark feed/category/all as read): check the body size limit             |
| PUT    | `/v1/entries/{id}/bookmark` | —                         | resolve the toggle into `SetStarred(value)` (**value, not toggle**) → `204`. Which value: see Q5                                    |
| POST   | `/v1/categories`            | `{title}`                 | `CreateCategory` → `201` + category. **Duplicate title → `400`** with `error_message` (the client's add-feed fallback relies on it) |
| PUT    | `/v1/categories/{id}`       | `{title}`                 | `RenameCategory` → `201` + category                                                                                                 |
| DELETE | `/v1/categories/{id}`       | —                         | **deviation**: move feeds to the default category, then delete → `204`                                                              |
| POST   | `/v1/feeds`                 | `{feed_url, category_id}` | `CreateFeed` + **synchronous** first fetch → `201` + `{feed_id}`                                                                    |
| PUT    | `/v1/feeds/{id}`            | `{title?, category_id?}`  | `RenameFeed` / `MoveFeed` → `201` + feed                                                                                            |
| DELETE | `/v1/feeds/{id}`            | —                         | `DeleteFeed` → `204`                                                                                                                |

A default category always exists (Miniflux has no uncategorized feeds) and cannot be
deleted. `POST /v1/import` is not needed: the client's OPML import is broken
upstream (§2.3); OPML goes through `nuntiared import-opml`.

### 3.5 Edge cases to map

| #   | Scenario                                                                         | Why it matters                                                                    | Expected facade behavior           | Where to test                                         |
| --- | -------------------------------------------------------------------------------- | --------------------------------------------------------------------------------- | ---------------------------------- | ----------------------------------------------------- |
| S1  | Unread or starred count is an exact multiple of 100                              | the client asks for one extra page                                                | empty page, `200`                  | resterm vs Miniflux                                   |
| S2  | Several entries share `published_at` across a page boundary                      | undefined order can skip an entry; the client marks it read locally               | tie-breaker in ordering            | resterm vs Miniflux, then shard                       |
| S3  | Entries change state while the client paginates                                  | offsets shift, an unread entry is skipped                                         | transient by design; minimize (Q7) | shard (replication applying ops during a client sync) |
| S4  | Star changed on another device between two client syncs, then toggled in the TUI | the toggle inverts the **current** shard value, the opposite of the user's intent | see Q5                             | two shards                                            |
| S5  | Offline star/unstar replayed as toggles over a changed state                     | same mechanism as S4                                                              | see Q5                             | two shards                                            |
| S6  | Duplicate category title; add a feed without a category                          | the client's fallback path                                                        | `400` + `error_message`            | resterm vs Miniflux                                   |
| S7  | `/healthcheck` returns an error                                                  | the client still treats the service as reachable                                  | see Q8                             | resterm vs Miniflux, then shard                       |
| S8  | Mark thousands of entries read in one request                                    | request body size                                                                 | accept large bodies                | shard                                                 |

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

- [x] **P0.1 Golden fixtures.** resterm collection (`contract/miniflux/`) recorded from
      real eilmeldung traffic and curated per endpoint; responses in `fixtures/miniflux/`.
      _Done when_ every read and write endpoint has at least one recorded response.
- [x] **P0.2 Close Q1 and Q2** (open questions, §7). Answered from the news-flash
      source code (§2.3).
- [ ] **P0.3 Report upstream bugs** to eilmeldung/news-flash (§2.3): category delete
      ignores `remove_children` (T12); `import_opml` uses GET and always returns an error.
- [ ] **P0.5 Map edge cases against Miniflux.** Add S1, S2, S6 and S7 (§3.5) to the
      resterm collection, with `@assert` on status codes and response shape.
      _Done when_ they run green against the Miniflux container and Q6 is answered.
- [ ] **P0.4 Repository bootstrap.** Check that `nuntiare` is free on crates.io and
      GitHub; Cargo workspace with the empty crates of §4.2, CI running
      `cargo fmt --check`, `cargo clippy -D warnings` and `cargo test`.

### Phase 1 — Single shard, consistent with the TUI

- [ ] **P1.1 Domain model** in `nuntiare-core`: IDs, `Feed`, `Category`, `Entry`,
      `Command`, `EntryQuery`, `Page`. _Done when_ it compiles with no I/O dependencies.
- [ ] **P1.2 HLC + op model.** Commands produce ops; merge engine applies ops to state.
      _Done when_ property tests prove apply is idempotent and order-independent.
- [ ] **P1.3 SQLite store.** Schema and migrations for ops + materialized tables,
      indexes from §3.3, WAL mode, single writer. _Done when_ restart preserves state
      and the op log replays to an identical state.
- [ ] **P1.4 Entry identity + fetcher.** `reqwest` + `feed-rs`, conditional GET,
      identity rule of §4.3, per-feed interval scheduler.
      _Done when_ refetching an unchanged feed creates no new entries.
- [ ] **P1.5 Synchronous fetch on feed creation** (finding 7).
- [ ] **P1.6 CLI.** `nuntiared token create|revoke`, `nuntiared import-opml`,
      `nuntiared export-opml`. _Done when_ the current OPML imports with all feeds and
      categories.
- [ ] **P1.7 Facade: auth + read endpoints** with deterministic `i64` mapping,
      `/healthcheck`, the status codes, error bodies and required fields of §3.2, and the
      ordering rules of §3.3. _Done when_ snapshot tests (`insta`) match the shape of the
      P0.1 fixtures and the resterm collection passes against `nuntiared`.
- [ ] **P1.8 Facade: write endpoints** with the semantics of §3.4 (toggle → value,
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
  - the same entry starred on A and unstarred on B;
  - S3, S4 and S5 of §3.5 with eilmeldung on top of each shard.
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

Decisions taken while designing (2026-10-17 and 2026-10-18), with the reason for each.

| #   | Decision                                                                            | Reason                                                                                                                                                                                                         |
| --- | ----------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| D1  | Peer shards, no master/slave, no central server                                     | must work offline away from home and converge on return; replaces manual copying of eilmeldung's state directory between desktop and notebook                                                                  |
| D2  | No leader election for fetching                                                     | deterministic entry IDs make duplicate fetches harmless; election under partitions is the hardest part of distributed systems and buys nothing here. A preferred shard is kept only as a fetch priority (§4.5) |
| D3  | Field-level merge (LWW + tombstones), not "newest shard wins"                       | comparing whole-shard timestamps loses half of the changes when both sides changed while apart                                                                                                                 |
| D4  | v1 speaks the Miniflux API as a facade                                              | news-flash already ships a Miniflux provider; validated end-to-end with eilmeldung 1.9.0 (§2)                                                                                                                  |
| D5  | Core decoupled from any API; v2 native API in the news-flash core                   | the Miniflux contract forces full-snapshot syncs and toggles; a native API can be incremental and set-based                                                                                                    |
| D6  | `X-Auth-Token`, never per-request password hashing                                  | measured: Basic auth doubled sync time (~8.5 s vs ~3.9 s)                                                                                                                                                      |
| D7  | One shard per device on `localhost`                                                 | the client never goes offline: no partial offline queue, no freeze on reconnect (T14/T15)                                                                                                                      |
| D8  | Hand-written op log over cr-sqlite                                                  | explicit control of semantics (toggle resolution, category deletion) and no native extension                                                                                                                   |
| D9  | Rust                                                                                | user's choice; also matches news-flash and eilmeldung, which eases the v2 provider                                                                                                                             |
| D10 | Rejected: Syncthing on eilmeldung's state directory                                 | syncing a live SQLite file on two machines risks conflicts and corruption                                                                                                                                      |
| D11 | Rejected for the long term: single Miniflux on the desktop + Tailscale              | zero code, but no new articles and an offline client whenever the desktop is off                                                                                                                               |
| D12 | Contract mapped with resterm (`.http` files), recorded from real eilmeldung traffic | assertions (`@assert`), JUnit output for CI, `record` mode as a proxy between client and server; replaced Posting, whose v3 has no scripting or assertions                                                     |
| D13 | Contract derived from the client's source code, not only from captures              | captures show what happened once; the serde models and the sync loop show what the client requires (status codes, required fields, pagination stop rule)                                                       |

## 7. Open questions

### Resolved

- **Q1 — More than 100 starred entries.** Resolved 2026-10-18 from the source: starred
  entries are paginated exactly like unread ones (`offset` until a page has fewer than
  100 items). The single page in the capture only meant there were fewer than 100.
- **Q2 — Offline queue contents.** Resolved 2026-10-18 from the source: `Read`,
  `Unread`, `Mark`, `Unmark`, `Tag`, `Untag`; structural changes are not queued.
  Opposite pairs cancel out; stars are replayed as toggles (§2.3).

### Open

- **Q3 — Entry identity for problematic feeds** (no GUID or changing GUIDs): collect
  examples from the podcast/blog feeds before freezing the rule.
- **Q4 — Retention window** (days) and whether starred entries are exempt
  (proposal: exempt). The client also drops local articles after `keep_articles_days`
  (30 by default in eilmeldung).
- **Q5 — Star intent behind a toggle (S4, S5).** The client sends a toggle only when the
  state _it saw at its last sync_ differs from what the user wants. With replication,
  the shard's current value may already differ from what the client saw, so inverting
  the current value can contradict the user. Candidate approaches to evaluate:
  invert the current shard value (simple, wrong in S4); invert the value last served
  to that client (needs per-entry "last served" state); accept the risk and shorten the
  client's sync interval.
- **Q6 — Defaults of `GET /v1/feeds/{id}/entries` without a query** (limit, order,
  status) on Miniflux. Verify in P0.5.
- **Q7 — Pagination consistency (S3).** How far to go: accept the transient effect,
  defer applying replicated ops while a client sync is in progress, or serve pages from
  a consistent snapshot.
- **Q8 — Health semantics (S7).** When should the shard's `/healthcheck` report an
  error, given that the client treats HTTP errors as reachable?

## 8. References

- Miniflux API: <https://miniflux.app/docs/api.html>
- Miniflux configuration: <https://miniflux.app/docs/configuration.html>
- eilmeldung: <https://github.com/christo-auer/eilmeldung>
- news-flash: <https://gitlab.com/news-flash/news_flash>
- news-flash 3.2.0 Miniflux provider: <https://docs.rs/crate/news-flash/3.2.0/source/src/feed_api_implementations/miniflux/mod.rs>
- miniflux_api 0.8.0 (client and models): <https://docs.rs/crate/miniflux_api/0.8.0/source/>
- resterm: <https://github.com/unkn0wn-root/resterm>

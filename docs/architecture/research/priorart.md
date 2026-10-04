# Keep clone: prior art, capacity model and NFR baseline

*Research date: 2026-10-04. Scope: a production-grade Google Keep clone. Clients are a React PWA plus React Native/Expo apps for iOS and Android, sharing one TypeScript sync/domain core. The backend is Node TS on Postgres, Valkey and S3-compatible storage. The app is local-first and uses CRDT merging.*

**How to read this document**
- Claims carry inline links to sources. Primary sources (vendor docs, engineering blogs, official repos) are preferred.
- **UNVERIFIED** marks a claim I could not confirm from a primary source, or where sources disagree.
- **ASSUMPTION** marks a modelling input I chose. Each one has a reason next to it.
- All money is USD unless marked €. Cloud prices are list prices seen in Sep–Oct 2026 and change often (see the Hetzner 2026 repricing in §2.9).

---

## Part 1: How real systems architect note sync and collaboration

### 1.1 Summary matrix

| System | Sync topology | Ordering / versioning | Bootstrap vs delta | Merge / conflict model | Presence | Permission-aware partial sync | Attachments |
|---|---|---|---|---|---|---|---|
| **Google Keep** | Client ↔ Google servers. Internal `changes` endpoint (reverse-engineered) | Account-level `targetVersion` → `toVersion` cursor. Per-node `baseVersion` | Paged delta (`truncated` flag). Server can force `forceFullResync` | Node tree (NOTE / LIST / LIST_ITEM / BLOB). Text-merge algorithm not public (**UNVERIFIED**) | Not public | Shares via `roleInfo` (owner/writer). Labels, colour, archive and reminders are **per user** | `BLOB` nodes, separate media endpoint |
| **Apple Notes** | Device ↔ CloudKit (iCloud) | CloudKit change tokens. CvRDT inside the note body | CloudKit zone change fetch | CRDT text and table structures inside a protobuf blob | Collaboration UI (implementation not public) | CloudKit sharing (`CKShare`) | Separate attachment records/assets |
| **Notion** | Client ↔ server. SQLite cache on all platforms (WASM SQLite on web) | Record versions plus per-page `lastDownloadedTimestamp` for offline pages | Pages explicitly "made available offline"; reconnect fetches only pages with newer server version | Offline pages migrated to a new **CRDT** data model | Yes (not detailed) | Offline set tracked per page, with "reasons" | Not detailed |
| **Linear** | Client ↔ sync server over WebSocket. IndexedDB locally | **Global monotonic `lastSyncId`** | Full / partial / local bootstrap, then delta packets | Server is source of truth. Client **rebases** pending transactions | Not central | **Sync groups** plus subscriptions | n/a |
| **Figma** | One multiplayer process per document over WebSocket | Server orders everything. Per-file **journal sequence numbers** | Full document on open, then live ops | **Per-property LWW** (server-arrival order); fractional indexing for order | Yes, ephemeral | n/a (whole file) | Images separate |
| **Evernote** | Client ↔ shard (state-based replication) | **Per-account USN** (update sequence number) | Full sync via paged `SyncChunk`s; incremental `afterUSN`; `fullSyncBefore` cut-off | Client-side: dirty flags, field merge or conflict copy | No | Linked/shared notebooks synced separately | **Resources** fetched separately by MD5 hash |
| **Simplenote / Simperium** | Client ↔ Simperium over WebSocket | Bucket change version `cv`. Per-object `v` | `i` index paging, then `cv` catch-up; `?` means rebuild | jsondiff (diff-match-patch for strings); `sv`/`ev`; `ccid` for idempotent acks | No | Per-bucket | n/a |
| **Obsidian Sync** | File-level sync of a vault | Per-file versions | File diffs | **diff-match-patch** for `.md`; JSON key merge for settings; LWW for binary; optional conflict files | No | Shared vaults | Files (LWW); size limits per plan |
| **Bear** | CloudKit | CloudKit | CloudKit | Not public | No | n/a | CloudKit assets. Per-note AES-GCM-256 for locked notes |
| **Craft** | Client (Realm) ↔ Craft Sync Service (socket.io) ↔ Postgres | Not published | Mobile syncs whole space; web syncs per document | Not published (**UNVERIFIED**) | Yes | Per space/document | Not detailed |
| **Excalidraw** | Relay server; E2E-encrypted payloads; no server coordination | Per-element `version` plus random `versionNonce` tie-break | Peers exchange full scene, then updates | Element-level "highest version wins"; `isDeleted` tombstones | Cursors | Room key | Files separate |
| **tldraw sync** | One authoritative `TLSocketRoom` per document (Cloudflare Durable Object in template) | Server-authoritative | Connect, then stream | Server is authority | Yes (presence records) | Room-scoped | **Separate `TLAssetStore`** |

### 1.2 System-by-system notes and reusable patterns

#### Google Keep
- **Official API is enterprise/admin only.** It is used for CASB-style audit and remediation (list, delete, permissions, download attachments) ([guide](https://developers.google.com/workspace/keep/api/guides)).
- **Data-model limits published in the API.** These are useful defaults for us ([Note resource](https://developers.google.com/workspace/keep/api/reference/rest/v1/notes)):
  - title < 1,000 chars
  - text body < **20,000 chars**
  - list item text < 1,000 chars
  - < **1,000 list items**
  - **one level of list nesting**
  - resources: `notes`, `notes.permissions` (batchCreate/batchDelete), `media` ([REST index](https://developers.google.com/keep/api/reference/rest))
- **Internal sync protocol, as reverse-engineered by `gkeepapi`.** Unofficial, so treat as indicative ([source](https://gkeepapi.readthedocs.io/en/latest/_modules/gkeepapi.html)).
  - `POST notes/v1/changes` carries `nodes` (dirty local nodes), `clientTimestamp` and `targetVersion` (the client's cursor).
  - The response carries `toVersion`, `nodes`, `truncated` (keep paging), `forceFullResync` and `upgradeRecommended`.
  - Notes are a **tree of nodes** of type `NOTE`, `LIST`, `LIST_ITEM` and `BLOB` ([node source](https://gkeepapi.readthedocs.io/en/latest/_modules/gkeepapi/node.html)).
  - Each node has `parentId`, `sortValue` (integer, for ordering), `baseVersion`, `serverId`, `superListItemId` (indent) and `timestamps{created, updated, trashed, deleted, userEdited}`.
  - Sharing uses `roleInfo` (O = owner, W = writer) and `shareRequests`.
  - Reminders used a separate `reminders/v1internal` API.
- **Per-user overlay on shared notes.** Collaborators "label, color, archive, or add reminders without changing the note for others". Deleting a shared note you own deletes it for everyone ([help](https://support.google.com/keep/answer/6101196?hl=en)).
- **Trash is kept 7 days** ([help](https://support.google.com/keep/answer/6262770?hl=en-GB)).
- **Reminders moved out of Keep.** In 2025–26 Keep reminders were migrated to **Google Tasks**:
  - Keep no longer sends reminder notifications itself; Tasks/Calendar do.
  - Location reminders were dropped ([9to5Google, Jan 2026](https://9to5google.com/2026/01/15/google-keep-reminders-migration-tasks-wide/); [Keep help](https://support.google.com/keep/answer/3187168?hl=en)).
  - Keep's default reminder presets are Morning 8 AM, Afternoon 1 PM and Evening 6 PM, user-configurable (secondary: [droid-life](https://www.droid-life.com/2020/09/09/random-tip-change-default-gmail-snooze-times-in-google-keep/)). This matters for load spikes (§2.7).
- Scale reference: 1B+ Play Store installs as of Dec 2020 ([9to5Google](https://9to5google.com/2020/12/08/google-keep-reaches-1-billion-downloads-on-the-play-store/)).

**Reusable patterns:**
- An account-level change cursor with paging and a server-forced full resync.
- Notes as a tree of small nodes, so each list item is independently versioned.
- Integer sort keys with gaps for ordering.
- Explicit tombstone timestamps (`trashed`, `deleted`).
- A per-user overlay for view state on shared notes.
- Reminders as a separate domain.

#### Apple Notes (CloudKit + CRDT)
- **Local storage.** A Core Data SQLite store. The note body is a **gzip/zlib-compressed protobuf** ([Simon Willison](https://simonwillison.net/2021/Dec/9/notes-on-notesapp/)).
- **CRDT inside the body.** Steve Dunham's reverse engineering describes a **CvRDT** for text with attribute runs. Tables are CRDTs too: rows and columns are *ordered sets of UUIDs* plus a cell map ([notesutils](https://github.com/dunhamsteve/notesutils/blob/master/notes.md)). Apple has not published the design. Internals are **UNVERIFIED** beyond reverse engineering.
- **CloudKit pattern (public, CKSyncEngine, WWDC23)** ([session](https://developer.apple.com/videos/play/wwdc2023/10188/)):
  - The app keeps a *pending changes* list.
  - The engine persists an opaque *state serialization*, which holds change tokens.
  - The OS schedules sync.
  - Whether Notes itself uses CKSyncEngine is **UNVERIFIED**.
- **Sharing.** CloudKit shares allow 100 participants plus the owner (Apple developer forum answer: [thread](https://developer.apple.com/forums/thread/783081)). A Notes-specific cap is **UNVERIFIED**.
- **Privacy.**
  - With Advanced Data Protection, Notes becomes end-to-end encrypted ([Apple security guide](https://support.apple.com/guide/security/sec973254c5f)).
  - "Recently Deleted" keeps notes for 30 days. In 2017 ElcomSoft found deleted notes retained server-side beyond that window ([9to5Mac](https://9to5mac.com/2017/05/19/apple-icloud-notes-deleted/)). That is a cautionary tale for purge pipelines.

**Reusable patterns:**
- The CRDT lives *inside* each note blob; the cloud stores opaque records.
- Structured content (tables, checklists) is modelled as ordered sets of stable IDs plus maps.
- The sync engine owns a persisted pending-changes queue and opaque server tokens.

#### Notion
- **WASM SQLite in the browser (2024)**
  - Uses the OPFS SyncAccessHandle Pool VFS.
  - A **SharedWorker with a single active tab** owns the DB; Web Locks handle failover.
  - Multiple writers corrupted data.
  - On slow devices they **race SQLite reads against the network**.
  - Result: **20% faster page navigation**, 28–33% in AU/CN/IN ([blog](https://www.notion.com/blog/how-we-sped-up-notion-in-the-browser-with-wasm-sqlite)).
- **Offline mode (Dec 11, 2025)** ([blog](https://notion.com/blog/how-we-made-notion-available-offline))
  - The best-effort SQLite cache became a persistent store, with `offline_page` and `offline_action` tables.
  - Each offline page records the *reasons* it is offline. A page is evicted only when its last reason disappears.
  - Offline pages are **migrated on the fly to a CRDT data model**.
  - Clients **subscribe to a push channel per offline page**, then fetch.
  - On reconnect they fetch only pages whose server version is newer than `lastDownloadedTimestamp`.
  - Database views auto-download up to 50 pages.
- **Server.** Postgres sharded into **480 logical shards** on 32 physical hosts (2021), re-sharded to 96 (2023). 480 was chosen because it is highly divisible ([sharding blog](https://notion.com/blog/sharding-postgres-at-notion); [data-lake blog](https://www.notion.com/blog/building-and-scaling-notions-data-lake)).
- **Trash** defaults to 30 days, configurable for enterprise ([help](https://www.notion.com/help/guides/notions-data-retention-settings)).

**Reusable patterns:**
- A single-writer local DB coordinator on web.
- An explicit "offline set" with reference-counted reasons.
- Push-then-pull invalidation (a cheap "something changed" ping, then a delta fetch).
- Logical shards decoupled from physical hosts.

#### Linear sync engine
- **Client side, per a detailed reverse-engineering study** ([wzhudev/reverse-linear-sync-engine](https://github.com/wzhudev/reverse-linear-sync-engine))
  - A **global, monotonically increasing `lastSyncId`** gives a total order over all transactions.
  - Bootstrap comes in three forms:
    - **full** (snapshot at a `lastSyncId`)
    - **partial** (newly visible sync groups)
    - **local** (from IndexedDB, then delta)
  - Models hydrate lazily.
  - Mutations are batched GraphQL transactions. The local DB is updated **only when the server's delta packet arrives**, so the server is the source of truth.
  - Pending transactions are **rebased** on incoming deltas.
  - Delta actions are Insert / Update / Delete / Archive / Unarchive.
- **Server side (Aug 18, 2026)** ([blog](https://linear.app/blog/rebuilding-delta-sync-read-path))
  - Linear rebuilt the **delta-sync read path**. Postgres stays authoritative for action payloads.
  - A secondary index (turbopuffer) holds **sync groups (permissions) and subscriptions as sorted posting lists of action IDs**.
  - Read is two-stage: a metadata scan, then late enrichment from Postgres.
  - Scale: the largest workspaces produce ~1M sync actions/day, with 20+ TB of sync actions in total and ~1 s p50 replication latency.
  - Key quote: the difference between "storing a change log and serving a change log".

**Reusable patterns:**
- Server-assigned sequence IDs.
- Three bootstrap modes.
- Lazy hydration.
- Optimistic local transactions with rebase.
- **Permissions materialized as indexable sets** rather than computed per read.

#### Figma multiplayer
- **Architecture (2019)** ([blog](https://www.figma.com/blog/how-figmas-multiplayer-technology-works/))
  - One process per open document, over WebSockets.
  - The client downloads the full doc.
  - **Per-property last-writer-wins, ordered by arrival at the server**. Figma calls this "CRDT-inspired, not true CRDTs".
  - **Unacknowledged local changes override incoming server values**, which avoids flicker.
  - The tree uses parent pointers. The server rejects reparenting that would create cycles.
  - Child order uses **fractional indexing**.
  - Offline edits are reapplied on reconnect.
- **Journal (2022)** ([blog](https://www.figma.com/blog/making-multiplayer-more-reliable/))
  - A write-ahead **journal in DynamoDB** with per-file sequence numbers.
  - Client updates arrive at ~30 FPS and are **batched** into journal entries.
  - Periodic checkpoints go to S3. Recovery is the checkpoint plus a replay of later entries.
  - Results: **95% of edits durable within 600 ms**, >2.2B changes/day, and potential loss cut from ~60 s to <1 s.
- **Non-document data** ([LiveGraph 100x, 2024](https://www.figma.com/blog/livegraph-real-time-data-at-scale/)): "LiveGraph" tails the Postgres WAL. It moved from mutation-based to **invalidation-based caching** keyed by "query shapes".
- **Postgres** was horizontally sharded into "colos" in 2024 ([blog](https://www.figma.com/blog/how-figmas-databases-team-lived-to-tell-the-scale/)).

**Reusable patterns:**
- Server-ordered LWW for scalar properties.
- Fractional indexing.
- Batching high-frequency ops into durable log entries.
- Checkpoint plus journal.
- A separate live-query system for non-document metadata.

#### Evernote (EDAM sync, USN)
- **Spec** ([edam-sync.pdf](https://dev.evernote.com/media/pdf/edam-sync.pdf), [NoteStore ref](https://dev.yinxiang.com/doc/reference/NoteStore.html))
  - **State-based replication.** The server keeps no per-client state and no fine-grained transaction log.
  - Each object carries a **USN that increases monotonically per account**. `updateCount` is the high-water mark.
  - Clients page through `getFilteredSyncChunk(afterUSN, maxEntries, filter)`.
  - Clients keep a **dirty flag** per object and do **all conflict resolution** themselves: field-by-field merge, or report a conflict.
  - `fullSyncBefore` forces a full resync when history was wiped or the server restored.
  - Note content and resources are fetched lazily and verified by **MD5 hash and length**.
- **Architecture (2011).** Shards of ~100k users each, with one Lucene index per user ([HighScalability](https://highscalability.com/evernote-architecture-9-million-users-and-150-million-reques/)): 9.5M users, 150M requests/day, 90 shards.
- **RENT (2024).** A rewritten metadata sync, "up to 3x faster", with a client DB "up to 70% smaller" ([AlternativeTo summary](https://alternativeto.net/news/2024/8/evernote-launches-new-sync-engine-rent-to-boost-sync-speed-app-startup-and-note-sharing); primary help page returned 403, so **UNVERIFIED** in detail).

**Reusable patterns:**
- Per-account (per-user) USN is the simplest delta cursor that scales with sharding by user.
- A server-controlled full-resync cut-off.
- Content-addressed attachments.
- Metadata sync separated from content sync.

#### Simplenote / Simperium
- **WebSocket protocol** ([docs](https://simperium.com/docs/websocket))
  - Bucket-level change version `cv` and per-object version `v`.
  - Each change carries `sv` (source version) → `ev` (end version), a **jsondiff** payload and a `ccid` (client change id, for idempotent ack and dedupe).
  - Offline catch-up sends `cv`. A `?` reply means the server no longer has that history, and the client must rebuild its index.
  - If a diff cannot be applied, the client resends the full object.
- **Status.** Automattic ended active development in March 2026; maintenance only ([AlternativeTo](https://alternativeto.net/news/2026/3/automattic-ends-simplenote-active-development-only-basic-maintenance-continues)).

**Reusable patterns:**
- A client-generated change id for exactly-once ack.
- Source/end version per object.
- An explicit "history compacted, please resync" response.
- A full-object fallback.

#### Obsidian Sync
- **Merge rules** ([troubleshoot](https://obsidian.md/help/sync/troubleshoot))
  - Markdown conflicts merge with **Google diff-match-patch**.
  - Settings JSON merges keys with local winning.
  - Binary files and canvases are **last-modified-wins**.
  - Since 1.9.7, users can opt to create **conflict files** instead.
- **Plans** ([plans](https://obsidian.md/help/sync/plans))
  - Standard: 1 GB, 5 MB max file, **1 month version history**.
  - Plus: 10–100 GB, 200 MB max file, **12 months history**.
  - History and attachments count against the quota.

**Reusable patterns:**
- Merge by content type.
- Version history as a paid, quota-counted feature.
- Opt-in conflict copies for users who distrust auto-merge.

#### Bear and Craft
- **Bear**
  - Syncs over **CloudKit**, so there is no separate account ([FAQ](https://bear.app/faq/syncing-privacy/)).
  - Locked notes use per-note **AES-GCM-256** keys and now cover attachments ([Bear 2.4](https://community.bear.app/t/bear-2-4-better-encryption-auto-todo-sorting-and-pin-within-tags/16571)).
  - Full E2EE relies on Apple ADP.
- **Craft** ([blog, Dec 2024](https://www.craft.do/blog/in-house-sync-protocol))
  - Moved from Realm Sync to an **in-house Sync Service** using socket.io, with Postgres (RDS) server-side and Realm on device.
  - Mobile syncs whole spaces for offline use; web syncs per document.
  - The conflict algorithm is unpublished (**UNVERIFIED**).

**Reusable patterns:**
- Opt-in per-note client-side encryption can coexist with server-side features for other notes.
- Different platforms can sync different scopes: full on mobile, per-doc on web.

#### Excalidraw and tldraw
- **Excalidraw** ([blog, 2020](https://plus.excalidraw.com/blog/building-excalidraw-p2p-collaboration-feature))
  - A relay server forwards **E2E-encrypted** messages without coordination.
  - Each element has `version` plus a random **`versionNonce`** for deterministic tie-breaks.
  - Deletes are **`isDeleted` tombstones**, so peers cannot resurrect them.
  - Merge is a union by element ID.
  - The decryption key is carried in the share link's URL fragment (**UNVERIFIED** in the cited post).
- **tldraw sync** ([docs](https://tldraw.dev/docs/sync))
  - **Exactly one authoritative `TLSocketRoom` per document**; two rooms cause divergence.
  - The Cloudflare template uses Durable Objects with built-in SQLite persistence.
  - **Assets live in a separate `TLAssetStore`** (bucket).
  - WebSocket hibernation avoids reconnect storms.

**Reusable patterns:**
- Version plus random-nonce LWW for element-level objects.
- Tombstones.
- Single-owner routing per document.
- Assets outside the sync channel.

#### Adjacent 2026 sync-engine landscape (for build-vs-buy)
- **Rocicorp Zero 1.0** (June 8, 2026) offers **query-driven sync**. A Postgres replica in zero-cache streams query results to the client cache, with permissions on mutators and queries ([InfoQ](https://infoq.com/news/2026/06/zero-version-1)).
- **ElectricSQL joined Databricks** and **Electric Cloud is shutting down** (Aug 11, 2026) ([PowerSync migration guide](https://docs.powersync.com/migration-guides/electric)). This is a vendor-risk lesson for hosted sync layers.
- **y-redis**, now "y/hub" ([repo](https://github.com/yjs/y-redis)):
  - Stateless Yjs servers.
  - **Redis streams as the hot log**, with workers compacting to S3/Postgres.
  - JWT auth with read/write/awareness permissions.
  - **AGPL/commercial, beta.** A useful reference design even if we don't adopt it.
- **Yjs protocol** ([y-protocols](https://github.com/yjs/y-protocols/blob/master/PROTOCOL.md))
  - SyncStep1 (state vector) → SyncStep2 (missing updates) → Update.
  - **Awareness is separate and ephemeral**: entries expire after **30 s** and are rebroadcast every ~15 s.
  - Read-only peers can be enforced by inspecting message type.

### 1.3 Pattern catalogue (what to reuse, mapped to our stack)

| Concern | Pattern to adopt | Prior art |
|---|---|---|
| Bootstrap | **Staged bootstrap**: (1) metadata + plaintext previews for pinned/recent notes, (2) the remaining metadata, (3) CRDT state lazily per note on open or idle. Local bootstrap from SQLite first, racing the network on slow devices | Linear (full/partial/local + lazy hydration), Notion (race SQLite vs network) |
| Delta cursor | **Per-user change feed with a server-assigned monotonic `seq`** ("USN") covering every note the user can see, plus every membership/ACL change. Client asks "everything after seq N" | Evernote USN, Keep `targetVersion`, Simperium `cv`, Linear `lastSyncId` |
| Per-object versions | Each note keeps a **server `rev` counter** (metadata), a **Yjs state vector** (content) and `client_change_id` for idempotency | Simperium `sv/ev/ccid`, Keep `baseVersion` |
| Server-authoritative order | Sync node appends client CRDT updates to the note's log in **server-arrival order**, then assigns the user feed `seq`. CRDT guarantees convergence; the server order gives cheap catch-up and audit | Figma, tldraw, Linear |
| Snapshots + log | **Append-only update log + periodic compacted snapshot** (on N updates or T idle). Recovery = snapshot + replay | Figma journal + checkpoint, y-redis workers |
| Escape hatch | `RESYNC_REQUIRED{epoch}` when the client cursor predates compaction/restore, or the schema changed | Evernote `fullSyncBefore`, Keep `forceFullResync`, Simperium `?` |
| Presence | Separate ephemeral channel (Yjs awareness over Valkey pub/sub), never persisted, 30 s TTL | Yjs awareness, Figma |
| Permission-aware partial sync | `note_member(user_id, note_id, role, joined_seq)` drives the feed. Gaining access = **partial bootstrap** of that note. Losing access = tombstone + local purge | Linear sync groups; Keep `roleInfo` |
| Merge semantics | Body/list text: **CRDT (Yjs Y.Text)**. Scalars (title, checked, colour): **LWW register** (Y.Map). Ordering: **fractional index**. **Per-user overlay** (colour, labels, pin, archive, reminder) outside the shared doc | Figma LWW + fractional indexing, Keep per-user overlays, Apple Notes ordered-set tables |
| Conflict UX | Invisible for text. For non-mergeable cases (e.g., attachment replaced on 2 devices) keep both versions; never silently drop | Obsidian, Evernote conflict copies |
| Attachments | **Content-addressed (SHA-256), immutable**, uploaded resumably and directly to object storage. The note references the hash. Thumbnails eager, originals lazy | Evernote MD5 resources, tldraw TLAssetStore |
| Local store | SQLite everywhere: expo-sqlite/op-sqlite on RN; WASM SQLite (OPFS) behind a single-writer SharedWorker on web, with IndexedDB fallback | Notion, Apple Notes, Linear (IndexedDB) |
| Sharding | Logical shards (e.g., 512) by **owner user_id**, mapped to few physical DBs at launch | Notion 480, Evernote 100k users/shard, Figma colos |

### 1.4 Twelve lessons that should shape our design

1. **Keep two channels: durable document ops and ephemeral presence.** Cursors and "who's here" go over a TTL'd channel (Yjs awareness: 30 s expiry) and never touch Postgres. Mixing them inflates write load and history ([y-protocols](https://github.com/yjs/y-protocols/blob/master/PROTOCOL.md)).
2. **The server assigns a monotonic sequence per sync scope, and delta sync is "give me everything after N".** This holds even with CRDTs. A per-user USN-style feed (Evernote) is the simplest form, and it shards cleanly by user ([Evernote spec](https://dev.evernote.com/media/pdf/edam-sync.pdf); [Linear](https://github.com/wzhudev/reverse-linear-sync-engine)).
3. **Ship a full-resync escape hatch from day one.** It needs a server "epoch" so restores, compaction or schema changes can invalidate client cursors safely (Evernote `fullSyncBefore`, Keep `forceFullResync`, Simperium `?`).
4. **Bootstrap in stages and hydrate lazily.** For 5k notes, ship metadata and previews first and CRDT state on demand. Race the local cache against the network on slow devices ([Linear](https://github.com/wzhudev/reverse-linear-sync-engine); [Notion WASM SQLite](https://www.notion.com/blog/how-we-sped-up-notion-in-the-browser-with-wasm-sqlite)).
5. **Model permissions as indexed membership that the change feed is built from.** Do not filter permissions at read time. Linear had to rebuild its read path around posting lists of action IDs per sync group, because *serving* a log is harder than *storing* one ([Linear 2026](https://linear.app/blog/rebuilding-delta-sync-read-path)).
6. **Choose merge semantics per field, not per app.** Use a CRDT for text, LWW registers for scalars, fractional indices for order, and a **per-user overlay** for view state (Keep's colour, labels, archive and reminders are per collaborator) ([Figma](https://www.figma.com/blog/how-figmas-multiplayer-technology-works/); [Keep help](https://support.google.com/keep/answer/6101196?hl=en)).
7. **Store a batched journal of updates plus periodic snapshots.** Recover by replay, and batch high-frequency client ops into one durable write. Figma got 95% of edits durable within 600 ms this way ([Figma](https://www.figma.com/blog/making-multiplayer-more-reliable/)).
8. **For millions of tiny documents, prefer stateless append-and-fan-out over "load every doc into a room process".**
   - Room-per-doc (Figma, tldraw) fits big, long-lived docs.
   - Keep notes are small and short-lived in edit sessions, so a y-redis-style stateless relay with worker compaction is cheaper.
   - Exactly-one-owner routing is still needed for any server-side doc mutation ([tldraw](https://tldraw.dev/docs/sync); [y-redis](https://github.com/yjs/y-redis)).
9. **Attachments are a separate, content-addressed, immutable pipeline.** Hash-addressed blobs give dedupe, integrity checks and simple caching. Never route bytes through the sync socket ([Evernote spec](https://dev.evernote.com/media/pdf/edam-sync.pdf); [tldraw assets](https://tldraw.dev/docs/sync)).
10. **The local DB is the product; make it SQLite with one writer.**
    - Notion saw multi-writer OPFS corruption and fixed it with SharedWorker + Web Locks single ownership.
    - Mobile gets native SQLite.
    - The shared TS core must abstract over the storage engine ([Notion](https://www.notion.com/blog/how-we-sped-up-notion-in-the-browser-with-wasm-sqlite)).
11. **Conflict UX should be invisible, and deletes are tombstones.**
    - Text and list merges should never prompt.
    - Deletes are tombstones (`trashed`/`deleted` timestamps, Excalidraw `isDeleted`), so offline peers can't resurrect notes.
    - Conflict copies are only for genuinely non-mergeable binaries (Obsidian, Evernote).
12. **Design sharding, retention and *real* purge up front.**
    - Use logical shards decoupled from hosts (Notion 480 → 96 physical).
    - Keep per-user data locality (Evernote ~100k users/shard).
    - Build an auditable purge path that reaches replicas, search, object storage and backups. Apple's 2017 retained-deleted-notes story is the counter-example ([Notion](https://notion.com/blog/sharding-postgres-at-notion); [9to5Mac](https://9to5mac.com/2017/05/19/apple-icloud-notes-deleted/)).

---

## Part 2: Capacity model (back-of-envelope)

Targets: **1M MAU at end of launch year (Y1)**, growing to **10M MAU by end of Y3** (ASSUMPTION for interpolation: Y2 end = 3.5M). All derived numbers come from the inputs below; change an input and the rest scales. Arithmetic was run in a script; results are rounded.

### 2.1 Inputs and assumptions

| # | Input | Planning value | Basis |
|---|---|---|---|
| A1 | DAU/MAU | **0.35** | Productivity apps run ~0.2–0.6; weekly-productivity ~0.2–0.3 ([Phiture](https://phiture.com/mobilegrowthstack/user-stickiness-guide/), [mwm.ai](https://mwm.ai/glossary/dau-mau)). Keep-like quick capture skews daily. ASSUMPTION |
| A2 | Registered accounts / MAU | **1.6** | Dormant accounts still store data. ASSUMPTION |
| A3 | New notes per DAU per day | **1.0** | MIT list.it field study: 42 participants added ~35 notes/day in aggregate ≈ **0.83/participant/day** ([Van Kleek et al., CHI'09](https://people.csail.mit.edu/emax/papers/listit-camera.pdf)) |
| A4 | Notes retained (not purged) | **80%** | list.it: notes "rarely revised or deleted". ASSUMPTION |
| A5 | Import on signup (Keep Takeout etc.) | **10% of accounts × 150 notes** | ASSUMPTION; a Keep clone must import |
| A6 | Note plaintext size | median ≈ **100 B**, mean **600 B**, hard cap 20k chars | list.it: median **29 chars**, mean **62 chars** (σ = 164). Keep allows up to 20k chars + lists, so planning mean is ~10× list.it ([paper](https://people.csail.mit.edu/emax/papers/listit-camera.pdf); [Keep API limits](https://developers.google.com/workspace/keep/api/reference/rest/v1/notes)) |
| A7 | CRDT snapshot size | **2 × text + 400 B** ≈ 1.6 KB mean | Yjs on a 104,852-char real editing trace (259,778 ops) encodes to **159,929 B ≈ 1.5 B/char** with GC ([crdt-benchmarks](https://github.com/dmonad/crdt-benchmarks)). Notes are lightly edited (40% of edits change 1–2 chars per list.it); add fixed per-doc structure for list items and props |
| A8 | History / update log (amortized) | **2 × snapshot** ≈ 3.2 KB | Raw updates retained ≤ 7 d, then compaction; 30-day version snapshots. ASSUMPTION |
| A9 | Search index + materialized plaintext | **1.0 KB + 0.6 KB** | Postgres FTS/trigram ≈ 1–1.5× text. ASSUMPTION |
| A10 | Row + index overhead | **0.5 KB/note** | ~300 B row + indexes. ASSUMPTION |
| A11 | Notes with images | **8%**, 1.3 images each | ASSUMPTION (receipts, whiteboards, photos) |
| A12 | Stored image size | **≈ 450 KB** (≤ 2048 px long edge, WebP/JPEG q≈80, + 2 thumbnails) | Raw 12 MP photo ≈ 2 MB HEIC / 3–6 MB JPEG ([SamMobile](https://www.sammobile.com/news/how-much-storage-do-photos-and-videos-actually-use/), [img2go](https://www.img2go.com/blog/what-are-heic-files)). Client-side downscale is a **policy decision** (see 2.4) |
| A13 | Other attachments (drawings, audio) | 2% of notes × 500 KB | ASSUMPTION |
| A14 | Active editing per DAU | **4 min/day**; client coalesces to **≤ 4 updates/s** while typing, avg **2/s** | list.it: 50% of notes captured in ≤ 10 s; edits are small. Throttled Yjs updates. ASSUMPTION |
| A15 | Metadata ops per DAU (pin, colour, label, archive, reorder, check) | **20/day** | ASSUMPTION |
| A16 | Peak-hour / daily-average rate | **3×** | Diurnal, US/EU-heavy user base. ASSUMPTION |
| A17 | Concurrent WebSocket share of DAU at peak | **18%** | 40% web × 50% keep a tab open × 0.6 peak overlap ≈ 12%; mobile foreground ≈ 0.5%; ×1.2 multi-tab, rounded up. ASSUMPTION |
| A18 | Devices per user | **1.8** | ASSUMPTION |
| A19 | Reminder fires | **0.2 per MAU per day**, 2.2 pushes per fire | ASSUMPTION |
| A20 | Egress per DAU per day | **1.5 MB** | Breakdown in §2.6 |

### 2.2 Notes per user (distribution)

Creation-driven totals (A3 × DAU × days × A4 + imports):

- Notes created: Y1 ≈ 64M, Y2 ≈ 287M, Y3 ≈ 862M (avg MAU per year × 0.35 × 365).
- **Stored notes:**
  - Y1 = 64M × 0.8 + 1.6M × 0.1 × 150 ≈ **75M**
  - Y3 = (64 + 287 + 862)M × 0.8 + 16M × 0.1 × 150 ≈ **1.21B**
- Mean ≈ **47 notes/account (Y1)** and **~76/account (Y3)**, or ~120 per MAU at Y3.

Heavy-tailed distribution consistent with that mean (Y3, ASSUMPTION):

| Notes per account | Share of accounts | Bucket mean | Contribution to mean |
|---|---|---|---|
| 0 (signed up, idle) | 15% | 0 | 0 |
| 1–10 | 35% | 5 | 1.8 |
| 11–100 | 38.35% | 35 | 13.4 |
| 101–1,000 | 10% | 250 | 25 |
| 1,001–5,000 | 1.5% | 1,800 | 27 |
| > 5,000 | 0.15% | 8,000 | 12 |
| **Total** | 100% | | **≈ 79** |

- Median ≈ 10 notes, which matches list.it's 10-day median of 11.
- p99 ≈ 1,500–2,000.
- p99.9 ≈ 8,000–10,000.
- **Client performance is designed at 5k notes and must degrade gracefully to 20k–50k.**

### 2.3 Per-note size and database size

| Component | Bytes (mean) |
|---|---|
| Row + indexes (A10) | 500 |
| CRDT snapshot (A7) | 1,600 |
| Update log + version history, amortized (A8) | 3,200 |
| Search index (A9) | 1,000 |
| Materialized plaintext/preview (A9) | 600 |
| **Total per note** | **≈ 6.9 KB** (before TOAST compression; lz4 ≈ 2× on text is headroom, not counted) |

| | Y1 (1M MAU) | Y3 (10M MAU) |
|---|---|---|
| Stored notes | 75M | 1.21B |
| Notes × 6.9 KB × 1.10 (labels, shares, reminders, devices, sessions, outbox) | **≈ 0.57 TB** | **≈ 9.2 TB** |
| With 1 sync standby + 1 read replica (×3) | ~1.7 TB provisioned | ~28 TB provisioned |
| WAL/PITR archive (35 days) | ~0.3–0.6 TB | ~3–6 TB |

**Implication.** At ~9 TB, a single Postgres primary is technically possible but bad for vacuum, restore time (RTO) and IOPS (Figma hit exactly this: [blog](https://www.figma.com/blog/how-figmas-databases-team-lived-to-tell-the-scale/)). So:
- Use **logical shards (e.g., 512) keyed by owner `user_id` from day one**.
- Run 1 physical cluster in Y1 and ~8 in Y3, at ~1.2 TB per shard.
- Put the high-churn update log in a **time-partitioned table** (daily partitions, dropped after compaction) to avoid vacuum churn.

### 2.4 Object storage

| | Y1 | Y3 |
|---|---|---|
| Images = notes × 8% × 1.3 | 7.8M | 126M |
| Processed images (450 KB) + other attachments | **≈ 4.3 TB** | **≈ 69 TB** |
| If originals are also kept (+3 MB/image) | ≈ 28 TB | ≈ 447 TB |

**Decision flag.** Keeping full-resolution originals multiplies object storage by ~6.5×. Recommended options:
- Store originals only for paid tiers, or
- Store originals with lifecycle to infrequent-access after 30 days (R2 IA $0.01/GB-mo; S3 IA).

### 2.5 Sync traffic: CRDT messages, connections, write QPS

**Upstream CRDT/ops messages per DAU per day** = 4 min × 60 s × 2 msg/s + 20 metadata ops = **500**.

| Metric | Formula | Y1 | Y3 |
|---|---|---|---|
| DAU | MAU × 0.35 | 350k | 3.5M |
| Upstream msgs/day | DAU × 500 | 175M | 1.75B |
| Upstream msgs/s (avg → **peak**) | /86,400, × 3 | 2.0k → **6.1k/s** | 20k → **61k/s** |
| Fan-out deliveries/s (peak) | × 0.3 online subscribers (own other devices + collaborators) | ~1.8k/s | ~18k/s |
| Server acks/s (peak) | 1 per upstream msg (coalescible) | ~6.1k/s | ~61k/s |
| Concurrently-edited notes at peak | DAU × (4/1440) × 3 | ~2.9k | ~29k |
| **Concurrent WebSockets (peak)** | DAU × 18% | **~63k** | **~630k** |
| Presence msgs/s (peak) | awareness heartbeat every 15 s, only for open shared notes (~10% of connections) | ~0.4k/s | ~4k/s |

**Postgres write QPS (peak).** The design stores every acked update durably, using group commit.

| Write type | Y1 rows/s | Y3 rows/s | Notes |
|---|---|---|---|
| CRDT update-log appends | 6.1k | 61k | Group-commit per sync node every ≤ 50 ms → ~20 tx/s/node (Y1 ~120 tx/s; Y3 ~720 tx/s). Spread over 8 shards ≈ 7.6k rows/s/shard at Y3 |
| Snapshot compactions (snapshot + note row + search doc + feed `seq`) | ~75 | ~730 | 6 notes touched/DAU/day × 3 peak |
| Metadata/overlay ops + change-feed rows | ~245 | ~2.4k | |
| Sessions, push tokens, reminders, shares | < 50 | < 500 | |
| **Total** | **≈ 6.5k rows/s** | **≈ 65k rows/s** | |

**Sanity check.** DBOS sustained **144k writes/s** on a single RDS db.m7i.24xlarge with io2 ([DBOS](https://dbos.dev/blog/benchmarking-workflow-execution-scalability-on-postgres)). So ~8k rows/s per shard on 8 vCPU shards is comfortable with batching.

**Alternative (Option B).** Buffer updates in Valkey Streams (y-redis pattern) and persist only compactions to Postgres. This cuts Postgres write load ~50–80×, but RPO for acked edits then depends on Valkey durability. Keep Option A (Postgres group commit) as the baseline; it is simpler to reason about for RPO = 0.

### 2.6 Egress bandwidth

Per DAU per day (ASSUMPTION; all items cache-friendly):

| Item | KB/DAU/day |
|---|---|
| CRDT deltas, acks, change-feed catch-ups | ~150 |
| WebSocket keepalives (TLS frames, ~30 s pings on long-lived web tabs) | ~100 |
| Image downloads (on other devices/collaborators; thumbnails eager, full size on open) | ~500 |
| Web app asset updates after deploys (PWA service-worker cache) | ~250 |
| New-device/reinstall bootstrap, amortized (≈1% of MAU/day × ~5 MB) | ~250 |
| REST/JSON (search, presign, auth, share) | ~100 |
| Headroom | ~150 |
| **Total** | **≈ 1.5 MB** |

- **Y1:** 350k × 1.5 MB × 30 ≈ **16 TB/month**.
- **Y3:** 3.5M × 1.5 MB × 30 ≈ **158 TB/month**.
- **Bootstrap payload, 5k-note user:**
  - metadata + previews ≈ 5k × 300 B = 1.5 MB raw (~0.5 MB gz)
  - CRDT snapshots ≈ 5k × 1.6 KB = 8 MB raw (~3–4 MB compressed; Yrs compresses ~2.5× per [Loro's comparison](https://www.loro.dev/docs/performance))
  - Total ≈ **4–5 MB**, which is about 4 s at 10 Mbps. This is why bootstrap is staged.

### 2.7 Reminders and notifications

| Metric | Y1 | Y3 |
|---|---|---|
| Reminder fires/day (0.2 × MAU) | 200k | 2M |
| Push sends/day (× 2.2 devices) | 440k | 4.4M |
| **Worst single minute** | ~12.5k fires → **~27.5k pushes/min** | ~125k fires → **~275k pushes/min** |

**Spike model (ASSUMPTION).** Worst minute = daily fires × 25% (share on the "Morning 8:00" preset) × 25% (share of users in the largest timezone band) ≈ 6.25% of daily fires. Keep's presets are 8 AM / 1 PM / 6 PM local.

**Limits that bound this:**
- FCM default quota is **600k messages/min per project**, with a 15–30-day notice for temporary increases.
- FCM per-device limit is 240 msgs/min (Android) ([FCM quotas](https://firebase.google.com/docs/cloud-messaging/throttling-and-quotas)).
- APNs raises HTTP/2 concurrent streams to ~1,000 per connection after auth; use multiple connections ([Pushy best practices](https://github.com/relayrides/pushy/wiki/Best-practices)).
- At Y3 the worst minute is at ~46% of the FCM default if all pushes went to FCM. Fine, but within ~2× of the cap once growth is included.

**Design consequences:**
- **Schedule reminders locally on each device** (Android AlarmManager/WorkManager, iOS `UNUserNotificationCenter`). Server push is only for web and as a fallback.
- iOS keeps only the **soonest 64 pending local notifications** per app, so the client must reschedule a rolling window ([Apple forums](https://developer.apple.com/forums/thread/811171)).
- Server-side: a timing-wheel / time-bucketed queue (Valkey sorted set per minute, or a Postgres `due_at` index), pre-fetched 5 min ahead, sending across the minute.
- Store reminders as **local wall time + IANA timezone** for DST-correct recurrences.

### 2.8 Search query rate

- **Local-first means most search runs on device**: SQLite FTS5 on RN, WASM SQLite FTS on web.
- Server search is used only for web cold start before bootstrap completes, attachment OCR text, and very large accounts.
- Assumptions (ASSUMPTION): 1.5 searches/DAU/day, ×5 search-as-you-type queries, 20% hit the server.
  - Peak server search ≈ **18 qps (Y1)** / **180 qps (Y3)**.
  - If 100% went to the server: 91 / 910 qps.
- Either is easy for per-user-scoped Postgres FTS (`WHERE note_id IN (user's notes)`) plus `pg_trgm` for CJK/substrings.
- **No dedicated search cluster is needed through Y3.**

### 2.9 Fleet sizing

**Per-node planning capacities (ASSUMPTION; validate with load tests):**
- **Sync node** (4 vCPU / 8 GiB, e.g., AWS c8g.xlarge):
  - 25k WebSocket connections
  - ~2k msgs/s
  - ~2k hot docs in memory
  - Yjs memory ≈ 23 B/char in JS, so ~50 KB per typical note doc ([Yjs benchmarks via crdt-benchmarks](https://github.com/dmonad/crdt-benchmarks))
  - The connection memory budget is generous. uWebSockets.js claims ~1M sockets in ~111 MB of user space, so the real limit is CPU and TLS, not socket memory ([uNetworking](https://unetworkingab.medium.com/millions-of-active-websockets-with-node-js-7dc575746a01))
- **API node** (4 vCPU): ~400 req/s at 50% CPU, for bootstrap/delta/search/presign/auth.
- **Worker node** (4 vCPU): compaction (~2–5 ms CPU per Yjs merge+encode), image processing (~200 ms/image), reminder dispatch, exports, email, purge jobs.

| Tier | Y1 (1M MAU) | Y3 (10M MAU) |
|---|---|---|
| **Sync nodes** | 63k conns / 25k = 3 → **6** (2 per AZ × 3 AZ; covers reconnect storms) | 630k / 25k = 26 → **36** (+25% headroom for deploy drain/storms; 12 per AZ) |
| **API nodes** | 730 req/s / 400 = 2 → **4** | 7.3k / 400 = 18 → **21** |
| **Workers** | **3** | **12** (compaction ~730/s peak ≈ 4 cores; images ~10/s peak; reminders/push; exports) |
| **Postgres** | **1 cluster**: primary db.r8g.2xlarge (8 vCPU, 64 GiB) Multi-AZ + 1 read replica; 512 logical shards; ~1 TB gp3 | **8 physical shards**, each db.r8g.2xlarge Multi-AZ (~1.2 TB data each) + 4 read replicas; or 4 × r8g.4xlarge |
| **Valkey** (presence, cross-node pub/sub fan-out, rate limits, routing, idempotency keys) | 1 shard: primary + replica, cache.r7g.large | 3 shards × (primary + replica), cache.r7g.large, sharded pub/sub (`SPUBLISH`) |
| **Object storage** | ~4.3 TB | ~69 TB |
| **CDN egress** | ~16 TB/mo | ~158 TB/mo |
| **L4 LB for WebSockets** | NLB: 63k active conns ≈ 0.6 NLCU on that dimension | 630k ≈ 6.3 NLCU (100k active conns per NLCU) ([pricing summary](https://spendark.com/blog/alb-vs-nlb-pricing/)) |

### 2.10 Monthly cost ballpark

**AWS (us-east-1, on-demand list).** Unit prices:
- c8g.xlarge **$0.1595/h ≈ $116/mo** ([Holori](https://calculator.holori.com/aws/ec2/c8g.xlarge/us-east-1))
- RDS PostgreSQL db.r8g.2xlarge **$698/mo** single-AZ; 1-yr RI ≈ $440/mo ([Bytebase dbcost](https://www.bytebase.com/dbcost/rds/instance/db.r8g.2xlarge/)). Multi-AZ assumed 2× (**UNVERIFIED** exact)
- RDS gp3 **$0.115/GB-mo** (×2 Multi-AZ) ([InfraTally](https://infratally.com/articles/aws-rds-pricing-explained-2026/))
- ElastiCache Valkey cache.r7g.large **$0.174/h** ([Upstash](https://upstash.com/blog/aws-elasticache-pricing-explained-2026-full-cost-breakdown))
- S3 Standard **$0.023/GB-mo** (first 50 TB); internet egress **$0.09 → $0.085 → $0.07 → $0.05/GB** tiers ([CloudForecast](https://www.cloudforecast.io/blog/amazon-s3-pricing-and-optimization-guide/))
- **CloudFront flat-rate Premium: $1,000/mo for 50 TB + 500M requests; $3,500 for 200 TB; up to $10,000 for 600 TB**, no overage charges. WebSockets supported. Includes WAF/DDoS and 5 TB of S3 credits ([AWS docs](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/flat-rate-pricing-plan.html))

| Line item (USD/mo) | Y1 | Y3 |
|---|---|---|
| Sync nodes (6 / 36 × $116) | 700 | 4,190 |
| API nodes (4 / 21) | 470 | 2,450 |
| Workers (3 / 12) | 350 | 1,400 |
| RDS instances (1 Multi-AZ + 1 replica / 8 Multi-AZ + 4 replicas) | 2,090 | 13,960 |
| RDS storage (gp3, incl. Multi-AZ doubling) | 350 | 3,450 |
| Valkey (2 / 6 nodes) | 250 | 760 |
| S3 storage + requests | 150 | 1,860 |
| Egress via CloudFront flat-rate (Premium 50 TB / 200 TB tier) | 1,000 | 3,500 |
| *(pay-as-you-go egress instead)* | *(~1,390)* | *(~11,700)* |
| Load balancers | 100 | 600 |
| Observability, logs, NAT, WAF extras, SES email, KMS, cross-region backup copies | 1,500 | 6,000 |
| **Total on-demand** | **≈ $7.0k** | **≈ $38k** |
| **With 1-yr commitments** (RDS RI ~37% off; ~30% compute/cache savings plan) | **≈ $5.7k** | **≈ $30k** |
| **Per MAU-month** | ~$0.006–0.007 | ~$0.003–0.004 |

**Cheaper provider: Hetzner dedicated (EU) + Cloudflare R2 + Cloudflare CDN, self-managed.**

Hetzner prices changed twice in 2026:
- **April 1**: cloud +30–35%.
- **June 15**: new-order prices for dedicated vCPU cloud roughly doubled to tripled. Germany is ~+99% on average, the US ~+158%, and existing contracts are grandfathered ([heise](https://heise.de/-11333037); [wz-it](https://wz-it.com/en/blog/hetzner-price-increase-june-2026-cpx-ccx-alternatives/)).

Sources conflict on the **AX102** price (16-core Ryzen 9 7950X3D, 128 GB ECC, 2 × 1.92 TB NVMe, unmetered 1 Gbit/s ([Hetzner](https://www.hetzner.com/dedicated-rootserver/ax102/))):
- **€257.30/mo** ([AgentDeals, Oct 2 2026](https://agentdeals.dev/hetzner-pricing-2026))
- **€452.30/mo** for "AX102-1" (search snippet)
- **UNVERIFIED**: check the configurator.

R2 costs **$0.015/GB-mo with zero egress fees** ([Cloudflare R2 pricing](https://developers.cloudflare.com/r2/pricing/)).

| Line item | Y1 | Y3 |
|---|---|---|
| Postgres (Patroni + pgBackRest → R2): 3 AX102 (primary, sync standby, async replica) / 20 AX102 (8 shards × 2 + 4 replicas) | €0.8k–1.4k | €5.1k–9.0k |
| App tier (sync + API + workers + Valkey in containers): 3 / 12 AX102 | €0.8k–1.4k | €3.1k–5.4k |
| Misc (AX42 observability €97 each, Hetzner LB, IPs) | ~€0.15k | ~€0.8k |
| R2 storage + ops | ~$0.1k | ~$1.3k |
| Cloudflare plan (CDN/WAF; WebSockets supported) | ~$0.25k | ~$0.25k–Enterprise (**UNVERIFIED**) |
| **Total infra** | **≈ €2.0k–3.2k** | **≈ €10.5k–16.5k** |

**AWS vs Hetzner + R2 verdict:**
- Infra is **~2–3× cheaper** on Hetzner + R2. The gap was ~4–5× before the 2026 repricing.
- But the Hetzner route needs **~1–2 more SRE/DBA FTEs**: HA Postgres, backups, failover drills, kernel/security patching. At Y1 that cancels the savings.
- Hetzner dedicated servers are EU-only. US users get ~90–110 ms transatlantic RTT, and Hetzner US is cloud-only and now ~+158%.
- **Recommendation:**
  - Y1: AWS (or another managed Postgres provider), with object storage on R2 to kill egress.
  - Y2+: re-evaluate hybrid (stateful sync/API on cheaper metal, managed Postgres) once the team has SRE depth.
  - Keep everything S3-API and Postgres-wire compatible so moving is a migration, not a rewrite.

---

## Part 3: Non-functional requirements baseline

### 3.1 Availability SLOs (monthly, per component)

| Component | SLO | Error budget / month | Notes |
|---|---|---|---|
| **Local editing (create/edit/search/delete on device)** | **100% (no server dependency)** | none | Offline-first is a product requirement. Server outage must not block any local action |
| Sync service (accept WebSocket + durably ack updates) | **99.95%** | 21.9 min | Clients queue and retry with jittered backoff. Outage = delayed sync, not data loss |
| Delta catch-up / bootstrap API | 99.95% | 21.9 min | |
| Auth (login, token refresh) | 99.95% | 21.9 min | Long-lived refresh tokens; offline clients keep working when auth is down |
| Sharing / invitations API | 99.9% | 43.8 min | |
| Attachment upload/download (presign + object store + CDN) | 99.9% | 43.8 min | Bounded by S3's 99.9% SLA; uploads queue locally |
| Server-side search | 99.5% | 3.6 h | Falls back to local search |
| Reminder delivery | **99.9% of reminders fire within 60 s of due time** | — | Local scheduling is primary on mobile; server push is a fallback and for web |
| Data export (DSAR) | 99% | 7.3 h | Async job |
| **Durability of acked edits** | **≥ 99.999999999% (no loss)** | — | Ack only after synchronous-standby commit. Client keeps unacked updates |

### 3.2 Sync latency targets

| Path | p50 | p99 | Rationale / reference |
|---|---|---|---|
| Keystroke → rendered locally | ≤ 16 ms (1 frame) | ≤ 50 ms | INP "good" ≤ 200 ms ([web.dev](https://web.dev/articles/vitals)) |
| Keystroke → persisted in local DB | ≤ 250 ms (debounced) | ≤ 1 s; flushed on background/blur | Crash safety |
| Client update → server durable ack (same region) | ≤ 150 ms | ≤ 600 ms | Figma: 95% of edits durable within 600 ms ([Figma](https://www.figma.com/blog/making-multiplayer-more-reliable/)) |
| **Cross-device / collaborator visible** (both online, same region) | **≤ 300 ms** | **≤ 1.5 s** | |
| Cross-device, cross-region | ≤ 500 ms | ≤ 2.5 s | |
| Reconnect catch-up (≤ 24 h offline, ≤ 500 changed notes) | ≤ 1 s | ≤ 5 s | Linear's index replication is ~1 s p50 ([Linear](https://linear.app/blog/rebuilding-delta-sync-read-path)) |
| Presence (cursor / "editing now") | ≤ 150 ms | ≤ 1 s | Ephemeral; may drop |
| Background mobile sync after push | ≤ 30 s | best effort | OS-controlled |
| Share grant → note appears for recipient (online) | ≤ 2 s | ≤ 10 s | Partial bootstrap of one note |

### 3.3 Startup and time-to-first-note (account with **5,000 notes**)

| Scenario | Target | Notes |
|---|---|---|
| Mobile cold start → first note cards painted (from local SQLite) | **p50 ≤ 0.8 s, p90 ≤ 1.5 s** on a mid-range Android (~4-year-old mid-tier); p50 ≤ 0.6 s on recent iPhone | Read only the first screen (pinned + recent ~50) via an indexed query; virtualized masonry grid |
| Mobile cold start → fully interactive (scroll, search) | p50 ≤ 1.5 s | |
| Local search over 5k notes | p95 ≤ 50 ms | FTS5 |
| Web/PWA warm start (SW cache + OPFS SQLite) → first notes | **LCP p75 ≤ 1.0 s**; INP p75 ≤ 200 ms; CLS ≤ 0.1 | Core Web Vitals "good" = LCP ≤ 2.5 s, INP ≤ 200 ms, CLS ≤ 0.1 ([web.dev](https://web.dev/articles/vitals)) |
| Web first visit, new device (5k notes) → first notes | p75 ≤ 1.5 s (first page from server) | Staged bootstrap: pinned/recent first |
| Full background bootstrap of 5k notes | ≤ 10 s on 10 Mbps; resumable | ~4–5 MB payload (§2.6) |
| Memory with 5k notes loaded | web ≤ 300 MB; RN ≤ 250 MB | Lazy CRDT hydration; only open notes hold a Y.Doc |
| Local DB size for 5k notes | ~20 MB (excl. images) | 5k × (1.6 KB CRDT + 0.6 KB text + 0.5 KB row + 1 KB FTS) |
| Graceful-degradation ceiling | 50k notes still usable (p50 open ≤ 3 s) | Power users |

### 3.4 RPO / RTO and backups

| Scenario | RPO | RTO | Mechanism |
|---|---|---|---|
| Single DB instance / AZ failure | **0** (acked writes) | **≤ 5 min** | Synchronous standby (Multi-AZ / Patroni sync replica); automatic failover |
| Logical corruption / bad migration on one shard | ≤ 5 min (to chosen point) | **≤ 2 h per shard** | PITR from continuous WAL archive. Shards kept ≤ ~1.5 TB so restore fits RTO |
| Region loss | **≤ 5 min** | **≤ 4 h** | Cross-region async replica + WAL archive (archive_timeout ≤ 60 s); warm standby; DNS failover |
| Object storage region loss | ≤ 15 min | ≤ 4 h | Cross-region replication (S3 RTC targets 15 min); content-addressed objects make re-upload idempotent |
| Valkey loss | n/a (ephemeral only) | ≤ 5 min | Presence and pub/sub rebuild themselves; no durable data in Valkey under Option A |
| **Server rolled back behind clients** | **0 for data still on any device** | — | Server **epoch** bump → clients re-upload unacked/newer CRDT state (CRDT merge is idempotent). This is lesson 3 |

**Backup strategy:**
- Continuous WAL archiving plus daily base backups.
- **PITR window 35 days.** This matches the RDS maximum automated-backup retention ([AWS docs](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_WorkingWithAutomatedBackups.html)).
- Monthly snapshot retained 90 days, encrypted with a separate KMS key and copied **cross-account and cross-region with Object Lock (immutable)** against ransomware and operator error.
- Object storage: versioning with 30-day non-current version expiry.
- **Quarterly restore drills** (one shard plus one bucket prefix, timed against RTO).
- **Deletion ledger.** A durable list of purged user/note IDs is re-applied automatically after any restore. This keeps GDPR erasure true across restores.

### 3.5 Data retention

| Data | Retention | Reference |
|---|---|---|
| Trash | **30 days**, then auto-purge (user can empty earlier) | Keep 7 d ([help](https://support.google.com/keep/answer/6262770?hl=en-GB)); Apple Notes 30 d ([Apple](https://support.apple.com/guide/icloud/mm2f42f05cb9/icloud)); Notion default 30 d ([help](https://www.notion.com/help/guides/notions-data-retention-settings)) |
| Archived notes | Indefinite (archive ≠ delete) | Keep semantics |
| Shared note deleted by owner | Deleted for everyone, into the owner's trash; collaborators lose access immediately | Keep ([help](https://support.google.com/keep/answer/6101196?hl=en)) |
| Collaborator "remove from my notes" | Removes only their membership and overlay | |
| Raw CRDT update log | Compacted within 24 h; raw partitions dropped after **7 days** | |
| Version history snapshots | 30 days (free); longer as a paid feature | Obsidian 1 / 12 months ([plans](https://obsidian.md/help/sync/plans)) |
| Account deletion | 14-day grace (undo), then purge from primaries ≤ 30 days; backups expire ≤ 35 days (+ 90-day monthly snapshot, covered by the deletion ledger) | |
| Application logs | 30 days, PII-scrubbed, **no note content** | |
| Security / audit logs | 1 year | |
| Product analytics | Pseudonymous, 14 months | |

### 3.6 GDPR / CCPA

| Requirement | Our SLA | Legal ceiling |
|---|---|---|
| Access / portability export | Self-serve. JSON + Markdown/HTML + attachments ZIP, plus a Keep-Takeout-compatible format. **Ready ≤ 24 h p95** | GDPR: 1 month, extendable by 2 months ([Art. 12(3)](https://gdpr-info.eu/art-12-gdpr/)). CCPA: 45 days + 45 ([§7021](https://prighter.com/resources/laws/ccpa-regulations/sections/7021)) |
| Erasure / account deletion | Access revoked immediately; primaries purged ≤ 30 days (target 7); backups age out ≤ 35 days (monthly ≤ 90) and the deletion ledger is re-applied on restore | GDPR "without undue delay", ≤ 1 month to respond; CCPA confirm receipt ≤ 10 business days, respond ≤ 45 days |
| Rectification | In-app editing | |
| Breach notification | Internal runbook ≤ 24 h to decision | GDPR Art. 33: 72 h to supervisory authority |
| Sale/share of personal info | None. Publish "we do not sell or share" (CCPA) | |
| Lawful basis | Contract for the core service; consent for analytics and marketing | |
| Data residency | EU shard placement option by Y2 (shard-by-user makes this a routing rule) | |
| Subprocessors | Published list (cloud, email, push providers, error reporting with content scrubbing) | |
| Children | Not directed at under-13s (COPPA); age gating where required | |

### 3.7 Accessibility: WCAG 2.2 AA

**Standards context:**
- WCAG 2.2 is now **ISO/IEC 40500:2025** ([W3C, Oct 2025](https://www.w3.org/WAI/news/2025-10-21/wcag22-iso)).
- The **European Accessibility Act** has applied since **28 June 2025**. It uses EN 301 549 (≈ WCAG 2.1 AA plus native-app clauses) ([Level Access](https://www.levelaccess.com/blog/eu-accessibility-requirements-and-eaa-compliance/)).
- Paid subscriptions sold online to EU consumers likely bring us into scope. Target **WCAG 2.2 AA on web and the equivalent on native**.

Concrete requirements:
- **Full keyboard operation of the masonry grid**: roving tabindex, arrow keys, documented shortcuts compatible with Keep's (e.g., `c` new note, `/` search).
- **2.5.7 Dragging Movements.** Every drag reorder has a non-drag alternative: "Move up/down" actions, plus RN accessibility actions.
- **2.5.8 Target Size (Minimum)**: ≥ 24×24 CSS px. Mobile uses 44/48 pt platform guidance.
- **2.4.11 Focus Not Obscured.** The sticky "Take a note…" bar and toasts must not cover focus.
- **3.3.8 Accessible Authentication.** Passkeys, password-manager paste and magic links; no cognitive puzzles.
- **Colour.**
  - All 12-ish note colours meet **4.5:1 text contrast** in light and dark themes.
  - Colour is never the only signal; colour names are exposed to assistive tech (AT).
  - Labels are shown as text.
- **Screen readers.**
  - Checklist items are real checkboxes.
  - Collaborator edits are announced politely (`aria-live="polite"`, throttled).
  - Sync state ("Saved", "Offline – changes will sync") is exposed as status.
- **Native.** VoiceOver/TalkBack labels and actions; Dynamic Type / font scale to 200%; respect Reduce Motion.
- **Process.** Axe/Lighthouse in CI, manual AT test passes per release, and an a11y statement plus feedback channel.

### 3.8 Internationalization and RTL
- **Text model.** UTF-8 storage. CRDT offsets in Yjs are JS string (UTF-16) indices, so cursor and selection logic must move by **grapheme cluster** (`Intl.Segmenter`) to avoid splitting emoji and combining marks.
- **Bidi.**
  - `dir="auto"` per paragraph and per list item (mixed Arabic/Hebrew + Latin notes are common).
  - Mirrored layout via CSS logical properties; RN `I18nManager` for RTL.
  - Mirror directional icons only.
- **Messages.** ICU MessageFormat (plurals, gender); no string concatenation; pseudo-localization in CI.
- **Dates/times.** Locale formatting via `Intl`. Reminders stored as **wall-clock time + IANA zone**, so recurring reminders survive DST and travel.
- **Search.**
  - Postgres default FTS parsers are poor for CJK/Thai, so use `pg_trgm` / n-grams on the server and an FTS5 trigram or ICU tokenizer on the client.
  - Normalize NFC and case-fold.
- **Collation.** ICU collations for label and title sorting.
- **Launch locales (suggested).** en, es, pt-BR, fr, de, hi, ja, ar (RTL validation from day one).

### 3.9 Abuse: spam and phishing via sharing

**Threat.** "Share note with any email" is a delivery channel for spam and phishing (as with document and calendar share spam on other platforms).

**Controls:**
- **Pending inbox.** Shares from non-contacts land in a "Shared with you – pending" area, not the main grid, until accepted. Recipients can **Block sender** and **Report spam** in one tap; this removes the note and feeds reputation.
- **Notification emails carry minimal content.** No note body, title truncated and sanitized, no clickable links from user content. Sent from a dedicated subdomain with SPF/DKIM/DMARC.
- **Quotas.**
  - ≤ 50 collaborators per note (CloudKit's cap is 100 ([forum](https://developer.apple.com/forums/thread/783081))).
  - New accounts (< 7 days) can share to ≤ 20 new recipients/day; established accounts ≤ 200/day.
  - Global per-IP/ASN limits.
- **Signup hardening.** Disposable-domain blocklist, email verification before sharing, device attestation (Play Integrity / App Attest), risk-based challenges.
- **Content scanning.**
  - URLs in shared notes are checked against Safe Browsing / Web Risk.
  - Attachments in shared notes are malware-scanned.
  - Images are **re-encoded server-side**, which strips EXIF/GPS and neutralizes polyglot files.
- **Protocol abuse.**
  - Max CRDT update 256 KB.
  - Max note CRDT state 2 MB (Keep caps text at 20k chars and 1,000 list items).
  - Per-connection rate limit (e.g., 50 msgs/s burst, 10/s sustained).
  - Read-only roles enforced at message-type level (as in y-protocols).
  - Schema validation in the compaction worker.
- **Operations.** Abuse queue with **≤ 24 h triage SLA** and transparency for appeals.

### 3.10 Privacy expectations
- **Encryption.** TLS 1.3 in transit. Encryption at rest (KMS) for DB, backups and objects; per-environment keys.
- **No employee access to note content** without a user-consented support ticket. Break-glass access is audited and alerted.
- **No ads and no model training on note content.** Any AI features (OCR, summaries) are opt-in, processed under a DPA, and not retained by vendors.
- **Content-free telemetry.** Crash reports and logs scrub note text, titles and attachment names.
- **Clear visibility.** Every shared note shows exactly who has access. Labels, colours, archive, pins and reminders are **private per user** on shared notes (Keep semantics).
- **Deletion is real.** Purge reaches replicas, search, objects, caches and backups (lesson 12).
- **EXIF/GPS stripped by default** on uploaded images.
- **E2EE roadmap.**
  - Not in v1, because server-side search, OCR and sharing get much harder.
  - The design reserves "locked notes": client-side AES-GCM-256 per-note keys, as Bear does ([Bear](https://community.bear.app/t/bear-2-4-better-encryption-auto-todo-sorting-and-pin-within-tags/16571)). Users comparing us with Apple Notes ADP, Bear or Obsidian will ask for it.
- **Location data** is not collected in v1 (Keep itself dropped location reminders in the Tasks migration).

---

### Appendix: items flagged UNVERIFIED
- Google Keep's internal text-merge algorithm and real-time collaboration transport (not public). Internal protocol details come from the unofficial `gkeepapi`.
- Apple Notes CRDT specifics (reverse-engineered only). Whether Notes uses CKSyncEngine. Notes-specific collaborator cap.
- Craft's conflict-resolution algorithm. Excalidraw's URL-fragment key (not restated in the cited post).
- Evernote RENT details (primary help page returned 403; secondary summary used).
- RDS Multi-AZ instance price assumed 2× single-AZ. Third-party price aggregators were used for EC2, RDS and ElastiCache list prices.
- Hetzner AX102 current price (sources show €257.30 vs €452.30/mo) and Cloudflare plan needed for WebSocket scale.
- All ASSUMPTION-tagged inputs in §2.1. These should be replaced with telemetry from a beta cohort.

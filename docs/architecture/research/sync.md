# Local-first / CRDT sync evaluation for a TypeScript Google Keep clone

*Research date: 2026-10-04. Versions, release dates and release counts come from the npm registry (`registry.npmjs.org/<pkg>`) and the GitHub REST API, both queried on 2026-10-04. "Release cadence" counts **stable** npm versions published between 2025-10-04 and 2026-10-04. Claims I could not confirm from a primary source are marked **UNVERIFIED**.*

---

## 0. TL;DR

1. **Do not pick one tool for all state.** A Keep clone has two different kinds of state:
   - **Collaborative text:** note body, title and checklist items. This needs a text CRDT.
   - **Relational, permissioned, mostly per-user state:** pin, archive, color, labels, reminders, grid order, settings. This needs a partial-replication sync engine over Postgres with server-side authorization.
2. **Recommended stack (rank 1):** **PowerSync** for all relational state, plus **Yjs 13.6.x, one `Y.Doc` per note**, synced by self-hosted **Hocuspocus 4** and persisted into Postgres. TipTap 3 is the web editor.
   - The grid, search and reminders read **server-derived projections** (title, preview, plain text) that PowerSync syncs. They never load the CRDT.
3. **Disqualified for the core:**
   - **Zero.** No offline writes. Its own docs say "Zero doesn't support offline writes."
   - **Automerge.** No supported React Native/Hermes path.
   - **InstantDB.** The cloud service is being shut down (team joined OpenAI; shutdown 2027-08-31).
   - **Triplit.** Dormant since mid-2025 (team joined Supabase).
   - **Replicache.** In maintenance mode.
   - **cr-sqlite.** No release since 2024-01.
   - **Jazz 2.** Still alpha.
   - **LiveStore.** Its auth docs still say "TODO", and it maps one eventlog to one store.
4. **Runner-up (rank 2):** **Electric + TanStack DB** in place of PowerSync, with the same Yjs content layer. **Watch:** **Loro**, because MovableList and MovableTree suit checklists, but its RN binding lags the core library.
5. **If end-to-end encryption becomes a hard requirement,** the server can no longer read note content. That rules out server-side search and reminders over content, server-built previews and server-side merging. You would then move to an **Evolu**-style design, which is a different architecture.

---

## 1. Hard requirements used as filters

| # | Requirement | Why it filters |
|---|---|---|
| R1 | **Full offline editing**, including creates, edits and deletes. Offline periods can last hours or days. | Removes engines that reject writes while disconnected. |
| R2 | **One TypeScript sync/domain core** shared by React web (PWA) and React Native/Expo. | Removes WASM-only CRDTs unless they have a maintained native binding. Hermes WASM support is **UNVERIFIED**, see §3.2. |
| R3 | **Per-user partial replication:** each user syncs only the notes they own or that are shared with them. **Revocation** removes the note from the revoked user's devices. | Needs server-side authorization of what is synced, not only of what is written. |
| R4 | **Postgres is the source of truth** (alongside Redis/Valkey and S3). | Engines with their own proprietary backends become lock-in risks. |
| R5 | **Concurrent edits to shared notes merge without conflicts.** | Requires a text/sequence CRDT for note content. |
| R6 | **Scales to millions of users.** | Rules out designs where every keystroke goes through a single replication pipeline, or where one log holds a whole user's data. |

---

## 2. Comparison table (verified 2026-10-04)

Verdict key: ✅ recommended · 🟡 viable with caveats · 👀 watch · ❌ not recommended

| Candidate | Category | Latest stable (date) | Stable releases, 12 mo | License | Maintenance health | R1 offline writes | Rich-text CRDT | RN/Expo | Postgres story | R3 partial sync + revocation | Verdict for Keep clone |
|---|---|---|---|---|---|---|---|---|---|---|---|
| **Yjs** (+y-protocols) | Text/JSON CRDT lib | `yjs` 13.6.33 (2026-09-23). v14 is RC as `@y/y` 14.0.0-rc.28 (2026-09-29) | 6 (+ v14 RCs) | MIT | Active, 22.9k★. One core maintainer (bus factor) | ✅ | ✅ Best ecosystem | ✅ Pure JS. Persistence is DIY | Via Hocuspocus, y-electric or rows in PowerSync | Per-document (room) auth | ✅ **Content layer** |
| **Hocuspocus** | Yjs WebSocket server | 4.7.0 (2026-09-09). 4.0.0 shipped 2026-04-23 | 19 | MIT | Active (Tiptap) | n/a | n/a | Provider works in RN | `Database` extension (fetch/store hooks) | `onAuthenticate` per document | ✅ **Content transport** |
| y-websocket / `@y/websocket-server` | Reference provider/server | 3.1.0 (2026-08-06) / 0.1.5 | 1 / 2 | MIT | Low churn | n/a | n/a | ✅ | DIY | DIY | 🟡 |
| y-sweet | Yjs server on S3 | 0.9.1 (2025-09-16) | 0 | MIT | Stale. Jamsocket joined Modal (2025-07-10). Last commit 2025-12-04 | n/a | n/a | ✅ | S3, not Postgres | Token-based | ❌ |
| y-indexeddb | Yjs web persistence | 9.0.12 (2023-11-02) | 0 | MIT | Stable but untouched | ✅ | n/a | ❌ (web only) | n/a | n/a | 🟡 Use on web |
| **Automerge** + automerge-repo | JSON/text CRDT + repo | `@automerge/automerge` 3.5.0 (2026-09-16). Repo 2.5.6 stable (2026-05-18), 2.6 in alpha | 13 / 8 | MIT | Active (Ink & Switch) | ✅ | ✅ (`@automerge/prosemirror` 0.2.0) | ❌ No supported RN path. WASM is 1.14 MB gz | No first-party Postgres adapter | Keyhive is alpha | ❌ (RN) / 👀 |
| **Loro** | Text/list/tree CRDT lib | `loro-crdt` 1.16.4 (2026-09-30) | 45 | MIT | Very active, 6.2k★ | ✅ | ✅ (`loro-prosemirror` 0.4.4) | 🟡 `loro-react-native` 1.10.3 (2025-12-09) lags core | DIY. `loro-protocol` ships minimal servers | DIY | 👀 |
| **Zero** (Rocicorp) | Query-driven sync engine | 1.9.0 (2026-08-14). GA March 2026 | 46 | Apache-2.0 | Very active | ❌ **No offline writes** | ❌ | ✅ (expo-sqlite / op-sqlite) | ✅ Native (logical replication) | ✅✅ Best-in-class | ❌ (fails R1) |
| Replicache | KV sync engine | 15.3.0 (2025-07-02) | 0 | Apache-2.0 (open-sourced) | **Maintenance mode** | ✅ | ❌ | 🟡 | DIY push/pull | DIY | ❌ |
| **PowerSync** | Postgres→SQLite sync engine | web 2.4.2 / RN 2.3.1 (2026-10-01). Service 1.27.0 (2026-09-30) | ~35 (web SDK) | SDKs Apache-2.0. **Service FSL-1.1-ALv2** | Very active, commercial vendor | ✅ Upload queue | Via Yjs-as-rows demo | ✅ (op-sqlite) | ✅ Logical replication. Writes go through your API | ✅ Sync Streams. Bucket limits apply | ✅ **Relational layer** |
| **Electric** + TanStack DB | Read-path Postgres sync + client store | `@electric-sql/client` 1.5.28 (2026-09-09). `@tanstack/db` 0.11.3 (2026-10-02) | 45 / 84 | Apache-2.0 / MIT | Active. **Focus moving to an "agent platform"** | 🟡 TanStack DB persistence + `offline-transactions` | Via `y-electric` | ✅ (op-sqlite persistence) | ✅ Read path only | ✅ Shapes with subqueries, behind your auth proxy | 🟡 **Runner-up** |
| Triplit | Full-stack syncing DB | 1.0.50 (2025-07-31) | 0 | AGPL-3.0 (client) | **Dormant.** Co-founder joined Supabase (Oct 2025) | ✅ | ❌ | ✅ | Own server | Own rules | ❌ |
| InstantDB | BaaS + sync | 1.0.67 (2026-08-31) | 247 | Apache-2.0 | **Cloud sunsetting.** Team joined OpenAI (2026-08-22). Shutdown 2027-08-31 | ✅ | ❌ | ✅ | Self-host possible | CEL-style rules | ❌ |
| Jazz | Local-first DB | Classic 0.20.19 (2026-07-03). **2.0.0-alpha.58** (2026-09-30) | 59 | MIT | Mid-rewrite with breaking alphas | ✅ | Classic only. 2.0 is per-field LWW | ✅ | Own server/cloud | Row-level policies (2.0) | ❌ (alpha) |
| LiveStore | Event-sourced SQLite | 0.4.0 (2026-06-02) | 1 (plus many snapshots) | Apache-2.0 | Active, pre-1.0 | ✅ | ❌ ("best handled via CRDTs") | ✅ Expo adapter | Via Electric/S2/Cloudflare backends | Auth docs say "TODO" | ❌ |
| RxDB | NoSQL local DB + replication | 17.5.0 (2026-08-20) | 9 | Apache-2.0 core. **Paid premium storages** | Active | ✅ | ❌ | 🟡 Good storages are premium | DIY replication endpoints | DIY | 🟡 (DIY-heavy) |
| TinyBase | Reactive store + MergeableStore CRDT | 10.0.1 (2026-09-24) | 40 | MIT | Active, single maintainer | ✅ | ❌ | ✅ (expo-sqlite / op-sqlite persisters) | Postgres persister (not multi-tenant) | ❌ No permission model | ❌ (multi-user) |
| Evolu | SQLite + CRDT, E2EE | `@evolu/common` 8.17.0 (2026-10-02) | 33 | MIT | Active | ✅ | ❌ | ✅ | ❌ Blind relay, no plaintext server | Cryptographic "owners" | 🟡 Only if E2EE is mandatory |
| DXOS | P2P framework (ECHO) | `@dxos/client` 0.11.1 (2026-08-05) | 4 | FSL-1.1-Apache-2.0 | Active, app-centric (Composer) | ✅ | 🟡 | ❌ UNVERIFIED | ❌ | HALO/spaces | ❌ |
| cr-sqlite | CRDT SQLite extension | 0.16.3 (2024-01-17) | 0 | MIT (GitHub) | **Unmaintained.** Community build fixes only (2026-08) | ✅ | ❌ | 🟡 | ❌ | ❌ | ❌ |
| *Also noted* | Liveblocks: hosted Yjs; sync engine + dev server open-sourced Feb 2026. Turso sync: `@tursodatabase/sync` 0.8.1, pre-1.0. Ditto: commercial CRDT DB. Convex: server-first, not offline-write. | | | | | | | | | | |

Primary sources for the table:
- npm package pages, e.g. [yjs](https://www.npmjs.com/package/yjs), [@y/y](https://www.npmjs.com/package/@y/y), [@hocuspocus/server](https://www.npmjs.com/package/@hocuspocus/server), [@automerge/automerge](https://www.npmjs.com/package/@automerge/automerge), [loro-crdt](https://www.npmjs.com/package/loro-crdt), [@rocicorp/zero](https://www.npmjs.com/package/@rocicorp/zero), [@powersync/web](https://www.npmjs.com/package/@powersync/web), [@electric-sql/client](https://www.npmjs.com/package/@electric-sql/client), [jazz-tools](https://www.npmjs.com/package/jazz-tools)
- GitHub repos: [yjs/yjs](https://github.com/yjs/yjs), [ueberdosis/hocuspocus](https://github.com/ueberdosis/hocuspocus/releases), [loro-dev/loro](https://github.com/loro-dev/loro), [powersync-ja/powersync-service LICENSE](https://github.com/powersync-ja/powersync-service/blob/main/LICENSE), [vlcn-io/cr-sqlite releases](https://github.com/vlcn-io/cr-sqlite/releases)

---

## 3. Per-candidate notes

### 3.1 Yjs ecosystem (recommended content CRDT)

**Version and health**
- `yjs` 13.6.33 was released 2026-09-23 and is MIT-licensed.
- **v14 is in release candidate under a new package name,** [`@y/y`](https://www.npmjs.com/package/@y/y), at 14.0.0-rc.28 (2026-09-29). It brings attribution/"blame", delta-based events and a reworked API. The y-prosemirror v2 equivalent is [`@y/prosemirror`](https://www.npmjs.com/package/@y/prosemirror) 2.0.0-14 (beta).
- The [GitHub releases page](https://github.com/yjs/yjs/releases) gives no date for v14 stable.
- The ecosystem has not moved yet. `@tiptap/extension-collaboration` 3.31.4 peer-depends on `yjs ^13` and `@tiptap/y-tiptap` (Tiptap's fork of y-prosemirror). `@hocuspocus/server` 4.7.0 peer-depends on `yjs ^13.6.8`. **Use 13.6.x today.**

**Production users** (from the [README](https://github.com/yjs/yjs)): Evernote, Proton Docs, AFFiNE, JupyterLab, GitBook, Linear, AWS SageMaker, NextCloud, Typst and others.

**Data model and conflict semantics**
- Shared types: `Y.Text` (rich-text attributes), `Y.XmlFragment` (ProseMirror tree), `Y.Map` (LWW per key) and `Y.Array`.
- **There is no native move operation in v13 or in v14-rc.28.** I checked the v14 source. Reordering must be modelled with a fractional-index field inside a `Y.Map` item. Delete-and-reinsert would duplicate or lose concurrent edits to the moved item.

**Editor bindings**
- ProseMirror, Tiptap, Lexical (`@lexical/yjs` 0.52.0), BlockNote, CodeMirror, Monaco, Quill and Slate.
- This is the widest ecosystem of any CRDT ([README list](https://github.com/yjs/yjs)).

**Server and Postgres**
- [Hocuspocus 4](https://github.com/ueberdosis/hocuspocus/releases): released 2026-04-23. Cross-runtime via `crossws`, ordered message processing, typed context, Node ≥ 22.
- The [`Database` extension](https://tiptap.dev/docs/hocuspocus/server/extensions/database) gives you `fetch()`/`store()` hooks over the binary Yjs state. Postgres `bytea` works.
- The [`Redis` extension](https://tiptap.dev/docs/hocuspocus/server/extensions/redis) fans out updates across instances. Its own docs say "all messages will be handled on all instances", so it gives high availability, **not** CPU scale-out. Scale out by routing each document to one instance (shard by note ID).
- Alternatives:
  - [`y-electric`](https://github.com/electric-sql/electric/tree/main/packages/y-electric) 0.1.54: updates are POSTed to your API, stored as `bytea` rows and streamed back via Electric shapes.
  - PowerSync's [Yjs demo](https://docs.powersync.com/client-sdks/advanced/crdts): updates stored as rows.
  - [Liveblocks](https://liveblocks.io/blog/open-sourcing-the-liveblocks-sync-engine-and-dev-server): hosted. Its server was open-sourced Feb 2026; the exact license is **UNVERIFIED**.

**Persistence on each platform**
- Web: [y-indexeddb](https://www.npmjs.com/package/y-indexeddb) 9.0.12. Last release 2023-11, but stable.
- RN: the community adapters are stale (`y-op-sqlite` 1.0.7 from 2024-06, `y-expo-sqlite` 0.1.0 from 2024-06). **Write your own.** It is about 150 LOC: append updates as blobs to an op-sqlite table, then periodically `Y.mergeUpdates` into a snapshot row.
- Yjs is pure JS, so it runs on Hermes. lib0 maps `webcrypto` to `isomorphic-webcrypto` under the `react-native` export condition, so you need a crypto shim (verified in the lib0 0.2.119 `package.json`).

**Performance and size**
- I measured `export * from 'yjs'` with esbuild: 93 KB minified, **28.8 KB gzip**, including lib0.
- Notes are small, so per-document cost is low. **The real cost is loading thousands of `Y.Doc`s at once. Don't.** See §6.

**History and GC**
- With `gc: true`, deleted content is reduced to compact GC structs, but item IDs (tombstone metadata) remain.
- Keep has no version history, so keep GC on. Take occasional snapshots to S3 if you want "undo after sync".

**Schema evolution**
- Documents are schemaless. Store `meta.schemaVersion` in the doc and write deterministic, idempotent migrations that run on open.

**E2EE**
- Feasible at the transport level (encrypted update relay, e.g. the secsync pattern listed in the README).
- It is **incompatible with a server that merges and reads documents** (Hocuspocus, server-side previews and search).

### 3.2 Automerge (3.x) + automerge-repo

**Status**
- [Automerge 3.0](https://automerge.org/blog/automerge-3/) (July 2025) cut memory use by more than 10× and kept the same file format.
- Hexane v1 storage (July 2026) gave 2–9× faster save/load ([July 2026 update](https://automerge.org/blog/2026-july/)). The latest is 3.5.0 (2026-09-16).
- automerge-repo: the latest *stable* release is 2.5.6 (2026-05-18). The npm `latest` dist-tag currently points at `2.6.0-alpha.3`, which is a packaging oddity to watch.

**Access control and sync**
- Keyhive, the access-control and E2EE layer, is alpha (`@keyhive/keyhive` 0.3.0-alpha.1, 2026-09-25). An "ARK" guide covering grant and revoke was published Aug 2026 ([Aug 2026 update](https://automerge.org/blog/2026-august/)).
- Subduction (`@automerge/subduction` 0.23.0) is a new sync server aimed at loading thousands of documents.

**Why it fails here: React Native**
- Core is Rust compiled to WASM. I measured `automerge.wasm` at 3.6 MB raw and **1.14 MB gzipped**.
- There is no official RN binding. The long-standing ["React Native?" issue](https://github.com/automerge/automerge/issues/574) was converted to a discussion on 2026-08-27 without shipping support.
- Third-party claims say WASM is landing in Hermes ([Callstack](https://www.callstack.com/events/react-native-0-84-and-other-news)). The official [RN 0.84 post](https://reactnative.dev/blog/2026/02/11/react-native-0.84) does not mention WASM. **Treat Hermes WASM as UNVERIFIED and not production-ready.**

**Strengths:** full history, branches and merges, a clean JSON model, and `@automerge/prosemirror` 0.2.0.

**Verdict:** ❌ for this stack today.

### 3.3 Loro

**Strengths**
- Most capable data types for Keep's checklist: **MovableList** and **MovableTree** (native move ops with concurrent-safe reorder and nesting), Fugue-based rich text, LWW map and counter.
- Shallow snapshots for history GC ([README](https://github.com/loro-dev/loro)).
- The 1.0 encoding format is declared stable. Releases are very frequent (45 stable in 12 months). MIT.

**Gaps**
- **React Native:** [`loro-react-native`](https://github.com/loro-dev/loro-react-native) (uniffi-bindgen-react-native) is at 1.10.3 (2025-12-09). It lags `loro-crdt` 1.16.x by six minor versions and has about 23★.
- **Server:** [`loro-protocol`](https://github.com/loro-dev/protocol) ships only minimal WS servers (Node, plus Rust with SQLite snapshotting). There is no Hocuspocus-grade server with auth hooks, Postgres persistence or scale-out.
- **Editor bindings:** `loro-prosemirror` 0.4.4 and `loro-codemirror` 0.3.3. Tiptap and Lexical bindings are less mature than Yjs's (**UNVERIFIED** depth).
- Same WASM size class as Automerge: about 1.12 MB gzip, measured.
- Production users: **UNVERIFIED**. I found no primary-source list, and loro.dev blocked automated fetches.

**Verdict:** 👀 Re-evaluate when the RN binding tracks core releases.

### 3.4 Zero (Rocicorp)

**Status:** GA in March 2026 ([status page](https://zero.rocicorp.dev/docs/status)). 1.9.0 (2026-08-14). Apache-2.0.

**Strengths**
- Query-driven partial sync with **server-side query and mutate endpoints for authorization**. This is the most elegant answer to R3.
- RN via an `expo-sqlite` or `op-sqlite` kvStore ([docs](https://zero.rocicorp.dev/docs/react-native)).

**Why it is disqualified: offline writes**
- [When to use](https://zero.rocicorp.dev/docs/when-to-use): "Zero doesn't support offline writes".
- [Connection docs](https://zero.rocicorp.dev/docs/connection): writes are rejected in the `disconnected` state, which is entered after 1 minute by default. "Zero is not designed for long periods offline."
- It also recommends datasets under about 100 GB in its `zero-cache` SQLite replica.

**Verdict:** ❌ for R1. The best choice if offline writes were not required.

### 3.5 Replicache

[replicache.dev](https://replicache.dev/): "now in maintenance mode … open-sourced the code and no longer charge for its use … migrate to Zero". The source is in `rocicorp/mono` under Apache-2.0. **Verdict:** ❌

### 3.6 PowerSync (recommended relational layer)

**Architecture**
- Postgres logical replication feeds the PowerSync Service, which stores bucket data in **MongoDB or Postgres** ([self-hosted config](https://docs.powersync.com/configuration/powersync-service/self-hosted-instances)). Clients hold a local SQLite database.
- **Writes go to your own API** via the client upload queue. Server-authoritative reconciliation is per-field LWW "as received by the server" by default, and you can customise it ([conflicts](https://docs.powersync.com/handling-writes/handling-update-conflicts)).

**Partial replication and permissions:** [Sync Streams](https://docs.powersync.com/sync/streams/bucket-count) replace the legacy Sync Rules.
- **Limit:** 1,000 buckets per user by default, configurable up to 10,000 on request ([limits](https://docs.powersync.com/resources/performance-and-limits)).
- **Gotcha for Keep:** a subquery over a membership table makes **one bucket per note**, which breaks at more than 1,000 notes.
- **Fix:** the documented ["many-to-many via JSON array column"](https://docs.powersync.com/sync/advanced/reducing-bucket-count) pattern. Keep a trigger-maintained `notes.member_ids` JSON array and `json_each()` over it, so buckets are keyed per user.
- **Revocation:** rows that leave a bucket are delivered as `REMOVE` operations, and the client deletes them ([service architecture](https://docs.powersync.com/architecture/powersync-service)).

**Platforms**
- RN uses `@op-engineering/op-sqlite`. Expo Go works with a JS adapter ([RN docs](https://docs.powersync.com/client-sdks/reference/react-native-and-expo)).
- Web uses wa-sqlite: `IDBBatchAtomicVFS` by default, or `OPFSCoopSyncVFS` (recommended for multi-tab and Safari) ([web docs](https://docs.powersync.com/client-sdks/reference/javascript-web)).
- Client FTS5 search, an attachments queue and TanStack DB/Drizzle/Kysely integrations are available.

**Limits** ([source](https://docs.powersync.com/resources/performance-and-limits))
- 15 MB per row.
- About 1M rows per client is "good".
- Replication of 2,000–4,000 operations per second for small rows. **This is why high-frequency CRDT updates should not go through it at scale.**
- More than 50k concurrent clients per instance.

**Storage v4 (beta)**
- Gives faster sync, incremental reprocessing and S3 offload.
- **Self-hosted, it requires MongoDB bucket storage** ([doc](https://docs.powersync.com/sync/advanced/storage-version-4)).

**License and lock-in**
- Client SDKs are Apache-2.0. The service is **FSL-1.1-ALv2**: you may use it for anything except offering a competing product ([LICENSE](https://github.com/powersync-ja/powersync-service/blob/main/LICENSE)).
- There is a self-hostable "Open Edition" (Docker), a paid Enterprise self-hosted edition, and PowerSync Cloud ([self-hosting](https://docs.powersync.com/intro/self-hosting)).
- Lock-in is moderate. Your data stays in Postgres and your write path is your own API, so you could swap the read path for Electric.

**Production:** customer logos on [powersync.com](https://www.powersync.com/) include Snackpass, Stanley Black & Decker, Christian Louboutin and Allianz PNB Life.

### 3.7 Electric (Postgres Sync) + TanStack DB (runner-up relational layer)

**Sync model**
- Electric is a **read-path-only** sync engine. Writes go through your API ([writes guide](https://electric.ax/docs/sync/guides/writes)). 1.0 shipped 2025-03-17. Apache-2.0.
- Shapes are single-table **with subqueries** for memberships and sharing. They should be requested through your backend for authorization ([shapes](https://electric.ax/docs/sync/guides/shapes)).
- There is CDN fan-out. Electric is used in production by Trigger.dev Realtime ([Trigger.dev docs](https://trigger.dev/docs/realtime/how-it-works)).

**Offline story**
- [TanStack DB 0.6](https://electric.ax/blog/2026/03/25/tanstack-db-0.6-app-ready-with-persistence-and-includes) (2026-03-25) added SQLite-backed persistence, including op-sqlite on RN, plus `@tanstack/offline-transactions` (1.0.61).
- TanStack DB itself is still 0.x (0.11.3), and you assemble more of the write path than with PowerSync.

**Strategic risk:** the site now leads with "the first agent platform built on sync" ([electric.ax](https://electric.ax/)).

**Other notes:** PGlite does not run on RN ([Expo page](https://electric.ax/docs/sync/integrations/expo)). Electric has [`y-electric`](https://github.com/electric-sql/electric/tree/main/packages/y-electric) for Yjs.

### 3.8 Triplit

The co-founder joined Supabase in October 2025 ([Supabase blog](https://supabase.com/blog/triplit-joins-supabase)). There has been no npm release since 2025-07-31 and the last commit was 2026-01-19. The client is AGPL-3.0. **Verdict:** ❌

### 3.9 InstantDB

1.0 shipped 2026-04-09 and the code is Apache-2.0. However:
- The **team joined OpenAI** ([essay](https://www.instantdb.com/essays/instant_team_joins_openai), announced 2026-08-22).
- New signups are closed and cloud apps shut down **2027-08-31**. The [docs banner](https://www.instantdb.com/docs) reads "Instant is sunsetting".
- Self-hosting is documented, but you would inherit the maintenance.

**Verdict:** ❌

### 3.10 Jazz

- [Jazz 2.0](https://github.com/garden-co/jazz) is an **alpha with an entirely new API**: a relational model with row-level policies.
- Conflict resolution is **per-field LWW with hybrid logical clocks** ([data model](https://jazz.tools/docs/concepts/local-first-data-model)). There is no text CRDT in 2.0's described model.
- Alphas break between versions (the docs include an "Upgrading from alpha.55" page).
- Classic Jazz (0.20.19) is still on npm.

**Verdict:** ❌ for now. 👀 after 2.0 GA.

### 3.11 LiveStore

- 0.4.0 (2026-06-02). Event-sourced SQLite.
- [Designed for](https://docs.livestore.dev/overview/when-livestore/) "10s / low 100s of users collaborating on the same thing for a given eventlog". All client data must fit in an in-memory SQLite database. It adds "a few hundred kB".
- [Syncing docs](https://docs.livestore.dev/building-with-livestore/syncing/): Auth is "TODO", compaction is upcoming, there is a 1:1 eventlog↔DB mapping, and "Rich text data is best handled via CRDTs".

Per-note sharing would need one eventlog per note. **Verdict:** ❌

### 3.12 RxDB

- 17.5.0. The core is Apache-2.0, but **the fast storages are paid** ([premium](https://rxdb.info/premium/)): OPFS, IndexedDB, SQLite and Expo filesystem from $99/month on annual billing.
- Replication means implementing pull/push/stream endpoints on your server, with a conflict handler and a JSON CRDT plugin.
- It has no Postgres-aware permissioned partial sync.

**Verdict:** 🟡. It works, but you build what PowerSync gives you.

### 3.13 TinyBase

- 10.0.1. MergeableStore CRDT. Synchronizers for WebSocket, Durable Objects and PartyKit. Persisters for expo-sqlite, op-sqlite, PGlite and Postgres ([site](https://tinybase.org/)). About 7.3 kB gzip core.
- **No multi-tenant permission model.** You would build per-user partitioning in a custom synchronizer server.

**Verdict:** ❌ for multi-user sharing.

### 3.14 Evolu

- 8.x. SQLite + CRDT, **E2EE by default**, with a **blind relay** that sees only owner IDs, timestamps and padded ciphertext ([privacy](https://www.evolu.dev/docs/privacy)).
- Sharing uses `SharedOwner` and `SharedReadonlyOwner` keys ([Owner API](https://www.evolu.dev/docs/api-reference/common/local-first/Owner)). A per-owner storage quota is configurable ([relay](https://www.evolu.dev/docs/relay)).
- **Revocation means key rotation, not a server-side ACL.** The server cannot search, index, send reminders based on content, or build previews.

**Verdict:** 🟡 only if E2EE is a product pillar.

### 3.15 DXOS and cr-sqlite

- **DXOS:** FSL-1.1-Apache-2.0 and Composer-centric ([repo](https://github.com/dxos/dxos)). RN support is **UNVERIFIED**. ❌
- **cr-sqlite:** the last release is v0.16.3 (2024-01-17) ([releases](https://github.com/vlcn-io/cr-sqlite/releases)). 2026 commits are only community build fixes. ❌

---

## 4. Deep-dive dimensions for the finalists

| Dimension | Yjs + Hocuspocus | Loro | Automerge-repo | PowerSync | Electric + TanStack DB | Zero (ref.) |
|---|---|---|---|---|---|---|
| **Data model** | Document CRDT (per note) | Document CRDT | Document CRDT + repo of docs | Relational (Postgres → SQLite) | Relational (shapes → collections) | Relational (ZQL) |
| **Conflict semantics** | YATA sequence CRDT. Map keys are LWW. No move op | Fugue text. MovableList/Tree. LWW map | RGA-style text. LWW map with conflict sets | Server-authoritative. Default per-field LWW by server arrival, customisable | Whatever your API does | Server-authoritative mutators with rebase |
| **Rich text / bindings** | Tiptap, ProseMirror, Lexical, BlockNote, CodeMirror… | loro-prosemirror, loro-codemirror | @automerge/prosemirror | None native (Yjs-as-rows demo) | y-electric | None |
| **Server & Postgres** | Hocuspocus `store()` writes `bytea` to Postgres. Redis gives HA | DIY | DIY (sync server + storage adapter) | Native logical replication. Bucket storage in Mongo/PG | Native read path | Native logical replication |
| **Partial sync & revocation** | Per-document auth on connect. You must kick live sockets on revoke | DIY | Keyhive alpha | Sync Streams. REMOVE ops on revoke | Shapes + auth proxy. Shape changes on revoke | Query/mutate endpoints |
| **RN storage** | DIY on op-sqlite / expo-sqlite | Native binding (lagging) | ❌ | op-sqlite | op-sqlite (TanStack DB persistence) | expo-sqlite / op-sqlite |
| **Web storage** | IndexedDB (y-indexeddb) | DIY (IndexedDB/OPFS) | IndexedDB adapter | wa-sqlite: IndexedDB or OPFS VFS | SQLite persistence (browser adapter, **UNVERIFIED** which VFS) | IndexedDB |
| **Thousands of docs/user** | Fine **if** documents load lazily. Use projections for the grid | Same | Subduction targets this; still heavy | ~1M rows/client "good" | Fine | Fine |
| **Bundle / runtime** | ~29 kB gz | ~1.1 MB gz WASM | ~1.1 MB gz WASM | wa-sqlite WASM on web (size **UNVERIFIED**) | Small JS. Persistence adds SQLite | Small JS |
| **History / GC** | GC structs. Snapshots optional | Shallow snapshots | Full history (compressed) | Bucket compaction | n/a | n/a |
| **Schema evolution** | In-doc version + lazy migrations | Same | Same | Client schema is views (no client migrations). Multiple-client-versions support | Your API/schema | Schema + backwards-compatible queries |
| **E2EE feasibility** | Only with a dumb relay (no server merge) | Same | Keyhive (alpha) | Low (server must read rows) | Low | Low |
| **Ops burden** | Medium: stateful WS tier, sharding by note | High (build server) | High | Medium: service + bucket DB (Mongo for v4) | Medium: Electric + your write API | Medium: zero-cache with NVMe replica |
| **Lock-in** | Low (MIT; the format is an open de facto standard) | Low (MIT) | Low (MIT) | Medium (FSL service, but data stays in PG) | Low (Apache) | Low–medium |

---

## 5. Recommendation matrix by kind of state

### (a) Note body text + checklist items: rich text, concurrent collaborators, reorder, nesting

| Option | Verdict | Notes |
|---|---|---|
| **Yjs `Y.Doc` per note** | ✅ **Recommended** | See the document layout below. |
| Loro doc per note | 👀 | `LoroMovableList`/`LoroTree` model checklists natively. Blocked by the RN binding lag and the lack of a production server. |
| Automerge doc per note | ❌ | No RN path. |
| LWW rows in a sync engine | ❌ | Concurrent edits to the same note would clobber each other (fails R5). |

**Recommended Yjs document layout:**
- `title: Y.Text`
- `body: Y.XmlFragment`, bound to Tiptap with a minimal Keep-like schema: paragraphs, bold/italic/underline, headings, links.
- `items: Y.Map<itemId, Y.Map{ text: Y.Text, checked: boolean, parentId: string|null, order: string /*fractional index*/ }>`
  - **Reorder** sets `order`, which is LWW per item, so there are no duplicates.
  - **Nesting** sets `parentId`. Keep allows one indent level; validate the depth when rendering.
  - "Checked items at bottom" is a view rule, not stored data.
- `meta: Y.Map{ kind: 'text'|'list', schemaVersion }`

### (b) Note metadata

Google Keep's own help says collaborators "can label, color, archive, or add reminders without changing the note for others". It also says that deleting a shared note you own deletes it for everyone ([Keep help](https://support.google.com/keep/answer/6101196?hl=en)). So **most metadata is per-user.**

| Field | Scope | Where | Merge rule |
|---|---|---|---|
| Title | Shared | **CRDT (`Y.Text`)**. Server materializes `notes.title` for grid and search | CRDT |
| Kind (text/list) | Shared | CRDT `meta` + materialized column | CRDT (LWW key) |
| Owner trash/delete | Shared (owner only) | `notes.deleted_at` via PowerSync, enforced by the API | Delete wins |
| Collaborator "remove" | Per-user | Delete from `note_members` (online API) | Server |
| Color | **Per-user** | `note_user_state.color` | Per-field LWW (client HLC) |
| Pinned | Per-user. Keep's docs don't name pin explicitly (**UNVERIFIED**); design choice | `note_user_state.pinned` | Per-field LWW |
| Archived | **Per-user** | `note_user_state.archived` | Per-field LWW |
| Labels on note | **Per-user** | `note_labels(user_id, note_id, label_id)` | Add/remove set; delete wins |
| Reminders | **Per-user** | `reminders(user_id, note_id, fire_at, rrule…)` | LWW. Server scheduler sends push |

**Recommended:** relational rows synced by **PowerSync**. Writes go through your API with a per-field `hlc` comparison where wall-clock ordering matters. PowerSync's default is "last received by server", so an old offline edit could otherwise overwrite a newer one.

**Avoid:** putting per-user fields inside the shared CRDT. Collaborators would see each other's labels and colors, and the server could not query reminders.

### (c) Ordering of notes in the grid (drag reorder)

| Option | Verdict | Notes |
|---|---|---|
| **Fractional index string per (user, note)** in `note_user_state.sort_key`, LWW | ✅ **Recommended** | Use [`fractional-indexing`](https://github.com/rocicorp/fractional-indexing) (v4.0.0, CC0) or the jittered variant to avoid two devices generating identical keys offline (Figma's approach). Keep separate key spaces for pinned and unpinned, or sort by `(pinned desc, sort_key)`. A new note gets `generateKeyBetween(null, firstKey)`. Run a server job to rebalance keys longer than N characters. |
| CRDT list in a per-user index doc | ❌ | Tombstones grow forever with every move. One document holds the entire account. Server-side queries and permissions become awkward. |
| Server-assigned ordering | ❌ | Needs the network to reorder (fails R1). |

### (d) Non-collaborative account data (settings, label definitions)

**Recommended:** PowerSync rows (`user_settings`, `labels`) with per-field LWW, in a single user-scoped stream (1 bucket).

**Label-name collisions:** two devices may create "Groceries" offline. Derive the label ID deterministically, e.g. UUIDv5(`user_id`, `lower(name)`), so both converge on one row. Renames are LWW. Deletes are soft deletes and the server cascades them to `note_labels`.

---

## 6. Architecture patterns compared

### Pattern 1 — Hybrid: CRDT for note content + relational sync engine (LWW) for metadata ✅

**Pros**
- Permissions, partial sync and revocation come from a mature engine over Postgres.
- Per-user fields stay private and queryable: the reminder scheduler, label filters and search all use SQL.
- The grid renders from SQLite projections without loading thousands of `Y.Doc`s, which keeps cold start and memory low.
- CRDT cost is confined to the text that actually needs it.
- Each layer can be replaced independently (PowerSync ↔ Electric; Yjs ↔ Loro later).

**Cons**
- **Two sync planes** to keep consistent:
  - A note row can exist without its document, or the reverse.
  - Authorization logic is duplicated (Sync Streams and Hocuspocus `onAuthenticate`).
  - Revocation must also close live WebSockets.
- Two offline outboxes: the PowerSync upload queue and the Yjs update store.
- Projections (title/preview) are eventually consistent with the CRDT.

### Pattern 2 — One CRDT document per note for everything (content + metadata) ❌/🟡

**Pros**
- Single sync mechanism and a single offline store.
- Atomic local updates.
- Simplest mental model for a single-user app.

**Cons**
- Per-user fields (color, labels, archive, reminders) must be nested per user inside a shared document. Collaborators can read them, and nothing stops a collaborator from editing another user's fields without server-side validation of CRDT ops.
- **The grid needs all documents loaded,** or a separate projection, which brings Pattern 1's second plane back anyway.
- Server features (reminder push, search, admin, analytics) must decode CRDTs.
- Per-user ordering doesn't fit in a per-note document.

### Pattern 3 — One CRDT doc per user (index: list, order, metadata) + per-note docs ❌

**Pros**
- Fully local-first, P2P-capable and E2EE-friendly.
- This is how automerge-repo-style apps (and Jazz Classic) model accounts.
- The index document gives instant grid load.

**Cons**
- The index document grows without bound (tombstones from every reorder and archive).
- Shared notes need an entry in each collaborator's index, which requires server or peer fan-out into documents the sharer can't write.
- Revocation means editing someone else's index.
- No SQL over metadata (reminders, search).
- Large single-document sync on new devices.
- Hard to scale on the server (one hot document per user).

---

## 7. Concrete design sketch for rank 1

### Postgres schema (source of truth)

```sql
notes(id uuid pk /*client UUIDv7*/, owner_id, kind, title, preview, search_text,
      checklist_summary jsonb, member_ids jsonb /*trigger-maintained*/,
      members_public jsonb /*avatars/names for chips*/, content_seq bigint,
      created_at, updated_at, deleted_at)
note_members(note_id, user_id, role, added_by, created_at)      -- ACL source of truth
note_user_state(user_id, note_id, color, pinned, archived, sort_key, hlc jsonb, updated_at)
labels(id /*uuidv5(user,lower(name))*/, user_id, name, deleted_at)
note_labels(user_id, note_id, label_id, deleted_at)
reminders(id, user_id, note_id, fire_at, rrule, done_at)
user_settings(user_id pk, ...)
note_docs(note_id pk, ydoc bytea, state_vector bytea, byte_size, updated_at)   -- NOT synced by PowerSync
attachments(id, note_id, s3_key, mime, bytes, created_by)        -- blobs in S3 (presigned)
```

### PowerSync Sync Streams (sketch)

The syntax must be checked against the [Sync Streams grammar](https://docs.powersync.com/sync/supported-sql) before use.

```yaml
streams:
  account:            # 1 bucket per user
    queries:
      - SELECT * FROM user_settings   WHERE user_id = auth.user_id()
      - SELECT * FROM labels          WHERE user_id = auth.user_id()
      - SELECT * FROM note_user_state WHERE user_id = auth.user_id()
      - SELECT * FROM note_labels     WHERE user_id = auth.user_id()
      - SELECT * FROM reminders       WHERE user_id = auth.user_id()
  notes:              # keyed by user via json_each, NOT one bucket per note
    query: SELECT notes.* FROM notes INNER JOIN json_each(notes.member_ids) AS m WHERE m.value = auth.user_id()
```

### Content plane

- **Sharding:** Hocuspocus 4 runs behind a router that shards by `hash(note_id)`. Clients multiplex all open notes of a shard over one WebSocket.
- **Auth:** `onAuthenticate` checks `note_members`. Opening read-only for viewers is optional.
- **`onStoreDocument` must merge, never overwrite blindly.** Inside a transaction:
  1. `SELECT ydoc … FOR UPDATE`
  2. `Y.mergeUpdates([db, incoming])`
  3. `UPDATE note_docs`
  4. Derive `title`, `preview`, `search_text` and `checklist_summary`, and bump `notes.content_seq`. PowerSync then replicates the projection.

  Merging makes any writer (HTTP catch-up, admin tools) safe against lost updates. Hocuspocus docs warn that `fetch()` must return the stored bytes and must not create a fresh Y.Doc.
- **Revocation:** a trigger on `note_members` delete emits `NOTIFY`/Redis pub-sub, and the owning shard closes that user's connections for the note.

### Clients (shared TypeScript core)

- **Local stores:** PowerSync SQLite (op-sqlite on RN; OPFSCoopSyncVFS on web), plus a local `ydoc_updates` table. On web this lives in y-indexeddb or the same SQLite database.
- **Grid and search:** rendered and searched from projections, with FTS5 over `search_text` for offline search.
- **Opening a note:** load the `Y.Doc` lazily and connect the provider.
- **Background worker:**
  1. Push dirty documents (local pending updates).
  2. Pull documents where `notes.content_seq` is greater than the locally applied sequence.
  3. When a note disappears from the PowerSync `notes` table (revoked or deleted), delete its local `Y.Doc` store.
- **Create offline:** insert `notes` + `note_user_state` through PowerSync and create the `Y.Doc` locally. The content worker connects only after the upload queue has acked the note row, so the server-side ACL check passes.
- **Rich text on RN:**
  - Body: Tiptap in a WebView ([`@10play/tentap-editor`](https://www.npmjs.com/package/@10play/tentap-editor) 1.0.1, or Expo DOM components), sharing the same ProseMirror schema and the `Y.Doc` through a bridge.
  - Checklist items: native `TextInput` bound to `Y.Text` through a diff → `insert`/`delete` adapter.

---

## 8. Final ranked recommendation

1. **✅ PowerSync (relational, permissions, offline outbox) + Yjs 13 per-note docs via self-hosted Hocuspocus 4 → Postgres (+ S3 for large snapshots and attachments), Tiptap 3 on web.**
   - Best fit for R1–R6 today.
   - Every component is actively released (Sep–Oct 2026).
   - Data stays in your Postgres.
   - Swappable layers.
2. **🟡 Electric Postgres Sync + TanStack DB (SQLite persistence + offline-transactions) + the same Yjs/Hocuspocus content plane.**
   - Fully Apache/MIT, with CDN fan-out.
   - Choose it if the FSL license or MongoDB-for-v4 is unacceptable.
   - Costs: more DIY write-path work, TanStack DB is pre-1.0, and Electric is shifting focus toward agents.
3. **🟡 Single-pipe variant: PowerSync (or Electric) carrying Yjs updates as rows**, with no Hocuspocus ([PowerSync Yjs demo](https://docs.powersync.com/client-sdks/advanced/crdts); [y-electric](https://github.com/electric-sql/electric/tree/main/packages/y-electric)).
   - One permission model, one outbox and background sync for free. Best for an MVP.
   - Higher latency, no live cursors, and write and replication amplification. PowerSync documents 2–4k ops/s replication per instance.
   - Plan to move hot documents to Hocuspocus as you scale. Keeping the content CRDT behind a `NoteContentStore` interface in the domain core makes that a transport swap.
4. **🟡 (Conditional) If E2EE is mandatory:** Evolu for metadata + encrypted Yjs updates over a blind relay, accepting the loss of server-side search, reminders, previews and merging.
5. **👀 Watchlist:**
   - **Loro**, as the content CRDT, once `loro-react-native` tracks core and a production server exists.
   - **Zero**, if offline writes ship; it has the best permissions model.
   - **Automerge**, once RN bindings and Keyhive land.
   - **Jazz 2** after GA.
   - **LiveStore** 1.0.
   - **Not recommended:** InstantDB, Triplit, Replicache, cr-sqlite, DXOS, TinyBase/RxDB (as the primary multi-user engine), y-sweet.

---

## 9. The 5 biggest risks of the recommended approach (and mitigations)

1. **Two sync planes drift apart.**
   - *Failure modes:* a note row without its document, or the reverse; ACL logic duplicated in Sync Streams YAML and Hocuspocus auth; revoked users keeping live sockets; deletes racing open editors.
   - *Mitigations:*
     - Make `note_members` the single ACL table.
     - Generate both the Sync Stream filters and the Hocuspocus check from one shared authorization module, with contract tests.
     - Use client-generated UUIDv7 IDs and an "upload-acked before content-connect" rule.
     - Push revocations via NOTIFY/Redis.
     - Run a client GC loop keyed on PowerSync row removal.
     - Accept that offline devices keep their copy until they reconnect. This is inherent to local-first.
2. **Rich-text editing on React Native.**
   - *The problem:* there is no mature native rich-text editor bound to Yjs. WebView-hosted Tiptap (10tap: last release 2025-11; Expo DOM) costs keyboard, scroll and perf "native feel", plus bridge complexity. `react-native-enriched` (0.8.1) is native but has no Yjs binding (**UNVERIFIED** if one exists).
   - *Mitigations:*
     - Keep a deliberately small Keep-like schema.
     - Use native `TextInput` for titles and checklist items.
     - Isolate the editor behind a platform adapter.
     - Prototype the editor first, in week 1.
3. **Yjs v13 → v14 transition and maintainer concentration.**
   - *The problem:* v14 is an RC under a new package name (`@y/y`) with API changes. Tiptap and Hocuspocus still pin `^13`. Wire compatibility between 13 and 14 is **UNVERIFIED**. Core Yjs has one primary maintainer.
   - *Mitigations:*
     - Pin 13.6.x.
     - Store a `crdtFormat` version beside every persisted document.
     - Wrap Yjs behind a domain interface.
     - Keep a migration test corpus.
     - Track the Tiptap/Hocuspocus v14 adoption before upgrading.
4. **PowerSync scaling, semantics and licensing constraints.**
   - *Constraints:*
     - The 1,000-bucket default forces the JSON-array membership pattern.
     - Replication runs at 2–4k ops/s per instance, so plan sharding by user cohort or tenant, and keep CRDT traffic off it.
     - Default LWW is by *server arrival*. Add HLC per field.
     - Storage v4 is beta and needs MongoDB bucket storage when self-hosted.
     - The service is FSL.
   - *Mitigations:*
     - Load-test early: 10k notes per user, 1M users, and a sharing fan-out of 50.
     - Budget either the MongoDB operations or PowerSync Cloud.
     - Keep the write API engine-agnostic so Electric is a fallback.
5. **Client-side scale and durability with thousands of notes.**
   - *Risks:*
     - Cold-start and memory blowups if `Y.Doc`s load eagerly.
     - Unbounded growth of the local update store and of long-lived documents.
     - Browser storage eviction losing unsynced edits. Safari's 7-day cap on script-writable storage for non-installed sites ([WebKit, 2020](https://webkit.org/blog/10218/full-third-party-cookie-blocking-and-more/), not re-verified today) makes the installed PWA path important.
   - *Mitigations:*
     - Render the grid and search from SQLite projections only.
     - Load documents lazily, with an LRU unload.
     - Compact local updates into snapshots periodically.
     - Call `navigator.storage.persist()`.
     - Prioritise uploads of dirty content in the background worker.
     - Show a visible "unsynced changes" indicator.

---

## 10. Things I could not verify

| Claim | Status |
|---|---|
| Hermes WebAssembly support | Callstack claims it is "landing"; the official RN 0.84 post doesn't mention it |
| Wire/update-format compatibility between Yjs 13 and `@y/y` 14 | Not verified |
| v14 stable date | None published |
| Loro production users | Unknown. loro.dev blocked automated fetch |
| Depth of Loro's Tiptap/Lexical bindings | Not verified |
| Liveblocks open-source server license | Not verified |
| Which browser SQLite VFS TanStack DB persistence uses | Not verified |
| wa-sqlite bundle size in PowerSync web | Not measured |
| Whether pin is per-user in Google Keep (label, color, archive and reminders are documented as per-user) | Not verified |
| DXOS React Native support | Not verified |
| A Yjs binding for `react-native-enriched` | None found |
| Exact Sync Streams syntax for `json_each(...) WHERE m.value = auth.user_id()` | Pattern documented; this exact form not verified |
| The npm `latest` tag for automerge-repo pointing at 2.6.0-alpha.3 | Observed in the registry; intent unknown |

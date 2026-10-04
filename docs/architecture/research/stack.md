# Keep Clone: TypeScript Platform Stack Research (as of 2026-10-04)

Scope: a production-grade Google Keep clone. It ships a responsive web app/PWA plus iOS and Android apps built with Expo, all sharing one TypeScript domain/sync core. It is local-first with CRDT sync, and the backend is Node TS + Postgres + Valkey + S3-compatible storage.

Method: version numbers come from the npm registry (`npm view <pkg> version` / `dist-tags`, queried 2026-10-04) unless another source is cited. Other facts come from official docs, release notes, and GitHub, linked inline. Anything I could not confirm against a primary source is marked **UNVERIFIED**.

---

## 1. Monorepo

**Pick: pnpm 12.9 workspaces + Turborepo 2.11** (`pnpm@12.9.1`, `turbo@2.11.7`).

- **Why:** Expo officially supports pnpm monorepos, and has auto-configured Metro for monorepos since SDK 52 ([Expo monorepo guide](https://docs.expo.dev/guides/monorepos/)). pnpm 12 (Aug 2026) rewrote pnpm in Rust but kept pnpm 11's CLI, lockfile, and `node_modules` layout ([Socket](https://socket.dev/blog/pnpm-12), [InfoQ](https://infoq.com/news/2026/09/pnpm-12-rust)). Turborepo adds task-graph caching and remote cache, and its configuration is small. 2.9 deprecated APIs ahead of 3.0; 2.10 added graceful task shutdown, composable `--affected`/`--filter`, and local cache eviction; 2.11 shipped 2026-09-18 ([Turborepo blog](https://turborepo.dev/blog)).
- **Main alternative:** Nx 23.2 (`nx@23.2.1`). Pick it if you want code generators, enforced module-boundary lint rules (`@nx/enforce-module-boundaries`), and affected-graph CI out of the box. It costs more configuration and plugin coupling.
- **Package boundaries (recommended):**
  ```
  apps/web          Vite SPA (React DOM)               -> domain, sync, storage-web, editor, ui-tokens, api-contract
  apps/mobile       Expo app (Expo Router)             -> domain, sync, storage-native, editor (via DOM component), ui-tokens, api-contract
  apps/server       Fastify HTTP API                   -> domain, api-contract, db
  apps/sync         WebSocket/CRDT sync service        -> domain, sync-protocol, db
  apps/worker       jobs (pg-boss/BullMQ)              -> domain, db
  packages/domain        pure TS: note/label/reminder types, zod schemas, fractional ordering, invariants (no platform deps)
  packages/sync          Yjs doc model, local outbox, sync client state machine, CRDT -> SQL projection (depends only on interfaces)
  packages/sync-protocol wire messages (binary framing, versions) shared by client and server
  packages/storage       interface (exec/query/tx) + migrations as plain SQL
  packages/storage-web   @sqlite.org/sqlite-wasm in a Worker (OPFS)
  packages/storage-native expo-sqlite (or op-sqlite) adapter
  packages/editor        TipTap schema/extensions + React editor component (used by web and by the RN DOM component)
  packages/ui-tokens     design tokens (TS) -> Tailwind theme (web) + Unistyles theme (native)
  packages/api-contract  oRPC contracts (shared by server, web, mobile)
  packages/db            Drizzle schema + migrations (server only)
  ```
- **Gotchas:**
  - Duplicate `react`/`react-native` copies break Metro builds. Expo's guide says to audit with `why` and pin via root overrides. Since SDK 54, pnpm uses isolated installs by default; if Metro resolution fails, fall back to `nodeLinker: hoisted` ([Expo](https://docs.expo.dev/guides/monorepos/)).
  - Ship internal packages as TS source with an `exports` map that has `react-native` and `browser` conditions. Metro and Vite both transpile TS, so this avoids per-package build steps. Keep `packages/domain` and `packages/sync` free of DOM and RN imports, and inject platform services (storage, crypto, clock, network) through interfaces.
  - Expo SDK 58 requires Node 22.13+, 24.3+, or 26+ ([SDK 58 beta](https://expo.dev/changelog/sdk-58-beta)). Pin Node with `packageManager`/`engines` in the repo.

## 2. Mobile (Expo)

**Pick: Expo SDK 57 now (`expo@57.0.26`, React Native 0.86, React 19.2). Move to SDK 58 when it goes GA**, which should be in days to weeks. The SDK 58 beta (RN 0.88 RC) has been out since 2026-09-15, and npm `next` is `58.0.3` ([Expo changelog](https://expo.dev/changelog), [SDK 57](https://expo.dev/changelog/sdk-57), [SDK 58 beta](https://expo.dev/changelog/sdk-58-beta)).

- **New Architecture:** it is the only option. Expo SDK 55 dropped the Legacy Architecture and removed the `newArchEnabled` flag ([SDK 55 beta](https://expo.dev/changelog/sdk-55-beta)). React Native 0.84 removed the Legacy Architecture components and made **Hermes V1** the default ([RN blog](https://reactnative.dev/blog)).
- **Expo Router:** `expo-router@57.0.24`, now versioned in lockstep with the SDK. Since SDK 56 the router no longer depends on React Navigation (it forked the parts it needs) ([SDK 56](https://expo.dev/changelog/sdk-56)). In SDK 58, data loaders, SSR, middleware, native tabs, and toolbars move from experimental to stable ([SDK 58 beta](https://expo.dev/changelog/sdk-58-beta)).
- **EAS:** EAS Build/Submit for store binaries. EAS Update for OTA, using `runtimeVersion: { policy: "fingerprint" }`, which hashes the native layer so a JS bundle only reaches compatible binaries (summarized by a secondary source; primary page not fetched: [Expo runtime versions](https://docs.expo.dev/eas-update/runtime-versions/)). Use **dev clients** (`expo-dev-client@57.0.19`). Unistyles, op-sqlite, passkeys, and similar native modules do not run in Expo Go, and Expo Go has required login since 2026-09-03 ([changelog](https://expo.dev/changelog)). `eas-cli@24.10.0`.
- **Why:** Expo is the default production path for RN. You get CNG/prebuild, config plugins, EAS Workflows (including a first-class Maestro job), and lockstep-versioned SDK modules (notifications, sqlite, location, background-task).
- **Main alternative:** bare React Native CLI (RN 0.87.1 is npm `latest`). It gives full native control but loses CNG, EAS Update ergonomics, and SDK module alignment.
- **Gotchas:**
  - **iOS 27 SDK** requires the UIKit scene-based lifecycle. SDK 57.0.23+ makes it opt-in (`ios.enableSceneSupport`); SDK 58 makes it mandatory, and iPhone apps become resizable ([SDK 57](https://expo.dev/changelog/sdk-57), [SDK 58 beta](https://expo.dev/changelog/sdk-58-beta)).
  - SDK 58 turns on Android R8 minification by default; test release builds early. It also makes `File.write()` async and removes libSQL from `expo-sqlite`.
  - SDK 57 had a Hermes V1 memory regression, fixed in `expo@57.0.9`. Stay on the latest patch.
  - SDK 56 raised the iOS minimum to 16.4 and Xcode to 26.4+ ([SDK 56](https://expo.dev/changelog/sdk-56)).
  - OTA updates may only change JS and assets consistent with the reviewed functionality. Native changes need a new build.
  - Always run `npx expo install` so Reanimated, Gesture Handler, and similar libraries match the SDK. SDK 57 pins Reanimated 4.5, while npm latest is 4.7.1, and Gesture Handler 3.x is a new major.

## 3. Web app

**Pick: Vite 8 SPA (`vite@8.3.2`, Rolldown bundler) + TanStack Router (`@tanstack/react-router@1.170`) + `vite-plugin-pwa@2.0.0` (Workbox), deployed as static assets on a CDN.** Put marketing/landing pages on a separate static site (Astro or Next.js static export) so the app shell stays pure client-side.

- **Why:** In a local-first app nearly all logic runs in the client: SQLite in a Worker, Yjs, and the sync socket. SSR adds server cost and hydration complexity for no user benefit behind login. Vite 8 has been stable since 2026-03-12 with Rolldown as its single bundler ([Vite 8](https://vite.dev/blog/announcing-vite8)). vite-plugin-pwa 2.0.0 shipped 2026-10-03 (its peer `@vite-pwa/assets-generator` moved to ^2), and 1.3.0 had already added Vite 8 support ([releases](https://github.com/vite-pwa/vite-plugin-pwa/releases)).
- **Main alternatives:**
  - **Next.js 16.3** (`next@16.3.8`; 16.3 shipped 2026-08-03 with "instant navigations"; Turbopack is the default) ([Next blog](https://nextjs.org/blog)). Good if you want marketing, SEO pages, and the app in one framework. Mark app routes client-only and add Serwist (`@serwist/next@9.5.12`) for the service worker.
  - **Expo web / React Native Web** (`react-native-web@0.21.3`, Metro). Maximizes screen sharing, but the web app is Keep's power-user surface (keyboard shortcuts, masonry grid, HTML5 drag-and-drop, rich text), and DOM-native libraries (TipTap, dnd-kit, TanStack Virtual) are better there. Expo's PWA story and `expo-sqlite` web (alpha, needs COOP/COEP) are less mature ([expo-sqlite docs](https://docs.expo.dev/versions/v57.0.0/sdk/sqlite/)). Share the core, not the screens.
  - TanStack Start (`@tanstack/react-start@1.168`) in SPA mode is a reasonable middle ground ([announcement](https://tanstack.com/blog/announcing-tanstack-start-v1)).
- **PWA/offline:**
  - Precache the app shell with Workbox and treat SQLite as the data source. The service worker should not cache API responses.
  - Call `navigator.storage.persist()` after install or sign-in. WebKit grants persistence by heuristics such as Home Screen installation. Origin quota is up to 60% of disk for browser apps, and eviction is LRU by origin under pressure or ITP ([WebKit storage policy](https://webkit.org/blog/14403/updates-to-storage-policy/)).
- **OPFS:** run official SQLite WASM in a dedicated Worker on the `opfs-sahpool` VFS. It needs no COOP/COEP headers, but allows only one connection per DB. The `opfs` VFS needs COOP/COEP for `SharedArrayBuffer`. The new `opfs-wl` VFS (3.53+) uses Web Locks and needs `Atomics.waitAsync` ([SQLite persistence](https://sqlite.org/wasm/doc/trunk/persistence.md)). Browser support: Chromium (mid-2022+), Firefox 111+, Safari 16.4+.
- **Gotchas:**
  - Avoid cross-origin isolation (COOP/COEP) if you can. It breaks OAuth popups and third-party embeds, which is why sahpool is the pick.
  - Multi-tab: elect one "DB owner" tab or worker with Web Locks (or use a SharedWorker), and have other tabs RPC to it.
  - On iOS, Web Push and the stronger persistence heuristics only apply once the PWA is installed to the Home Screen.

## 4. Cross-platform UI and styling

**Pick: one `packages/ui-tokens` source (TS) that generates a Tailwind CSS v4 `@theme` (CSS variables) for web and a Unistyles 3 theme for native (`react-native-unistyles@3.4.0`).** Web and mobile keep separate component layers that share the tokens and naming.

- **Why:** Keep's UI is small (cards, chips, toolbars, dialogs, color palettes, dark mode), so sharing tokens gets most of the consistency with no runtime abstraction. Unistyles 3 updates styles through C++/JSI with no re-renders. It supports themes and breakpoints, which matter for per-note color themes, dark mode, and tablets, and it supports web/SSR ([Unistyles](https://www.unistyl.es/v3/start/getting-started)).
- **Main alternatives:**
  - **NativeWind:** `nativewind@4.2.7` is stable; **5.0.0-rc.0** (2026-09-13) targets Expo 57 / RN 0.86 / Reanimated 4.5 ([release](https://newreleases.io/project/npm/nativewind/release/5.0.0-rc.0)). Pick NativeWind if you want the same `className` vocabulary on web and native. NativeWind 4 is built on Tailwind v3 while the web would be on Tailwind v4, so wait for v5 GA to align them.
  - **Tamagui 2.7** (`tamagui@2.7.7`; 2.0 shipped 2026-05; v3 in beta). Use it if you want to share actual components across web and native through its compiler.
  - **gluestack-ui v5** (`@gluestack-ui/core@5.0.15`, built on NativeWind; v5 was announced as alpha in [this discussion](https://github.com/gluestack/gluestack-ui/discussions/3366)) for prebuilt accessible components.
  - **react-strict-dom** is still `0.0.55`, last published 2026-01-09. Meta uses it internally, but the 0.0.x API is not a stable foundation yet ([repo](https://github.com/facebook/react-strict-dom)).
- **Gotchas:**
  - Unistyles 3 requires the New Architecture, RN 0.81+/Expo 54+, a pinned `react-native-nitro-modules` version, a Babel plugin, and a dev client (no Expo Go) ([getting started](https://www.unistyl.es/v3/start/getting-started)).
  - Keep's 12 note colors need light and dark variants. Model them as semantic tokens (`note.bg.coral`), not hex values in notes, so the theme can remap them.

## 5. Rich-text editor with CRDT binding (web + RN)

**Pick (one strategy, two note kinds):**

1. **Text notes:** **TipTap 3** (`@tiptap/react@3.31.4`) with `@tiptap/extension-collaboration` + **`@tiptap/y-tiptap@3.0.9`** + **Yjs 13.6.33**. TipTap v3's collaboration install uses `@tiptap/y-tiptap` (its y-prosemirror fork) and disables the StarterKit UndoRedo in favor of Yjs undo ([TipTap collaboration](https://tiptap.dev/docs/editor/extensions/functionality/collaboration)).
   - **On RN**, render the *same* `packages/editor` React component through an **Expo DOM component** (`'use dom'`). Since SDK 56 it uses `@expo/dom-webview` by default. Expo's docs call out rich text as a good fit; on web the component renders as normal DOM ([Expo DOM components](https://docs.expo.dev/guides/dom-components/)).
   - The WebView holds a **Y.Doc replica**. It exchanges binary Yjs updates (base64) with the RN-side Y.Doc owned by `packages/sync` through async "native action" props. Two Yjs replicas converge by construction, so the bridge is just another sync transport.
2. **Checklist notes:** not TipTap task lists. Use a **structured CRDT**: `Y.Array<Y.Map{ id, text: Y.Text, checked, indent(0|1), order }>`, rendered with **native `TextInput`s on RN** (diff each change into Y.Text) and simple inputs on web.
   - This matches Keep's behavior: list notes are a separate type with check/uncheck, one indent level, "move checked items to bottom", and drag to reorder items. It also gives a fully native-feeling list editor without a WebView.

- **CRDT library:** **Yjs**. It is pure JS, so it runs on Hermes, and it has the most mature ProseMirror binding plus server ecosystem (Hocuspocus). Loro (`loro-crdt@1.16.4`, `loro-prosemirror@0.4.4`) and Automerge (`@automerge/automerge@3.5.0`) are Rust/WASM-based. **UNVERIFIED:** whether current Hermes runs WASM in production RN; the RN 0.84–0.87 posts don't mention it ([RN blog](https://reactnative.dev/blog)).
- **Main alternatives:**
  - **10tap-editor** (`@10play/tentap-editor@1.0.1`) puts TipTap in a WebView behind a typed bridge. Its last release was 2025-11-27 (a TipTap v3 history fix), so maintenance cadence is a risk, and it has no built-in Yjs support ([releases](https://github.com/10play/10tap-editor/releases)).
  - **react-native-enriched-html** (`1.1.1`, Software Mansion) is a fully native editor for iOS/Android/Web, but its I/O is HTML and it has no CRDT/delta API. Collaborative binding would mean lossy HTML↔Y diffing ([repo](https://github.com/software-mansion/react-native-enriched)).
  - **Lexical** (`lexical@0.52.0`, `@lexical/yjs`) has no RN renderer.
- **Gotchas:**
  - DOM components run standard JS, not Hermes bytecode, so startup is slower. Keep a warm, hidden editor instance and pre-mount it on the note screen.
  - Props and actions are async and serializable only, and children can't be passed ([Expo DOM](https://docs.expo.dev/guides/dom-components/)).
  - Keyboard avoidance and scroll inside a WebView need care.
  - The ProseMirror schema must be **versioned and identical** across clients. An older app version silently drops unknown node types it can't parse, so gate new node types behind a schema version flag carried in the doc.
  - Keep's text formatting is modest (H1/H2, bold, italic, underline, links). Keep the schema minimal.

## 6. Masonry grid + drag-and-drop with 1000s of notes

**Web pick: `@tanstack/react-virtual@3.14` with `lanes` (masonry) + `useWindowVirtualizer` + `measureElement`, and `@dnd-kit/react@0.5.0` for sortable drag-and-drop.**

- **Why:** TanStack Virtual's `lanes` puts each item in the shortest lane and measures real heights with `measureElement`. `laneAssignmentMode` can defer lane caching until measurement ([TanStack Virtual](https://tanstack.com/virtual/latest/docs/api/virtualizer)). dnd-kit's new framework-agnostic core is actively developed: 0.5.0 shipped 2026-06-11 and a 0.5.1 beta followed in 2026-09. The legacy `@dnd-kit/core@6.3.1` has not been published since 2024-12.
- **Alternatives:**
  - `@atlaskit/pragmatic-drag-and-drop@4.0.0` uses native HTML5 drag-and-drop, which is robust with virtualization, but you build the sortable animations yourself.
  - `@virtuoso.dev/masonry@1.4.3` is a ready-made virtualized masonry, but it can show visible re-arrangement on fast scroll ([docs](https://virtuoso.dev/masonry/)).
- **CSS masonry status:** `display: grid-lanes` ships only in Safari 26.4+. Chrome/Edge (140+) and Firefox have it behind flags, and global support is about 10% ([OpenReplay overview](https://blog.openreplay.com/css-grid-lanes-masonry-layout/); secondary source). Treat it as a progressive enhancement only. It also doesn't virtualize.

**RN pick: FlashList v2 (`@shopify/flash-list@2.3.3`) with the `masonry` prop + Reanimated 4 + Gesture Handler (SDK-pinned versions); `react-native-sortables@1.10.1` for reorder mode.**

- **Why:** FlashList v2 made masonry a prop (`MasonryFlashList` is deprecated). It measures real item sizes with no estimates and supports `overrideItemLayout` spans ([FlashList masonry](https://shopify.github.io/flash-list/docs/guides/masonry), [Shopify Eng](https://shopify.engineering/flashlist-v2)). react-native-sortables supports Grid and Flex layouts, auto-scroll, haptics, Reanimated 3/4, and web ([repo](https://github.com/MatiPl01/react-native-sortables)).
- **Alternative:** a custom drag layer. Long-press + pan with Gesture Handler lifts the card into a Reanimated overlay, and the drop index is computed from FlashList layouts. Alternatively, `react-native-reanimated-dnd` (Reanimated 4).
- **Gotchas:**
  - FlashList's `optimizeItemArrangement` (on by default) **reorders items to balance columns**. Disable it, or Keep's user-defined order breaks.
  - **UNVERIFIED:** whether react-native-sortables virtualizes. Its docs don't say. With 1000s of notes, enter an explicit reorder mode that limits the sortable window to the visible range, or build the custom overlay.
  - **Order must be CRDT-safe.** Store a fractional index string per note (`fractional-indexing@4.0.0` or `jittered-fractional-indexing@1.0.1`) rather than integer positions, so concurrent moves on two devices both survive. Keep pinned notes as a separate section and order.
  - Masonry position depends on measured heights. Lay out deterministically from (order, height), and don't animate everything on each remote change.

## 7. Local database

**Web pick: `@sqlite.org/sqlite-wasm@3.53.4` (official build) in a dedicated Worker on the `opfs-sahpool` VFS.**

- **Why:** you get real SQL, transactions, and **FTS5**. The canonical WASM build includes FTS5 but not FTS3/4 ([SQLite forum](https://sqlite.org/forum/info/28402061cb), [building docs](https://sqlite.org/wasm/doc/trunk/building.md)). sahpool needs no COOP/COEP ([persistence docs](https://sqlite.org/wasm/doc/trunk/persistence.md)).
- **Alternatives:**
  - **wa-sqlite** has a richer VFS set (IDBBatchAtomicVFS, OPFSCoopSyncVFS, AccessHandlePoolVFS, OPFSWriteAheadVFS…) and is active on GitHub, but npm `wa-sqlite@1.0.0` has not been published since 2024-01. Consume it from GitHub or a maintained fork ([repo](https://github.com/rhashimoto/wa-sqlite)).
  - **Dexie 4.4.6 / IndexedDB** is simpler, but has no SQL joins or FTS5. It is a fallback only.

**RN pick: `expo-sqlite@57.0.3`.**

- **Why:** it is first-party and versioned with the SDK. The `enableFTS` config plugin option defaults to **true** (FTS3/4/5). It also offers SQLCipher, session/changeset APIs, sqlite-vec, a `kv-store`, and Drizzle support ([expo-sqlite](https://docs.expo.dev/versions/v57.0.0/sdk/sqlite/)).
- **Alternative:** **op-sqlite** (`@op-engineering/op-sqlite@18.2.5`) is a JSI library tuned for raw performance. It has FTS5/Rtree/sqlite-vec plugins, SQLCipher/libsql targets, reactive queries, and custom tokenizers ([repo](https://github.com/op-engineering/op-sqlite)). Switch if profiling shows expo-sqlite is the bottleneck.

**Gotchas:**

- Store Yjs data as an **append-only update log (BLOB) plus periodic snapshot compaction** per note. Keep a **projection table** (`notes(id, title, plain_text, color, pinned, archived, trashed_at, order_key, updated_at…)`) maintained from the CRDT for list queries and FTS5.
- Share plain-SQL migrations across web and native from `packages/storage`.
- On web, a single sahpool connection means **one writer per origin**. Coordinate tabs with Web Locks or a SharedWorker.
- `expo-sqlite` web support is alpha and needs COOP/COEP, which is another reason to use sqlite-wasm directly on web.
- SDK 58 removes libSQL from expo-sqlite.

## 8. Notifications and reminders

**Pick:**

- **Device-local scheduling** with `expo-notifications@57.0.21` for the user's own reminders (works offline).
- **Server-side scheduled push** as the cross-device and web path: direct **APNs/FCM** from the worker using native device tokens (`getDevicePushTokenAsync`), via `firebase-admin@14.5.0` and APNs HTTP/2 (e.g., `@parse/node-apn@8.1.0`).
- **Web Push (VAPID)** with `web-push@3.6.7`.

Verified platform limits:

- **iOS: 64 pending local notifications per app.** The system keeps the soonest-firing 64 and discards the rest; a repeating notification counts once ([Apple UILocalNotification docs](https://developer.apple.com/documentation/uikit/uilocalnotification), [Apple forums](https://developer.apple.com/forums/thread/811171)).
  - Use a **rolling window**: schedule the next ~50 reminders to leave headroom, and re-plan on foreground, after sync, and from a background task.
- **Android exact alarms:**
  - `SCHEDULE_EXACT_ALARM` is **not pre-granted on fresh installs on Android 13/14+**. Check `canScheduleExactAlarms()`, send the user to `ACTION_REQUEST_SCHEDULE_EXACT_ALARM` with a rationale, and listen for permission-state broadcasts ([Android docs](https://developer.android.com/develop/background-work/services/alarms/schedule)).
  - `USE_EXACT_ALARM` is auto-granted but limited to specific use cases (alarm clock/calendar-type core functionality) by Google Play policy. **UNVERIFIED** whether a notes app with reminders qualifies; I could not load the Play policy page. Assume `SCHEDULE_EXACT_ALARM` plus a graceful fallback.
  - Inexact alarms on Android 12+ fire within about 1 hour; `setWindow` has a 10-minute minimum window.
  - expo-notifications needs `SCHEDULE_EXACT_ALARM` declared for exact-time triggers ([expo-notifications](https://docs.expo.dev/versions/latest/sdk/notifications/)). SDK 58 shows foreground notifications by default ([SDK 58 beta](https://expo.dev/changelog/sdk-58-beta)).
- **Background execution:** `expo-background-task@57.0.21` uses WorkManager (15-minute minimum interval) on Android and BGTaskScheduler (system-decided timing) on iOS. Both need sufficient battery and network ([docs](https://docs.expo.dev/versions/latest/sdk/background-task)). Use it for best-effort re-planning only, never for firing reminders.
- **Location reminders (geofencing):**
  - **iOS caps an app at 20 monitored regions.** CLMonitor (iOS 17+) keeps the same 20-condition limit ([Apple region monitoring](https://developer.apple.com/Library/ios/documentation/UserExperience/Conceptual/LocationAwarenessPG/RegionMonitoring/RegionMonitoring.html), [Apple forum](https://origin-devforums.apple.com/forums/thread/769113)).
  - **Android allows 100 geofences per app per device user** ([Android geofencing](https://developer.android.com/training/location/geofencing)).
  - Register only the nearest N geofences and re-plan on significant location change. This needs "Always"/background location permission, and on Google Play a background-location declaration.
  - `expo-location@57.0.20` provides geofencing; SDK 58 adds an opt-in rewrite preview.
- **Web Push:**
  - On iOS/iPadOS it works only for **Home Screen web apps** (iOS 16.4+) ([WebKit](https://webkit.org/blog/13878/web-push-for-web-apps-on-ios-and-ipados/); not re-fetched this session).
  - **Declarative Web Push** (Safari/iOS 18.4+, installed web apps) can show notifications without running service-worker JS ([WebKit Safari 18.4](https://webkit.org/blog/16574/digital-credentials-api/), [Meet Declarative Web Push](https://webkit.org/?p=16535)).
- **Expo Push Service vs direct:**
  - Expo's service is simple and free, but limited to **600 notifications/second per project** ([Expo push FAQ](https://docs.expo.dev/push-notifications/faq/)).
  - At millions of users with time-clustered reminders (e.g., 9:00 local), send directly to APNs/FCM with your own rate limiting. Expo Push is fine for an MVP.

**Gotchas:**

- Deduplicate local and remote firings by reminder ID plus a `fired_at`/`dismissed_at` state in the CRDT.
- Store reminders in UTC **plus the IANA timezone** so "9am" stays 9am when the user travels.
- Recurring reminders: one repeating local trigger counts as one of iOS's 64 slots.

## 9. Auth

**Pick: Better Auth 1.7 (`better-auth@1.7.7`), self-hosted on your Postgres, with `@better-auth/expo@1.7.7` and `@better-auth/passkey@1.7.7`.**

- **Why:**
  - It is TS-native and framework-agnostic, and keeps users and sessions in your own DB.
  - The Expo plugin caches the session in `expo-secure-store` and handles deep-link OAuth callbacks.
  - It supports native ID-token sign-in for Apple and Google, and its passkey plugin works with Expo ([Better Auth Expo](https://www.better-auth.com/docs/integrations/expo)).
  - Its **JWT plugin** issues 15-minute JWTs (EdDSA by default) with a JWKS endpoint, which suits the WebSocket server ([JWT plugin](https://www.better-auth.com/docs/plugins/jwt)).
- **Ecosystem status:**
  - **Auth.js** is now maintained by the Better Auth team (2025-09), which recommends Better Auth for new projects ([announcement](https://www.better-auth.com/blog/authjs-joins-better-auth)). `next-auth@4.24.15` is still npm latest; v5 is still `beta`.
  - **Better Auth joined Vercel** (2026-07-07) and says it stays open source and platform-agnostic ([post](https://better-auth.com/blog/better-auth-joins-vercel)).
  - **Lucia** was deprecated as a library in March 2025 and is now a learning resource ([repo](https://github.com/lucia-auth/lucia)).
- **Alternatives:**
  - **Clerk** (`@clerk/expo@4.8.0`): fastest to build, hosted, priced per MAU, but you don't own the user table.
  - **WorkOS AuthKit:** free up to 1M MAU, then $2,500 per additional 1M ([pricing](https://workos.com/pricing)).
  - **Supabase Auth:** best if you're on Supabase anyway.
- **Sign in with Apple (App Store 4.8):** if you offer Google or another social login for the primary account, you must also offer an equivalent login that limits data to name and email, allows a private email, and does no ad tracking without consent. SIWA is the simplest way to comply. Apps that **exclusively use their own account system** are exempt. Separately, **in-app account deletion is mandatory** (5.1.1(v)) ([guidelines](https://developer.apple.com/app-store/review/guidelines/)).
- **Token strategy:**
  - **HTTP:** opaque, DB-backed, rotating sessions. On mobile, send the session as a bearer header, stored in `expo-secure-store`.
  - **WebSocket:** fetch a **short-lived JWT** (≤15 min) and send it **in the first WS message**, not the URL query, so it doesn't leak into logs. Verify it against cached JWKS with no DB hit. Before expiry, the server requests re-auth **in-band** and the client sends a fresh token without reconnecting.
  - **Revocation** (logout, collaborator removed): publish over Valkey pub/sub so sync nodes drop affected subscriptions.
  - **Authorize per document subscription** (note ACL), not just at connect.
- **Gotchas:**
  - Passkeys on RN need `react-native-passkey@3.6.2` (or similar) plus iOS Associated Domains and an Android `assetlinks.json`.
  - Better Auth on RN doesn't auto-attach cookies to `fetch`. Add the session header manually, as the docs show.

## 10. Backend

**Pick: Node.js 24 LTS now; move to Node 26 when it becomes Active LTS on 2026-10-28** ([Node Release schedule](https://github.com/nodejs/Release)). **Fastify 5.12** (`fastify@5.12.5`) for HTTP, **oRPC 1.15** (`@orpc/server@1.15.4`, `@orpc/openapi`) for typed contracts, and a **separate sync service on Hocuspocus 4.7** (`@hocuspocus/server@4.7.0`) running on `ws` or uWebSockets.js.

- **Why:**
  - **Fastify** is the fastest mainstream Node HTTP framework in its own benchmarks (~88k req/s vs Hono ~83k on Node; [benchmarks](https://www.fastify.dev/benchmarks/)). It has mature schema/serialization, plugins, and OTel instrumentation.
  - **oRPC** gives tRPC-style end-to-end types and also generates **OpenAPI** for REST consumers, so third parties and future integrations don't need a second API ([oRPC](https://orpc.dev/docs/getting-started)).
  - **Hocuspocus v4** (2026-04) runs on Node with `ws` **or uWebSockets.js** through crossws, as well as on Bun and Deno. It scales horizontally with `@hocuspocus/extension-redis` (pub/sub). `sessionAwareness` lets many document providers share **one WebSocket** ([releases](https://github.com/ueberdosis/hocuspocus/releases)).
- **Alternatives:**
  - **Hono 4.13** (`@hono/node-server@2.1.3`) if you want runtime portability (edge, Bun, Workers).
  - **NestJS 12** if you want enforced DI/modules; it's heavier.
  - **tRPC 11.19**: the most popular option, but without first-class OpenAPI.
  - **ts-rest 3.52**: last published 2025-06; momentum has slowed.
  - **Custom sync on uWebSockets.js** (v20.71.0, 2026-09-16; distributed from **GitHub, not npm**; dropped Node 20/25 and added Node 26; [releases](https://github.com/uNetworking/uWebSockets.js/releases)) for maximum connection density.
  - **Bun 1.4** (2026-08-20) was rewritten in Rust and targets Node 26.3 compatibility ([Bun blog](https://bun.sh/blog)). It's promising, but the rewrite is weeks old, so keep production on Node and use Bun for tooling at most.
- **Gotchas:**
  - Keep has **thousands of notes per user**, so don't open one provider per note. Use a two-level protocol:
    - (a) a per-user **index channel** that streams note metadata (color, pinned, archived, labels, order key, reminder) with a server cursor;
    - (b) per-note Yjs content sync, on demand plus background catch-up of "changed since cursor".
  - Route sockets by user, or consistent-hash by doc, to cut cross-node fanout.
  - Implement graceful drain with a "reconnect elsewhere" hint plus client jittered backoff.

## 11. Database and ORM

**Pick: PostgreSQL 18 (18.6) + Drizzle ORM (`drizzle-orm@0.45.3` stable; plan the move to 1.0, now `1.0.0-rc.4`) with drizzle-kit SQL migrations (`drizzle-kit@0.31.11`).**

- **Postgres versions:** PG 18 is current stable. 18.6 shipped 2026-08-13, and PG 19 is at **Beta 4** (2026-09-24) with no GA date yet ([postgresql.org](https://www.postgresql.org/)). PG 18 has been on **RDS since 2025-11-14** (18.6 on RDS since 2026-08-25) and on **Aurora since 2026-06-11** (18.6 on Aurora since 2026-09-29) ([RDS calendar](https://docs.aws.amazon.com/AmazonRDS/latest/PostgreSQLReleaseNotes/postgresql-release-calendar.html), [Aurora calendar](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraPostgreSQLReleaseNotes/aurorapostgresql-release-calendar.html)). PG 14 reaches community EOL on 2026-11-12.
- **Why Drizzle:** SQL-first and type-safe, with zero codegen runtime. drizzle-kit generates reviewable SQL migrations. The same team also supports expo-sqlite, which helps if you want typed queries on the client too. The v1 RC series brings codec-based mappers, JIT row mappers, and a reported 25–30% latency reduction, and removes RQB v1 for Postgres ([Drizzle v1 upgrade](https://orm.drizzle.team/docs/upgrade-v1)).
- **Alternatives:**
  - **Kysely 0.29.6:** a pure query builder, best for complex SQL. Pair it with dbmate/Atlas migrations.
  - **Prisma 7.10:** the stable line. **Prisma 8 is an RC with GA expected October 2026**. It splits into DB-specific packages (`@prisma/orm-postgres`), and the RC still lacks `$extends`, JSON filters, atomic increment, most nested writes, and isolation levels ([Prisma release status](https://www.prisma.io/docs/orm/release-status)). Note that npm `prisma@latest` already points at `8.0.0-rc.19` while `@prisma/client@latest` is 7.10.0, so pin exact versions.
- **Gotchas:**
  - Store CRDT updates as `bytea` in an append-only `note_updates(note_id, seq, update, created_at)` table and compact into `note_snapshots` with a background job.
  - Use PG 18's native `uuidv7()` for time-ordered keys.
  - Plan hash partitioning by `owner_id` for the update log.
  - Use RDS Proxy or PgBouncer in transaction mode for the API. Transaction pooling breaks session features such as `LISTEN/NOTIFY` and session-level prepared statements, so give job workers direct connections.
  - Run zero-downtime migrations as expand → backfill → contract.

## 12. Jobs and scheduling

**Pick: pg-boss 12 (`pg-boss@12.36.0`) as the default queue; add BullMQ 6 (`bullmq@6.3.11`) on Valkey for high-throughput push delivery.**

- **Why pg-boss:**
  - It is Postgres-native (`SKIP LOCKED`), so jobs can be **enqueued in the same transaction** as business writes (outbox semantics).
  - It supports cron and **RRULE** scheduling, deferral, singleton policies, retries with backoff, dead-letter queues, priority, batching, groups/concurrency, and OTel. It requires Node 22.12+ and PG 13+ ([pg-boss](https://github.com/timgit/pg-boss)).
  - It covers trash purge (Keep deletes trash after 7 days), CRDT compaction, thumbnails/OCR/transcription, and account deletion.
- **Why BullMQ for push:** it uses no polling (Lua scripts), and offers rate limiting, deduplication, flows, and job schedulers ([BullMQ](https://docs.bullmq.io/)). Use it for spiky "9:00 local time" fan-out to APNs/FCM.
- **Alternatives:**
  - **Graphile Worker 0.18:** very fast, PG-native, still 0.x.
  - **Temporal** (`@temporalio/client@1.24.0`): durable workflows; overkill for Keep.
  - **Inngest 4.21 / Trigger.dev 4.7:** hosted-first with great DX, but per-run pricing at reminder scale and an external dependency in the hot path.
- **Reminder fan-out pattern:** keep a `reminders(next_fire_at, …)` table with a partial index. A scheduler tick (every 15–30 s) claims due rows in batches with `SKIP LOCKED`, enqueues send jobs, and computes the next occurrence for recurring reminders. Don't pre-create one delayed job per future reminder.
- **Gotchas:**
  - BullMQ **requires `maxmemory-policy noeviction`**; AOF persistence is recommended ([BullMQ production](https://docs.bullmq.io/guide/going-to-production)). Don't share that Valkey with an evicting cache.
  - pg-boss load lands on your primary DB. Watch bloat, tune autovacuum on job tables, and split to a dedicated queue DB if needed.

## 13. Search

**Pick: SQLite FTS5 on every client as the primary search (offline, instant), plus Postgres FTS (`tsvector` + GIN) with `pg_trgm` server-side for cold-start and web-before-sync, scoped by owner/ACL.**

- **Why:** a single user's corpus (thousands of notes) is small, so client FTS5 gives instant, offline results. FTS5 is in the sqlite-wasm canonical build and enabled by default in expo-sqlite (see §7). Server-side, Postgres FTS plus trigram fuzzy/substring search filtered by `note_access(user_id, note_id)` needs no extra system and no extra sync pipeline.
- **Alternatives:**
  - **Meilisearch 1.54** (2026-10-01; [releases](https://github.com/meilisearch/meilisearch/releases)): tenant tokens embed per-user filters ([tenant tokens](https://www.meilisearch.com/docs/learn/security/tenant_tokens)).
  - **Typesense v30.x** ([releases](https://github.com/typesense/typesense/releases); the release dates on that page looked inconsistent: **UNVERIFIED**).
  - **ParadeDB `pg_search`** (BM25 in Postgres) is not available on RDS/Aurora. Run it as a logical replica.
- **Gotchas:**
  - Index a **plain-text projection** extracted from the Y.Doc, refreshed on compaction, not the CRDT bytes. Include label names, checklist item text, OCR text, and transcripts.
  - Use the `simple` config plus `unaccent` (and FTS5 `unicode61 remove_diacritics` or the `trigram` tokenizer) instead of English stemming, for multilingual notes.
  - Shared notes must appear in collaborators' indexes. Fan out ACL rows on share and unshare.

## 14. Media (images, drawings, audio)

**Pick:**

- **Storage:** **S3 + CloudFront** if hosting on AWS; **Cloudflare R2** if egress cost dominates.
- **Uploads:** **direct-to-bucket presigned uploads**.
- **Processing:** **sharp 0.35.5** (libvips) in the worker for thumbnails, outputting **AVIF + WebP** (JPEG fallback).
- **OCR:** **on-device** (Apple Vision / ML Kit) first.
- **Speech-to-text:** on-device (`expo-speech-recognition`), with server Whisper as a fallback.

Details:

- **Storage trade-offs:** R2 costs $0.015/GB-month (Standard) with **no egress fees**. Class A operations cost $4.50 per million and Class B $0.36 per million ([R2 pricing](https://developers.cloudflare.com/r2/pricing/)). R2 presigned URLs support GET/HEAD/PUT/DELETE but **not presigned POST**, so you can't use a `content-length-range` policy. They also **don't work on custom domains**, and expire in at most 7 days. You can pin `Content-Type` in the signature ([R2 presigned URLs](https://developers.cloudflare.com/r2/api/s3/presigned-urls/)). On S3, use **presigned POST** with `content-length-range` to enforce size limits (`@aws-sdk/client-s3@3.1146.0`). GCS is the alternative if hosting on GCP.
- **Upload flow:**
  1. The client asks the API for an upload slot, and the server records a `pending` attachment.
  2. The client PUTs/POSTs directly to the bucket.
  3. The client confirms, which triggers a job that validates magic bytes, strips EXIF/GPS, and generates thumbnails.
  4. The attachment ID is inserted into the note CRDT.
  
  Attachments upload offline-first via the local outbox.
- **Images:** sharp's prebuilt binaries list JPEG, PNG, Ultra HDR, WebP, AVIF, TIFF, GIF, and SVG input, and need Node ≥ 20.9 ([sharp install](https://sharp.pixelplumbing.com/install)). HEIC (HEVC) is **not** in that list, so **convert HEIC → JPEG/WebP on device** (`expo-image-manipulator@57.0.20`) before upload. Render on RN with `expo-image@57.0.5`.
- **OCR ("Grab image text"):**
  - On-device: Apple Vision `RecognizeTextRequest`, plus iOS 26 `RecognizeDocumentsRequest` (structure-aware, 26 languages; secondary source: [Kodeco](https://www.kodeco.com/ios/paths/apple-ai-models/49307523-vision-framework/04-unveiling-new-vision-framework-features-in-ios-18-ios-26/05)); and ML Kit Text Recognition v2 on Android. Use them via `expo-text-extractor@2.0.0` or `@react-native-ml-kit/text-recognition@2.0.0`.
  - Web: `tesseract.js@7.0.0` in a Worker, or server-side Google Cloud Vision / AWS Textract for quality.
- **Voice notes:**
  - `expo-speech-recognition@57.1.0` wraps SFSpeechRecognizer and Android SpeechRecognizer. It does on-device recognition on iOS 17+ and Android 13+ (model download required on Android), **transcribes audio files**, and also supports web ([repo](https://github.com/jamsch/expo-speech-recognition)).
  - Alternatives: on-device `whisper.rn@0.7.4` (whisper.cpp) or `react-native-executorch@0.10.4`; server-side self-hosted faster-whisper/whisper.cpp on GPU, or a hosted transcription API (current model names **UNVERIFIED**).
- **Gotchas:**
  - Store originals privately and serve them through signed or short-TTL CDN URLs.
  - Keep's drawings: store vector strokes (JSON in the CRDT) plus a rendered PNG/WebP thumbnail.
  - Cap audio length and size, and record AAC/M4A.

## 15. Observability and testing

**Pick:**

- **Tracing/metrics:** **OpenTelemetry** (`@opentelemetry/api@1.9.1`, `@opentelemetry/sdk-node@0.222.0`, `@opentelemetry/auto-instrumentations-node@0.80.0`) on all services, exported to any OTLP backend.
- **Errors/replay:** **Sentry** (`@sentry/node@11.4.0`, `@sentry/browser@11.4.0`, `@sentry/react-native@8.29.0`) across clients and servers. Upload source maps for EAS Update bundles.
- **Mobile performance:** **EAS Observe**, generally available since 2026-08-20, for real-world app performance ([Expo changelog](https://expo.dev/changelog)).
- **Unit/integration:** **Vitest 5** (`vitest@5.0.3`; 5.0 announced 2026-09-03; [Vitest blog](https://vitest.dev/blog)) for `domain`, `sync`, the server, and web. Keep `jest-expo` for RN component tests; RN 0.85 moved its Jest preset to a separate package ([RN blog](https://reactnative.dev/blog)).
- **E2E:** **Playwright 1.63** (`@playwright/test@1.63.0`) for web, using multiple browser contexts for two-user collaboration and `context.setOffline()` for offline/merge scenarios. **Maestro** (CLI 2.11.0, 2026-09-29; [releases](https://github.com/mobile-dev-inc/maestro/releases)) for mobile, run natively in **EAS Workflows** (`type: maestro` jobs; insights dashboard since 2026-06-24) ([EAS E2E](https://docs.expo.dev/eas/workflows/examples/e2e-tests/)).

Notes:

- **Alternative:** **Detox 20.51.4** for gray-box RN E2E with JS-thread synchronization. It is more setup and less EAS-integrated.
- **CRDT tests:** property-based convergence tests (fast-check). Generate random concurrent ops across N simulated replicas with random delivery order and offline gaps, then assert identical state and projections.
- **Gotchas:**
  - Propagate trace context through the WebSocket. Add a `traceparent` field to sync frames so client edit → server apply → fan-out spans link up.
  - Sample WS message spans heavily; per-keystroke spans are too much volume.

## 16. Hosting

**Pick: AWS.**

- **Compute:** **ECS** (Fargate to start; EC2 capacity providers for WS density) behind an **ALB**.
- **Database:** **Aurora PostgreSQL 18** (or RDS PG 18.6).
- **Cache/queues:** **ElastiCache for Valkey 9.1**.
- **Media:** **S3 + CloudFront**.
- **Web app:** the static SPA on CloudFront (or Cloudflare).

- **Why:** this setup is designed for millions of users. It gives predictable long-lived connections, horizontal WS scaling, managed failover and PITR, and IAM. The ALB supports WebSockets with an idle timeout of up to 4000 s; tune `deregistration_delay` for draining ([websocket.org ALB guide](https://websocket.org/guides/infrastructure/aws/alb/); secondary source). ElastiCache added Valkey 9.0 (2026-05-05) and **Valkey 9.1** (2026-06-23), which brings new I/O threading, up to 17% more throughput, and up to 20% less memory for small strings ([AWS what's new](https://aws.amazon.com/about-aws/whats-new/2026/06/amazon-elasticache-valkey-9-1/)).
- **Alternatives:**
  - **Fly.io:** great DX and global placement. The proxy sends WebSocket `1001` GOAWAY on restarts, so recycle connections every 10–15 minutes. `kill_timeout` can be up to 5 minutes on shared CPU and 24 hours on dedicated ([Fly blog](https://fly.io/blog/graceful-vm-exits-some-dials/), [community](https://community.fly.io/t/long-lived-tcp-connections-are-dropped/4074)).
  - **Render / Railway:** simplest for MVP stage.
  - **GCP Cloud Run:** supports WebSockets, but the **maximum request timeout is 60 minutes**, so connections are force-closed hourly. Session affinity is best-effort, an instance takes up to 1,000 concurrent connections, and an instance with any open socket incurs instance-based billing. It needs Memorystore pub/sub across instances ([Cloud Run WebSockets](https://docs.cloud.google.com/run/docs/triggering/websockets)).
- **Managed Postgres options:**
  - **Aurora/RDS:** scale and ecosystem. Aurora takes major versions up to 8 months after the community `.1` release; RDS takes about 30 days.
  - **Neon:** acquired by Databricks in 2025; scale-to-zero and branching are excellent for preview environments. Post-acquisition pricing changes are **UNVERIFIED**; they come from secondary sources only.
  - **Supabase:** bundled Postgres, auth, and storage.
  - **Crunchy Bridge:** Crunchy Data was acquired by Snowflake in June 2025 and its service is becoming "Snowflake Postgres". The status of standalone Bridge is **UNVERIFIED**.
- **Redis vs Valkey:** Valkey is BSD-licensed and is AWS's default engine. Redis 8 added AGPLv3 alongside RSAL/SSPL ([The Register](https://www.theregister.com/2025/05/01/redis_returns_to_open_source/)). Both work with BullMQ and Hocuspocus's Redis extension.
- **Gotchas:**
  - Never put the sync service on Lambda or other request-scoped serverless.
  - Deploys must drain sockets gradually (stagger and send reconnect hints) to avoid a thundering herd of reconnects plus catch-up syncs.
  - Size Valkey pub/sub per active doc channel, not per user.
  - Place the web app's static assets on a separate origin from the API only if CORS and cookies are planned for it. With Better Auth cookie sessions on web, prefer same-site subdomains.

---

## Recommended stack at a glance

| Area | Recommended pick (version, 2026-10-04) | Main alternative | Key gotcha |
|---|---|---|---|
| Monorepo | pnpm 12.9 workspaces + Turborepo 2.11 | Nx 23.2 | Single React/RN copy; `nodeLinker: hoisted` fallback for Metro |
| Shared core | `domain` / `sync` / `storage` / `editor` / `ui-tokens` / `api-contract` packages, TS source exports | (share screens via Expo web) | Keep core free of DOM/RN imports; inject platform services |
| Mobile | Expo SDK 57 (RN 0.86, React 19.2) → SDK 58 at GA (RN 0.88); New Arch only; Hermes V1; Expo Router 57.x; EAS Build/Submit/Update (fingerprint); dev clients | Bare RN 0.87 | iOS 27 scene lifecycle; R8 default in SDK 58; Node ≥22.13 |
| Web | Vite 8.3 SPA + TanStack Router 1.170 + vite-plugin-pwa 2.0; static marketing site | Next.js 16.3 (client-only app routes + Serwist) | Avoid COOP/COEP; persist storage; iOS install required for push |
| UI/styling | Shared tokens → Tailwind v4 (web) + Unistyles 3.4 (native) | NativeWind 4.2.7 / 5.0 RC; Tamagui 2.7 | NativeWind 4 uses Tailwind v3; react-strict-dom still 0.0.x |
| Editor | TipTap 3.31 + @tiptap/y-tiptap 3.0 + Yjs 13.6; RN via Expo DOM component (same editor); checklists as Y.Array of Y.Map with Y.Text + native inputs | 10tap-editor 1.0.1; react-native-enriched-html 1.1 (no CRDT) | Version the PM schema; WebView warm-up; async bridge |
| CRDT | Yjs 13.6 (pure JS, runs on Hermes) | Loro 1.16 / Automerge 3.5 (WASM; Hermes WASM UNVERIFIED) | Append-only update log + compaction |
| Masonry + DnD (web) | @tanstack/react-virtual 3.14 `lanes` + @dnd-kit/react 0.5 | pragmatic-drag-and-drop 4.0; @virtuoso.dev/masonry 1.4 | CSS grid-lanes Safari-only; fractional index ordering |
| Masonry + DnD (RN) | FlashList 2.3 `masonry` + Reanimated 4 + RNGH (SDK-pinned) + react-native-sortables 1.10 | Custom Reanimated drag overlay | Disable `optimizeItemArrangement`; sortables virtualization UNVERIFIED |
| Local DB (web) | @sqlite.org/sqlite-wasm 3.53 + `opfs-sahpool` in Worker (FTS5 built in) | wa-sqlite (GitHub/fork); Dexie 4.4 | One connection → leader tab via Web Locks |
| Local DB (RN) | expo-sqlite 57 (FTS on by default) | op-sqlite 18.2 | expo-sqlite web is alpha; libSQL removed in SDK 58 |
| Reminders/notifications | expo-notifications 57 local (rolling ≤~50 on iOS) + direct APNs/FCM (firebase-admin 14.5) + web-push (VAPID) | Expo Push Service (600/s/project) | iOS 64 pending cap; Android exact-alarm permission not pre-granted; geofences 20 (iOS) / 100 (Android) |
| Auth | Better Auth 1.7.7 (+ expo, passkey, JWT plugins) | Clerk; WorkOS AuthKit (free ≤1M MAU) | SIWA required if offering social login (4.8); in-app deletion; WS JWT in first frame |
| Backend | Node 24 LTS (→26 LTS on 2026-10-28) + Fastify 5.12 + oRPC 1.15 | Hono 4.13; tRPC 11.19; NestJS 12; Bun 1.4 | Two-level sync protocol (user index + per-note docs) |
| WebSocket/sync server | Hocuspocus 4.7 (ws or uWebSockets.js via crossws) + Redis/Valkey extension | Custom on uWebSockets.js 20.71 (GitHub-only) | Multiplex docs on one socket; graceful drain |
| DB/ORM | PostgreSQL 18.6 + Drizzle 0.45 (→1.0 RC) + drizzle-kit migrations | Kysely 0.29; Prisma 7.10 (8 RC, GA ~Oct 2026) | Transaction poolers break LISTEN/NOTIFY; uuidv7; partition update log |
| Jobs | pg-boss 12.36 (default) + BullMQ 6.3 on Valkey (push fan-out) | Graphile Worker 0.18; Temporal 1.24; Inngest 4 / Trigger.dev 4 | BullMQ needs `noeviction`; scan `next_fire_at` instead of pre-creating jobs |
| Search | Client SQLite FTS5 + Postgres FTS/GIN + pg_trgm (ACL-scoped) | Meilisearch 1.54 (tenant tokens); Typesense v30 | Index plain-text projection incl. OCR/transcripts |
| Media | S3 + CloudFront, presigned POST; sharp 0.35 → AVIF/WebP; on-device OCR (Vision/ML Kit); expo-speech-recognition 57.1 | Cloudflare R2 (zero egress; no presigned POST); whisper.rn / server Whisper | Convert HEIC on device; strip EXIF/GPS |
| Observability/testing | OpenTelemetry + Sentry (node/browser 11.4, RN 8.29) + EAS Observe; Vitest 5.0; Playwright 1.63; Maestro 2.11 on EAS Workflows | Detox 20.51 | Trace context in WS frames; CRDT convergence property tests |
| Hosting | AWS ECS (Fargate/EC2) + ALB, Aurora PG 18, ElastiCache Valkey 9.1, S3/CloudFront | Fly.io / Render / Railway (early); Cloud Run (60-min WS cap) | Never serverless for WS; staggered drains |

### Items flagged UNVERIFIED

- Whether Hermes in current RN runs WebAssembly in production; this affects Loro/Automerge on RN.
- Google Play's exact eligibility rules for `USE_EXACT_ALARM` for a notes app with reminders (the policy page did not load).
- Whether react-native-sortables virtualizes items for very large grids.
- Typesense's latest release dates (the GitHub release page showed inconsistent dates).
- Neon post-acquisition pricing changes, and the status of standalone Crunchy Bridge after the Snowflake acquisition.
- Current hosted speech-to-text model names.
- EAS Update `fingerprint` policy details were confirmed only via a secondary source (the primary Expo page was not fetched).
- The ALB 4000 s idle-timeout figure comes from a secondary source.
- CSS grid-lanes browser status comes from secondary sources.

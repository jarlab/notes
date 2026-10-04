# Keep Clone: Architecture

The architecture for a production-grade Google Keep clone: a web PWA plus iOS and Android apps, written in TypeScript end to end. It is local-first, with CRDT sync, sharing, reminders, search and attachments, and it is designed to scale to 10M MAU.

**Status (2026-10-04):** the spine is at v1.3. The detail docs are drafts and are partly reconciled with it (see [Known gaps](#known-gaps)).

## How to read this

1. **[spine.md](spine.md): the single source of truth.** Every decision (D-xx), invariant (INV-xx), product rule (P-xx), cross-cutting rule (X-xx), scale trigger (T-xx), risk and milestone is defined there. If a detail doc disagrees with the spine, the spine wins.
2. **[detail/](detail/):** one design doc per subsystem (index below).
3. **[research/](research/):** verified research briefs with citations: the Keep feature inventory, sync technologies, the 2026 stack, and prior art with a capacity model.

## The architecture in one page

- **Clients.** A Vite SPA/PWA and Expo (React Native) apps share one TypeScript core:
  - packages: `domain`, `note-model`, `sync-client`, `storage`, `editor`, `authz`;
  - every action commits to local SQLite first: sqlite-wasm on OPFS in a Worker on web, expo-sqlite on mobile;
  - the grid, search (FTS5) and reminders read only local data.
- **Data split, following Keep's real semantics.**
  - **Shared note content** is one **Yjs doc per note**: title, rich body, checklist items and attachment references.
  - **Per-user state** is relational rows merged by per-field HLC last-writer-wins: color, pin, archive, grid order, labels and reminders. Keep keeps these per user even on shared notes.
- **One custom sync plane ("KSP")** over one WebSocket per device carries two things:
  - a per-user change feed (USN cursor);
  - per-note sequenced Yjs update logs.

  The server acks only after the Postgres commit, and authorization runs inside the write transaction. We chose this over PowerSync + Hocuspocus to avoid two sync planes, a vendor ceiling and duplicated ACLs.
- **Server.** One Node image runs three Fargate services (`api`, `sync` and `worker`, on Fastify + oRPC). It uses one Aurora PostgreSQL cluster, Valkey for pub/sub only, and S3 + CloudFront. Auth is Better Auth with passkeys, Sign in with Apple, Google and email OTP.
- **Scale is designed in, not deployed.** It is built in from day 1:
  - client-minted UUIDv7 IDs carry a 12-bit logical shard;
  - `shard_id` leads every key;
  - cross-shard fan-out always goes through an outbox relay.

  Second clusters, partitions, uWS/NLB, BullMQ and Aurora Global are switched on only at measured triggers (spine §3.1).
- **Reminders fire locally on every mobile device.** A server ledger and direct APNs/FCM/Web Push cover web and devices that aren't armed. One device rings, and an action on any device clears the others.
- **Correctness is a deliverable.** INV-1 to INV-18 are properties in a deterministic sync simulator that gates every sync change, plus real-Postgres isolation tests.

**Day-1 cost:** about $0.8k/month at beta and about $1.5k/month at public MVP. At planning scale, about $9–10k/month at 1M MAU and $45–50k/month at 10M MAU (spine §2.5).

## Roadmap (spine §8)

| Milestone | Scope |
|---|---|
| M0 Foundations | Monorepo, core packages, simulator skeleton, 7 de-risking spikes (mobile WebView editor, masonry at 5k cards, Safari OPFS, group commit, …) |
| M1 Private alpha | Single user, multiple devices: text and list notes, labels, color, pin, archive, trash, search, full sync |
| M2 Public MVP | Images, import/export, account deletion, restore/DR, store launch |
| M3 Collaboration and reminders | Sharing, live co-editing, reminders and push, abuse controls |
| M4 v1 GA | Link previews, OCR, filters, widgets and share targets, locales, Aurora Global |

**Effort, honestly:** about 9 months to MVP and about 14 months to GA for 2–3 engineers. A solo builder should plan for at least 2.5× that, follow the spine's "solo path", and use its cut list.

## Detail docs

| # | Doc | Owns |
|---|---|---|
| 01 | [domain-model](detail/01-domain-model.md) | IDs, HLC, ordering, limits, NoteDoc schema, Projector, recurrence |
| 02 | [sync-protocol](detail/02-sync-protocol.md) | KSP frames, ops, error codes, client state machine, simulator properties |
| 03 | [sync-server](detail/03-sync-server.md) | Gateway, group commit, compactor, fan-out relay |
| 04 | [client-core](detail/04-client-core.md) | sync-client, local schema, LiveQuery, CoreApi, platform services |
| 05 | [editor](detail/05-editor.md) | TipTap schema, DocPort, mobile EditorSheet, ChecklistController |
| 06 | [web-app](detail/06-web-app.md) | PWA, masonry grid, shortcuts, CSP, web performance budgets |
| 07 | [mobile-app](detail/07-mobile-app.md) | Expo app, native modules, widgets and share-extension formats, EAS |
| 08 | [sharing-and-authz](detail/08-sharing-and-authz.md) | `authz` package, invites, revocation, abuse caps |
| 09 | [reminders-and-push](detail/09-reminders-and-push.md) | Planner, coverage leases, ledger, PushSender, payloads |
| 10 | [media](detail/10-media.md) | Uploads, renditions, signed URLs, quotas, OCR, link unfurl |
| 11 | [search](detail/11-search.md) | Client FTS5, server Postgres FTS |
| 12 | [identity-and-devices](detail/12-identity-and-devices.md) | Sessions, JWTs, device identity, account deletion |
| 13 | [data-platform](detail/13-data-platform.md) | Full DDL, sharding and shard moves, backups, DR, restore journal |
| 14 | [infra-and-operations](detail/14-infra-and-operations.md) | CDK, CI/CD, observability, paging, runbooks, cost |
| 15 | [security-privacy-compliance](detail/15-security-privacy-compliance.md) | Threat model, GDPR/CCPA, App Store and Play compliance |
| 16 | [verification](detail/16-verification.md) | Simulator, isolation tests, E2E matrix, performance gates |

## How this was produced

1. Four research briefs.
2. Four competing end-to-end proposals, each optimizing one lens: fidelity, simplicity, scale or platform.
3. A four-judge panel, followed by synthesis into the spine.
4. Two adversarial reviews (spine v1.1).
5. Sixteen detail docs, whose authors' findings were folded back into spine v1.2 (91 changes) and v1.3 (consumer-doc findings, logged in spine §14).

## Known gaps

- **Reconciliation with spine v1.3 is incomplete.** It was interrupted by a usage limit.
  - Docs 01, 02, 03, 04, 05, 12 and 13 were mid-update when it stopped. They carry an "Aligned with spine v1.3" marker, but some sections may still describe v1.1 behavior.
  - Doc 08 has not been reconciled yet.
  - Docs 06, 07, 09, 10, 11, 14, 15 and 16 were written against v1.2 and have not yet been checked against v1.3 changes C-201 and later.
- Each doc's "Spine issues" and "Cross-doc issues" sections list its known open items.
- Open questions, including hands-on verification of unverified Keep behaviors, are in spine §10.

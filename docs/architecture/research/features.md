# Google Keep: verified feature inventory (as of 2026-10-04)

These are the requirements for a Keep clone. They cover web (keep.google.com), Android, iOS/iPadOS, Wear OS and the Chrome extension.

**Method.** I checked each claim against the Google Keep Help Center, the Google Workspace Updates blog, the official Keep REST API reference and blog.google, then against reputable news and APK-teardown reports from 2022 to 2026. I used the open-source reverse-engineered client [gkeepapi](https://github.com/kiwiz/gkeepapi) only as evidence of the wire and data shape. Citations are inline.

**Evidence labels**
- **[P]**: primary Google source (Help Center, Workspace Updates, API docs, blog.google, Chrome Web Store).
- **[S]**: secondary source (reputable press, or gkeepapi for data shape).
- **[T]**: APK teardown. Code exists but the feature may not have shipped.
- **UNVERIFIED**: I could not confirm it. It may be my own observation or an inference, and needs a hands-on check.

**Platform codes:** W = web, A = Android (phone and tablet), i = iOS/iPadOS, WO = Wear OS, CE = Chrome extension.

**Priority for the clone:** MVP (launch), v1 (first major release), later, or exclude.

---

## 0. Platform snapshot

| Surface | Status on 2026-10-04 | Evidence |
|---|---|---|
| Web (keep.google.com) | Main desktop client. **No offline access on computers.** Supports the two most recent versions of Chrome, Firefox, Edge and Safari. | [P] [help 10183819](https://support.google.com/keep/answer/10183819), [P] [help 9050190](https://support.google.com/keep/answer/9050190) |
| Android | Has the most features: rich text since 2023, sort (2025), Find in note (2026), widgets, AI features. Material 3 Expressive redesign shipped Aug 2025. | [S] [9to5 2025 recap](https://9to5google.com/2026/01/03/recap-google-keep-2025/) |
| iOS/iPadOS | Core features work. **Cannot display rich-text formatting.** Has a share extension. | [P] [help 2888246 (iOS)](https://support.google.com/keep/answer/2888246?hl=en&co=GENIE.Platform%3DiOS) |
| Wear OS | Create notes and lists, reminders, pin, archive, tiles and complications. | [P] [Wear OS help 7664632](https://support.google.com/wearos/answer/7664632) |
| Chrome extension | Actively maintained: v4.26391.540.1, updated 2026-10-01, about 7M users. | [P] [Chrome Web Store](https://chromewebstore.google.com/detail/google-keep-chrome-extens/lpcaedmchfhocbbapmcbpinfpgnhiddi) |
| Chrome *app* (packaged, offline) | Support ended in early 2021. | [P] [help 10183819](https://support.google.com/keep/answer/10183819) |
| Apple Watch | App removed in June 2025 (v2.2025.26200). | [S] [9to5google](https://9to5google.com/2025/06/30/google-keep-apple-watch/) |
| Public API | Keep REST API v1 (Workspace) offers notes create/get/list/delete and permission batch create/delete. | [P] [API reference](https://developers.google.com/workspace/keep/api/reference/rest/v1/notes) |

---

## 1. Note types and content

| Feature | Precise behavior, edge cases, limits | Platforms | Priority | Architectural implication |
|---|---|---|---|---|
| **Text note** | Optional title plus body. Title must be **< 1,000 chars**; body text **< 20,000 chars** [P] [API](https://developers.google.com/workspace/keep/api/reference/rest/v1/notes). Press reports put the practical cap at 19,999 chars and suggest moving to Docs [S] [cloudwards](https://www.cloudwards.net/google-keep-review/). | W A i WO CE | MVP | `Note.body` is a union of `text` and `list` ([API `Section`](https://developers.google.com/workspace/keep/api/reference/rest/v1/notes)). Enforce the limits on both server and client. |
| **Checklist (list) note** | Its own body type. See §3. | W A i WO | MVP | Each list item is its own entity (see §3). In gkeepapi, `NodeType` = Note, List, ListItem, Blob [S] [node.py](https://github.com/kiwiz/gkeepapi/blob/master/src/gkeepapi/node.py). |
| **Image note / images in a note** | Create a note from an image, or attach images to any note. Repeat to add more. Each image can be removed [P] [help 6395566](https://support.google.com/keep/answer/6395566). Images must be **< 10 MB and < 25 MP**. That limit comes from a secondary source and does not appear in the current help page [S] [NIE](https://learn.nie.edu.sg/etoolsNIE/info.aspx?id=38). On web, only local files can be uploaded [S]. **Max images per note: UNVERIFIED.** In the 2025 redesign, cards show images in a carousel [S] [9to5](https://9to5google.com/2025/08/21/google-keep-material-3-expressive-redesign/). On Android you can drag images out into other apps (2022) [S] [Chrome Unboxed](https://chromeunboxed.com/you-can-now-drag-images-out-of-keep-into-other-apps-on-android/). | W A i | MVP (attach) | Blob service with presigned uploads, size and megapixel checks, thumbnails and EXIF normalization. In gkeepapi, `NodeImage` has `width`, `height`, `byte_size`, `extracted_text` and `extraction_status`, which means OCR runs as an async server job [S] [node.py](https://github.com/kiwiz/gkeepapi/blob/master/src/gkeepapi/node.py). |
| **Drawing note / add drawing to a note** | Tools: Pen (thin), Marker (thick), Highlighter, Eraser (per shape), Select (shapes) and Grid (grid or rule lines). Each tool has its own color and size [P] [help 6395656](https://support.google.com/keep/answer/6395656). Secondary sources say 28 colors and 6 tip sizes [S] (2015-era; UNVERIFIED today). Handwriting in drawings can be searched [S]. | W A i | later | Store vector strokes (ink JSON) plus a rasterized snapshot for cards and for clients that cannot render ink. gkeepapi's `NodeDrawing.drawingInfo.snapshotData` and `extracted_text` show Google stores both [S]. A handwriting OCR job is needed. |
| **Draw on (annotate) an image** | Open the image, tap the pen icon and draw over it [P] [help 6395656](https://support.google.com/keep/answer/6395656). Whether the original image is preserved or flattened: **UNVERIFIED**. | W A i | later | Keep the original blob plus an annotation layer (non-destructive), or create a derived blob. Decide which up front. |
| **Audio note + transcription** | Recording works only on mobile. Web can play recordings but not record them [S] [LaptopMag](https://www.laptopmag.com/articles/how-to-use-google-keep). Keep transcribes while you speak and saves both the audio clip and the text, and the text can be edited [S]. **Max recording length: UNVERIFIED.** Android exposes "Audio" in the FAB menu, and the quick-capture widget has an audio button [S] [9to5](https://9to5google.com/2025/04/02/google-keep-text-notes/). | A i (playback on W) | later | Audio blob (gkeepapi `NodeAudio.length`). Speech-to-text runs on-device (Android SpeechRecognizer, iOS Speech) or on a server. The transcript is written into normal note text, not as linked metadata. |
| **Default creation type** | "Create text notes by default" (Android, Apr 2025): tapping the FAB opens a text note, and a long press shows Text/List/Drawing/Image/Audio [S] [9to5](https://9to5google.com/2025/04/02/google-keep-text-notes/). A larger FAB rolled out in June 2026 (v5.26.222) [S] [9to5](https://9to5google.com/2026/06/04/google-keep-large-fab/). | A | v1 | Client-only UX setting. |
| **Edited timestamp** | The note footer shows when it was last edited. **UNVERIFIED** (observed UI; not in any help page I fetched). The API exposes `createTime`, `updateTime` and `trashTime` [P]. | W A i | MVP | Store `createdAt`, `updatedAt` (content), `userEditedAt` and per-user `stateUpdatedAt` separately, so a per-user pin does not change the shared "edited" time. |

---

## 2. Rich text and editing

| Feature | Precise behavior, edge cases, limits | Platforms | Priority | Architectural implication |
|---|---|---|---|---|
| **Rich text formatting** | Options: **Bold, Italic, Underline, H1, H2, Normal text, Remove formatting** [P] [help 2888246 (Android)](https://support.google.com/keep/answer/2888246?hl=en&co=GENIE.Platform%3DAndroid). **Release dates:** Android new notes from **2023-08-24**, existing notes from **Oct 2023** [P] [Workspace Updates](https://workspaceupdates.googleblog.com/2023/08/google-keep-notes-android-rich-text-formatting.html). Web from **2025-05-09** [P] [Workspace Updates](https://workspaceupdates.googleblog.com/2025/05/release-notes-05-09-2025.html). **iOS still has none.** The current help page says "If you open a note with formatting on an iOS device, you won't be able to view this formatting" [P] [help (iOS)](https://support.google.com/keep/answer/2888246?hl=en&co=GENIE.Platform%3DiOS). Font sizes, strikethrough, bullet lists and inline links were not shipped: **UNVERIFIED** (2022 teardown hints at font sizes [S] [9to5](https://9to5google.com/2022/06/14/google-keep-text-formatting/)). Whether checklist items can be formatted: **UNVERIFIED**. The public API exposes `TextContent.text` as plain text only [P]. | A W (not i, not WO) | v1 (choose the storage format in MVP) | Pick a portable rich-text model now, such as marks or spans, or ProseMirror/Yjs JSON, with a plain-text projection for search, limits, the API and old clients. **Clients that can't render formatting must preserve it on save.** Keep's iOS gap shows the risk of lossy round-trips. |
| **Title** | Separate optional field, < 1,000 chars [P] [API](https://developers.google.com/workspace/keep/api/reference/rest/v1/notes). An AI "Generate title" button was found in Android v5.25.382 (Oct 2025) [T] [jetstream](https://jetstream.blog/?p=207489). | all | MVP | Separate field. Titles take part in search and conflict resolution independently of the body. |
| **Undo / Redo** | Undo and Redo buttons in the editor on Computer, Android and iOS [P] [help 2888246](https://support.google.com/keep/answer/2888246). Undo snackbars after archive or delete: **UNVERIFIED** (observed). | W A i | MVP | Client-local operation stack per editing session. Not synced. A CRDT text model makes local undo much easier. |
| **#hashtag → label** | Typing `#` in a note autocompletes existing labels and creates the label when picked (2016) [S] [TechCrunch](https://techcrunch.com/2016/04/20/google-keep-gets-support-for-labels-and-a-new-chrome-extension). | W A i | v1 | The editor needs a mention/autocomplete plugin backed by the user's label list. |
| **Find in note** | Search within one open note, with up/down navigation through matches. Rolling out on Android from about Aug 2026 (v5.26.331) [S] [SammyGuru](https://sammyguru.com/google-keep-rolls-out-a-much-needed-in-note-search-feature/), [S] [Android Authority](https://www.androidauthority.com/google-keep-find-in-note-3685188/). | A | later | Client only. |
| **Inline to-do line inside a text note** | Turns one line into a checkbox without converting the whole note. **Unreleased**, seen in v5.26.341 (Aug 2026) [T] [Android Authority](https://www.androidauthority.com/google-keep-to-do-lists-notes-apk-teardown-3703191/). | (A, T) | later | Argues for a block-based body (paragraph and checkbox blocks) rather than a strict text-or-list union. Consider building it in from the start. |
| **Note links / backlinks** | Link to other notes, with "Links to this note" and "Links from this note" panels. Seen in v5.25.402 (Oct 2025) [T] [jetstream](https://jetstream.blog/?p=207719). Shipping status: **UNVERIFIED**. | (A, T) | later | Needs a `NoteLink` edge table and an ACL check before showing a backlink's title (a linked note may not be shared with the viewer). |

---

## 3. Checklists

| Feature | Precise behavior, edge cases, limits | Platforms | Priority | Architectural implication |
|---|---|---|---|---|
| Create list / convert | "New list". Convert a note with "Show checkboxes" and back with "Hide checkboxes" [P] [help 6395451](https://support.google.com/keep/answer/6395451). Shortcut **Ctrl/⌘+Shift+8** toggles checkboxes [P] [shortcuts](https://support.google.com/keep/answer/12862970). | W A i WO | MVP | Converting must keep text losslessly: lines become items and items become lines. Indented children need flattening rules. |
| **Indent / nesting** | **Only one level of nesting** [P] [API `ListItem.childListItems`](https://developers.google.com/workspace/keep/api/reference/rest/v1/notes). Drag an item right to indent; **the first item cannot be indented** [P] [help 6395451](https://support.google.com/keep/answer/6395451). Shortcut **Ctrl/⌘+]** and **[** [P] [shortcuts](https://support.google.com/keep/answer/12862970). On mobile, swipe or drag right [S]. Checking a parent checks all its children and moves them to Completed [S] [Android Central](https://www.androidcentral.com/how-google-keep-sublist). | W A i | MVP | The server enforces depth ≤ 1 by storing a `parentItemId` that may only point at a top-level item (gkeepapi `superListItemId` [S]). Checking a parent is one operation that cascades. |
| Item order / reorder | Drag to reorder [P] [help 6395451](https://support.google.com/keep/answer/6395451). Keyboard: **n/p** to move between items, **Shift+n/p** to move an item [P] [shortcuts](https://support.google.com/keep/answer/12862970). | W A i | MVP | Fractional index (`sortValue` in gkeepapi [S]) per item. Order is **shared** content. |
| **Add new items to top/bottom** | Setting under "List behavior" [P] [help 6358550](https://support.google.com/keep/answer/6358550). Also on Wear OS [P] [Wear help](https://support.google.com/wearos/answer/7664632). | W A i WO | MVP | Account-level preference applied when an item is inserted. gkeepapi also has a per-node `newListItemPlacement` (Top/Bottom) [S]. |
| **Move checked items to bottom** | Setting under "List behavior" [P] [help 6358550](https://support.google.com/keep/answer/6358550). When on, checked items go to a collapsible checked/"Completed" section. When off, they stay in place, struck through (UNVERIFIED wording). gkeepapi has `CheckedListItemsPolicy` = Default or Graveyard, and `GraveyardState` = Expanded or Collapsed [S] [node.py](https://github.com/kiwiz/gkeepapi/blob/master/src/gkeepapi/node.py). Wear OS: "Hide checked items" [P]. | W A i WO | MVP | Make it a display projection over a stable shared order. Do **not** physically reorder shared items when one user checks something, or collaborators with different settings will fight over the order. Store the collapsed/expanded state per user and per device. |
| Uncheck all / Delete checked items | Wear OS: "Reload" unchecks all items and restores the original list [P] [Wear help](https://support.google.com/wearos/answer/7664632). On W/A/i the menu items "Uncheck all items" and "Delete checked items" are **UNVERIFIED** (observed, not in a fetched help page). | WO (+W A i UNVERIFIED) | v1 | Batch operation over items. Bulk delete of items must sync as one op. |
| Limits | **< 1,000 items** per list. Item text **< 1,000 chars** [P] [API](https://developers.google.com/workspace/keep/api/reference/rest/v1/notes). | — | MVP | Validate on the server. |
| Grocery autocomplete | Grocery item suggestions (2016) [S] [GSMArena](https://www.gsmarena.com/new_google_keep_update_brings_link_previews_grocery_auto_complete-blog-18570.php). gkeepapi has `SuggestValue.GroceryItem` [S]. | A (UNVERIFIED elsewhere) | later | Suggestion dictionary or ML. |
| Check off from widget | The single-note widget toggles checkboxes without opening the app [P] [Workspace Updates](https://workspaceupdates.googleblog.com/2023/03/google-keep-notes-available-on-home-screen-android-devices.html). | A | v1 | The widget process needs write access to the local DB and the sync queue. |

---

## 4. Organization: color, background, pin, archive, trash, labels

| Feature | Precise behavior, edge cases, limits | Platforms | Priority | Architectural implication |
|---|---|---|---|---|
| **Color** | "Change color" picks from a color or background gallery and applies to one or many notes [P] [help 6191044](https://support.google.com/keep/answer/6191044). Enum: Default/White, Red, Orange, Yellow, Green, Teal, Blue, DarkBlue, Purple, Pink, Brown, Gray (12 values) [S] [gkeepapi](https://github.com/kiwiz/gkeepapi/blob/master/src/gkeepapi/node.py). Colors adapt to dark mode. **Per-user on shared notes** [P] [help 6101196](https://support.google.com/keep/answer/6101196). | W A i WO(view) | MVP | Store a semantic color token (not hex) in **per-user** note state. Clients map tokens to light and dark palettes. |
| **Background image/theme** | Illustrated backgrounds launched July 2021, 9 designs that adapt to light and dark. At launch they were on Android and iOS only and did not show on web [S] [9to5](https://9to5google.com/2021/07/28/google-keep-background-notes-image/). Web now has a "background gallery" [P] [help 6191044](https://support.google.com/keep/answer/6191044). Per-user scope is **UNVERIFIED** (inferred because it lives in the same picker as color). | W A i | later | Background token in per-user state. Assets ship with the client in light and dark variants. |
| **Pin** | Pinned notes go to a "Pinned" section above "Others" [P] [help 6191044](https://support.google.com/keep/answer/6191044). Shortcut **f** [P] [shortcuts](https://support.google.com/keep/answer/12862970). Wear OS supports pin [P]. Per-user scope is **UNVERIFIED but strongly inferred** (see §Shared vs per-user). Pinning an archived note moves it back to Notes: **UNVERIFIED**. | W A i WO | MVP | `pinned` flag plus its own fractional sort key in per-user state. The pinned section has its own order. |
| **Archive** | Hides a note from the main view. Archive view in the nav drawer. Unarchive from the note [P] [help 6262765](https://support.google.com/keep/answer/6262765). Shortcut **e**. Bulk archive by checking several notes. **Per-user** on shared notes [P] [help 6101196](https://support.google.com/keep/answer/6101196). Archiving a note with a reminder has no effect on the reminder/task [P] [help 3187168](https://support.google.com/keep/answer/3187168). Archived notes still appear in search: **UNVERIFIED**. | W A i WO | MVP | `archived` flag in per-user state. The main feed query is `!archived && !trashed`. |
| **Trash (delete)** | Delete sends the note to Trash. **You have 7 days to recover it**, then it is permanently deleted [P] [help 6262765](https://support.google.com/keep/answer/6262765). Restore through More → Restore. **Empty Trash** permanently deletes everything in Trash. Bulk delete via More → Delete notes. **"Deleted notes are also deleted for anyone you've shared them with"** and "If you delete a shared note that you own, it'll be deleted for everyone" [P] [help 6101196](https://support.google.com/keep/answer/6101196). Shortcut **#**. API: "If trashed, the note is eventually deleted" [P] [API](https://developers.google.com/workspace/keep/api/reference/rest/v1/notes). API `notes.delete` needs OWNER, is immediate and irreversible, and "collaborators will lose access" [P] [API delete](https://developers.google.com/workspace/keep/api/reference/rest/v1/notes/delete). Deleting one note forever from Trash, and notes in Trash being read-only: **UNVERIFIED** (observed). I found no admin or Vault recovery for Keep beyond the 7-day window (**UNVERIFIED**). | W A i (WO UNVERIFIED) | MVP | Soft delete with `trashedAt` on **shared** content when the owner deletes. A **server cron purges after 7 days** with cascading blob deletion. Empty Trash is a hard-delete job. Tombstones must reach every member's change feed so devices purge their local copies. |
| **Labels** | **Up to 50 labels**. Create (type a name, then Create), rename and delete via Menu → Edit labels. One note can have many labels, and labels can be applied to many notes at once [P] [help 6191044](https://support.google.com/keep/answer/6191044). Each label gets a view in the nav drawer. Labels can be added from the Chrome extension [P] [CWS](https://chromewebstore.google.com/detail/google-keep-chrome-extens/lpcaedmchfhocbbapmcbpinfpgnhiddi). **Labels are personal and are not shared with collaborators** [P] [help 6101196](https://support.google.com/keep/answer/6101196). Labels are flat (not nested); max label name length is **UNVERIFIED**. gkeepapi `Label` fields: `mainId`, `name`, `timestamps`, `lastMerged`. Labels sync in a per-account `userInfo.labels` block, not on notes [S] [gkeepapi `__init__`](https://github.com/kiwiz/gkeepapi/blob/master/src/gkeepapi/__init__.py). | W A i WO CE | MVP | `Label` belongs to a user (cap 50, unique name per user). The note-label join table is **(userId, noteId, labelId)**. Rename is an id-stable metadata change. Deleting a label removes the join rows only. The `lastMerged` field suggests Google merges duplicate labels created offline; the clone needs dedupe-by-name on sync. |

---

## 5. Reminders

**Important for 2025-2026:** Keep's own reminder engine has been replaced by Google Tasks. Location reminders no longer exist.

| Feature | Precise behavior, edge cases, limits | Platforms | Priority | Architectural implication |
|---|---|---|---|---|
| Date/time reminder | Bell icon on a note. Presets **Later today / Tomorrow morning / Tomorrow evening / Next week**, plus "Pick date & time" [S] [9to5 Nov 2025](https://9to5google.com/2025/11/28/google-keep-tasks-rollout/). Users can change the preset times for morning, afternoon and evening [P] [help 3187168](https://support.google.com/keep/answer/3187168). | W A i WO | v1 | Store reminders in the user's timezone with an explicit `tz`. Preset times are user settings. |
| Recurring | Does not repeat / Daily / Weekly / Monthly / Yearly / Custom, where Custom can end on a date (added 2015) [S] [Droid Life](https://www.droid-life.com/2015/03/25/google-keep-gets-labels-and-recurring-reminders/). Since the Tasks migration the **maximum interval is 1,000 days**: anything longer (e.g. 2,000 days) is clamped to 1,000 [P] [Tasks help 16540694](https://support.google.com/tasks/answer/16540694). | W A i | v1 | Store an RRULE subset with a validator. The scheduler computes the next occurrence after each fire or completion. |
| **Location reminders** | **Removed.** "You can no longer create or get location-based reminders." Old location text is copied into the task description but is not an active geofence [P] [Tasks help 16540694](https://support.google.com/tasks/answer/16540694). Home, Work and Pick a place are gone [S] [9to5 Nov 2025](https://9to5google.com/2025/11/28/google-keep-tasks-rollout/). | none now | exclude (or later as a differentiator) | If built, it is client-side geofencing (Android GeofencingClient, iOS CLMonitor) with OS permission flows. No server job. |
| **Saved to Google Tasks** | From **2025-10-13** (extended rollout), "Keep reminders will be automatically saved to Tasks". Existing date/time reminders were migrated [P] [Workspace Updates](https://workspaceupdates.googleblog.com/2025/10/google-keep-reminders-now-saved-to-tasks.html). The prompt reads "Reminders are now Google Tasks" [S] [9to5](https://9to5google.com/2025/11/28/google-keep-tasks-rollout/). Reminders can be viewed and edited in Keep, Calendar, Tasks and Gemini. Changing a Keep note's title does **not** update the reminder title [P] [help 3187168](https://support.google.com/keep/answer/3187168). Deleting the reminder in Calendar or Tasks leaves the note in place. Non-repeating reminders more than a year old moved to an "Old Google Keep Reminders" list [P] [Tasks help](https://support.google.com/tasks/answer/16540694). | W A i | — | Shows the right separation: **the reminder is its own entity that links to a note**, not a field on the note. Even before the migration Keep used a separate `reminders/v1internal` API [S] [gkeepapi](https://github.com/kiwiz/gkeepapi/blob/master/src/gkeepapi/__init__.py). |
| Notifications / surfacing across devices | "You no longer get reminder notifications from Google Keep". The Calendar or Tasks app delivers them [P] [Tasks help](https://support.google.com/tasks/answer/16540694). Notifications fire at the set time **and again 24 hours later** [P] [help 3187168](https://support.google.com/keep/answer/3187168). | (via Tasks/Calendar) | v1 | The clone needs its own **scheduler service**: a durable delayed-job queue that fans out to FCM, APNs and Web Push, with device dedupe. It should also schedule local notifications on mobile for offline reliability. |
| Snooze | Notification snooze is now handled by the Tasks/Calendar notification. Keep's historical snooze UI is **UNVERIFIED**. | — | v1 | A snooze is a one-off override of the next fire time and does not change the RRULE. |
| Reminders view | "Reminders" stays as a view in Keep's nav drawer [S] [9to5 Nov 2025](https://9to5google.com/2025/11/28/google-keep-tasks-rollout/). Separation into Upcoming and Sent: **UNVERIFIED**. | W A i | v1 | Index on (userId, nextFireAt). |
| Reminders on shared notes | **Per-user.** Collaborators "add reminders without changing the note for others" [P] [help 6101196](https://support.google.com/keep/answer/6101196). | — | v1 | Key reminders on (userId, noteId). |
| Wear OS | Schedule new reminders and change existing reminder times [P] [Wear help](https://support.google.com/wearos/answer/7664632). | WO | later | — |

---

## 6. Collaboration and sharing

| Feature | Precise behavior, edge cases, limits | Platforms | Priority | Architectural implication |
|---|---|---|---|---|
| Invite collaborators | From the note's Collaborator action, by name or email, **including Google Groups** [P] [help 6101196 (Android)](https://support.google.com/keep/answer/6101196?hl=en&co=GENIE.Platform%3DAndroid). The API can grant to users, groups and families [P] [API guide](https://developers.google.com/workspace/keep/api/guides/modify-permissions). | W A i | v1 (data model in MVP) | `NoteMember(noteId, principalType{user,group,family,pendingEmail}, principalId, role)`. Pending invites must resolve when the invitee signs up. |
| Roles | Only **OWNER** and **WRITER** exist. OWNER: "full access. This role cannot be added or removed". WRITER: "contribute content and modify note permissions" [P] [API](https://developers.google.com/workspace/keep/api/reference/rest/v1/notes). Collaborators can only be given WRITER [P] [API guide](https://developers.google.com/workspace/keep/api/guides/modify-permissions). **No viewer or commenter role and no link sharing.** For read-only sharing, send a copy through another app [P] [help 6101196](https://support.google.com/keep/answer/6101196). | — | v1 | Two-role ACL. Writers may add or remove other writers but never the owner (API). Ownership transfer does not exist. |
| What collaborators can do | Edit "text, lists, images, drawings, and audio recordings" [P] [help 6101196](https://support.google.com/keep/answer/6101196). Real time: "create and edit notes collaboratively in real-time" [P] [Workspace product page](https://workspace.google.com/products/keep/). | W A i | v1 | Live push channel, plus a merge strategy for text and list items (see Top 15). |
| Family group | Share with the whole family group at once. Shows a family icon. **"Anyone in the family can edit or delete notes you share with the family group."** Adding non-family members reveals your family manager's name [P] [help 7313121](https://support.google.com/keep/answer/7313121). | W A i | exclude (Google-specific) | If replaced with a "household/group" principal, decide whether group members may delete. Google says yes. |
| Remove collaborator / unshare | Click **Remove** next to a collaborator [P] [help 6101196](https://support.google.com/keep/answer/6101196). The removed user loses the note ("The owner may have stopped sharing the note") [P] [help 6102239](https://support.google.com/keep/answer/6102239). Whether the removed user keeps a copy: none is kept (**UNVERIFIED**, inferred). | W A i | v1 | Revoking access must emit a tombstone into that user's change feed so their devices purge the content and its blob cache. Their per-user state (labels, color, reminder) for that note is garbage-collected. |
| Owner deletes shared note | Deleted for everyone [P] [help 6101196](https://support.google.com/keep/answer/6101196), [P] [help 6262765](https://support.google.com/keep/answer/6262765). | — | v1 | Shared trash state. Collaborators can't restore (UNVERIFIED). |
| Collaborator deletes shared note | **UNVERIFIED.** No primary source. The family-group help says family members can delete. Leaving a note voluntarily ("remove me") is undocumented. | — | v1 | **Recommendation:** treat a non-owner delete as "leave note" (remove own membership), and only an owner delete trashes for all. Flag for product decision. |
| Sharing on/off setting | "Enable sharing": "If you turn sharing off, future notes can't be shared with you. You can still share notes with others" [P] [help 6358550](https://support.google.com/keep/answer/6358550). | W A i | v1 | Account flag checked during invite. |
| Max collaborators per note | **UNVERIFIED.** No official number found. Press claims "no limit" [S], which is unreliable. | — | v1 | Pick a cap, e.g. 50, to bound feed fan-out. |
| Collaborator indicator | Shared notes show collaborator avatars or an icon, including on the widget [P] [Workspace Updates](https://workspaceupdates.googleblog.com/2023/03/google-keep-notes-available-on-home-screen-android-devices.html). | W A i | v1 | Member list (with avatars) in the note payload. |
| People filter | Search filter for "notes that you've shared with specific people" [P] [help 2888263](https://support.google.com/keep/answer/2888263). | W A i | v1 | Index notes by member principal. |

---

## 7. Search, filters, views and ordering

| Feature | Precise behavior, edge cases, limits | Platforms | Priority | Architectural implication |
|---|---|---|---|---|
| Full-text search | Search box at the top. Shortcut **/** [P] [shortcuts](https://support.google.com/keep/answer/12862970). Matches text in images (OCR) and handwriting [S] [Google blog: 8 tips](https://blog.google/products-and-platforms/products/workspace/8-tips-help-you-keep-google-keep/). Whether Archive and Trash are included: **UNVERIFIED**. | W A i | MVP (text), later (OCR) | On mobile, a local FTS index (SQLite FTS5) gives instant offline search. A server index (Postgres FTS or OpenSearch) serves web. OCR and transcript text are indexed fields. |
| Filters | **Types** (reminders, lists, images, drawings, URLs, recordings), **Labels**, **Things** (categories such as books, music, travel), **People**, **Colors** [P] [help 2888263](https://support.google.com/keep/answer/2888263). Things categories: Books, Food, Movies, Music, Places, Quotes, Travel, TV [S] [gkeepapi `CategoryValue`](https://github.com/kiwiz/gkeepapi/blob/master/src/gkeepapi/node.py). | W A i | v1 (Types, Labels, Colors, People); later (Things) | Derived facets: has-image, has-audio, has-drawing, has-url, has-reminder, is-list. "Things" needs an ML classification job that writes `annotations.category`. |
| Grid vs list view | Toggle with **Ctrl/⌘+G** on web [P] [shortcuts](https://support.google.com/keep/answer/12862970). Android has single or multi-column views (UNVERIFIED wording). | W A i | MVP | Per-device UI preference. Masonry layout on web. |
| Manual order (drag) | Drag notes. **Shift+J/K** moves a note to the next or previous position [P] [shortcuts](https://support.google.com/keep/answer/12862970). | W A i | MVP | Per-user fractional sort key. Per-user scope is **UNVERIFIED but inferred**. |
| Sort (Android) | **Custom** (default), **Date created**, **Date modified**. When sorted by date, an indicator shows in the search field and the custom order is kept. Android only (rolled out ~Aug 2025, v5.25.312). Not on iOS or web [S] [9to5](https://9to5google.com/2025/08/13/google-keep-sort-notes-android/). | A | v1 | Query-time sort over `createdAt` and `updatedAt`. Custom order is retained independently. |
| Pinned section ordering | Pinned notes form their own section on top. Reordering within it: drag (UNVERIFIED detail). | W A i | MVP | Sort key per (user, pinned bucket). |
| Navigation views | Notes, Reminders, each label, Edit labels, Archive, Trash [P] [help 6262765](https://support.google.com/keep/answer/6262765), [help 6191044](https://support.google.com/keep/answer/6191044). | W A i | MVP | Each view is a query. Keep counts and pagination cheap. |
| Keyboard navigation | **j/k** move to the next or previous note, **Enter** opens [P] [shortcuts](https://support.google.com/keep/answer/12862970). | W | v1 | — |

---

## 8. Multi-select, copy, send and export

| Feature | Precise behavior, edge cases, limits | Platforms | Priority | Architectural implication |
|---|---|---|---|---|
| Multi-select / bulk actions | Hover and check several notes, then Archive, or More → Delete notes [P] [help 6262765](https://support.google.com/keep/answer/6262765). Color or labels for several notes [P] [help 6191044](https://support.google.com/keep/answer/6191044). Shortcuts **x** (select) and **Ctrl/⌘+A** (select all) [P] [shortcuts](https://support.google.com/keep/answer/12862970). Bulk pin, remind and copy: **UNVERIFIED**. | W A i | MVP | Batch mutation endpoint with idempotency keys. Applied atomically per note and partially tolerated across notes. |
| Make a copy | In the note's ⋮ menu [S] [Dummies](https://www.dummies.com/article/sharing-notes-with-g-suites-keep-app-272760). Whether the copy keeps labels or collaborators: **UNVERIFIED**. The copy is presumably private and owned by the copier. | W A i | v1 | Server-side deep clone of content and blobs (copy-on-write blob refs). New owner. Per-user state optional. |
| Copy to Google Docs | More → "Copy to Google Docs", then "Open Doc" [P] [help 6320648](https://support.google.com/keep/answer/6320648). iOS got it in 2015 [S] [PCWorld](https://www.pcworld.com/article/424301/google-keep-for-ios-gets-closer-to-evernote-with-ease-of-use-features.html). | W A i | exclude / replace | Replace with export to Markdown, DOCX or PDF, or an "open in…" integration. |
| Send to other apps | Android share intent and iOS share sheet. This is the documented way to share without edit rights [P] [help 6101196](https://support.google.com/keep/answer/6101196), [help 6320648](https://support.google.com/keep/answer/6320648). | A i | v1 | Serialize to plain text or Markdown plus attachments. |
| Takeout export | Google Takeout zip with an HTML and a JSON file per note, plus attachments. Includes color, `isPinned`, `isArchived`, `isTrashed`, labels and collaborators [P] [help 10017039](https://support.google.com/keep/answer/10017039), [S] [How-To Geek](https://www.howtogeek.com/694042/how-to-export-your-google-keep-notes-and-attachments/). | W | v1 (own export); **v1 import from Takeout** | Own async export job, which is also needed for GDPR. **A Takeout importer is the main migration path into the clone.** |

---

## 9. Media intelligence

| Feature | Precise behavior, edge cases, limits | Platforms | Priority | Architectural implication |
|---|---|---|---|---|
| Grab image text (OCR) | Image → ⋮ → "Grab image text" inserts the recognized text into the note [S] [Google blog: 8 tips](https://blog.google/products-and-platforms/products/workspace/8-tips-help-you-keep-google-keep/). Confirmed on web and Android. iOS: **UNVERIFIED**. | W A (i?) | later | OCR job (ML Kit or Apple Vision on-device, or Tesseract or a cloud service on the server). Store `extracted_text` and `extraction_status` per blob (gkeepapi [S]). One button copies the result into the body. |
| Searchable image and handwriting text | See §7 [S]. | W A i | later | Same OCR output feeds the search index. |
| Link previews | A pasted URL produces a preview card (thumbnail, title, snippet) under the note. Cards can be removed individually. Setting "Display rich link previews" [P] [help 6358550](https://support.google.com/keep/answer/6358550), [S] [GSMArena 2016](https://www.gsmarena.com/new_google_keep_update_brings_link_previews_grocery_auto_complete-blog-18570.php). | W A i | v1 | Server unfurl job with **SSRF protection**, caching and a timeout. Previews are stored as note annotations (gkeepapi `WebLink` annotation [S]). The "removed" state of a preview lives on the note. |
| "Things" auto-categorization | See §7. Topics are created automatically [S] [9to5mac](https://9to5mac.com/guides/google-keep/). | W A i | later / exclude | Async ML classifier writing category annotations. |
| Voice transcription | See §1. | A i | later | — |

---

## 10. History, conflicts and sync behavior

| Feature | Precise behavior, edge cases, limits | Platforms | Priority | Architectural implication |
|---|---|---|---|---|
| Version history | More → Version history → choose a version → **Download** as a text file. **Text changes only; images are not included** [P] [help 13820887](https://support.google.com/keep/answer/13820887). Web only at launch (Aug 2023). No in-app restore [S] [9to5](https://9to5google.com/2023/08/17/google-keep-version-history/). | W | later | Append-only text snapshots per note, throttled (e.g. one per N minutes of editing). Shared across collaborators (inferred). Retention policy needed. |
| Conflicting edits | Editing on two devices while one is offline (or on a weak connection) can raise **"Conflicting edits found"**. The user picks **"Keep selected"** or **"Keep both"** [P] [help 6102239](https://support.google.com/keep/answer/6102239). Dates back to 2014 [S] [Android Police](https://www.androidpolice.com/2014/04/04/neat-as-of-the-latest-update-google-keep-will-let-you-resolve-conflicting-edits/). | A (web noted [S]) | MVP (some strategy) | Keep uses **optimistic concurrency with a per-node `baseVersion`** [S] [gkeepapi](https://github.com/kiwiz/gkeepapi/blob/master/src/gkeepapi/node.py) and surfaces body conflicts to the user. The clone can match this (fork a conflict copy), or do better with a text CRDT. |
| Sync protocol (as observed) | Single `POST notes/v1/changes`. The client sends `targetVersion` plus dirty `nodes`. The server returns `toVersion`, changed `nodes`, `userInfo.labels`, a `truncated` flag for paging, and `forceFullResync` [S] [gkeepapi `__init__`](https://github.com/kiwiz/gkeepapi/blob/master/src/gkeepapi/__init__.py). The client advertises capabilities (color, pinning, labels, annotations, sharing, drawings, trash, indentation…) [S]. | all | MVP | Per-user monotonic change cursor, delta pull with paging, full-resync escape hatch, and **capability negotiation** so older clients (e.g. ones without rich text) degrade without losing data. |
| Reload / resync | "Reload" re-uploads local notes and then re-downloads [P] [help 6102239](https://support.google.com/keep/answer/6102239). | A | v1 | Client "reset local store" path that first flushes the outbox. |

---

## 11. Settings, theming, accessibility, keyboard

| Feature | Precise behavior, edge cases, limits | Platforms | Priority | Architectural implication |
|---|---|---|---|---|
| Settings | Add new items to bottom; Move checked items to bottom; Enable sharing; Display rich link previews; Enable Dark Mode [P] [help 6358550](https://support.google.com/keep/answer/6358550). Reminder default times [P] [help 3187168](https://support.google.com/keep/answer/3187168). Create text notes by default (Android) [S]. | W A i | MVP/v1 | `UserSettings` synced per account. Device-only preferences (grid/list, sort) stay local. |
| Dark mode | Android "Enable dark theme" since May 2019, dark gray rather than true black [S] [9to5](https://9to5google.com/2019/05/17/google-keep-dark-mode-rolling-out/). Web "Enable Dark Mode" [P] [help 6358550](https://support.google.com/keep/answer/6358550). iOS date: **UNVERIFIED**. | W A i | MVP | Theme tokens. Note colors need dark variants. |
| Material 3 Expressive | Android and Wear OS redesign in Aug 2025: new search bar, containers, image carousel, Wear tiles [S] [9to5](https://9to5google.com/2025/08/21/google-keep-material-3-expressive-redesign/). | A WO | — | Visual only. |
| Screen reader support | Documented screen-reader flows, e.g. "Checkboxes shown/hidden" announcements and the Ctrl+/ shortcut dialog [P] [help 16914649](https://support.google.com/keep/answer/16914649). | W | v1 | ARIA live regions; roving focus in the masonry grid. |
| Keyboard shortcuts (web) | Full list below [P] [help 12862970](https://support.google.com/keep/answer/12862970). | W | v1 | Global key handler that is suppressed inside editable fields (the help says shortcuts work only when focus is outside the edit field [P] [help 16914649](https://support.google.com/keep/answer/16914649)). |

**Keyboard shortcuts** ([P] [help 12862970](https://support.google.com/keep/answer/12862970); use ⌘ for Ctrl on Mac)

| Group | Action | Keys |
|---|---|---|
| Navigation | Next / previous note | `j` / `k` |
| Navigation | Move note to next / previous position | `Shift+j` / `Shift+k` |
| Navigation | Next / previous list item | `n` / `p` |
| Navigation | Move list item to next / previous position | `Shift+n` / `Shift+p` |
| Application | Compose new note | `c` |
| Application | Compose new list | `l` |
| Application | Search notes | `/` |
| Application | Select all notes | `Ctrl+a` |
| Application | Open keyboard shortcut help | `?` (screen-reader page also lists `Ctrl+/` [P] [help 16914649](https://support.google.com/keep/answer/16914649)) |
| Application | Send feedback | `@` |
| Actions | Archive note | `e` |
| Actions | Trash note | `#` |
| Actions | Pin / unpin | `f` |
| Actions | Select note | `x` |
| Actions | Open note | `Enter` |
| Actions | Toggle list / grid view | `Ctrl+g` |
| Editor | Finish editing | `Esc` or `Ctrl+Enter` |
| Editor | Toggle checkboxes | `Ctrl+Shift+8` |
| Editor | Indent / dedent list item | `Ctrl+]` / `Ctrl+[` |
| Editor (a11y page) | Jump to title from start of note | `Shift+Tab` [P] [help 16914649](https://support.google.com/keep/answer/16914649) |

---

## 12. Platform surfaces and OS integration

| Feature | Precise behavior, edge cases, limits | Platforms | Priority | Architectural implication |
|---|---|---|---|---|
| Android widgets | **Quick capture** widget (redesigned Apr 2025): text, list, audio, drawing and camera buttons, in 5×1, 4×1 and 3×1 sizes [S] [9to5](https://9to5google.com/2025/04/10/google-keep-widget-redesigned-wide/). It is also available on the lock screen [S] [9to5 recap](https://9to5google.com/2026/01/03/recap-google-keep-2025/). **Note collection** widget. **Single note** widget (Mar 2023): pin a note or list to the home screen, toggle checkboxes in place, shows colors, reminders and a collaborator icon [P] [Workspace Updates](https://workspaceupdates.googleblog.com/2023/03/google-keep-notes-available-on-home-screen-android-devices.html). | A | v1 | Jetpack Glance widgets read the local DB through a ContentProvider or shared Room DB. Writes go through the same outbox. Widgets update when sync completes. |
| iOS widgets | Today-view widget for new note, list, photo or voice, plus recent notes (2015-16) [S] [PCWorld](https://www.pcworld.com/article/424301/google-keep-for-ios-gets-closer-to-evernote-with-ease-of-use-features.html). Whether a modern WidgetKit home or lock-screen widget exists: **UNVERIFIED**. | i | v1 | App Group shared container plus WidgetKit timeline and App Intents for interactive checkboxes (iOS 17+). |
| Share target (into Keep) | iOS share extension for web pages, photos and other content [S] [PCWorld](https://www.pcworld.com/article/424301/google-keep-for-ios-gets-closer-to-evernote-with-ease-of-use-features.html). Android receives shares through the share sheet (intent; long-standing, help-page citation **UNVERIFIED**). | A i | v1 | Android `ACTION_SEND`/`SEND_MULTIPLE` intent filters; iOS Share Extension with an App Group for queued writes made before the main app runs. |
| Android Notes role / lock screen / stylus | Keep supports Android 14's Notes role (launch from the lock screen, stylus button) behind a flag. Shown as "coming soon" in Dec 2023 [S] [9to5](https://9to5google.com/2023/12/14/google-keep-android-14-note-app-lock-screen/). "Content capture for notes" (screenshot into Keep) found Oct 2025 [T] [jetstream](https://jetstream.blog/?p=207489). Current shipping status: **UNVERIFIED**. | A | later | `android.app.role.NOTES` plus `ACTION_CREATE_NOTE`. Lock-screen mode must hide existing notes. |
| Wear OS app | Create notes or lists by **voice, keyboard, handwriting or emoji**. Check and uncheck items, hide checked items, Reload (uncheck all), new-item placement setting, schedule or change reminders, pin, archive. **Tiles** ("Create note", "Single note") and **complications** ("Add note", "Add list") [P] [Wear help 7664632](https://support.google.com/wearos/answer/7664632), [S] [9to5 Aug 2025](https://9to5google.com/2025/08/21/google-keep-material-3-expressive-redesign/). Color, labels and delete on the watch: **UNVERIFIED**. | WO | later | Standalone Wear app that talks to the backend directly over Wi-Fi or LTE, or relays through the phone over the Data Layer. Small local cache. Tiles API and ComplicationDataSourceService. |
| Chrome extension | One-click save of the page URL, selected text or an image, with a note and labels added in the popup. Syncs to all platforms [P] [Chrome Web Store](https://chromewebstore.google.com/detail/google-keep-chrome-extens/lpcaedmchfhocbbapmcbpinfpgnhiddi). Notes linked to the source site (2016) [S] [TechCrunch](https://techcrunch.com/2016/04/20/google-keep-gets-support-for-labels-and-a-new-chrome-extension). | CE | later | MV3 extension with context menus, OAuth via `chrome.identity`, a "create note" API, and a `sourceUrl` field on the note. |
| Workspace side panels | Keep in the side panel of Docs, Sheets, Slides, Drawings, Calendar and Gmail. "Save to Keep notepad" from Docs [P] [Workspace Updates 2018](https://workspaceupdates.googleblog.com/2018/08/use-quick-access-side-panel-to-do-more.html). | W | exclude | — |
| Google Messages → Keep | Shortcut to save messages to Keep, seen in Aug 2026 Messages teardown: **UNVERIFIED** [T] (reported by jetstream.blog). | A | exclude | — |

---

## 13. Gemini / AI features

| Feature | Precise behavior, edge cases, limits | Platforms | Priority | Architectural implication |
|---|---|---|---|---|
| "Help me create a list" | Prompt → Gemini generates checklist items to insert. Tested in Workspace Labs (Feb 2024), then broad Android rollout in 2024. Free on Pixel; other devices needed AI Premium or Labs [S] [Android Central](https://androidcentral.com/apps-software/how-to-use-google-keep-to-help-me-create-a-list), [S] [9to5google](https://9to5google.com/?p=637266). | A | later | LLM gateway with structured output (JSON list items), quotas and plan gating. |
| Gemini app Keep extension | From the Gemini app: create notes and lists, add to notes, add or remove list items. Announced at I/O 2024, launched with Pixel 9, broader from Oct 2024 [S] [9to5](https://9to5google.com/2024/09/17/google-tasks-keep-gemini-extensions/), [P] [Workspace Updates](https://workspaceupdates.googleblog.com/2024/10/gemini-app-extensions-calendar-keep-tasks-beta.html). Gemini Live can add notes to Keep, including from camera input (Jun 2025) [S] [9to5](https://9to5google.com/2025/06/25/gemini-live-google-keep/). | (Gemini) | later | Expose a public, OAuth-scoped API or MCP server so third-party assistants can act on notes. |
| Generate title | Gemini-style "Generate title" button (Oct 2025, v5.25.382) [T] [jetstream](https://jetstream.blog/?p=207489). | A | later | — |
| **Keep Live / "Talk to Keep"** | Talk freely and Keep "turn[s] your thoughts into structured lists and notes" in the background. Announced at I/O in **May 2026**. Launched **2026-09-03** for Google AI Plus, Pro and Ultra; "coming soon" for Workspace business [P] [blog.google](https://blog.google/products-and-platforms/products/workspace/voice-features-gmail-docs-keep). Teardown name "Talk to Keep" (codename "Ramble") in Android v5.26.301 [T] [jetstream](https://jetstream.blog/en/google-keep-talk-to-keep-app-teardown/). That "Talk to Keep" and "Keep Live" are the same feature is **UNVERIFIED** (very likely). Platforms: **UNVERIFIED**. | A (others UNVERIFIED) | later | Streaming STT → LLM structuring → multiple notes/lists created as one batch. Needs an "AI-generated" provenance flag and feedback capture. |

---

## 14. Security and privacy features

| Feature | Behavior | Platforms | Priority | Implication |
|---|---|---|---|---|
| Note lock (password or biometric) | **Does not exist** in Keep [S] [newsoftwares](https://www.newsoftwares.net/blog/privacy-control-in-google-keep-can-you-password-protect-your-notes/). I found no 2025-26 announcement. | — | later (differentiator) | Real locking means client-side E2E encryption of the body, which breaks server search, OCR, previews and sharing. Treat it as a separate "vault" note class. |
| Sharing kill-switch | "Enable sharing" setting (§6). | W A i | v1 | — |
| Lock-screen exposure | Android lock-screen capture (§12) must not reveal existing notes (UNVERIFIED Keep behavior; OS requirement). | A | later | — |

---

## Shared vs per-user state on a shared note

**Main primary evidence:** "You can share a note with other people so they can edit text, lists, images, drawings, and audio recordings. **Anyone you share with can label, color, archive, or add reminders without changing the note for others.**" ([help 6101196, Computer and Android variants](https://support.google.com/keep/answer/6101196)).

Supporting evidence for the data shape: gkeepapi's single sync payload delivers per-user fields (`color`, `isArchived`, `isPinned`, `labelIds`) on the same node object as shared content. That implies **the server projects a per-user view of each shared note**. Label definitions travel separately in a per-account `userInfo.labels` block ([gkeepapi](https://github.com/kiwiz/gkeepapi/blob/master/src/gkeepapi/__init__.py)).

| State | Scope | Evidence | Confidence |
|---|---|---|---|
| Title | **SHARED** | "edit text…" [P] [6101196](https://support.google.com/keep/answer/6101196) | High |
| Body text plus rich formatting | **SHARED** | same | High |
| Checklist items, text, **checked state**, **item order**, **indent** | **SHARED** | same; "watch items get checked off in real time" ([Workspace page](https://workspace.google.com/products/keep/)) | High |
| Images, drawings, audio (+ transcript text) | **SHARED** | same | High |
| OCR text, link previews | **SHARED** (derived from shared content) | inferred | Medium |
| Collaborator list / ACL | **SHARED** (one ACL per note; owner fixed) | [P] [API `permissions[]`](https://developers.google.com/workspace/keep/api/reference/rest/v1/notes) | High |
| Owner-initiated trash / delete | **SHARED** ("deleted for everyone") | [P] [6101196](https://support.google.com/keep/answer/6101196), [6262765](https://support.google.com/keep/answer/6262765) | High |
| Version history | **SHARED** (history of shared text) | inferred | Medium |
| **Labels** (definitions and assignment) | **PER-USER** | [P] [6101196](https://support.google.com/keep/answer/6101196); labels can't be shared | High |
| **Color** | **PER-USER** | [P] [6101196](https://support.google.com/keep/answer/6101196) | High |
| **Archived** | **PER-USER** | [P] [6101196](https://support.google.com/keep/answer/6101196) | High |
| **Reminders** | **PER-USER** (now a task in each user's own Google Tasks) | [P] [6101196](https://support.google.com/keep/answer/6101196), [WU Oct 2025](https://workspaceupdates.googleblog.com/2025/10/google-keep-reminders-now-saved-to-tasks.html) | High |
| **Background image** | **PER-USER** (same picker as color) | inferred from [6191044](https://support.google.com/keep/answer/6191044) | Medium (UNVERIFIED) |
| **Pinned** | **PER-USER** | Not stated explicitly. Inferred because it is an organizational action delivered like `isArchived` and color in the same per-user projection | Medium-high (UNVERIFIED) |
| **Position / custom order in the grid** (and in the Pinned section) | **PER-USER** | Inferred: each user has their own board; `sortValue` is on the projected node | Medium (UNVERIFIED) |
| Non-owner "delete" | Probably **per-user** (leave) for ordinary collaborators. Family-group members **can delete for all** ([P] [7313121](https://support.google.com/keep/answer/7313121)) | partial | Low (UNVERIFIED) |
| Checked-items policy, graveyard collapsed, new-item placement | Account settings ([P] [6358550](https://support.google.com/keep/answer/6358550)), though gkeepapi also has per-node `nodeSettings` | Recommend per-user display preference | Low (UNVERIFIED) |
| Sort mode, grid/list, widget pinning | **Per-device / per-user UI** | [S] [9to5 sort](https://9to5google.com/2025/08/13/google-keep-sort-notes-android/) | High |

**Resulting data model (for the clone):**
- `Note` (shared): `id, ownerId, type, title, body(rich), createdAt, updatedAt, version, trashedAt, deletedAt`
- `ListItem` (shared): `id, noteId, parentItemId(≤1 level), text, checked, sortKey, updatedAt`
- `Attachment` (shared): `id, noteId, kind{image,drawing,audio}, blobRef, meta, extractedText, extractionStatus`
- `NoteMember` (shared ACL): `noteId, principal, role{OWNER,WRITER}, addedBy, addedAt, removedAt`
- `UserNoteState` (per user × note): `userId, noteId, color, background, pinned, archived, sortKey, pinnedSortKey, leftAt, stateUpdatedAt`
- `Label` (per user): `id, userId, name, createdAt`, plus `NoteLabel(userId, noteId, labelId)`
- `Reminder` (per user × note): `id, userId, noteId, dueAt, tz, rrule, snoozedUntil, doneAt`
- `UserSettings` (per user)

---

## Non-functional expectations users have

| Expectation | What Keep sets as the bar | Evidence | Target for the clone |
|---|---|---|---|
| Instant open / instant capture | Widgets, FAB defaulting to a text note, lock-screen and Wear capture. Users expect to be typing within about 1 s | [S] [9to5](https://9to5google.com/2025/04/02/google-keep-text-notes/) | Local-first render from the on-device DB. Cold start to editable note in under 1 s on mid-range phones. Never block on network. |
| Offline | Mobile: full offline, "Changes sync as soon as you're connected". **Web: no offline** | [P] [Workspace page](https://workspace.google.com/products/keep/), [P] [10183819](https://support.google.com/keep/answer/10183819) | Mobile must work fully offline (MVP). Web offline via a PWA (IndexedDB plus service worker) is optional and would beat Keep. |
| Multi-device sync latency | "Syncs in real-time across all your devices". No published SLA (**UNVERIFIED** numbers) | [P] [Workspace page](https://workspace.google.com/products/keep/) | p95 under 2 s between online devices through a push channel. Background mobile sync through silent push. |
| Real-time collaboration | Shared list check-offs appear "in real time" | [P] [Workspace page](https://workspace.google.com/products/keep/) | Item-level operations broadcast live. Presence is optional (Keep shows none; UNVERIFIED). |
| No data loss on conflicts | Conflicts surface as "Keep selected / Keep both" | [P] [6102239](https://support.google.com/keep/answer/6102239) | Never silently drop text. Use a CRDT for body text, or conflict copies. |
| Recoverability | 7-day trash. Text version history on web | [P] [6262765](https://support.google.com/keep/answer/6262765), [13820887](https://support.google.com/keep/answer/13820887) | Same, plus server backups with point-in-time recovery. |
| Data portability | Takeout (HTML + JSON + media) | [P] [10017039](https://support.google.com/keep/answer/10017039) | Self-serve export plus Takeout import. |
| Scale per user | Thousands of notes; note count effectively unlimited [S] [cloudwards](https://www.cloudwards.net/google-keep-review/); 20k chars per note; 50 labels | API, help | Plan for 10k+ notes per user with paginated delta sync and incremental search indexing. |
| Browser support | Two latest versions of Chrome, Firefox, Edge and Safari | [P] [9050190](https://support.google.com/keep/answer/9050190) | Same. |
| Accessibility and i18n | Screen-reader flows. Extension ships in 55 languages | [P] [16914649](https://support.google.com/keep/answer/16914649), [CWS](https://chromewebstore.google.com/detail/google-keep-chrome-extens/lpcaedmchfhocbbapmcbpinfpgnhiddi) | WCAG 2.2 AA. RTL layouts. ICU message formatting. |
| Battery and data (Wear, mobile) | Wear M3E promises battery gains [S] | [S] [9to5](https://9to5google.com/2025/08/21/google-keep-material-3-expressive-redesign/) | Batched sync. No polling. |

---

## Google-specific integrations: exclude or replace

| Google integration | Clone decision | Replacement |
|---|---|---|
| Google Account sign-in, Workspace domain policies, Admin console | Replace | Own auth (email + passkeys/OAuth via Apple/Google/Microsoft as IdPs). Org/admin features later. |
| Google Tasks / Calendar as the reminder backend and notifier | Replace | Own reminder scheduler plus FCM, APNs, Web Push and local notifications. Optional iCal (ICS) feed export. |
| Location reminders (removed by Google) | Exclude (or later) | Client geofencing if ever needed. |
| Copy to Google Docs | Replace | Export to Markdown, DOCX or PDF. Share-sheet "open in". |
| Google Groups and Family group principals | Exclude initially | Own "groups/households" later. |
| Gemini ("Help me create a list", Generate title, Keep Live, Gemini app extension, Gemini Live) | Replace (later) | Pluggable LLM provider behind a server AI gateway, user opt-in. A public API or MCP server lets external assistants act on notes. |
| Google Lens / server OCR, Google speech-to-text | Replace | On-device ML Kit or Apple Vision and SpeechRecognizer or Speech framework; server fallback with Tesseract or Whisper. |
| "Things" ML categorization, grocery suggestions | Exclude (later) | Simple classifier later. |
| Workspace side panels (Gmail, Docs, Calendar, Drive), "Save to Keep notepad" | Exclude | — |
| Google Takeout | Replace | Own export. **Build a Takeout importer** for migration. |
| Google Drive / Google One storage quota | Replace | Own per-user storage quota and blob store (S3-compatible). |
| Google Messages "save to Keep", Assistant | Exclude | The OS share target covers this. |
| Apple Watch app | Exclude (Google dropped it in 2025) | — |
| Vault / eDiscovery | Exclude | — |

---

## Top 15 architectural implications

1. **Split shared content from per-user state from day one.** `Note`, `ListItem`, `Attachment` and `NoteMember` are shared. `UserNoteState` (color, background, pinned, archived, sort keys), `Label`/`NoteLabel` and `Reminder` are per user ([help 6101196](https://support.google.com/keep/answer/6101196)). Every read is a join of the two, and every write must say which side it touches. Adding this after launch would mean a painful migration.
2. **Local-first clients with an outbox.** Mobile must create, edit and search fully offline, because users expect instant capture. Each client keeps a local DB (SQLite/Room/Core Data or IndexedDB) and a durable queue of mutations with client-generated IDs (UUIDv7) and idempotency keys.
3. **Delta sync over a per-user change feed.** Mirror Keep's own shape ([gkeepapi](https://github.com/kiwiz/gkeepapi/blob/master/src/gkeepapi/__init__.py)): the client sends `cursor` plus dirty entities, and the server returns changed entities, a new cursor, a `hasMore` flag and a `forceFullResync` flag. A change to a shared note must fan out into **every member's** feed. Per-user state changes go only to that user's feed.
4. **Fine-grained sync units for lists.** Each checklist item is its own entity with a fractional `sortKey` and a `parentItemId` (depth ≤ 1). Two people checking different items never conflict. The server enforces the one-level nesting rule ([API](https://developers.google.com/workspace/keep/api/reference/rest/v1/notes)) and the first-item-can't-indent rule ([help 6395451](https://support.google.com/keep/answer/6395451)).
5. **Choose a body-text merge strategy explicitly.** Keep uses per-node `baseVersion` optimistic concurrency and shows "Conflicting edits found → Keep selected / Keep both" ([help 6102239](https://support.google.com/keep/answer/6102239)). For a clone that promises real-time collaboration, use a text CRDT (e.g. Yjs/Automerge) for title and body. Otherwise do version checks plus automatic conflict copies. Either way, never silently lose text.
6. **Real-time transport.** Use a WebSocket or SSE channel for open clients and silent push (FCM/APNs) to wake mobile clients. Updates to shared notes and list check-offs must arrive in seconds ([Workspace page](https://workspace.google.com/products/keep/)).
7. **A forward-compatible rich-text format.** Store structured rich text (marks for B/I/U and block types for H1/H2/paragraph, plus future checkbox blocks; see the [inline to-do teardown](https://www.androidauthority.com/google-keep-to-do-lists-notes-apk-teardown-3703191/)) with a plain-text projection for search, limits and the API. Use **capability negotiation** so a client that can't render a feature keeps it intact on save. Keep's iOS app still can't show formatting ([help](https://support.google.com/keep/answer/2888246?hl=en&co=GENIE.Platform%3DiOS)).
8. **Fractional ordering everywhere.** Notes in the grid, the pinned section and list items all use string fractional indexes (LexoRank-style). Drag or `Shift+J/K` reorders write only the moved entity. Note order is per user; item order is shared.
9. **Blob pipeline separate from note sync.** Use presigned direct uploads with limits enforced (10 MB / 25 MP for images [S]), thumbnails and renditions, a CDN and content-addressed dedupe for "Make a copy". Attachments sync as metadata. Bytes download lazily and are cached on the device.
10. **Async enrichment workers.** Separate jobs handle OCR (`extracted_text`/`extraction_status`), handwriting recognition on drawings, audio transcription, link unfurling (with SSRF protection) and later AI categorization. Each writes derived fields back to shared content and re-indexes for search. These jobs must be idempotent and retryable.
11. **Search spanning both scopes.** Facets combine shared derived fields (type, has-image, has-url, OCR text) and per-user fields (labels, color, archived), plus people ([help 2888263](https://support.google.com/keep/answer/2888263)). Mobile uses local FTS for offline. The server index for web must update every member's view when shared content changes.
12. **A reminder scheduler as its own service.** Reminders are per user and link to a note ([help 6101196](https://support.google.com/keep/answer/6101196)). Store time plus timezone plus a constrained RRULE (interval cap ≤ 1,000 days, per [Tasks help](https://support.google.com/tasks/answer/16540694)). Use a durable delayed-job queue for push, mirrored by local notifications on the device, with dedupe across devices, snooze overrides and a "Reminders" query view. Keep both apart from note sync: Google itself moved reminders out to Tasks in 2025 ([Workspace Updates](https://workspaceupdates.googleblog.com/2025/10/google-keep-reminders-now-saved-to-tasks.html)).
13. **Trash and deletion lifecycle jobs.** Owner delete sets `trashedAt` on shared content and hides the note for everyone. A **7-day purge cron** ([help 6262765](https://support.google.com/keep/answer/6262765)) hard-deletes the note, its blobs and its search docs. Empty Trash is immediate. Revoking a member, or a collaborator leaving, sends tombstones to that user's feed and clears their per-user rows. Also handle account-deletion cascades.
14. **A simple ACL model enforced on every mutation.** Roles are OWNER and WRITER only. Writers can edit content and manage other writers but cannot remove the owner ([API](https://developers.google.com/workspace/keep/api/reference/rest/v1/notes)). There is no link sharing. Invites by email include pending invites for people not yet signed up. The "Enable sharing" setting blocks incoming shares ([help 6358550](https://support.google.com/keep/answer/6358550)). Cap collaborators to bound fan-out.
15. **Platform extension surfaces share the local store and outbox.** Android Glance widgets (including toggling checkboxes in a widget), iOS WidgetKit, Share Extension and App Intents through an App Group, Android share intents and the Notes role, the Wear OS app with tiles and complications, and an MV3 Chrome extension all need the same domain/sync core: a shared Kotlin Multiplatform or Rust core, or at least a shared schema and protocol package. Otherwise these surfaces drift and corrupt state.

---

## UNVERIFIED items to check by hand before finalizing requirements

- Max collaborators per note. Max images per note. Max audio length. Max label-name length.
- Pinned, background and custom-order scope on shared notes (inferred per-user).
- What a non-owner "delete" does on a shared note, and whether a "leave note" action exists.
- Whether archived or trashed notes appear in search. Whether pinning an archived note unarchives it. Whether trashed notes are read-only. Whether a single note can be "Delete forever" from Trash.
- "Uncheck all items" and "Delete checked items" menu wording on W/A/i.
- Whether checklist items accept rich formatting.
- Whether Grab image text exists on iOS. Whether Keep has a modern iOS WidgetKit widget. iOS dark-mode date.
- Whether Note links, Generate title, the Android Notes role and lock-screen capture have shipped (all teardown-only).
- Whether "Talk to Keep" and "Keep Live" are the same feature, and which platforms have it.
- Whether "Make a copy" keeps labels, collaborators or reminders.

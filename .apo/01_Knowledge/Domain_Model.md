---
note_type: knowledge
template_version: 1
contract_version: 1
knowledge_id: "KNOW-0003"
category: architecture
title: "Domain Model"
status: in_progress
owner: ""
created: '2026-06-30'
updated: '2026-06-30'
reviewed_on: ""
related_notes: []
tags: [apovault, knowledge, architecture]
---

# Domain Model

The "entities" here are the two persistent data artifacts (registry + mapping) and the Figma
nodes they describe. There is no database or runtime model — these are JSON shapes.

## Entities

- **Flow** — one screen flow, the unit of work. In the mapping each flow records `name`,
  `fileKey`, `fileName`, `page`, `iosSectionNodeId`, `androidSectionNodeId`, `screenPairs[]`,
  `createdAt`, `lastSyncedAt`. **Source:** `mappings.example.json:4-18` (read 2026-06-30).
- **ScreenPair** — one iOS↔Android screen correspondence: `{ name, ios, android }` (Figma
  node ids). The mapping's `screenPairs[]` is the join table between platforms.
  **Source:** `mappings.example.json:12-15` (read 2026-06-30).
- **Section** — a Figma section frame, one per platform per flow (iOS section + Android
  section), siblings on one page. **Source:** `skills/ios-to-android/SKILL.md:33-43` (read 2026-06-30).
- **Registry component** — a design-system constant: `{ key, library, type, note/variant }`,
  e.g. `statusBar`, `topAppBar`, `iconClose`, `iconBack`, `materialButton` (forward) and the
  `androidToIos.components` set (`iosStatusBar`, `iosModalNav`, `sheetGrabber`, `homeIndicator`).
  **Sources:** `registry/components.json:8-14` (read 2026-06-30), `registry/components.json:52-57` (read 2026-06-30).
- **Library** — a published Figma library referenced by key: `tidepoolLoopAndroid`,
  `material3` (forward); `coreIos17` (reverse; `tidepoolLoopIos`/`appleHig` ship empty).
  **Sources:** `registry/components.json:4-7` (read 2026-06-30), `registry/components.json:46-51` (read 2026-06-30).

## Relationships

- **Flow 1:1 file/page**, **1:1 iOS section**, **1:1 Android section**.
  **Source:** `mappings.example.json:5-11` (read 2026-06-30).
- **Section 1:N screens**; **iOS screen 1:1 Android screen** via `screenPairs[]`.
  **Source:** `mappings.example.json:12-15` (read 2026-06-30).
- **A flow may be built in either direction** and reuses one entry — `create-ios` reuses the
  flow's entry, treating `androidSectionNodeId` as source and `iosSectionNodeId` as the new
  section. **Source:** `commands/create-ios.md:29` (read 2026-06-30).
- **Registry component → library** by the component's `library` field (a key into
  `libraries`/`iosLibraries`). **Source:** `registry/components.json:9-13` (read 2026-06-30).

## Invariants

- **Derive, don't copy** — a target section is derived only from the source section + the
  target DS; never read a pre-existing target section to drive the design (only its container
  structure may be mirrored, compared after). **Source:** `skills/ios-to-android/SKILL.md:25-29` (read 2026-06-30).
- **Sections stack vertically, left-aligned, source on top**, sharing `firstScreenX` +
  `screenPitch` so screen pairs column-align. **Source:** `skills/ios-to-android/SKILL.md:54-65` (read 2026-06-30).
- **Discover screens by enumerating a section's direct children, not by name-regex** — off-
  convention frames (e.g. `End Screen (ADTL)`) would otherwise be silently dropped.
  **Source:** `skills/ios-to-android/SKILL.md:193-206` (read 2026-06-30).
- **Idempotency:** if a target section already exists for a flow, stop and ask — never
  silently duplicate. **Source:** `commands/create-android.md:15-16` (read 2026-06-30).
- **Mapping is created/updated only after a successful build.**
  **Source:** `skills/ios-to-android/SKILL.md:232-237` (read 2026-06-30).

## References

- [[Code_Map]], [[Integration_Map]], [[Coding_Standards]] (Theme Tokens)

## Verification status

Fully cited from the mapping schema, registry, command, and skill files. No `(verify)` items.

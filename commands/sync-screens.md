---
description: (PLANNED) Propagate changes from iOS screens to their Android counterparts using the saved mapping
argument-hint: "[flow name or file key]"
---

> **STATUS: PLANNED — not yet implemented.** Do not make changes yet. Explain the intended
> behavior to the user, confirm scope, and stop unless they explicitly ask to prototype it.

Sync edits made on iOS screens over to their Android counterparts for the flow in
`$ARGUMENTS`, using the recorded screen-node pairs.

Intended behavior:
1. Read `~/.figma-sync/mappings.json`; locate the flow and its `screenPairs` (iOS↔Android).
2. Preflight the Figma MCP (`whoami`).
3. For each pair, detect what changed on the iOS screen since `lastSyncedAt` (text, images,
   structure) and apply the equivalent change to the Android screen — **re-running the
   `figma-sync:ios-to-android` translation** for changed content (re-font, re-clone, etc.)
   rather than blind-copying iOS pixels.
4. Leave chrome/CTA/layout conventions intact; only propagate genuine content deltas.
5. Update `lastSyncedAt`; screenshot-verify; report per-screen.

Open design questions: how to diff iOS reliably (node-hash snapshots stored in the mapping?
content fingerprints?), how to handle Android-only manual tweaks without clobbering them, and
conflict resolution.

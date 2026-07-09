---
description: Create an iOS (Apple HIG) section from an existing Android section in a Figma file
argument-hint: "[figma url or file key] [optional freeform hints]"
---

Create a new **IOS Section** that translates an existing Android flow, in the Figma file referenced by: $ARGUMENTS

Follow the **`figma-sync:android-to-ios`** skill as the authoritative playbook. Do not deviate from its conventions. Work through these phases:

## 1. Preflight (do not skip)
- Confirm the Figma MCP is available and authenticated: call `mcp__plugin_figma_figma__whoami`. If it fails, stop and tell the user to open Figma desktop, enable the MCP server, and sign in. (This plugin rides on the official Figma MCP — it does **not** provide its own.)
- Resolve the target from `$ARGUMENTS` (a figma.com URL or a file key). If none was given, ask the user for the file URL.
- Capture any **freeform hint text** trailing the file reference in `$ARGUMENTS` (e.g. "titles are sentence case", "skip the End screen"). Honor it as **additional per-run guidance**, subordinate to the golden rule (derive, don't copy) and the skill's conventions / the registry source-of-truth — a hint steers the build, it never overrides them.
- Read the design-system constants from `${CLAUDE_PLUGIN_ROOT}/registry/components.json` — the shared grid + `tokens.brandPrimary`, and the `androidToIos` block (iOS layout, typography, component keys). **iOS chrome keys ship empty:** if any `androidToIos.components.*.key` is blank, discover it via `search_design_system` against the iOS DS library and **commit it back to the registry** before building.

## 2. Idempotency guard
- Resolve the counterpart via the **identity stamp**, not page position or name (per the `figma-sync:page-layout` skill §1): look for a section whose `figmaSyncPairId` matches the Android source's and whose `figmaSyncRole` is `ios`. If a stamped iOS counterpart exists, **stop and ask** whether to update or recreate — never silently duplicate. This command is for creating one where none exists.
- If the Android **source** is itself unstamped (built before stamping existed), adopt it first (`page-layout` §6): write its stamp, then proceed. A pre-existing but **unstamped** iOS section is a legacy-adoption case — hand it to `page-layout` (§6) to pair + stamp; do not treat it as absent and build a duplicate.

## 3. Golden rule
- Derive the iOS design **only** from the Android section + the iOS/Apple HIG design systems. If a finished iOS section exists elsewhere as a reference, **do not look at it to drive the design.**

## 4. Build (per the skill)
- Set up the `IOS Section`: background `#565656`, **black** sidebar cloned from the Android `Status` (recolored black fill / white text), 64px section radius, 192px top/bottom padding. **Placement is owned by the `figma-sync:page-layout` packer** — below the Android source, left-aligned, in the managed column band (never to the right); do **not** hardcode a fixed `y` offset. **Write the identity stamp** on the new section (`figmaSyncFlowId` / `figmaSyncRole=ios` / `figmaSyncPairId`, per `page-layout` §1).
- **Column-align the two sections:** place iOS screens at the shared `x = firstScreenX + order*screenPitch` (1184 + order*460). Keep true device widths (Android 412 / iOS 375), left-aligned; equalize the two section widths (use the wider).
- Build **screen 1 first** and show the user for sign-off before batching the rest.
- For each Android screen: assemble iOS status bar + grabber (if a sheet) + nav (inline centered title; Back chevron ‹ / Close per the header rule) + cloned `Content AL` re-fonted to SF Pro (Text/Display by size, preserve Android metrics) + full-width iOS brand button (r8) + home indicator. iOS screen radius 0. Device-vs-scroll height logic at 812.
- **Nav title = the flow name**, not a screen's content H1 — the Android source's app-bar title may be a placeholder; don't copy it blindly. After building, verify each nav title reads the flow name and fix any that duplicated the content headline (load font → set `characters`).
- Lay screens out per the section-layout rules (fixed 460 pitch, y=192 baseline, recompute section width/height).

## 5. Record the mapping
- After a successful build, write/update `~/.figma-sync/mappings.json` (create the dir/file if missing). Reuse the flow's entry (a flow may be built in either direction): `flowId` (matching the identity stamp), `name`, `fileKey`, `page`, `androidSectionNodeId` (source), `iosSectionNodeId` (the new section), and the `screenPairs` (android↔ios node ids). See `${CLAUDE_PLUGIN_ROOT}/mappings.example.json` for the schema.
- **Capture initial per-side snapshots** (drift-sync §4 — `texts / imageRefs / structHash / ctaLabel / screenFill`) for each pair at record time. Both sides agree at create, so this seeds the drift baseline and lets the **first** `sync-screens` **attribute direction** instead of degrading to live-only detection.
- **Reconcile related mappings:** if another flow with the **same `fileKey`** shares this section, update it too; a flow with the **same node ids but a different `fileKey`** is a separate duplicate file — leave it (flag only). Gate on `fileKey`, never bare node ids.

## 6. Report
- Share the iOS section node link and a short summary. Flag anything that needs human judgment (ambiguous titles, non-"Continue" CTA labels, terminal/menu screens with no nav/CTA).

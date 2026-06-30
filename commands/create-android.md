---
description: Create an Android (Material 3) section from an existing iOS section in a Figma file
argument-hint: "[figma url or file key]"
---

Create a new **ANDROID Section** that translates an existing iOS flow, in the Figma file referenced by: $ARGUMENTS

Follow the **`figma-sync:ios-to-android`** skill as the authoritative playbook. Do not deviate from its conventions. Work through these phases:

## 1. Preflight (do not skip)
- Confirm the Figma MCP is available and authenticated: call `mcp__plugin_figma_figma__whoami`. If it fails, stop and tell the user to open Figma desktop, enable the MCP server, and sign in. (This plugin rides on the official Figma MCP — it does **not** provide its own.)
- Resolve the target from `$ARGUMENTS` (a figma.com URL or a file key). If none was given, ask the user for the file URL.
- Read the design-system constants from `${CLAUDE_PLUGIN_ROOT}/registry/components.json` (library + component keys, tokens, layout values). Use those keys — do not re-discover by search unless a key is missing/stale.

## 2. Idempotency guard
- Inspect the page. If an `ANDROID Section` already exists for this flow, **stop and ask** whether to update or recreate — never silently duplicate. This command is for creating one where none exists.

## 3. Golden rule
- Derive the Android design **only** from the iOS section + the Android/Material design systems. If a finished Android section exists elsewhere as a reference, **do not look at it to drive the design.**

## 4. Build (per the skill)
- Set up the `ANDROID Section` **stacked directly below the iOS section and left-aligned** (`x = ISO.x`, `y = ISO.y + ISO.height + 800`) — never to the right. Gray `#BCBCBC`, white sidebar cloned from iOS `Status` (stroke removed), 64px section radius, 192px top/bottom padding.
- **Column-align the two sections:** place Android screens at the shared `x = 1184 + order*460`, and **re-space the iOS screens to the same 460 pitch** (canvas spacing only) so each iOS screen sits directly above its Android counterpart. Keep true device widths (375 / 412), left-aligned; equalize the two section widths (use the wider).
- Build **screen 1 first** and show the user for sign-off before batching the rest.
- For each iOS screen: assemble status bar + centered Top app bar (back ←/close ✕ per the header rule) + cloned `Content AL` re-fonted to Roboto (preserve iOS metrics) + brand pill CTA + gesture nav. 18px screen radius. Device-vs-scroll height logic.
- Lay screens out per the section-layout rules (fixed 460 pitch, y=192 baseline, recompute section width/height).

## 5. Record the mapping
- After a successful build, write/update `~/.figma-sync/mappings.json` (create the dir/file if missing) with this flow's entry: `name`, `fileKey`, `page`, `iosSectionNodeId`, `androidSectionNodeId`, and the `screenPairs` (iOS↔Android node ids). See `${CLAUDE_PLUGIN_ROOT}/mappings.example.json` for the schema. This is what `sync-screens` and `apply-ds-update` will read later.

## 6. Report
- Share the Android section node link and a short summary. Flag anything that needs human judgment (ambiguous titles, non-"Continue" CTA labels, terminal/End screens).

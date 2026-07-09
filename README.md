# figma-sync

A Claude Code **plugin** for translating Figma flows between **iOS (Apple HIG)** and
**Android (Material 3)** — in **either direction** — and keeping the two platforms' screens in
sync as content changes (screen-level content sync shipped; a design-system roll-out is still
planned). Built for Tidepool's design team.

It rides on the **official Figma MCP** (it does not ship its own MCP server), and encodes
the team's iOS→Android conventions as a reusable skill + slash commands.

## Prerequisites
- **Claude Code** with the **official Figma MCP** installed, enabled, and **signed in**
  (Figma desktop). Verify with the `/mcp` command or by asking Claude to run `whoami`.
- Edit access to the target Figma file.

## Install
```
/plugin marketplace add tidepool-org/figma-sync
/plugin install figma-sync@tidepool
```
Update later with `/plugin update`. (Repo path during development:
`~/src/github/tidepool/figma-sync`.)

## Commands
| Command | Status | What it does |
|---|---|---|
| `/figma-sync:create-android [figma url]` | ✅ ready | Create an Android section translating an existing iOS flow |
| `/figma-sync:create-ios [figma url]` | ✅ ready | Create an iOS section translating an existing Android flow |
| `/figma-sync:sync-screens [flow]` | ✅ ready | Detect content drift and propagate approved edits in either direction, with snapshot/rollback |
| `/figma-sync:apply-ds-update [ios\|android] [change]` | ✅ ready | Roll a committed registry (design-system) change across every mapped file for a platform, via a full re-apply that preserves project overrides, with approval gate + per-file backup |
| `/figma-sync:build-page [figma url] [ios\|android]` | ✅ ready | Reconcile a whole page of many flows: build each missing platform section below its source, relocate misplaced ones (geometry only), leave unmanaged content untouched — approval gate + reversible layout backup |

## How it works
- **`skills/ios-to-android/`** and **`skills/android-to-ios/`** — the authoritative conventions
  (layout, typography, header rules, component usage, gotchas) for each direction. Claude follows
  the matching one when building.
- **`skills/drift-sync/`** — the sync playbook: detect where a mapped flow's **content** has
  diverged, present a reviewable plan, and — **only after you approve** — propagate the deltas
  in the chosen direction with a backup/restore safety net.
- **`skills/ds-update/`** — the design-system update playbook: roll a committed registry change
  across every mapped file for a platform via a full per-screen re-apply that preserves project
  overrides, with an approval gate, per-file backup, and per-flow version stamping.
- **`skills/page-layout/`** — the multi-section-page playbook: a stable in-file identity stamp,
  a read-only classifier, a declarative packer that lays each section out **below its source**
  and makes room by pushing lower sections down, geometry-only relocation of misplaced sections,
  legacy-section adoption, and a reversible layout backup. `create-android` / `create-ios` defer
  to it for placement + identity; `build-page` drives it across a whole page.
- **`registry/components.json`** — versioned design-system constants (library + component
  keys, tokens, layout values). Update this when iOS/Android ship new specs.
- **`~/.figma-sync/mappings.json`** — per-designer, user-global record of which files/sections
  have been built and the iOS↔Android screen-node pairs. Each screen pair may also carry
  **per-side content snapshots** — one record that serves as both the drift baseline and the
  rollback point. Created/updated automatically by the commands; no git knowledge required.
  Schema: [`mappings.example.json`](./mappings.example.json).

**Syncing screens.** `sync-screens` is **bidirectional** (default iOS → Android). It reads the
mapping, diffs live content against the per-side snapshots to attribute *who* changed, and
presents a plan; on explicit approval it backs up the target (the snapshot for content-only
deltas, a hidden in-canvas duplicate before a screen removal or wholesale re-clone, the built
node id for a new screen), propagates the approved **content and screen-level structural**
deltas through the matching direction skill, verifies, and re-baselines the snapshots. A bad
result restores from the backup. Chrome and font family differ by platform *by design* and are
never treated as drift. **Structural changes are in scope**: a screen the source added is built
on the target and a screen it removed is deleted from the target — each approved per screen (a
removal is destructive), backed up, and reversible.

**Applying a DS update.** When iOS or Android ships a new design-system spec, reflect it into
`registry/components.json` and **commit** it — that commit is the auditable record. Then
`apply-ds-update [ios|android] [summary]` reads the delta from the registry `git diff`, builds
the work set of every mapped file for the platform (grouped by `fileKey`, **including cross-file
duplicates** — the reverse of `sync-screens`), and detects deliberate project overrides to
preserve. After you approve, it backs up each file (a hidden in-canvas duplicate), fans out one
agent per file to **fully re-apply** each screen's chrome from the current DS while keeping the
overrides, verifies (restoring on mismatch), and stamps each updated flow with the applied
`dsVersion` so a re-run skips files already current.

**Building a whole page.** A page often holds **many flows**, not one. `build-page [figma url]
[ios|android]` reconciles the whole page for the chosen target platform: it classifies every
section (read-only), then for each flow **builds** a missing counterpart, **relocates** a
misplaced one below its source (geometry only — never rebuilding its content), or leaves an
in-place one alone. Placement is computed by a declarative packer that **makes room by pushing
lower sections down**; unmanaged content (type-system boards, unrelated frames) is never touched.
Sections are identified by a durable in-file **identity stamp**, so counterparts are matched
reliably even with many flows on one page; a pre-existing unstamped section is **adopted** (its
pairing inferred, then confirmed by you) before it joins the layout. Like the other write
commands it runs read-only first, presents a plan, and mutates only after you approve — with a
**layout backup** that restores the whole arrangement. After adopting/relocating a section it may
**offer** a `sync-screens` pass to reconcile that pair's content.

## Notes
- **Derive, don't copy.** The target design is derived only from the source screens + the
  target platform's design system — never from a pre-existing target section.
- The two sections are **stacked vertically, left-aligned** — **source on top** (iOS for
  `create-android`, Android for `create-ios`) — for easy comparison. On a page with many flows,
  `build-page` keeps this below-the-source layout and makes room by pushing lower sections down.

See [`ARCHITECTURE.md`](./ARCHITECTURE.md) for the design rationale and roadmap.

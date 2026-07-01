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
| `/figma-sync:apply-ds-update [ios\|android] [change]` | 🚧 planned | Roll a design-system change across all mapped files |

## How it works
- **`skills/ios-to-android/`** and **`skills/android-to-ios/`** — the authoritative conventions
  (layout, typography, header rules, component usage, gotchas) for each direction. Claude follows
  the matching one when building.
- **`skills/drift-sync/`** — the sync playbook: detect where a mapped flow's **content** has
  diverged, present a reviewable plan, and — **only after you approve** — propagate the deltas
  in the chosen direction with a backup/restore safety net.
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
deltas, a hidden in-canvas duplicate for structural ones), propagates only the **content**
deltas through the matching direction skill, verifies, and re-baselines the snapshots. A bad
result restores from the backup. Chrome and font family differ by platform *by design* and are
never treated as drift; structural propagation (adding/deleting screens) is deferred.

## Notes
- **Derive, don't copy.** The target design is derived only from the source screens + the
  target platform's design system — never from a pre-existing target section.
- The two sections are **stacked vertically, left-aligned** — **source on top** (iOS for
  `create-android`, Android for `create-ios`) — for easy comparison.

See [`ARCHITECTURE.md`](./ARCHITECTURE.md) for the design rationale and roadmap.

# figma-sync

A Claude Code **plugin** for translating iOS-designed Figma flows into matching
**Android (Material 3)** sections — and (planned) keeping platforms in sync as design
specs evolve. Built for Tidepool's design team.

It rides on the **official Figma MCP** (it does not ship its own MCP server), and encodes
the team's iOS→Android conventions as a reusable skill + slash commands.

## Prerequisites
- **Claude Code** with the **official Figma MCP** installed, enabled, and **signed in**
  (Figma desktop). Verify with the `/mcp` command or by asking Claude to run `whoami`.
- Edit access to the target Figma file.

## Install
```
/plugin marketplace add tidepool/figma-sync
/plugin install figma-sync@tidepool
```
Update later with `/plugin update`. (Repo path during development:
`~/src/github/tidepool/figma-sync`.)

## Commands
| Command | Status | What it does |
|---|---|---|
| `/figma-sync:create-android [figma url]` | ✅ ready | Create an Android section translating an existing iOS flow |
| `/figma-sync:sync-screens [flow]` | 🚧 planned | Propagate iOS screen edits to their Android counterparts |
| `/figma-sync:apply-ds-update [ios\|android] [change]` | 🚧 planned | Roll a design-system change across all mapped files |

## How it works
- **`skills/ios-to-android/`** — the authoritative conventions (layout, typography, header
  rules, component usage, gotchas). Claude follows this when building.
- **`registry/components.json`** — versioned design-system constants (library + component
  keys, tokens, layout values). Update this when iOS/Android ship new specs.
- **`~/.figma-sync/mappings.json`** — per-designer, user-global record of which files/sections
  have been built and the iOS↔Android screen-node pairs. Created/updated automatically by the
  commands; no git knowledge required. Schema: [`mappings.example.json`](./mappings.example.json).

## Notes
- **Derive, don't copy.** The Android design is derived only from the iOS screens + the
  Android/Material design systems — never from a pre-existing Android section.
- The two sections are **stacked vertically, left-aligned** (iOS on top) for easy comparison.

See [`ARCHITECTURE.md`](./ARCHITECTURE.md) for the design rationale and roadmap.

# figma-sync — Architecture & Roadmap

Design rationale for packaging Tidepool's iOS↔Android Figma workflow as a Claude Code
plugin, and the roadmap for the workflows beyond the first.

---

## 1. Why a plugin (not a bare skill)

"Skill vs plugin" is a false choice: **a plugin is the container; skills live inside it.**
We chose a plugin because we need:

- **Team install + versioned updates** — `/plugin install` once, `/plugin update` for new
  conventions. No "which file is current?" confusion.
- **Namespaced slash commands** as entry points (`/figma-sync:create-android`) — a bare skill
  can't provide these.
- **Room to grow** — multiple workflows (create in both directions / sync / apply-DS-update)
  plus shared state, all under one installable unit.

A bare project-skill would only make sense for one-off personal use. It can't be installed
team-wide as a unit and offers no command surface or versioning.

## 2. Key decision: ride on the official Figma MCP

The workflow runs entirely through the **official Figma MCP** (`use_figma`,
`get_design_context`, etc.), which authenticates **interactively via Figma desktop**.
Plugin-bundled MCP servers **cannot** do interactive auth, and every designer already has
the Figma MCP installed. Therefore:

- This plugin **does not declare its own MCP server.** Its skills call the already-present
  `mcp__plugin_figma_figma__*` tools.
- It **documents the prerequisite** and each command runs a `whoami` **preflight**.

This removes all token/secret handling from the plugin.

## 3. Repository layout (single repo = marketplace + plugin)

```
figma-sync/
├── .claude-plugin/
│   ├── plugin.json            # name, version (semver), description
│   └── marketplace.json       # makes the repo installable (name: "tidepool")
├── commands/                  # the verbs (entry points)
│   ├── create-android.md      # ✅ ready  (iOS → Android)
│   ├── create-ios.md          # ✅ ready  (Android → iOS)
│   ├── sync-screens.md        # 🚧 planned
│   └── apply-ds-update.md     # 🚧 planned
├── skills/
│   ├── ios-to-android/SKILL.md  # forward-direction conventions playbook
│   └── android-to-ios/SKILL.md  # reverse-direction conventions playbook
├── agents/
│   ├── screen-builder.md      # offload/parallelize per-screen building (→ Android)
│   └── screen-builder-ios.md  # offload/parallelize per-screen building (→ iOS)
├── registry/
│   └── components.json        # versioned DS constants (keys, tokens, layout)
├── mappings.example.json      # schema for the user-global mapping
├── README.md
└── ARCHITECTURE.md
```

Install: `/plugin marketplace add tidepool-org/figma-sync` → `/plugin install figma-sync@tidepool`.

## 4. The backbone: registry + mapping

Two data artifacts turn three separate scripts into one coherent system.

### `registry/components.json` (committed, versioned)
The stable design-system constants — library keys, component keys, brand tokens, and layout
values. **`apply-ds-update` edits this file**, so its git history becomes the auditable
record of every design-system change. Keeps the build from re-discovering keys each run.

### `~/.figma-sync/mappings.json` (user-global, auto-managed)
Per-designer record of work done: for each flow, the `fileKey`, page, iOS + Android section
node ids, and the **iOS↔Android `screenPairs`**.

- `create-android` **writes** it after a build.
- `sync-screens` **reads** the pairs to know which Android node mirrors a changed iOS node.
- `apply-ds-update` **reads** it to enumerate target files.

**Why user-global, not repo-local:** designers install via standard Claude commands and may
have no git workflow at all. A file at `~/.figma-sync/mappings.json` works regardless of any
repo, survives across projects, and has zero risk of being committed. (`mappings.example.json`
in the repo documents the schema.)

## 5. Workflows

| Workflow | Command | Status | Core idea |
|---|---|---|---|
| Create Android section | `create-android` | ✅ | Translate an iOS flow where no Android exists; write the mapping |
| Create iOS section | `create-ios` | ✅ | Reverse direction: translate an Android flow where no iOS exists; write the mapping |
| Sync iOS → Android | `sync-screens` | 🚧 | Re-run the translation for content that changed on iOS, per the mapping |
| Apply DS update | `apply-ds-update` | 🚧 | Edit the registry, fan out one agent per mapped file to re-apply |

The create workflows share the registry (constants) and the mapping (targets/pairs); each uses
its direction's skill (`ios-to-android` / `android-to-ios`). The two skills are mirrors —
forward keys live at the top of the registry, reverse keys under `androidToIos`.

## 6. Distribution & versioning
- Single GitHub repo is both the marketplace and the plugin (`source: "./"`).
- Semantic version in `plugin.json`; bump on convention/registry changes; team runs
  `/plugin update`.
- The `registry/components.json` `version` tracks DS-spec revisions independently.

## 7. Roadmap / phasing
- **Phase 1 (now):** `create-android` wrapping the proven workflow + the skill + registry +
  mapping write. Installable; designers can use it today.
- **Phase 2:** `sync-screens`. Needs a reliable **iOS-change detection** strategy — likely
  storing per-screen content fingerprints in the mapping at build time and diffing on sync.
  Must avoid clobbering intentional Android-only tweaks (conflict handling).
- **Phase 3:** `apply-ds-update`. Multi-file fan-out (agent per file), delta semantics
  (targeted patch vs full re-apply), idempotency, and rollback.

## 8. Open questions (to resolve before Phase 2/3)
- iOS-diff representation: node-hash snapshots vs content fingerprints in the mapping.
- Protecting manual Android edits during sync (merge/conflict policy).
- DS-update "delta" semantics and per-file versioning/rollback.
- Whether to add a `validate`/`audit` command (drift check between iOS, Android, and registry).

## 9. Decisions captured
- **Plugin**, not a bare skill. *(team install + growth)*
- **Ride on the official Figma MCP**; do not bundle one. *(interactive auth)*
- **Mapping is user-global** at `~/.figma-sync/mappings.json`. *(no git knowledge needed)*
- **Registry is committed & versioned** as the DS source of truth and DS-change audit log.
- **Sections stacked vertically, left-aligned** (source on top) for comparison.
- **Derive, don't copy** from any existing target-platform reference.
- **Bidirectional, mirrored skills.** `create-ios` mirrors `create-android`; reverse iOS DS
  constants live under the registry's `androidToIos` block and ship with empty component keys
  to be discovered against the iOS DS library and committed on first run.

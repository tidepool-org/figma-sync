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
│   ├── sync-screens.md        # ✅ ready  (bidirectional content sync + rollback)
│   ├── apply-ds-update.md     # ✅ ready  (roll a committed DS change across mapped files)
│   └── build-page.md          # ✅ ready  (reconcile a whole page of many flows)
├── skills/
│   ├── ios-to-android/SKILL.md  # forward-direction conventions playbook
│   ├── android-to-ios/SKILL.md  # reverse-direction conventions playbook
│   ├── drift-sync/SKILL.md      # detect drift, plan, propagate + rollback
│   ├── ds-update/SKILL.md       # roll a committed DS change across mapped files
│   └── page-layout/SKILL.md     # multi-section identity + packer + reconcile
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

Three data artifacts turn the separate commands into one coherent system.

### `registry/components.json` (committed, versioned)
The stable design-system constants — library keys, component keys, brand tokens, and layout
values. **`apply-ds-update` edits this file**, so its git history becomes the auditable
record of every design-system change. Keeps the build from re-discovering keys each run.

### `~/.figma-sync/mappings.json` (user-global, auto-managed)
Per-designer record of work done: for each flow, the `fileKey`, page, iOS + Android section
node ids, and the **iOS↔Android `screenPairs`**.

- `create-android` **writes** it after a build.
- `sync-screens` **reads** the pairs to find each side's counterpart, **reads** the per-side
  content snapshots as the drift baseline + rollback record, and **rewrites** the snapshots +
  `lastSyncedAt` / `lastSyncDirection` after a successful sync.
- `apply-ds-update` **reads** it to enumerate target files (grouped by `fileKey`) and
  **writes** each updated flow's `dsVersion` / `dsAppliedAt` stamp after a successful re-apply.
- `build-page` **reads** the identity stamp + mapping to reconcile a whole page and **writes**
  each flow's `flowId` and relocated/built section node ids (recording adopted pairings).

**Why user-global, not repo-local:** designers install via standard Claude commands and may
have no git workflow at all. A file at `~/.figma-sync/mappings.json` works regardless of any
repo, survives across projects, and has zero risk of being committed. (`mappings.example.json`
in the repo documents the schema.)

### In-file identity stamp (per section, `setPluginData`)
Each managed section carries a durable identity written into the node via Figma
`setPluginData`: `figmaSyncFlowId`, `figmaSyncRole` (`ios`/`android`), and `figmaSyncPairId`
(links the two counterpart sections). It is the source of truth for a flow's identity and its
iOS↔Android pairing — so the commands recognise an existing counterpart and reconcile a page
**reliably even when many flows share it**, where spatial/name matching cannot.
`create-android` / `create-ios` / `build-page` write it; the mapping's `flowId` cross-links the
user-global record to the in-file stamp. It holds identity only — never layout coordinates (the
`build-page` layout backup lives in-canvas) or chrome/content.

## 5. Workflows

| Workflow | Command | Status | Core idea |
|---|---|---|---|
| Create Android section | `create-android` | ✅ | Translate an iOS flow where no Android exists; write the mapping |
| Create iOS section | `create-ios` | ✅ | Reverse direction: translate an Android flow where no iOS exists; write the mapping |
| Sync screens (either direction) | `sync-screens` | ✅ | Detect content drift, present a plan, then propagate the approved deltas either way with a snapshot/rollback safety net |
| Apply DS update | `apply-ds-update` | ✅ | Read the delta from the committed registry diff, fan out one agent per `fileKey` to fully re-apply each screen's chrome from the current DS (preserving project overrides), with approval gate, per-file backup, and per-flow version stamping |

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
- **Phase 2 (shipped):** `sync-screens`. Content-change detection stores **per-side content
  snapshots** in the mapping and diffs live content against them to attribute *who* changed.
  Sync is **bidirectional**, propagates **content and screen-level structural** deltas (chrome
  and type family differ by platform by design and are never drift), and protects intentional
  counterpart edits via **conflict handling** (both-sides-edited → ask; unresolved conflicts
  skipped). **Structural propagation is in scope**: a screen the source added is built on the
  target via the `create-*` path, a screen it removed is deleted from the target — each approved
  per screen (a removal is destructive), and `screenPairs` reconciled afterward. Rollback is
  **plugin-side**: content-only deltas restore from the snapshot; a screen removal (or wholesale
  re-clone) from a hidden in-canvas duplicate; a screen build by removing the built node
  (keep-last-1). It deliberately **does not** use Figma's REST version-history API — there is no
  node-level edit history and no programmatic version create/restore, so the snapshot doubles as
  the drift baseline and the rollback record.
- **Phase 3 (shipped):** `apply-ds-update`. The command is a thin orchestration that defers to
  the **`ds-update`** skill as its authoritative playbook — the same command↔skill split as
  `sync-screens` → `drift-sync`. The operator reflects an upstream DS change into
  `registry/components.json` and **commits** it (the auditable record); the command reads the
  delta from the registry `git diff`, builds the work set of every mapped file for the platform
  **grouped by `fileKey`** (including cross-file duplicates — the *reverse* of `sync-screens`,
  which flags duplicates rather than editing them), detects deliberate **project overrides** to
  preserve, and — after an approval gate — fans out **one agent per `fileKey`** to **fully
  re-apply** each screen's chrome from the current DS while keeping the overrides. Delta
  semantics resolved to **full re-apply** (not a targeted patch): a DS change ripples through
  chrome in ways a field patch would miss, and overwrite-to-current-DS is idempotent. Backup
  **reuses the sync-screens mechanism** — a hidden in-canvas duplicate of the affected section,
  keep-last-1 per flow — and verify restores on mismatch. Each updated flow is stamped with the
  applied `dsVersion` / `dsAppliedAt` so a re-run skips files already current and a partial
  fan-out resumes.
- **Phase 4 (shipped):** `build-page`. The command is a thin orchestration that defers to the
  **`page-layout`** skill — the same command↔skill split as the others. It makes a Figma page
  holding **many flows** a first-class unit, dropping the earlier one-page-per-flow assumption.
  Sections gain a durable in-file **identity stamp** (`setPluginData`), so idempotency and
  iOS↔Android pairing stop relying on fragile spatial/name matching; `create-android` /
  `create-ios` were retrofitted to write and resolve it. A read-only **classifier** tags each
  section iOS / Android / non-flow by structural signals, and a **declarative packer** recomputes
  the canonical below-the-source layout from flow order every run — idempotent and resumable
  across atomic batches — inside a **managed column band** that leaves unmanaged content
  untouched. Per flow it **builds** a missing counterpart, **relocates** a misplaced one
  (geometry only — never rebuilding content), or no-ops; a pre-existing unstamped section is
  **adopted** (pairing inferred, then human-confirmed) first. Chosen layout strategy is **below +
  make-room reflow** (over parallel-column or a dedicated page). A **layout backup** of every
  managed section's position makes the whole reflow reversible, and after adopting/relocating a
  pair the command may **offer** a `sync-screens` pass for its content (fresh builds are in-sync
  by construction). Behind the same read-only-then-approve gate + per-flow fan-out as
  `apply-ds-update`.

## 8. Open questions

**Resolved for Phase 2:**
- **Diff representation** → per-side **content snapshots** in the mapping (text / imageRefs /
  structHash / CTA label / screen fill), not node hashes.
- **Protecting manual edits** → both-sides-edited **conflicts** are flagged and asked;
  unresolved conflicts are skipped, never clobbered.

**Resolved for Phase 4:**
- **Flow identity on a shared page** → a durable in-file `setPluginData` **identity stamp**
  (`figmaSyncFlowId` / `figmaSyncRole` / `figmaSyncPairId`), not spatial/name matching.
- **Placement without collisions** → a **declarative packer** (recompute canonical layout from
  flow order each run) with a **make-room reflow** inside a **managed column band**.

**Still open:**
- Whether to add a `validate`/`audit` command (drift check between iOS, Android, and registry).

## 9. Decisions captured
- **Plugin**, not a bare skill. *(team install + growth)*
- **Ride on the official Figma MCP**; do not bundle one. *(interactive auth)*
- **Mapping is user-global** at `~/.figma-sync/mappings.json`. *(no git knowledge needed)*
- **Registry is committed & versioned** as the DS source of truth and DS-change audit log.
- **Sections stacked vertically, left-aligned** (source on top) for comparison.
- **Multi-section pages** reconciled by a **declarative packer** (below + make-room reflow),
  with a per-section in-file **identity stamp** so pairing/idempotency never rely on position or
  name. *(a page holds many flows)*
- **Derive, don't copy** from any existing target-platform reference.
- **Bidirectional, mirrored skills.** `create-ios` mirrors `create-android`; reverse iOS DS
  constants live under the registry's `androidToIos` block and ship with empty component keys
  to be discovered against the iOS DS library and committed on first run.
- **Sync is bidirectional** (default iOS → Android), routed through the matching direction
  skill; propagates **content and screen-level structural deltas**.
- **Rollback is plugin-side** — the snapshot for content-only deltas, a hidden in-canvas
  duplicate before a removal or wholesale re-clone, the built node id for a new screen —
  **never** Figma's REST version API.
- **Plan-then-approve gate** — no mutation before the designer explicitly approves the
  presented sync plan.
- **Conflicts and structural changes ask the designer** — no silent clobber and no auto
  build/delete on a guess; structural actions are proposed in the plan and run only on explicit
  approval (a screen **removal** is destructive and approved per screen), backed up and
  reversible.

# figma-sync

A Claude Code **plugin** for translating Figma flows between **iOS (Apple HIG)** and
**Android (Material 3)** — in **either direction** — keeping the two platforms' screens in sync
as content changes, and rolling design-system updates across every file. Built for Tidepool's
design team.

It rides on the **official Figma MCP** (it does not ship its own MCP server), and encodes the
team's platform conventions as reusable **skills** that a set of **slash commands** drive.

## Prerequisites
- **Claude Code** with the **official Figma MCP** installed, enabled, and **signed in**
  (Figma desktop). Verify with the `/mcp` command or by asking Claude to run `whoami`.
- Edit access to the target Figma file.

## Install
```
/plugin marketplace add tidepool-org/figma-sync
/plugin install figma-sync@tidepool
```
Update later with `/plugin marketplace update tidepool` then `/reload-plugins`. (Repo path
during development: `~/src/github/tidepool/figma-sync`.)

---

## Which command do I use?

| I want to… | Command |
|---|---|
| Translate **one** flow to the other platform | `create-android` / `create-ios` |
| Reconcile a **whole page of many flows** for a platform | `build-page` |
| Propagate **content edits** between a flow's two platforms | `sync-screens` |
| Roll a **design-system change** across every mapped file | `apply-ds-update` |

**The shared safety model.** Every command that writes runs the same way: it works **read-only
first**, presents a **plan**, and mutates **only after you explicitly approve** in a following
turn — each behind its own **backup** (layout, content snapshot, or in-canvas duplicate) so any
change is reversible. Detection never edits; approval is never implied by an echoed plan.

**The golden rule.** The target design is always **derived** from the source screens + the
target platform's design system — never copied from a pre-existing target section. Chrome (status
bar, app bar/nav, gesture/home indicator) and font family differ by platform **by design** and
are never treated as drift.

---

## Commands

### `/figma-sync:create-android` · `/figma-sync:create-ios`
**Purpose:** translate a **single** existing flow to the other platform, building a new sibling
section next to the source.
- `create-android` — iOS source section → new **Android (Material 3)** section.
- `create-ios` — Android source section → new **iOS (Apple HIG)** section.

**Arguments:** `[figma url or file key] [optional freeform hints]`
(e.g. a node URL for the page holding the source flow; hints like "the CTA says Get Started").

**When to use:** you have one flow on one platform and want its counterpart. For a page that
holds **many** flows, use `build-page` instead.

**How it runs:** preflight (Figma MCP `whoami`) → discover the source's screens by enumerating
the section's children (never by name pattern) → derive each target screen from the source + the
target design system → place the new section **directly below its source, left-aligned** (via the
`page-layout` packer) → write a durable **identity stamp** on the section → record the flow in
`~/.figma-sync/mappings.json`. Idempotent: if a stamped counterpart already exists it won't
duplicate it.

---

### `/figma-sync:build-page`
**Purpose:** the page-level driver for a page that holds **many flows**. Ensures every flow has
its target-platform section **directly below its source**, building what's missing and tidying
what's misplaced — without touching anything you don't manage.

**Arguments:** `[figma url or file key] [ios|android]`
The platform is the side to **ensure**: `android` builds/relocates an Android section below each
iOS source; `ios` does the reverse.

**When to use:** a page has several flows, or a legacy platform column you want to adopt and
reconcile into a clean below-the-source layout.

**How it runs (read-only → approve → write):**
1. **Classify** every top-level section read-only — platform by structural signals (screen width
   375 vs 412, sidebar `Status` fill, name as a weak tiebreak), stamped vs. legacy, non-flow
   (skipped). Sections whose **name** doesn't say iOS/Android are flagged **unlabeled** with a
   proposed label.
2. **Plan** — per-flow reconcile (**build** a missing counterpart / **relocate** a misplaced one,
   geometry only / **no-op**), inferred **adoption** pairings for legacy sections, a placement
   preview, and the proposed section labels.
3. **Approve** — nothing mutates until you say so. Adoption pairings and section labels apply only
   if you confirm them.
4. **Write** — take a reversible **layout backup**, reflow/relocate existing sections (a
   declarative packer that **makes room by pushing lower sections down**), fan out one builder
   agent per missing flow, verify each sits in its slot, stamp + map. Unmanaged content
   (type-system boards, unrelated frames) is never moved or collision-checked.
5. **Offer** (never auto-run) a `sync-screens` pass for each **adopted / relocated** pair to
   reconcile its content.

---

### `/figma-sync:sync-screens`
**Purpose:** detect **content** drift between a mapped flow's iOS and Android screens and
propagate approved edits in the chosen direction, with a backup/rollback safety net.

**Arguments:** `[flow name or file key] [optional freeform hints]`
The flow name is required to pick which mapped flow (the mapping can hold many). Hints can steer
direction (e.g. "the Android side is ahead"); **default direction is iOS → Android**.

**When to use:** after you've edited content on one platform and want the other to catch up, or
after `build-page` adopts a legacy pair whose sides have diverged.

**How it runs:** preflight (needs a section-level mapping) → **detect** read-only, comparing
content only (text, images, structure, CTA label, screen fill — ignoring chrome and font) →
present a **per-screen plan** with conflicts and structural adds/removes flagged → **approve** →
back up the target → apply the approved **content and screen-level structural** deltas through the
matching direction skill → verify (restore on mismatch) → re-baseline snapshots → reconcile any
sibling mappings. Structural changes are in scope — a screen the source added is **built** on the
target, one it removed is **deleted** (destructive → per-screen approval).

**Adopted pairs (first sync):** a pair adopted by `build-page` has no recorded screen pairs, so
the first run **pairs the screens live** (by name, then order), proposes those pairings for you to
confirm/repair, and — having no baseline — asks direction. On approval it persists the pairings +
snapshots, so later syncs behave normally. **No manual setup required.**

---

### `/figma-sync:apply-ds-update`
**Purpose:** roll a committed **design-system** change across **every mapped file** for a
platform, via a full per-screen re-apply that preserves deliberate project overrides.

**Arguments:** `[ios|android] [summary of what changed]`

**When to use:** iOS or Android ships a new spec. First reflect it into
`registry/components.json` and **commit** — that commit is the auditable record and the version
the run stamps onto each updated flow.

**How it runs:** read the DS delta from the registry `git diff` → build the work set of every
mapped file for the platform (grouped by `fileKey`, **including cross-file duplicates**) →
detect deliberate project overrides to preserve → **approve** → back up each file (a hidden
in-canvas duplicate) → fan out one agent per file to fully re-apply each screen's chrome from the
current DS while keeping overrides → verify (restore on mismatch) → stamp each flow with the
applied `dsVersion` so a re-run **skips files already current** (resumable).

---

## How it works
- **`skills/ios-to-android/`** and **`skills/android-to-ios/`** — the authoritative per-direction
  conventions (layout, typography, header rules, component usage, gotchas). Claude follows the
  matching one when building screens.
- **`skills/page-layout/`** — the multi-section-page playbook: a stable in-file **identity stamp**,
  a read-only **classifier**, a **declarative packer** that lays each section out below its source
  and makes room by pushing lower sections down, geometry-only relocation, legacy-section
  **adoption**, and a reversible **layout backup**. `create-android` / `create-ios` defer to it for
  placement + identity; `build-page` drives it across a whole page.
- **`skills/drift-sync/`** — the content-sync playbook `sync-screens` defers to: detect where a
  mapped flow's content diverged (including live-pairing for adopted pairs), plan, and — only after
  approval — propagate deltas with backup/restore.
- **`skills/ds-update/`** — the design-system playbook `apply-ds-update` defers to: per-fileKey
  work set, override preservation, full per-screen re-apply, and per-flow version stamping.
- **`registry/components.json`** — versioned design-system constants (library + component keys,
  tokens, layout values). Update this when iOS/Android ship new specs; it's the source of truth
  for values, and the skills reference its keys by name.
- **`~/.figma-sync/mappings.json`** — per-designer, user-global record of which files/sections
  have been built and the iOS↔Android screen-node pairs. Each pair may also carry **per-side
  content snapshots** — one record that serves as both the drift baseline and the rollback point.
  Written automatically by the commands. Schema: [`mappings.example.json`](./mappings.example.json).

## Notes
- **Identity is a stamp, not a position or name.** Sections carry an in-file `figmaSyncFlowId` /
  `figmaSyncRole` / `figmaSyncPairId` stamp, so counterparts are matched reliably even with many
  flows on one page or duplicated files that reuse node ids.
- The two sections are **stacked vertically, left-aligned** — **source on top** — for easy
  comparison. On a page with many flows, `build-page` keeps this below-the-source layout and makes
  room by pushing lower sections down; unmanaged content is left exactly where it is.

See [`ARCHITECTURE.md`](./ARCHITECTURE.md) for the design rationale and roadmap.

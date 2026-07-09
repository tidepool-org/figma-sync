---
name: page-layout
description: Build and reconcile platform sections on a Figma page that holds MANY flows (not one page per flow). The authoritative playbook the build-page command defers to, and the placement/identity authority create-android / create-ios defer to. Covers the in-file identity stamp, a read-only classifier preflight, the managed column band, a declarative packer with a make-room reflow, the three-state (build / move / no-op) reconcile with geometry-only relocation, legacy-section adoption via inferred+confirmed pairing, the layout backup, and the optional sync-screens hand-off. Detection is read-only; placement/reflow writes via the Figma MCP only after the command's approval gate. Use during build-page and whenever create-android / create-ios place or reconcile a section.
---

# Multi-section page layout & identity

The authoritative playbook for treating a **Figma page that holds many flows** as a first-class
unit. The translation commands were designed one-page-per-flow: each placed a new platform
section at a blind fixed offset directly below its source, and found an existing counterpart by
inspecting the page spatially / by name. On a page with many flows stacked together, that offset
overlaps the neighbour below, and spatial/name identification cannot say which counterpart
belongs to which source. This skill replaces both with **stable in-file identity** and a
**declarative layout packer**.

`build-page` defers to this skill the way `apply-ds-update` defers to `ds-update` and
`sync-screens` defers to `drift-sync`. `create-android` / `create-ios` also defer here for
**section placement and identity** — they still own their per-screen build conventions
(chrome, typography, header rule, CTA); this skill owns *where a section goes* and *how it is
identified*, not *how a screen is built*.

**This skill owns layout, not content.** Reconciling the content inside a pair of screens is
`sync-screens` / `drift-sync`; this skill only *offers* that hand-off (§8) and never performs it.

**The halves differ on writes.** The classifier (§2), building the work set, and computing the
target layout (§3–§4) make **no Figma edits**. Placement, the reflow, builds, and relocations
(§4–§7) write, and — when driven by `build-page` — **only ever after the command's explicit
approval gate**; a layout backup is taken before the first move and can restore it.

> Prerequisite: the official **Figma MCP** must be running and authenticated
> (`mcp__plugin_figma_figma__whoami`). This skill drives **read-only** `use_figma` for the
> classifier and the layout computation, and **atomic** `use_figma` writes for placement,
> reflow, builds, and relocations.
>
> **Source of truth for values:** layout constants live in
> `${CLAUDE_PLUGIN_ROOT}/registry/components.json` under `layout` — read them from there. This
> doc teaches the *method* and names the registry keys; any literal shown inline is
> **illustrative**, not authoritative. When prose and registry disagree, the registry wins.

---

## 0. Golden rule — one page, many flows, nothing else touched

Operate only on the flows you are asked to manage, inside their **column band** (§3). Every
other frame on the page — a type-system board, scratch work, an unrelated section — is left
exactly where it is. The page may be organised unpredictably; never assume a layout, **read it**
(§2), and move only what identity and the band say is yours.

---

## 1. Identity stamp — the source of truth for a flow

Every managed section carries a durable identity written into the node with Figma
`setPluginData`, under the namespace `figma-sync`:

- `figmaSyncFlowId` — the flow's stable slug (e.g. `a-day-in-the-life`), shared by both
  counterpart sections of a flow.
- `figmaSyncRole` — `ios` or `android`.
- `figmaSyncPairId` — links the two counterpart sections; both sides of a flow share the same
  `figmaSyncPairId`.

**Read/write it via `use_figma`:** `node.setPluginData('figmaSyncFlowId', id)` /
`node.getPluginData('figmaSyncFlowId')`. Resolve two questions from the stamp, never from
position or name:

- **Idempotency** — "does a counterpart already exist for this flow?" → look for a section whose
  `figmaSyncPairId` matches the source's and whose `figmaSyncRole` is the target platform.
- **Pairing** — "which iOS section goes with which Android section?" → equal `figmaSyncPairId`.

Every builder writes the stamp: `create-android`, `create-ios`, and `build-page`. The `flowId`
is also recorded in `~/.figma-sync/mappings.json` (the mapping's `flowId` field) so the
user-global mapping cross-links to the in-file stamp. The stamp holds **identity only** — never
layout coordinates (the layout backup lives in-canvas, §7) or chrome/content.

> A pre-existing section built before stamping exists carries **no** stamp; it is **adopted**
> (§6) — its pairing inferred and confirmed, then the stamp written — before it is treated as a
> managed section.

---

## 2. Classifier preflight (read-only)

Before any write, build the authoritative picture of the page by **enumerating the page's
top-level section frames and their direct children** — never by name-pattern (a name-regex
silently drops off-convention frames and mislabels). For each section:

1. **Enumerate direct children** (id / name / type / x / y / w / h) via a read-only `use_figma`.
   Drop the `Status` sidebar; the remaining child frames are its screens (the same
   discover-by-children rule the direction skills use).
2. **Classify the platform** from structural signals, not the name alone:
   - **Screen width** — iOS screens are `layout.iosScreenWidth` (375), Android are
     `layout.androidScreenWidth` (412).
   - **Sidebar `Status` fill** — iOS sidebar is black (white text); Android is white (dark text).
   - **Section name** — a weak tiebreak only (e.g. an "ANDROID" / "iOS" label), never decisive.
3. **Tag** each section `ios`, `android`, or `non-flow` (a type-system board, scratch, anything
   whose children aren't a screen row). Read any existing identity stamp (§1) at the same time.

**Confirm ambiguous sections with the designer.** If the signals disagree (e.g. 412-wide screens
under a black sidebar) or a section can't be confidently classified, present it for a decision
rather than guessing. The classifier makes **no edits**; its output is the table
`build-page` presents in its plan: which sections are managed, their platform, whether each is
stamped or a legacy-adoption candidate, and which are non-flow (skipped).

---

## 3. The managed column band

The managed flows occupy a single vertical **column band** — the x-range spanned by the sections
you manage (source sections plus their target slots), left-aligned at `layout.firstScreenX` less
its leading inset. The band is the contract that protects everything else:

- The packer (§4) computes positions and checks collisions **only within the band**.
- Sections **outside** the band — a right-hand type-system board, a legacy Android column you
  are not adopting this run — are **never moved, resized, or collision-checked**. They are
  untouched *by construction*, not by case-by-case avoidance.
- If a managed section's width would grow into another band (e.g. re-spacing widens it past a
  neighbour to the right), **flag it** for the designer — do **not** auto-move the cross-band
  neighbour.

A legacy section living outside the band (the right-column Android in a typical file) enters the
band only when it is **adopted** (§6) and then **relocated** into its canonical slot (§5).

---

## 4. Declarative packer & make-room reflow

Placement is computed by **recomputing the entire canonical layout from the logical flow order
every run**, then moving each managed section to its computed target — never by imperatively
"inserting one section and shoving the neighbours down by a delta" (that double-shifts on re-run
and corrupts on a mid-batch failure).

**Canonical order** — interleave each flow's counterpart directly under its source, in the
source sections' existing top-to-bottom order:

```
[ source A, target A, source B, target B, source C, target C, … ]
```

**Compute targets** — stack them down the band with the registry gaps:

```
y = bandTop
for section in canonicalOrder:
    target[section].x = sourceColumnX        # left-aligned in the band
    target[section].y = y
    y += section.height + gap                # gap = layout.sectionStackGap between a flow's
                                             # source/target; a flow-to-flow gap of the same
```

- **Below, left-aligned, never to the side.** A target section sits directly below its source
  (`x = source.x`), preserving the screen-for-screen comparison the direction skills produce.
  This is the retained-and-generalised form of the old fixed-offset rule.
- **Column alignment feeds heights.** Any screen re-spacing to `layout.screenPitch` (for
  column-aligned comparison, per `layout.columnAlignAcrossSections`) changes a section's height,
  so **re-space first, then measure, then pack** — heights are packer inputs.
- **Screens sit at `layout.screenBaselineY`** within each section; `layout.sidebarWidth` /
  section radius / top+bottom padding are unchanged from the direction skills.

**Idempotent + resumable.** Because the layout is recomputed from order every run, re-running
with all counterparts already in slot **reasserts identical positions** (a no-op), and a run
interrupted mid-reflow simply **completes on re-run**. This is what lets the reflow cross the
atomic `use_figma` boundary safely: it is a sequence of independent "move section S to
`target[S]`" writes in **atomic ≤10-op batches**, screenshotting between batches. Moving a
section *frame* translates its children rigidly — safe, unlike a *resize* (see §9).

**Make room by pushing down.** When a target slot would overlap the section beneath it, the
recomputed stack has already accounted for it: every section below the insertion point gets a
larger `y` and moves down. There is never a "no room → place to the side" fallback — the packer
makes room within the band.

---

## 5. Three-state reconcile

For each managed flow, reconcile its target-platform section to its canonical slot. Build and
move are the **same** "place into `target[S]`" operation under the packer; the only difference is
whether the section already exists:

| State | Condition | Action |
|---|---|---|
| **Build** | no counterpart (no matching `figmaSyncPairId`) | build the section via the direction skill, stamp it (§1), place it in its slot |
| **Move** | counterpart exists but not in its slot | **relocate it — geometry only** (§5.1) |
| **No-op** | counterpart already in its slot | nothing |

### 5.1 Relocation moves geometry only

A move changes **canvas geometry only** — the section's `x` / `y`, and if needed its screens'
pitch for column alignment. It **must not rebuild, re-derive, or clobber the section's content**,
even if that content looks stale: an existing counterpart may carry deliberate hand-work.
Content divergence is handled separately by the **optional `sync-screens` hand-off** (§8), never
inline here. (This is the same preserve-what-the-designer-made ethos the DS re-apply follows for
project overrides.)

Re-runs converge: once a section is in its slot it is a no-op, so the reconcile is idempotent.

---

## 6. Legacy-section adoption

A counterpart built before identity stamping exists (e.g. the right-hand Android column in an
older file) has no stamp, so it can't be matched by §1. Adopt it — **once, human-gated** — before
it can be reconciled:

1. **Infer the pairing** to its source flow: first by **flow-name match**; if that's ambiguous,
   by **order** (the Nth target section ↔ the Nth source section) or **screen count** as a
   tiebreak.
2. **Present the inferred pairing** in the classifier table for the designer to **confirm** —
   never adopt or pair silently.
3. On confirmation, **write the identity stamp** (§1) onto both sections (shared
   `figmaSyncPairId`) and record the `flowId` in the mapping.

Once adopted, the section is an ordinary managed section: relocatable (§5), stamped, mapped. An
adopted counterpart may differ in content from its source — exactly the case §8 targets.

---

## 7. Layout backup & restore

The reflow is high-blast-radius: it moves many pre-existing sections, including freshly-adopted
ones that may have been hand-placed. Before the **first** move:

- **Snapshot** every managed section's `id → (x, y)` (and width/height) — cheap coordinates only.
  Record it for the run (keep-last-1 per run, like the content backups).
- **No layout backup → do not reflow.**

**Restore** re-writes each section's saved `(x, y)` — a pure inverse of the reflow. This is
distinct from, and lighter than, the in-canvas hidden-duplicate content backup: that backup
restores a section's *internals*; this one restores the page-level *positions* of many sections.
A failed or aborted `build-page` run can roll the arrangement back without touching any section's
internals.

---

## 8. Optional `sync-screens` hand-off

`build-page` owns layout; content reconciliation belongs to `sync-screens` / `drift-sync`. After
a counterpart has been **relocated, stamped, and mapped**, the command may **offer** — never
auto-run — a per-flow `sync-screens` hand-off, and **only for adopted / relocated pairs**:

- A **freshly built** counterpart is in-sync with its source by construction — never offer it.
- The offer routes through `sync-screens`' own approval gate.
- **Sequence strictly:** relocate → stamp → write mapping → *then* offer (`sync-screens` reads
  the mapping).

**Caveat — first sync on an adopted pair is live-only.** `create-*` seed a drift baseline at
build time (both sides agree then); an adopted pair has no trustworthy baseline, so its first
`sync-screens` runs in **live-only detection** and cannot auto-attribute drift direction — it
surfaces the diffs and asks direction. This is expected, not a bug.

---

## 9. Gotchas

- **Never name-regex the sections or screens** — enumerate direct children and classify by
  structural signals (§2). Off-convention frames (terminal screens, type-system boards) defeat a
  name pattern.
- **Identity is the stamp, not the position.** Two flows on one page, dev copies that reuse node
  ids across files — only the stamp disambiguates. Keep pairing keyed **within a `fileKey`**; a
  different `fileKey` with the same node ids is a separate duplicate file.
- **`setPluginData` travels with a node on duplication** — a duplicated file carries the same
  stamp values, consistent with the "dev copies reuse node ids" reality. This is why pairing is
  scoped per `fileKey`.
- **Re-space screens *before* packing** — screen pitch changes section height, and height is a
  packer input (§4). Packing on pre-respace heights leaves gaps or overlaps.
- **Move, don't resize, to reflow.** Translating a section frame moves its children rigidly.
  *Resizing* a section moves children with center/scale constraints (the sidebar drifts ~half the
  width delta) — after any unavoidable resize, re-snap the sidebar to `x=0` with
  `constraints = {horizontal:'MIN', vertical:'STRETCH'}`.
- **Reflow in atomic ≤10-op batches, screenshot between** — `use_figma` is atomic; a failed batch
  makes no change. The declarative packer makes a partial reflow safe to resume (§4).
- **Relocation is geometry only** (§5.1) — never rebuild an existing counterpart's content; route
  content drift to the §8 hand-off.
- **A width that grows past the band is flagged, not auto-fixed** (§3) — never move a cross-band
  neighbour to make room horizontally.
- **New sections land on `currentPage`, which resets every `use_figma` call.** An invocation
  starts on the *first* page, so a section created before `setCurrentPageAsync` — or a top-level
  node appended while `currentPage` is stale — silently lands on the wrong page. **Parent every
  new (and relocated) section to the target page by id** (`targetPage.appendChild(section)`) and
  **assert `section.parent.id === pageId`**. Verify placement by **parent page, not a
  screenshot** — a screenshot renders a node on any page and hides the error, so a screenshot-only
  check passes a misparented section.
- **App-bar / nav titles** — a freshly built section inherits the direction skills' title rule
  (title = flow name, not a content H1); verify after building.

---

## 10. Reference

- Placement / identity authority for `create-android` (`ios-to-android`) and `create-ios`
  (`android-to-ios`); the page driver is the `build-page` command.
- Content sync is `sync-screens` / `drift-sync`; DS-state sync is `apply-ds-update` / `ds-update`.
- Layout constants: `registry/components.json` → `layout` (`iosScreenWidth`, `androidScreenWidth`,
  `screenPitch`, `firstScreenX`, `screenBaselineY`, `sidebarWidth`, `sectionStackGap`,
  `columnAlignAcrossSections`).
- Figma REST has no node-level edit history and no programmatic version create/restore, so the
  layout backup (§7) — like the content/DS backups — is **plugin-side**, held in-canvas / in the
  run record.

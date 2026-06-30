---
name: drift-sync
description: Detect iOS↔Android content drift for a mapped Figma flow and produce a reviewable sync plan — read-only. Covers screen enumeration, the content-vs-chrome boundary, the divergence algorithm, the per-side snapshot format, direction handling, and the sync-plan format the sync-screens command presents. Use during sync-screens detection (via the Figma MCP, read-only use_figma).
---

# Drift detection & sync-plan generation

The authoritative playbook for the **read-only half** of the sync workflow: read a mapped
iOS↔Android flow, detect where the two platforms' **content** has diverged, and emit a
structured **sync plan** for the designer to refine. This skill **makes no Figma edits** —
it enumerates, reads, compares, and reports. Propagation and rollback are a separate concern
and are not taught here.

> Prerequisite: the official **Figma MCP** must be running and authenticated
> (`mcp__plugin_figma_figma__whoami`). This skill drives **read-only** `use_figma`.
>
> **Source of truth for values:** component keys, design tokens, and layout constants live
> in `${CLAUDE_PLUGIN_ROOT}/registry/components.json` — read them from there. This doc teaches
> the *method* and names the registry keys; any literal shown inline is **illustrative**, not
> authoritative. When prose and registry disagree, the registry wins.

---

## 0. Golden rule — compare content, not chrome

Detection looks **only** at content. **Chrome** (status bar, top app bar / nav bar, gesture
nav / home indicator) and **font family** differ between the platforms *by design* — the
target was derived from the source under exactly those platform rules — so a difference there
is **never drift** and is never reported. This is the mirror of the translation skills' golden
rule: the platforms are *meant* to differ in chrome and type family; they are meant to **agree
in content**.

---

## 1. Screen enumeration & pairing

Discover screens the same way the translation skills do — **enumerate each section's direct
children, never by name-pattern** (flows include off-convention terminal screens that a regex
silently drops).

1. Read-only `use_figma` returning `iosSection.children` and `androidSection.children`
   (`id / name / type / x / y / w / h`). Drop the `Status` sidebar; sort each by `x`.
2. **Pair by the stored `screenPairs` node ids** in the mapping — not by name or order.
3. Reconcile against the live children:
   - A live child whose id is in **neither** side's `screenPairs` → an **added** screen.
   - A stored `screenPairs` id **absent** from the section's live children → a **removed** screen.
   - Added/removed screens are **flagged and asked**, never built or deleted in this workflow.

---

## 2. What counts as content (the comparison surface)

See **§3 (screen skeleton / anatomy)** of the translation skills (`ios-to-android` and its
mirror `android-to-ios`) for where each piece lives. For each paired screen, compare **only**:

- **Text** — the `characters` of every text node inside the `Content AL` subtree, in document
  order. **Ignore `fontName`** (family/weight differ by platform by design).
- **Images** — the `imageHash` of every `IMAGE` paint in the content subtree (fills and image
  nodes), in order → the `imageRefs` list.
- **Structure** — child count / order / node types of the content subtree → folded into
  `structHash` (§4).
- **CTA label** — the `characters` of the **visible** `Button / Primary` label. Read it **only
  from the visible subtree, checking *ancestor* visibility** — hidden sibling frames carry
  stale duplicate labels (e.g. a hidden pagination frame holding "Finish").
- **Screen fill** — the screen frame's own fill.

**Explicitly NOT content** (never reported as drift): status bar, app bar / nav, gesture nav /
home indicator, CTA radius/shape, font family, and content margins — all platform chrome.

---

## 3. Divergence algorithm

Live divergence is the **core signal** — there is no node-level edit history to read, so
"what changed" is reconstructed from the live tree plus any stored snapshot. For each pair:

1. **Live cross-platform diff.** Compare the source-side content against the counterpart's
   content **now**. Because the target was *derived* from the source, the content
   (text / images / structure / CTA label / fill) **should match**; a mismatch is a candidate
   drift. Font family differing is expected and ignored (§0).
2. **Snapshot attribution** (when a stored per-side snapshot exists): diff **each side against
   its own snapshot** to attribute the change:
   - source changed vs its snapshot, counterpart unchanged → **clean delta** (propose
     source → counterpart).
   - counterpart changed, source unchanged → **reverse delta** (the counterpart is ahead;
     propose counterpart → source, or flag, per the resolved direction).
   - **both** changed vs their snapshots → **conflict**: flag which side(s) and **ask the
     designer** — never auto-clobber an edited counterpart.
   - neither changed vs snapshot but a live diff exists → the snapshot is stale / pre-dates an
     edit; treat as a live-only candidate and ask.
3. **No snapshot → live-only detection.** Report the live diff as a candidate delta and rely
   on the designer to confirm direction. (Older mapping entries with no snapshot degrade to
   this path — that is the documented back-compat behavior.)

---

## 4. Snapshot format

Mirror the mapping schema: an optional per-side `iosSnapshot` / `androidSnapshot` under each
`screenPair`, each `{ texts[], imageRefs[], structHash, ctaLabel, screenFill }`. All captured
**read-only** via `use_figma`:

- **`texts`** — ordered `characters` of the content text nodes.
- **`imageRefs`** — ordered `imageHash` of the `IMAGE` paints in the content subtree.
- **`structHash`** — walk the content subtree in document order, emit one compact token per
  node (its `type` plus its order among siblings), join, and store a short **stable** hash of
  that string. **Exclude** volatile data — node ids, absolute coordinates, and `fontName` — so
  that re-fonting or repositioning does **not** register as a structure change.
- **`ctaLabel`** — the visible `Button / Primary` label (ancestor-visibility checked), or
  `null` when the screen has no CTA (terminal screens).
- **`screenFill`** — the screen frame's fill, or `null`.

**Content only** — never store chrome or font family. The snapshot doubles as the drift
baseline and the rollback record, so it must capture exactly the content surface §2 compares.

---

## 5. Direction

Default **iOS → Android**, selectable. Detection itself is **symmetric** — the divergence
algorithm runs identically in either direction; the direction only decides which side is the
"source" for the proposed action and orients the live diff. The resolved direction is recorded
on the flow (`lastSyncDirection`) when a sync actually runs — **not in this read-only pass**.

---

## 6. Sync-plan format

The plan is the deliverable the command presents for refinement — **and then it stops; no
mutation**. Structure it per screen pair, plus an overall summary.

**Per screen pair:**
- **pair** — screen name + the two node ids; **direction**.
- **detected delta fields** — which of `text / images / structure / CTA label / screen fill`
  diverged.
- **conflict flag** — whether both sides were edited, and **which side(s)**.
- **structural notes** — added / removed screens; terminal screen with no CTA / no top chrome.
- **proposed action** — e.g. *propagate text + image, source → counterpart*; or *conflict —
  needs a decision*; or *no change*.

**Overall summary:** number of pairs; how many carry deltas; how many are conflicts; added /
removed screen counts; the resolved direction. Close by inviting refinement and stating
plainly that **no edits were made**.

---

## 7. Gotchas

- **Enumerate by section children, never by name-regex** — terminal / off-convention screens
  (e.g. an end screen) get silently dropped otherwise, and the plan misses a pair.
- **`use_figma` is read-only in this workflow** — detection performs no writes. Work in small
  read steps; `use_figma` is atomic, so a failed read changes nothing.
- **CTA label needs the *ancestor*-visibility check** — a node's own `visible=true` is not
  enough; a hidden ancestor frame can carry a stale duplicate label.
- **Ignore `fontName` in text comparison** — family differs by platform by design; comparing it
  produces false drift on every screen.
- **`structHash` must exclude node ids, absolute coords, and fonts** — otherwise every re-font
  or canvas reposition false-positives as a structure change.
- **Terminal / menu screens have no CTA and no top chrome** — `ctaLabel` is `null`; do **not**
  report a "missing CTA" as drift (the source lacks it too).
- **Cloned frames carry source strokes / fills** — compare the screen frame's intended fill,
  not an inherited stroke artifact.
- **Snapshots are content-only** — a chrome or font-family difference is never drift, with or
  without a snapshot.

---

## 8. Reference

- Forward translation method and screen anatomy (§3): the `ios-to-android` skill.
- Reverse translation method (mirror): the `android-to-ios` skill.
- Mapping + per-side snapshot schema: `${CLAUDE_PLUGIN_ROOT}/mappings.example.json`.

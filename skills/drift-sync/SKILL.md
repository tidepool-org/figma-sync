---
name: drift-sync
description: Detect iOS↔Android content drift for a mapped Figma flow, produce a reviewable sync plan, and — once the designer approves — propagate the content and screen-level structural deltas in either direction with an in-canvas/snapshot rollback. Covers screen enumeration and structural reconciliation, the content-vs-chrome boundary, the divergence algorithm, the per-side snapshot format, direction handling, the sync-plan format, backup/restore, content and structural delta application, and conflict execution. Detection is read-only; propagation writes via the Figma MCP. Use during sync-screens.
---

# Drift detection, sync-plan generation & propagation

The authoritative playbook for the **full** sync workflow: **detect** where a mapped
iOS↔Android flow's **content** has diverged and emit a reviewable **sync plan** (read-only),
then — after the designer approves — **propagate** the content and screen-level structural
deltas in the chosen direction with a backup/rollback safety net.

**The two halves differ on writes.** Detection (§0–§6) makes **no Figma edits** — it
enumerates, reads, compares, and reports. Propagation (§7–§10) writes, and **only ever after
an approved plan**; it backs up before it mutates and can restore. The `sync-screens` command
runs detection first, stops at the reviewed plan, and executes the mutation half only on
explicit approval.

> Prerequisite: the official **Figma MCP** must be running and authenticated
> (`mcp__plugin_figma_figma__whoami`). This skill drives **read-only** `use_figma` for
> detection and **atomic** `use_figma` writes for propagation.
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
3. Reconcile against the live children — added / removed screens are **structural deltas**
   (screen-set changes, as opposed to a paired screen's content), and they are **in scope**
   for propagation (§8.1). Classify each, oriented by the resolved direction (source → target):
   - A live **source** child whose id is in no `screenPair` → a **source-add**: the source
     gained a screen the target lacks → **build** the target twin (additive).
   - A live **target** child whose id is in no `screenPair` → a **target-orphan**: the target
     has a screen the source lacks → **remove** it from the target to mirror the source (or, if
     the designer flips the direction, reverse-add it to the source).
   - A `screenPair` whose **source** id is absent live → the source screen was **deleted** →
     remove the now-orphaned target twin and drop the pair.
   - A `screenPair` whose **target** id is absent live → the target twin was deleted while the
     source remains → **rebuild** the target twin (additive) and refresh the pair.
   Structural deltas are **carried into the plan** with a proposed **build** / **remove** action
   (§6) and are **never mutated before approval + backup** (§7). A **removal is destructive**,
   so it is executed only when the designer **explicitly approves that screen** (§9).

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

> **Locating the content subtree — don't give up when `Content AL` is absent.** Most screens
> wrap content in a `Content AL` node, but **terminal / off-convention screens do not** — an
> End / menu screen may wrap it in a `Container` (iOS) or bare `Content` (Android) frame
> instead. If you find `Content AL` on one side and **nothing** on the other, you'll
> **false-positive the whole screen as text + structure drift** (this bit a real run on the End
> screen). Fall back: when `Content AL` is missing, take the screen's content region as **all
> text / image under the screen minus the chrome containers** (status bar + its clock, app bar /
> nav, gesture nav / home indicator). If both sides' content-minus-chrome match, it is **not**
> drift.

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

> **Do the whole detection in one read.** Run every pair's signature diff **and** the structural
> reconciliation (§1) inside a **single** read-only `use_figma` that takes the `screenPairs` +
> section ids and **returns only the deltas** — skip the content of clean pairs. One round trip
> beats one-per-screen, and returning full content for the (usually many) clean pairs just burns
> context. Include per-pair which fields diverged and the differing values only.

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
- **structural note** — a structural delta (§1): the proposed **build** (missing target twin)
  or **remove** (orphaned target screen) action, its direction, and — for a removal — an
  explicit **destructive** flag; plus terminal-screen notes (no CTA / no top chrome).
- **proposed action** — e.g. *propagate text + image, source → counterpart*; or *build target
  twin, source → target*; or *remove orphaned target screen (destructive)*; or *conflict —
  needs a decision*; or *no change*.

**Overall summary:** number of pairs; how many carry content deltas; how many are conflicts;
added / removed screen counts (each with its proposed build / remove action); the resolved
direction. Close by inviting refinement and stating plainly that **no edits were made**.

---

## 7. Backup before mutating

**Nothing is mutated until a backup exists.** Take the backup **once per run**, before the
first write, and key it to what the delta will change:

- **Content-only deltas** (text / image / fill / CTA label, no content-structure change): the
  per-side content snapshot in `mappings.json` (§4) **is** the backup — no canvas node is
  needed. Restore re-writes those snapshot fields back onto the target's content surface.
- **Structure-changing deltas** — a content delta whose **structure** field changed (the sync
  must re-clone `Content AL` wholesale), **or** a **screen removal** (§8.1): **duplicate** the
  affected target screen(s) into a **hidden frame** (`visible=false`) on the page, named for
  the run (e.g. `backup — <flow> — <run>`), and record the backup node id **plus** the
  snapshot in `mappings.json`. Restore swaps the duplicate back **wholesale** — re-link it into
  the section and re-snap it to the pair's x/y slot; for a removal, this un-deletes the screen.
- **Screen builds** (§8.1 — a source-add or a target-twin rebuild): nothing pre-exists to
  duplicate, so the rollback record is the **built node id** — record it in `mappings.json`.
  Restore **removes** the built node and drops its new `screenPair`.

The hidden frame keeps the column-aligned section layout undisturbed (it never occupies a
screen slot). **keep-last-1:** on a successful run, retain only the most recent backup per flow
and delete any older backup frame.

---

## 8. Applying an approved content delta

For each **clean** screen delta in the approved plan, re-run **only the content steps** of the
matching direction skill — leaving chrome and platform conventions intact:

- re-clone / refresh `Content AL` from the source,
- re-font per the typography rule (Roboto for → Android, SF Pro Text/Display for → iOS;
  preserve the source's metrics),
- re-copy the screen fill,
- update the CTA label.

Route by direction: iOS → Android through the `ios-to-android` skill; Android → iOS through the
`android-to-ios` skill. Default **iOS → Android** unless the plan resolved otherwise.

Apply is **overwrite-to-source-content** — it sets the target's content surface to match the
source's *current* content, so it is inherently **idempotent**: running it twice converges on
the same target state, and there is no incremental diff to double-apply.

The **golden rule** still holds — derive from the source + the target design system, and
**never invent UI the source lacks** (a terminal screen has no CTA / toolbar; do not add one).
Work in **atomic ≤10-op batches** and **screenshot between batches**.

### 8.1 Applying an approved structural delta

Structural deltas change the target's **screen set**, not just a screen's content. Execute only
the actions the designer approved (§9), in the resolved direction:

- **Build** (source-add / target-twin rebuild): construct the target screen the **same way the
  `create-*` commands do** — route to the matching direction skill (`ios-to-android` for
  iOS → Android, `android-to-ios` for the reverse), or offload to the `screen-builder` /
  `screen-builder-ios` agent — passing the source screen node id, its order, and `hasBack`.
  Slot the new screen into the target section at the **column-aligned x/y** for its order
  (registry `firstScreenX` + `screenPitch`, `screenBaselineY`), re-spacing neighbors if needed.
  This is a **`create-*` build, not a content sync** — build full chrome + CTA from the target
  DS while deriving the content from the source; **never invent UI the source lacks**.
  - **Mid-flow insert recipe (source-add at column *k*):** the source may insert a screen
    *between* existing ones, not just append. Derive *k* from the source screen's **order in the
    flow**, not its raw x. To keep both sections column-aligned: (1) shift every target screen at
    column ≥ *k* **right by one `screenPitch`**; (2) build the twin into the opened column at the
    shared `x = firstScreenX + k*screenPitch`, `y = screenBaselineY`; (3) grow the target
    **section width by one `screenPitch`** (and sidebar height only if the built screen is taller
    than the current max screen bottom + `sectionBottomPadding`).
  - **Verify the built screen's app-bar / nav title reads the flow name**, not the source's
    content H1 — see the §11 gotcha.
- **Remove** (target-orphan / source-deleted twin): after the backup exists (§7), delete the
  target screen and re-close the column gap so the layout stays aligned. **Destructive —
  approved per screen only.**

Then reconcile the mapping (§10): **add** a `screenPair` (with fresh both-side snapshots) for
each built screen; **drop** the `screenPair` for each removed screen. Work in **atomic ≤10-op
batches** and **screenshot between**.

---

## 9. Conflict execution

Apply **exactly** the resolution the designer chose during plan refinement (overwrite / keep /
per-field). **Never re-ask** at execution time and **never fall back to a silent default.**

- A conflict the plan left **unresolved** is **not eligible** for mutation — skip that pair and
  report it; do not guess.
- **Structural deltas** (added / removed screens, §1 & §8.1) are **in scope**: execute the
  **build** / **remove** the designer approved. A structural delta the plan left **unapproved**
  is **skipped** and reported — never auto-build or auto-delete on a guess. A **removal**
  requires the designer's **explicit per-screen approval** (it is destructive) and a backup
  (§7) before it runs.

---

## 10. Post-sync: rewrite snapshots & verify

After a screen's content is applied and screenshot-verified:

1. **Rewrite both per-side snapshots** for each synced pair — source and target now agree, so
   this re-baselines the drift comparison **and** refreshes the rollback record (§4).
2. **Reconcile `screenPairs` for structural deltas** — **add** a new pair (both node ids +
   fresh both-side snapshots) for each **built** screen; **remove** the pair for each
   **removed** screen.
3. **Update the flow** `lastSyncedAt` (today) and `lastSyncDirection` (the direction just run).
4. **Verify each touched screen** against the source's content — for a **build**, verify the
   new target screen against its source; for a **removal**, verify the screen is gone and the
   column re-closed. On a mismatch, **restore from the backup** (§7) and report the failure —
   **never leave a silent partial apply.**
5. **Reconcile related mappings** (treat as part of the run, not optional). After this flow's
   entry is updated, check every **other** flow in `mappings.json` for staleness from the deltas
   you just applied — especially **structural** ones. A sibling whose **`fileKey` matches** the
   edited flow references the *same physical nodes*: update its `screenPairs` / snapshots the
   same way. A flow with the **same node ids but a different `fileKey`** is an **independent
   duplicate file** (Figma duplicates preserve node ids across files) — your change did **not**
   reach it; at most **flag** it for its own run, never edit it. Gate "shared" on `fileKey`,
   never on bare node ids. State the outcome (siblings updated, duplicates flagged, or nothing
   affected) in the report.

Run the **keep-last-1** backup cleanup (§7) only after the whole run succeeds.

---

## 11. Gotchas

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
- **Never mutate before a backup exists** — content-only deltas back up via the snapshot; a
  screen removal (or a structure-changing content delta) needs the hidden duplicate frame
  first; a screen build records its built node id as the rollback record (§7).
- **A content delta applies content only** — re-clone `Content AL` + re-font + fill + CTA label;
  do **not** rebuild chrome or re-translate the whole screen. Rebuilding a whole screen is a
  **structural** delta (§8.1) that routes through the `create-*` build path — not a content sync.
- **Structural deltas are in scope, but a removal is destructive** — build a missing target
  twin via the `create-*` path and slot it at the column-aligned x/y; delete an orphaned target
  screen only after the hidden-duplicate backup and the designer's **explicit per-screen**
  approval, then re-close the column gap. Never auto-build or auto-delete on a guess.
- **A built screen needs a new `screenPair` (with snapshots); a removed screen's pair is
  dropped** — otherwise the next detection pass re-flags the same structural delta.
- **Execute the designer's conflict choice verbatim** — no re-asking, no silent default; an
  unresolved conflict is skipped and reported, never guessed.
- **Backups live in a hidden frame** (`visible=false`) so they never occupy a screen slot or
  disturb the column-aligned layout; **keep-last-1** after a successful run.
- **Re-baseline both snapshots after a successful apply** — a stale post-sync snapshot would
  make the next detection pass re-report the change you just propagated.
- **A built screen's app-bar / nav title must be the flow name, not the content H1** — the
  source's nav title slot is often an unedited placeholder (literally "Title"), so a builder can
  fall back to the screen's content headline and duplicate it in the app bar. After any build,
  screenshot and confirm the title reads the flow name; fix it (load font → set `characters`) if
  not.
- **Off-convention screens hide their content outside `Content AL`** — an End / menu screen may
  wrap content in `Container` / bare `Content`; comparing a found `Content AL` against an empty
  result false-positives the whole screen (§2). Fall back to content-minus-chrome.
- **Reconcile siblings by `fileKey`, not node id** — duplicated Figma files reuse node ids, so a
  flow with matching node ids but a **different `fileKey`** is a *separate file* your change
  never touched. Update same-`fileKey` siblings; only **flag** different-`fileKey` duplicates
  (§10 step 5).

---

## 12. Reference

- Forward translation method and screen anatomy (§3): the `ios-to-android` skill.
- Reverse translation method (mirror): the `android-to-ios` skill.
- Mapping + per-side snapshot schema: `${CLAUDE_PLUGIN_ROOT}/mappings.example.json`.

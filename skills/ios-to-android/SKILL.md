---
name: ios-to-android
description: Translate an iOS-designed Figma flow into a matching Android (Material 3) section. Conventions, design-system component keys, section/screen layout, typography, header rules, and per-screen build steps. Use when creating or extending an Android version of iOS screens in Figma via the Figma MCP (use_figma).
---

# iOS → Android Figma translation

The authoritative playbook for translating an iOS-designed screen flow into a matching
**Android (Material 3)** section in Figma. We design iOS-first, then derive Android.
These conventions keep the Android output faithful to iOS intent **and** correct for
Material 3 — without copying any pre-existing Android work.

> Prerequisite: the official **Figma MCP** must be running and authenticated
> (`mcp__plugin_figma_figma__whoami`). This skill drives `use_figma`.
>
> **Source of truth for values:** component keys, design tokens, and layout constants live
> in `${CLAUDE_PLUGIN_ROOT}/registry/components.json` — read them from there. This doc teaches
> the *method* and names the registry keys; any literal number or hex shown inline is
> **illustrative** (a current value from the Loop flow), not authoritative. When prose and
> registry disagree, the registry wins. Keeping the literals out of the prose is what stops
> this skill from going stale as the design system evolves.

---

## 0. Golden rule — derive, don't copy
Derive the Android flow **only** from (1) the iOS screens and (2) the Android / Material 3
design-system libraries. If a finished Android section exists elsewhere as a reference,
**do not look at it to drive the design** — only its container structure may be mirrored,
and only compared *after* you're done.

---

## 1. Page & container structure

One Figma page per flow, holding **two sibling section frames, stacked vertically and
left-aligned** — iOS on top, Android directly below — for screen-for-screen comparison
with minimal horizontal scrolling:

| Section | Name | Screen width | Section background | Sidebar |
|---|---|---|---|---|
| Source | iOS section (existing) | 375 | as-is (read its fill) | black `Status`, white text |
| Target | `ANDROID Section` | 412 | `tokens.sectionBackground` | **white** `Status`, dark text |

### Sidebar (`Status` frame)
- Left rail, `layout.sidebarWidth` wide, full section height; flow title inherited from the
  cloned iOS sidebar (Work Sans Bold 92 in Loop — comes along with the clone, not set here).
- Android: **white fill**, dark text (`tokens.sidebarText`), flush at `x=0, y=0`.
- **Remove the stroke** after cloning the iOS sidebar (it carries a black stroke that
  is invisible on black but shows as a border on white): `strokes = []`.

### Section layout rules
Every constant below is a key in `registry.layout` — read them at runtime; parenthetical
numbers are the current Loop values, for orientation only.
- **Placement:** `ANDROID Section` is left-aligned with the iOS section (`x = iOS.x`) and
  sits **directly below it** (`y = iOS.y + iOS.height + sectionStackGap`). Stack vertically —
  never place it to the right of iOS.
- **Section frame:** `cornerRadius = sectionCornerRadius`, `clipsContent = true`,
  background `tokens.sectionBackground`.
- **Screen row:** single horizontal row in flow order at fixed pitch —
  `screen.x = firstScreenX + order * screenPitch` (Android 412 + 48 gutter → 460 step).
- **Column alignment across sections:** both sections share the same `firstScreenX` and
  `screenPitch` so each iOS screen sits directly above its Android counterpart. If the iOS
  screens sit at a tighter pitch, **re-space them to `screenPitch`** (canvas spacing only —
  no design change) so pairs left-align. Keep true device widths (iOS 375 / Android 412),
  left-aligned (not centered). **Equalize the two section widths** (use the wider).
- **Vertical baseline:** every screen at `y = screenBaselineY`, top-aligned.
- **Right inset:** `section.width = (rightmost screen right edge) + (firstScreenX −
  sidebarWidth)` — i.e. match the leading inset (288 in Loop).
- **Section height:** `max(screen.y + screen.height) + sectionBottomPadding`.
- **Sidebar height** tracks section height.

> Constant (in `registry.layout`): pitch, baseline, section radius, top/bottom padding, and
> the stacked left-aligned placement. Flexible (derived per build): section width/height and
> per-screen heights.

---

## 2. Design-system components
Keys live in `registry/components.json`. Import with `importComponentByKeyAsync` (single
component) or `importComponentSetByKeyAsync` (variant set), then `.createInstance()`.

- `statusBar` — Material 3 punch-hole status bar (412×52)
- `topAppBar` — Tidepool Loop: Android Top app bar set; use `Configuration=small-centered, Elevation=flat`
- `iconClose` / `iconBack` — Material 3 glyphs, swapped into the app-bar icon buttons via the `appBarIconSwapProp` (`Icon#52081:6`)
- `materialButton` — reference only; the CTA is a brand pill primitive (§4.4)

---

## 3. Android screen skeleton
Vertical auto-layout frame, 412 wide, `itemSpacing: 0`, `cornerRadius 18`, `clipsContent`.
**Background: copy the iOS screen frame's own fill — do not hardcode white.** Most screens
are white, but some carry an intentional tint (e.g. the terminal index screen is `#F2F2F7`,
a slight blue-gray, with white content cards on top). Read `iosScreen.fills` and apply it to
the Android screen frame **and** the gesture nav (the M3 status bar is transparent and
inherits it); cloning a white template silently drops the tint.

```
ADITL / NN  (FRAME, vertical auto-layout, 412 × H, bg = iOS fill, r18, clip)
├─ status bar     (instance, FILL width)                ← Material 3
├─ Top app bar    (instance, FILL width)                ← small-centered, white surface
├─ Content        (auto-layout, FILL, pad L/R 24, top 16, gap 24)
│   └─ Content AL (cloned from iOS, re-fonted to Roboto)
├─ Spacer         (FILL width; FILL height on device screens)
├─ Bottom Actions (auto-layout, FILL, pad L/R 24, top 8, bottom 24)
│   └─ Button/Primary (full-width brand pill)
└─ Gesture Nav    (FILL width, 412×24, centered handle pill)
```

Each iOS screen = `Content AL` + `Bottom Actions` + a top chrome instance. **Clone
`Content AL`**, **rebuild** chrome + CTA from the DS, add the gesture nav.

**Height (device vs scroll):** build hug-height (`primaryAxisSizingMode='AUTO'`) with a
`Spacer`. Measure: if `< 892`, set frame **FIXED 892** and `Spacer.layoutSizingVertical='FILL'`
(pins CTA to bottom). If `≥ 892`, leave hug-height (content flows).

---

## 4. Conventions & decisions

### 4.1 Typography — preserve iOS metrics, swap family only
Goal: visual parity with iOS. **Swap family only; keep iOS size, weight, line-height,
letter-spacing.** The source family prefixes, target family, and per-weight map all live in
`registry.typography` (Loop: SF Pro / Work Sans → Roboto, Semibold→SemiBold, Heavy→Black,
…). Map weight **per text segment** so mixed runs survive (Italic preserved).

Implementation: iterate `getStyledTextSegments(['fontName'])`; for each segment whose family
matches a `sourceFamilyPrefixes` entry, `setRangeFontName(start, end, {family: targetFamily,
style: mapped})`. **Never** set `fontSize`/`lineHeight` — keep the cloned iOS values, so the
type ramp (e.g. a ~34px Bold title over ~17px body) carries over without being hardcoded here.

The CTA label is the exception: the pill is rebuilt, not cloned, so its label is *specified*,
not preserved — Roboto Medium 16 white (§4.4).

**Material divergence — decision rule:** if an iOS pattern intentionally breaks the iOS DS
(e.g. an oversized Large-Title weight Material would set lighter), carry that break to
Android; otherwise follow each platform's DS, and where the Android DS is silent, match the
iOS weight/size/spacing.

### 4.2 Layout
- Content horizontal margins: iOS 16dp → **Android 24dp**. Content width = `412 − 48 = 364`.
- Dividers span content width (FILL).

### 4.3 Header (Top app bar)
- Variant **`Configuration=small-centered`** (centered title, matches iOS), `Elevation=flat`.
- **White surface** (`fills=[white]`) — override the themed lavender tint.
- **Title = flow name**, centered. Set it by editing the visible `headline` text node's
  `characters` directly (load font first). The component's `Headline` property does **not**
  drive the visible node in this variant.
- **Leading = back arrow (←) top-left:** the `small-centered` leading default is a
  **hamburger** — swap it to `iconBack` via `appBarIconSwapProp`. **Hide** (`visible=false`)
  when there's no back action.
- **Trailing = close (✕) top-right:** enable `Show 1st trailing icon`, swap the trailing
  icon-button's glyph to `iconClose`.

**Per-screen header rule:** close ✕ on **every** screen; back ← on every screen **except
the first** (iOS shows Back top-left / Close top-right; the first screen has close only).

**Build tip:** configure the header once on screen 1, then **clone that app-bar instance**
for the rest — it carries the white fill, centered variant, and close ✕. Per screen: set
the headline text node, and show+swap the back arrow (skip on screen 1).

### 4.4 Primary CTA button
Build a **full-width filled pill primitive** (not the Material `Button` instance): horizontal
auto-layout, `cornerRadius 100`, padding 16/24, centered, FILL width, brand fill, label
Roboto Medium 16 white. (The iOS CTA is full-width + brand-colored; the stock Material
button is standard-width + M3-palette, so it would need width+color overrides anyway and
recoloring instance internals is fragile.) If governance requires the DS component, swap in
Material `Button` (Filled/enabled), FILL width, override container fill to brand.

### 4.5 Brand color
`tokens.brandPrimary` (Loop blue `#657FF7`), sourced from the iOS primary button fill.

### 4.6 Status bar & gesture nav
- Status bar: Material 3 `statusBar` instance, FILL width.
- Gesture nav: white frame 412×24 with a centered handle pill (108×4, `cornerRadius 100`,
  `~#1A1A1A`). **No bottom navigation bar** in an onboarding flow.

---

## 5. Procedure (per flow)
1. **Preflight:** `whoami`; read `registry/components.json`; enumerate the iOS Section's
   direct children to build the authoritative screen list (see §6 — don't name-regex).
2. **Section setup:** create `ANDROID Section` stacked below iOS, left-aligned; gray bg;
   clone iOS `Status` sidebar → white fill, dark text, **stroke removed**, full height.
3. **Screen 1 first**, get sign-off before batching.
4. **Per screen:** status bar → centered Top app bar (header rule) → cloned `Content AL`
   re-fonted (§4.1) → Spacer → brand pill CTA (label from the iOS `Button / Primary`) →
   gesture nav. Apply radius + height logic; place per §1 layout.
5. **Record mapping** (§7) and report. Compare to iOS only after building.

---

## 6. Gotchas
- **Discover screens by enumerating the iOS Section's direct children — never by
  name-pattern.** Run a read-only `use_figma` returning `isoSection.children`
  (id/name/type/x/y/w/h), drop the `Status` sidebar, sort by `x` → that is the
  authoritative screen list. Do **not** grep metadata for `ADITL / NN`: flows include
  off-convention frames (e.g. the terminal `End Screen (ADTL)`) that a name-regex
  silently drops, leaving the Android section short a screen. Such terminal/menu screens
  are also structurally non-standard — **mirror the iOS screen's own chrome, don't impose
  the standard skeleton.** The terminal index screen has *no top toolbar* (no back/close)
  and *no `Button / Primary` CTA* in its iOS frame, so the Android version gets **neither**
  a Top app bar nor a CTA — just the cloned/re-fonted content plus device chrome (status
  bar, gesture nav). Adding back/close or a Continue button here invents UI the source
  lacks (golden rule). Check whether each screen actually has a chrome instance before
  rebuilding one, and flag terminal screens for human judgment.
- **`use_figma` is atomic** — failed scripts make no changes; fix and retry. Work in small
  steps (≤10 ops) and screenshot between them.
- **Re-fonting:** load a node's *current* font before mutating, or it throws "unloaded font".
  Wrap per-node re-font in try/catch (text locked inside instances will throw — skip it).
- **Centered headline ≠ `Headline` property** — set the text node's `characters` directly.
- **Switching the app-bar `Configuration` variant resets overrides** — re-apply, or (better)
  clone a pre-configured instance.
- **`small-centered` leading glyph defaults to hamburger** — always swap to `iconBack`.
- **Title — don't scrape heuristically:** iOS chrome instances bundle example toolbars
  (e.g. a stray "Settings") and body charts reuse generic node names like `Title`. The
  app-bar title is the flow name (a known constant for the run) — set it directly.
- **CTA label — read only from the visible `Button / Primary`** subtree, checking *ancestor*
  visibility (not just the node's own `visible`). Hidden sibling frames can carry a stale
  duplicate label — e.g. in the Loop file a hidden `Pagination` frame holds "Finish" with
  its own `visible=true`.
- **Cloned iOS frames carry iOS strokes/fills** — audit and clear (the black sidebar stroke).
- **Screen background ≠ always white** — when cloning a white template screen, re-copy the
  iOS screen frame's fill onto the new frame + gesture nav, or an intentional tint (e.g.
  `#F2F2F7`) is silently lost. See §3.
- **Resizing a section moves children with center/scale constraints** (the sidebar drifted
  ~half the width delta). After any section resize, re-snap the sidebar to `x=0` and set
  `constraints = {horizontal:'MIN', vertical:'STRETCH'}` so it stays flush-left, full-height.

---

## 7. Mapping (persistent state)
After a successful build, write/update **`~/.figma-sync/mappings.json`** (user-global; create
if missing) with the flow's entry — `name`, `fileKey`, `page`, `iosSectionNodeId`,
`androidSectionNodeId`, and `screenPairs` (iOS↔Android node ids). Schema:
`${CLAUDE_PLUGIN_ROOT}/mappings.example.json`. This is what `sync-screens` and
`apply-ds-update` read later.

---

## 8. Reference
- Material 3 Design Kit — figma.com/community/file/1035203688168086460
- Loop iOS customizations — file `m8iprZw0FBO1DDZq0QlpUw`
- Colors are authored in Figma 0–1 RGB in scripts; hex above is the human-facing form.

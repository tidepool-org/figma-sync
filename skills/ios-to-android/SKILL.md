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
> (`mcp__plugin_figma_figma__whoami`). This skill drives `use_figma`. Stable
> component keys / tokens / layout values live in `${CLAUDE_PLUGIN_ROOT}/registry/components.json` —
> read them from there rather than re-discovering by search.

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
| Source | `ISO Section` | 375 | `#565656` | black `Status`, white text |
| Target | `ANDROID Section` | 412 | `#BCBCBC` | **white** `Status`, dark text |

### Sidebar (`Status` frame)
- Left rail, **896** wide, full section height, flow title in **Work Sans Bold 92**.
- Android: **white fill**, dark text (`#1A1A1A`), flush at `x=0, y=0`.
- **Remove the stroke** after cloning the iOS sidebar (it carries a black stroke that
  is invisible on black but shows as a border on white): `strokes = []`.

### Section layout rules (constants from `registry.layout`; read off `ISO Section` to confirm)
- **Placement:** `ANDROID Section` is left-aligned with `ISO Section` (`x = ISO.x`) and
  sits **directly below it** (`y = ISO.y + ISO.height + 800`). Stack vertically — never
  place it to the right of iOS.
- **Section frame:** `cornerRadius = 64`, `clipsContent = true`, background `#BCBCBC`.
- **Screen row:** single horizontal row, flow order, fixed pitch. 412 wide + 48 gutter →
  **460px step**. First screen `x = 1184`. Formula: `screen.x = 1184 + order * 460`.
- **Column alignment across sections:** both sections share the **same start x (1184) and
  pitch (460)** so each iOS screen sits directly above its Android counterpart. iOS screens
  (375) are usually at a tighter pitch (~415); **re-space the iOS screens to 460** (canvas
  spacing only — no design change) so pairs left-align. Keep true device widths (375 / 412),
  left-aligned (not centered). **Equalize the two section widths** (use the wider).
- **Vertical baseline:** every screen at `y = 192` (iOS top padding), top-aligned.
- **Right inset:** `section.width = (rightmost screen right edge) + 288` (match leading inset).
- **Section height:** `max(screen.y + screen.height) + 192` (192 bottom padding).
- **Sidebar height** tracks section height.

> Constant: pitch, baseline (192), section radius (64), 192/192 padding, and the stacked
> left-aligned placement. Flexible: section width/height and per-screen heights.

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
Vertical auto-layout frame, 412 wide, white, `itemSpacing: 0`, `cornerRadius 18`, `clipsContent`:

```
ADITL / NN  (FRAME, vertical auto-layout, 412 × H, white, r18, clip)
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
Goal: visual parity with iOS. **Swap family SF Pro → Roboto; keep iOS size, weight,
line-height, letter-spacing.** Map weight 1:1 **per text segment** (so mixed runs survive):
Regular→Regular, Medium→Medium, Semibold→**SemiBold**, Bold→Bold, Light→Light,
Black/Heavy→Black (+ Italic preserved). For this flow: title = Roboto Bold 34, body/list =
Roboto Regular 17, CTA label = Roboto Medium 16 (white) — see §4.4.

Implementation: iterate `getStyledTextSegments(['fontName'])`; for each segment whose
family starts with `SF Pro` (or `Work Sans`), `setRangeFontName(start, end, {family:'Roboto', style: mapped})`.
**Never** set `fontSize`/`lineHeight` — keep the cloned iOS values.

**Material divergence (accepted):** body/labels at 16–17px match Material Body Large fine.
The one deliberate break is the **34px Bold** title (iOS Large Title) — Material would use
Regular/Medium at that size. Decision rule: if an iOS pattern intentionally breaks the iOS
DS, carry that break to Android; otherwise follow each platform's DS, and where the Android
DS doesn't specify, match the iOS weight/size/spacing.

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
Loop blue **`#657FF7`** (`rgb(0.396, 0.502, 0.969)`), from the iOS primary button fill.

### 4.6 Status bar & gesture nav
- Status bar: Material 3 `statusBar` instance, FILL width.
- Gesture nav: white frame 412×24 with a centered handle pill (108×4, `cornerRadius 100`,
  `~#1A1A1A`). **No bottom navigation bar** in an onboarding flow.

---

## 5. Procedure (per flow)
1. **Preflight:** `whoami`; read `registry/components.json`.
2. **Section setup:** create `ANDROID Section` stacked below iOS, left-aligned; gray bg;
   clone iOS `Status` sidebar → white fill, dark text, **stroke removed**, full height.
3. **Screen 1 first**, get sign-off before batching.
4. **Per screen:** status bar → centered Top app bar (header rule) → cloned `Content AL`
   re-fonted (§4.1) → Spacer → brand pill CTA (label from the iOS `Button / Primary`) →
   gesture nav. Apply radius + height logic; place per §1 layout.
5. **Record mapping** (§7) and report. Compare to iOS only after building.

---

## 6. Gotchas
- **`use_figma` is atomic** — failed scripts make no changes; fix and retry. Work in small
  steps (≤10 ops) and screenshot between them.
- **Re-fonting:** load a node's *current* font before mutating, or it throws "unloaded font".
  Wrap per-node re-font in try/catch (text locked inside instances will throw — skip it).
- **Centered headline ≠ `Headline` property** — set the text node's `characters` directly.
- **Switching the app-bar `Configuration` variant resets overrides** — re-apply, or (better)
  clone a pre-configured instance.
- **`small-centered` leading glyph defaults to hamburger** — always swap to `iconBack`.
- **Title extraction:** the iOS chrome instance bundles example toolbars (e.g. "Settings"),
  and body charts reuse the name `Title`. For this flow the toolbar title is the constant
  flow name — don't scrape it heuristically.
- **CTA label:** read only from the **visible `Button / Primary`** subtree (check ancestor
  visibility). The hidden `Pagination` frame holds a stale label (e.g. "Finish") with its
  own `visible=true`.
- **Cloned iOS frames carry iOS strokes/fills** — audit and clear (the black sidebar stroke).
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

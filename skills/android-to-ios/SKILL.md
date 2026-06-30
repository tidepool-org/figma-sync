---
name: android-to-ios
description: Translate an Android (Material 3) Figma flow into a matching iOS (Apple HIG) section. The reverse of ios-to-android — conventions, design-system component discovery, section/screen layout, typography (Roboto → SF Pro), header rules, and per-screen build steps. Use when creating or extending an iOS version of Android screens in Figma via the Figma MCP (use_figma).
---

# Android → iOS Figma translation

The authoritative playbook for translating an **Android (Material 3)** screen flow into a
matching **iOS (Apple HIG)** section in Figma. This is the mirror of `ios-to-android`: here
we treat **Android as the source** and derive iOS. Same rigor, same comparison layout — the
platform specifics are flipped.

> Prerequisite: the official **Figma MCP** must be running and authenticated
> (`mcp__plugin_figma_figma__whoami`). This skill drives `use_figma`.
>
> **Source of truth for values:** component keys, design tokens, and layout constants live
> in `${CLAUDE_PLUGIN_ROOT}/registry/components.json` — read them from there. This doc teaches
> the *method* and names the registry keys; any literal number or hex shown inline is
> **illustrative** (a current value from the Loop flow), not authoritative. When prose and
> registry disagree, the registry wins.
>
> **iOS keys may need discovery first.** The registry's `androidToIos` block scaffolds the iOS
> design-system component keys but ships them **empty** — the forward workflow only needed
> Android keys. On the first real run, if `androidToIos.components.*.key` is blank, discover the
> iOS chrome via `search_design_system` against the iOS DS library, then **persist the keys back
> to the registry** so later runs (and `sync-screens`) don't re-discover. See §2.

---

## 0. Golden rule — derive, don't copy
Derive the iOS flow **only** from (1) the Android screens and (2) the iOS / Apple HIG
design-system libraries. If a finished iOS section exists elsewhere as a reference, **do not
look at it to drive the design** — only its container structure may be mirrored, and only
compared *after* you're done. (Cloning a DS *chrome instance* for structure is fine; studying
a hand-finished iOS screen to copy its choices is not.)

---

## 1. Page & container structure

One Figma page per flow, holding **two sibling section frames, stacked vertically and
left-aligned** — **source on top, derived target directly below** — for screen-for-screen
comparison. In this direction the **Android section is the source (top)** and the **iOS
section is the target (below)**:

| Section | Role | Name | Screen width | Section background | Sidebar |
|---|---|---|---|---|---|
| Android | source (top) | existing | 412 | as-is (`#BCBCBC`) | white `Status`, dark text |
| iOS | target (below) | `IOS Section` | 375 | `androidToIos.tokens.iosSectionBackground` (`#565656`) | **black** `Status`, **white** text |

### Sidebar (`Status` frame)
- Left rail, `layout.sidebarWidth` wide (896), full section height; flow title inherited from
  the cloned Android sidebar (Work Sans Bold 92 — comes along with the clone).
- iOS: **black fill**, **white** text (`androidToIos.tokens.sidebarTextOnDark`), flush at
  `x=0, y=0`. (Mirror of the Android sidebar, which is white/dark.)
- Clone the Android `Status` sidebar and recolor: `fills = [black]`, retint the title text to
  white. The black fill needs no stroke.

### Section layout rules
Every grid constant below is a key in `registry.layout` and is **shared with the forward
workflow** — read at runtime; parentheticals are current Loop values, for orientation only.
- **Placement:** `IOS Section` is left-aligned with the Android section (`x = AND.x`) and sits
  **directly below it** (`y = AND.y + AND.height + sectionStackGap`). Stack vertically — never
  to the right.
- **Section frame:** `cornerRadius = sectionCornerRadius` (64), `clipsContent = true`,
  background `androidToIos.tokens.iosSectionBackground` (`#565656`).
- **Screen row:** single horizontal row in flow order at fixed pitch —
  `screen.x = firstScreenX + order * screenPitch` (iOS 375 + gutter → the shared 460 step).
- **Column alignment across sections:** both sections share the same `firstScreenX` (1184) and
  `screenPitch` (460) so each Android screen sits directly above its iOS counterpart. Keep true
  device widths (Android 412 / iOS 375), left-aligned (not centered). **Equalize the two section
  widths** (use the wider).
- **Vertical baseline:** every screen at `y = screenBaselineY` (192), top-aligned.
- **Right inset:** `section.width = (rightmost screen right edge) + (firstScreenX − sidebarWidth)`
  — match the leading inset (288 in Loop).
- **Section height:** `max(screen.y + screen.height) + sectionBottomPadding` (192).
- **Sidebar height** tracks section height.

> Constant (in `registry.layout`): pitch, baseline, section radius, padding, sidebar width, and
> the stacked left-aligned placement. Flexible (derived per build): section width/height and
> per-screen heights.

---

## 2. Design-system components (iOS)
Keys live in `registry.androidToIos`. Import with `importComponentByKeyAsync` /
`importComponentSetByKeyAsync`, then `.createInstance()`. The Android source screens are
**read/cloned for content**; the iOS chrome is **rebuilt from the iOS DS**.

Needed iOS pieces (see `androidToIos.components`):
- `iosStatusBar` — iOS status bar (light), 375 wide (often bundled inside the modal-nav chrome).
- `iosModalNav` — the iOS modal-stack nav chrome: centered **inline title** + leading Back
  chevron + trailing **Close**. (In Loop this is an `… / Modal Stack / Light` component.)
- `sheetGrabber` — the gray grabber pill at the top of a presented sheet (~36×5, systemGray).
- `homeIndicator` — the iOS home indicator (375-wide area; centered bar ~134×5, `cornerRadius
  100`). The iOS counterpart to Android's gesture nav.

**First-run discovery:** if any `key` is blank, call `search_design_system` against the iOS DS
library for the component above, set the key, and **write it back to the registry**
(`androidToIos.components.<name>.key` + the resolved `iosLibraries` key). If a component truly
isn't in the DS (grabber, home indicator), draw it as a small primitive per §4.6 and leave the
key blank with a note. Do **not** silently fall back to guessed keys.

---

## 3. iOS screen skeleton
Vertical auto-layout frame, 375 wide, `itemSpacing: 0`, `clipsContent`. **iOS screen corner
radius is `0`** (these are flat sheet frames, not device-rounded) — but as always, **copy the
Android screen frame's own fill** rather than hardcoding white; an intentional tint (e.g.
`#F2F2F7`) must carry over (read `androidScreen.fills`, apply to the iOS frame **and** home
indicator).

```
ADITL / NN  (FRAME, vertical auto-layout, 375 × H, bg = Android fill, r0, clip)
├─ status bar     (instance, FILL width)                 ← iOS status bar (light)
├─ Sheet grabber  (FILL width; centered gray pill)       ← only for modal/sheet flows
├─ Nav bar        (instance, FILL width)                 ← inline centered title + Back/Close
├─ Content        (auto-layout, FILL, pad L/R 16, top 8, gap …)
│   └─ Content AL (cloned from Android, re-fonted to SF Pro)
├─ Spacer         (FILL width; FILL height on device screens)
├─ Bottom Actions (auto-layout, FILL, pad L/R 16, bottom …)
│   └─ Button/Primary (full-width filled iOS button, r8)
└─ Home indicator (FILL width; centered handle bar)
```

Each Android screen = `Content` (with `Content AL`) + `Bottom Actions` + top chrome. **Clone
the Android `Content AL`**, **rebuild** chrome + CTA from the iOS DS, add the home indicator.

**Height (device vs scroll):** build hug-height (`primaryAxisSizingMode='AUTO'`) with a
`Spacer`. Measure: if `< iosDeviceHeight` (**812**), set frame **FIXED 812** and
`Spacer.layoutSizingVertical='FILL'` (pins CTA to bottom). If `≥ 812`, leave hug-height
(content flows). *(The forward workflow uses 892 for the taller Android device; iOS is 812.)*

---

## 4. Conventions & decisions

### 4.1 Typography — preserve Android metrics, swap family only
Goal: visual parity with Android. **Swap family only; keep Android size, weight, line-height,
letter-spacing.** Source prefixes, target families, and the per-weight map live in
`androidToIos.typography` (Roboto → SF Pro). Map weight **per text segment** so mixed runs
survive (Italic preserved).

**SF Pro is two families by size — pick per segment:** `SF Pro Text` for `fontSize < 20`,
`SF Pro Display` for `fontSize ≥ 20` (`displaySizeThreshold`). And the SemiBold style is spelled
**`Semibold`** in SF Pro (one word, lowercase b) — *not* Roboto's `SemiBold`. Weight map:
Thin→Thin, Light→Light, Regular→Regular, Medium→Medium, **SemiBold→Semibold**, Bold→Bold,
Black→Black (+ Italic preserved).

Implementation: iterate `getStyledTextSegments(['fontName', 'fontSize'])`; for each segment whose
family matches a `sourceFamilyPrefixes` entry, choose the family by size, then
`setRangeFontName(start, end, {family, style: mapped})`. **Never** set `fontSize`/`lineHeight`
— keep the cloned Android values, so the type ramp carries over without being hardcoded here.

The CTA label is the exception: the button is rebuilt, not cloned, so its label is *specified* —
SF Pro Text Semibold 17 white (§4.4).

**HIG divergence — decision rule:** same rule as the forward direction, flipped. If an Android
pattern intentionally breaks the Android DS, carry that break to iOS; otherwise follow each
platform's DS, and where the iOS DS is silent, match the Android weight/size/spacing.

### 4.2 Layout
- Content horizontal margins: Android 24dp → **iOS 16pt**. Content width = `375 − 32 = 343`.
- Dividers span content width (FILL).

### 4.3 Header (iOS nav / toolbar)
- iOS status bar (light) on top; for a presented sheet, a **grabber pill** sits just below it.
- **Inline centered title** = flow name, SF Pro Text Semibold 17. Set the visible title text
  node's `characters` directly — **don't scrape it**: the iOS modal-nav component bundles stale
  example titles (e.g. "Attach Infusion Assembly"), exactly like the Android app bar did.
- **Leading = back affordance**, top-left, per `androidToIos.header.back` — render the `icon`
  (e.g. chevron `‹`) and, when `label` is set, the label beside it; tint via `back.tint`
  (`tokens.brandPrimary`). When a label is present and `labelStyleMatchesTrailing` is true, style
  the label to match the trailing action's text node (same family/size/weight). **Hide** when
  there's no back action.
- **Trailing action**, top-right, per `androidToIos.header.trailing` — a text button (e.g.
  "Close", or ✕) tinted `trailing.tint`, shown per `showOnEveryScreen`.

**Per-screen header rule (mirror of forward):** the trailing action follows
`header.trailing.showOnEveryScreen`; the back affordance appears on every screen **except the
first** (`header.back.showOnFirstScreen: false`).

**Build tip:** configure the nav once on screen 1, then **clone that nav instance** for the rest;
per screen set the title text node and show the back affordance (skip on screen 1). When the back
uses a label, set it from `header.back.label` in the trailing action's text style.

### 4.4 Primary CTA button
Build a **full-width filled iOS button** (or the iOS DS button if governance requires): vertical
auto-layout, centered, **`cornerRadius 8`** (the iOS value — *not* the Android pill's 100),
~`54` tall, FILL width, fill `tokens.brandPrimary`, label SF Pro Text Semibold 17 white. (The
Android source CTA is a full-width brand pill; iOS uses the same full-width brand fill but the
HIG rounded-rect radius.) Read the label only from the **visible** Android `Button / Primary`.

### 4.5 Brand color
`tokens.brandPrimary` (Loop blue `#657FF7`) — shared across both platforms; on Android it fills a
pill, on iOS a radius-8 button and the nav tint.

### 4.6 Status bar & home indicator
- Status bar: iOS `iosStatusBar` instance (light), FILL width.
- Home indicator: 375-wide frame (~34pt area) with a centered handle bar (~134×5, `cornerRadius
  100`, near-black). The iOS counterpart to Android's gesture nav. **No tab bar** in an
  onboarding flow.

---

## 5. Procedure (per flow)
1. **Preflight:** `whoami`; read `registry/components.json`; **enumerate the Android Section's
   direct children** to build the authoritative screen list (see §6 — don't name-regex);
   discover + persist any missing iOS DS keys (§2).
2. **Section setup:** create `IOS Section` stacked below the Android section, left-aligned;
   `#565656` bg; clone the Android `Status` sidebar → **black fill, white text**, full height.
3. **Screen 1 first**, get sign-off before batching.
4. **Per screen:** iOS status bar → grabber (if a sheet) → nav (header rule) → cloned `Content AL`
   re-fonted to SF Pro (§4.1) → Spacer → iOS brand button (label from the visible Android
   `Button / Primary`) → home indicator. Apply r0 + height logic (812); place per §1 layout.
5. **Record mapping** (§7) and report. Compare to any existing iOS only after building.

---

## 6. Gotchas
- **Discover screens by enumerating the Android Section's direct children — never by
  name-pattern.** Run a read-only `use_figma` returning `androidSection.children`
  (id/name/type/x/y/w/h), drop the `Status` sidebar, sort by `x` → authoritative screen list.
  Flows include off-convention frames (e.g. a terminal `End Screen`) that a name-regex silently
  drops. **Mirror the source screen's chrome, don't impose the skeleton:** if an Android screen
  has no top app bar and no CTA (a terminal/menu screen), its iOS counterpart gets **no nav bar
  and no button** — just cloned/re-fonted content plus device chrome (status bar, home
  indicator). Adding nav or a button there invents UI the source lacks (golden rule). Flag
  terminal screens for human judgment.
- **`use_figma` is atomic** — failed scripts make no changes; fix and retry. Work in small steps
  (≤10 ops) and screenshot between them.
- **Re-fonting Roboto → SF Pro:** load each node's *current* font before mutating (or it throws
  "unloaded font"); wrap per-node in try/catch (locked text in instances throws — skip it).
  Pick **`SF Pro Text` vs `SF Pro Display` by `fontSize`** (≥20 → Display), and use the style
  spelling **`Semibold`** (not `SemiBold`). Load the *target* SF Pro fonts before
  `setRangeFontName`.
- **Inline title ≠ a `title` property** — set the visible nav title text node's `characters`
  directly, and don't scrape it (the modal-nav component carries stale example titles).
- **Trailing action is on every screen; the back affordance is hidden on screen 1** — when cloning
  a configured nav, remember to *show* the back accessory (icon + label per
  `androidToIos.header.back`) on screens 2+.
- **CTA label — read only from the visible `Button / Primary`** subtree, checking *ancestor*
  visibility. Hidden sibling frames can carry a stale duplicate label.
- **Cloned Android frames carry Android strokes/fills** — audit and clear.
- **Screen background ≠ always white** — re-copy the Android screen frame's fill onto the iOS
  frame + home indicator, or an intentional tint (e.g. `#F2F2F7`) is silently lost. See §3.
- **iOS CTA radius is 8, not 100** — don't carry the Android pill radius across; that's a
  platform tell.
- **Resizing a section moves children with center/scale constraints** — after any section
  resize, re-snap the sidebar to `x=0` with `constraints = {horizontal:'MIN', vertical:'STRETCH'}`
  so it stays flush-left, full-height.

---

## 7. Mapping (persistent state)
Reuse the **same** per-flow entry in `~/.figma-sync/mappings.json` (a flow may be built in either
direction). After a successful build, write/update `name`, `fileKey`, `page`,
`androidSectionNodeId` (the **source** here) and `iosSectionNodeId` (the **new** section), and the
`screenPairs` (android↔ios node ids). Schema: `${CLAUDE_PLUGIN_ROOT}/mappings.example.json`. This
is what `sync-screens` and `apply-ds-update` read later.

---

## 8. Reference
- Apple **Human Interface Guidelines** — developer.apple.com/design/human-interface-guidelines
- iOS native components in Figma — the community **iOS 17 / SF Pro UI kits**
- Loop iOS source — file `m8iprZw0FBO1DDZq0QlpUw`
- Colors are authored in Figma 0–1 RGB in scripts; hex above is the human-facing form.
- This is the mirror of `ios-to-android`; consult that skill for the forward-direction rationale.

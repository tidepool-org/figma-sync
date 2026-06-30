---
note_type: knowledge
template_version: 1
contract_version: 1
knowledge_id: "KNOW-0006"
category: standards
title: "Coding Standards"
status: in_progress
owner: ""
created: '2026-06-30'
updated: '2026-06-30'
reviewed_on: ""
related_notes: []
tags: [apovault, knowledge, standards]
---

# Coding Standards

This repo authors **Markdown + JSON**, not compiled code. The conventions below are about
file naming, the registry token shape, and the skill/registry split — there is no lint,
build, or test toolchain.

## Lint / format (vault-wide)

- **None present.** No `.eslintrc*`, `biome.json`, `.prettierrc*`, `.editorconfig`,
  `tsconfig.json`, or `package.json` in the repo. The only config is `.gitignore` (OS/editor
  noise + a note that the mapping lives outside the repo).
  **Sources:** root `find -maxdepth 2` (read 2026-06-30), `.gitignore:1-15` (read 2026-06-30).
- **Prose wrapping:** authored Markdown wraps body text at ~95 columns (skill/command files);
  match the surrounding file when editing. **Source:** `skills/ios-to-android/SKILL.md` (read 2026-06-30, observed).

## Theme Tokens (Observed)

The token system is a **custom JSON registry**, not a JS theme module or Tailwind/CSS — so
`apo mine theme-sources` reports none; the real source is `registry/components.json`.
**Source:** `apo mine theme-sources --json` + `registry/components.json` (read 2026-06-30).

- **Canonical source:** `registry/components.json`. Single source; no secondary token files.
- **Shape — flat, two scopes:**
  - Forward (iOS→Android): `tokens.*` — `brandPrimary` `#657FF7`, `sidebarText` `#1A1A1A`,
    `sectionBackground` `#BCBCBC`, `divider` `#F5F5F5`. **Source:** `registry/components.json:16-21` (read 2026-06-30).
  - Reverse (Android→iOS): `androidToIos.tokens.*` — `iosSectionBackground` `#565656`,
    `sidebarTextOnDark` `#FFFFFF`. **Source:** `registry/components.json:76-79` (read 2026-06-30).
- **Sibling categories in the same file:** `libraries`, `components`, `appBarIconSwapProp`,
  `typography`, `layout`, and the whole `androidToIos` mirror block (`iosLibraries`,
  `components`, `typography`, `layout`, `tokens`, `header`).
  **Source:** `registry/components.json:4-85` (read 2026-06-30).
- **Access shape:** colors are authored as **hex**; scripts convert to Figma 0–1 RGB at apply
  time. Skills reference tokens by dotted key (`tokens.brandPrimary`,
  `tokens.sectionBackground`). **Sources:** `registry/components.json:2` (read 2026-06-30), `skills/ios-to-android/SKILL.md:42,171` (read 2026-06-30).
- **Layout/typography constants live alongside tokens** (e.g. `layout.screenPitch` 460,
  `layout.firstScreenX` 1184, `typography.targetFamily` "Roboto"). Treat them the same way:
  read the key, don't hardcode. **Source:** `registry/components.json:22-43` (read 2026-06-30).
- **Sample is not exhaustive** — grep `registry/components.json` for the exact key before use.

## Build / run commands (vault-wide)

- **No build.** Distributed as a Claude Code plugin:
  `/plugin marketplace add tidepool-org/figma-sync` → `/plugin install figma-sync@tidepool`;
  update via `/plugin update`. **Source:** `README.md:15-21` (read 2026-06-30).
- **Runtime preflight:** each command calls `mcp__plugin_figma_figma__whoami` and stops if the
  Figma MCP is unavailable. **Source:** `commands/create-android.md:11` (read 2026-06-30).

## Naming Conventions (Observed) — figma-sync

**Subsystem path:** `.` (repo root). **Scanned:** 2026-06-30. UI-primitive suffix scan
(`apo mine conventions --path .`): **all zero** — this is not a UI-component codebase.

- **Patterns NOT present:** `*Dialog`, `*Modal`, `*Card`, `*Form`, `*Section`, `*Drawer`,
  `*Panel`, `*Page`, `*View`, `*Sheet`, `*Popover`, `*Tooltip` (0 each).
  **Source:** `apo mine conventions --path . --json` (read 2026-06-30).
- **Commands:** kebab-case `<verb>.md` in `commands/` — `create-android`, `create-ios`,
  `sync-screens`, `apply-ds-update`. **Source:** `commands/` listing (read 2026-06-30).
- **Skills:** `skills/<direction-kebab>/SKILL.md` — `ios-to-android/`, `android-to-ios/`.
  **Source:** `skills/` listing (read 2026-06-30).
- **Agents:** kebab-case `<role>.md` in `agents/`, iOS mirror suffixed `-ios` —
  `screen-builder.md`, `screen-builder-ios.md`. **Source:** `agents/` listing (read 2026-06-30).
- **Direction convention:** forward = iOS→Android (no suffix); reverse = Android→iOS, named
  `<x>-ios` / `androidToIos`. **Source:** `ARCHITECTURE.md:93-96` (read 2026-06-30).
- **Figma node names (set by the build):** sections `ANDROID Section` / `IOS Section`;
  screens follow the source naming (`ADITL / NN`) — discovered by children, not regex.
  **Sources:** `skills/ios-to-android/SKILL.md:41,98` (read 2026-06-30), `skills/ios-to-android/SKILL.md:193-206` (read 2026-06-30).

## API-Usage Patterns (Observed) — figma-sync

| Newer | Older | Verdict |
|---|---|---|
| — | — | N/A — no code APIs. The only programmatic surface is the Figma MCP, accessed uniformly through `use_figma`. |

**Source:** `apo mine subsystems` (file_count 0 for code dirs) + `commands/`/`skills/` content
(read 2026-06-30). No modern-vs-legacy API split exists to record.

## File organization (observed) — figma-sync

- Flat, type-partitioned top level: `commands/`, `skills/`, `agents/`, `registry/`,
  `.claude-plugin/` + docs at root. One file (or one dir for skills) per artifact.
  **Source:** `ARCHITECTURE.md:36-59` (read 2026-06-30).
- **Registry layout:** forward keys at top level; reverse keys mirrored under `androidToIos`;
  shared grid constants documented as shared (not duplicated).
  **Source:** `registry/components.json:44-45` (read 2026-06-30).

## Test conventions (observed)

- **No test framework, no test files.** Verification is manual / screenshot-based in Figma
  (skills instruct screenshotting between `use_figma` steps; commands "screenshot-verify and
  report"). **Sources:** root tree (read 2026-06-30), `skills/ios-to-android/SKILL.md:208` (read 2026-06-30), `commands/sync-screens.md:21` (read 2026-06-30).

## References

- [[Domain_Model]] (token entities), [[Prompt_Standards]] (theme-token rail), [[Code_Map]]

## Verification status

Fully cited. The "no lint / no tests" findings are negative facts confirmed by the directory
tree and the mining verbs. No `(verify)` items.

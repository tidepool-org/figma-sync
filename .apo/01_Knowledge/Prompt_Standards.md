---
note_type: knowledge
template_version: 1
contract_version: 1
knowledge_id: "KNOW-0007"
category: standards
title: "Prompt Standards"
status: in_progress
owner: ""
created: '2026-06-30'
updated: '2026-06-30'
reviewed_on: ""
related_notes: []
tags: [apovault, knowledge, standards]
---

# Prompt Standards

## Agent-facing rails (from the plugin's own skills)

The repo has no project-level `CLAUDE.md` / `AGENTS.md` / `.cursorrules`; the skill files are
the authoritative agent rails. The cross-cutting ones for any figma-sync work:

- **Do derive, don't copy.** Build a target section only from the source section + the target
  DS; never read a pre-existing target section to drive the design.
  **Source:** `skills/ios-to-android/SKILL.md:25-29` (read 2026-06-30).
- **Do treat the registry as the source of truth for values.** Skills teach method and name
  keys; inline literals are illustrative. When prose and registry disagree, the registry wins.
  Do not hardcode numbers/hex that belong in `registry/components.json`.
  **Source:** `skills/ios-to-android/SKILL.md:16-21` (read 2026-06-30).
- **Do preflight the Figma MCP** (`whoami`) and stop with guidance if it fails. Do not bundle
  or assume a plugin-local MCP. **Source:** `commands/create-android.md:11` (read 2026-06-30).
- **Do work atomically** (`use_figma` is atomic; ≤10 ops/batch; screenshot between). Do not
  run large multi-op scripts blind. **Source:** `skills/ios-to-android/SKILL.md:207-208` (read 2026-06-30).
- **Do build screen 1 first and get sign-off** before batching the rest.
  **Source:** `commands/create-android.md:24` (read 2026-06-30).
- **Do discover screens by enumerating section children, not by name-regex.** Do not assume
  every screen matches `ADITL / NN`; mirror non-standard terminal screens' own chrome.
  **Source:** `skills/ios-to-android/SKILL.md:193-206` (read 2026-06-30).
- **Do flag for human judgment:** ambiguous titles, non-"Continue" CTA labels, terminal/menu
  screens. **Source:** `commands/create-android.md:32` (read 2026-06-30).

## Vault-artifact citations in generated content

- **Do not** embed apo-vault structural IDs or paths in code, comments, test titles, runtime
  strings, or `01_Knowledge/*` rail bodies — no `DEC-NNNN`, `OBS-NNNN`, `PHASE-NN`,
  `STEP-NN-NN`, `SESSION-*`, or `02_Work/**` / `01_Knowledge/_pending/**` /
  `01_Knowledge/_archive/**` wikilinks.
- **Do** keep vault IDs in their structural homes: file names, frontmatter, and `02_Work/**`
  cross-references. Domain item IDs (`TASK-NNNN` / `BUG-NNNN`) referenced as kanban items are
  allowed.
- Enforced by `/apo:lint` (the "vault-artifact citations" check). This rail ships on every
  project; it is an apo-vault standard, not a figma-sync-specific finding.

## Theme tokens

The token source is `registry/components.json` (flat `tokens.*` + `androidToIos.tokens.*`);
see [[Coding_Standards]] § Theme Tokens for the shape.

- **Do** read `registry/components.json` (or grep the key) before referencing a token, color,
  layout constant, or component key in generated work.
- **Do not** invent token names. Cite the actual key — e.g. `tokens.brandPrimary`
  (`#657FF7`), not `tokens.loopBlue`.
- **Do** prefer the closest sibling's token usage when extending, and keep the forward/reverse
  scope right (`tokens.*` for iOS→Android, `androidToIos.tokens.*` for Android→iOS).
- **Do not** flatten or rename across scopes (`androidToIos.tokens.iosSectionBackground` is
  not `tokens.iosSectionBackground`).

## References

- [[Coding_Standards]], [[Agent_Workflow]]

## Verification status

Cited from the skill/command files and the registry. No project `CLAUDE.md`/`AGENTS.md` exists,
so agent rails are sourced from the skills. No `(verify)` items.

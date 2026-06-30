---
note_type: knowledge
template_version: 1
contract_version: 1
knowledge_id: "KNOW-0008"
category: definition_of_done
title: "Definition of Done"
status: in_progress
owner: ""
created: '2026-06-30'
updated: '2026-06-30'
reviewed_on: ""
related_notes: []
tags: [apovault, knowledge, definition_of_done]
---

# Definition of Done

A figma-sync change (registry, skill, command, or agent) is **done** when all gates below
pass. **Confirmed by owner on 2026-06-30** (clint@tidepool.org).

## Verification gate

- **Build-run, screenshot-verified.** Run the affected `/figma-sync:create-*` command
  end-to-end against a real test Figma file in **each affected direction**, and visually
  confirm the screenshots before calling the change done. A diff-only self-review is not
  sufficient. This mirrors the skills' own screenshot-between-steps discipline (see
  [[Coding_Standards]] § Test conventions).

## Mirror rule (shared conventions)

- When a change touches a convention **shared by both directions**, update **both** mirror
  skills (`skills/ios-to-android/SKILL.md` and `skills/android-to-ios/SKILL.md`) and **both**
  registry blocks (top-level + `androidToIos`), then **build-verify both directions**. Never
  land a shared-convention change against only one direction. (The two skills are explicit
  mirrors — see [[Code_Map]] § Module Boundaries.)

## Versioning gate

- **Bump `plugin.json` semver** on any command / skill / convention change (so the team's
  `/plugin update` picks it up).
- **Move `registry/components.json` `version`** only on a **DS-spec change** (new/changed
  keys, tokens, or layout values) — its git history is the auditable DS-change record.
- The two versions move independently (per `ARCHITECTURE.md` §6).

## Workflow / merge gate

- **Feature branch + PR.** Land changes via a feature branch and a pull request — not direct
  commits to `main`. Merge only after the gates above pass.
- **PR description must include:**
  - Screenshot evidence of the verified build-run (each affected direction).
  - The version bump(s) made (`plugin.json` and/or registry `version`).
  - For a design-rationale change, a reference to the relevant `ARCHITECTURE.md` decision
    (§9) or a note that a new decision was added.

## References

- [[Coding_Standards]] (verification + versioning), [[Code_Map]] (mirror skills), `ARCHITECTURE.md` §6, §9

## Verification status

Policy confirmed by the project owner on 2026-06-30. The mechanical hooks it references
(version bump rules, mirror skills, screenshot verification) are cited to the repo; the gate
choices (build-run requirement, feature-branch+PR workflow) are owner decisions.

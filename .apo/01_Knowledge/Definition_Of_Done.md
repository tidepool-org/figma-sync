---
note_type: knowledge
template_version: 1
contract_version: 1
knowledge_id: "KNOW-0008"
category: definition_of_done
title: "Definition of Done"
status: planned
owner: ""
created: '2026-06-30'
updated: '2026-06-30'
reviewed_on: ""
related_notes: []
tags: [apovault, knowledge, definition_of_done]
---

# Definition of Done

> **(verify)** — team-owned; not codebase-extractable. Define what "done" means for a
> figma-sync change.
>
> Expected answers when this section is filled:
> - [ ] What gates a registry/skill change? (peer design review? a real build-run against a
>       Figma file before merge?)
> - [ ] Must `plugin.json` semver be bumped on convention/registry changes, and must the
>       `registry/components.json` `version` move on DS-spec changes?
> - [ ] What must a PR description include (ARCHITECTURE decision refs, screenshots of a
>       verified build, mapping diff)?
> - [ ] Manual reviewer checks before merge (run `/figma-sync:create-*` end-to-end? confirm
>       both directions still build?).
> - [ ] Merge/deploy gate — who runs `/plugin update` for the team, and when?
>
> Look at: `ARCHITECTURE.md` §6 (versioning), `README.md` (install/update), team norms.

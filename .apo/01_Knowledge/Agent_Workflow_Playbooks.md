---
note_type: knowledge
template_version: 1
contract_version: 1
knowledge_id: "KNOW-0009"
category: playbooks
title: "Agent Workflow Playbooks"
status: planned
owner: ""
created: '2026-06-30'
updated: '2026-06-30'
reviewed_on: ""
related_notes: []
tags: [apovault, knowledge, playbooks]
---

# Agent Workflow Playbooks

> **(verify)** — recurring recipes for changing figma-sync itself (not for running a build —
> that lives in the skills). Capture after the first few `/apo:plan` dogfood runs.
>
> Expected answers when this section is filled:
> - [ ] **Adding a new command** — what files (`commands/<verb>.md` + a skill? + an agent?),
>       frontmatter shape (`description`, `argument-hint`), README + ARCHITECTURE table
>       updates, semver bump.
> - [ ] **Updating a DS constant** — edit `registry/components.json`, bump its `version`,
>       (planned) run `/figma-sync:apply-ds-update`; what to verify.
> - [ ] **Discovering + committing a new component/library key** — `search_design_system`
>       flow, where the key goes (forward vs `androidToIos`), filling an empty reverse key.
> - [ ] **Adding a new direction/platform** — what the mirrored skill + agent + registry block
>       must contain.
> - [ ] **Editing a skill** — keep literals out of prose (registry-wins rule); update both
>       mirror skills when a shared convention changes.
>
> Look at: `ARCHITECTURE.md` §5 (workflows) and §7 (roadmap), the existing
> `commands/*.md` + `skills/*/SKILL.md` as templates to mirror.

---
note_type: knowledge
template_version: 1
contract_version: 1
knowledge_id: "KNOW-0010"
category: taxonomy
title: "Bug Taxonomy"
status: planned
owner: ""
created: '2026-06-30'
updated: '2026-06-30'
reviewed_on: ""
related_notes: []
tags: [apovault, knowledge, taxonomy]
---

# Bug Taxonomy

> **(verify)** — team-owned; define the severity ladder, categories, and tracker for
> figma-sync defects.
>
> Expected answers when this section is filled:
> - [ ] Severity ladder (sev-1 … sev-4) in team terms — e.g. is "wrong component key produces
>       a broken Figma build" sev-1, vs "title slightly off" sev-3?
> - [ ] Which categories apply (logic / integration / regression / ux / docs / …)? Likely
>       figma-sync-specific buckets: stale registry key, layout/positioning, typography
>       mapping, header-rule, mapping-file corruption.
> - [ ] Lifecycle steps (new → … → closed) and who verifies a design-output fix.
> - [ ] Primary tracker — vault items (`apo item`), JIRA, or GitHub issues?
>
> Look at: the `gotchas` sections in both `skills/*/SKILL.md` (they enumerate the real failure
> modes), team conventions.

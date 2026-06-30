---
description: (PLANNED) Apply a design-system spec change across all mapped Figma files for a platform
argument-hint: "[ios|android] [summary of what changed]"
---

> **STATUS: PLANNED — not yet implemented.** Do not make changes yet. Explain the intended
> behavior to the user, confirm scope, and stop unless they explicitly ask to prototype it.

Roll a design-system change (e.g. a new Material spec, a token change, a component update)
across every Figma file this team has mapped, for the platform in `$ARGUMENTS`.

Intended behavior:
1. Update `${CLAUDE_PLUGIN_ROOT}/registry/components.json` to reflect the new spec (keys,
   tokens, layout values). This commit is the auditable record of the DS change.
2. Read `~/.figma-sync/mappings.json` for the set of target files/sections.
3. Preflight the Figma MCP (`whoami`).
4. Fan out **one agent per file** (see `agents/screen-builder`) to apply the delta — e.g.
   re-instantiate updated components, rebind tokens, adjust layout constants — following the
   `figma-sync:ios-to-android` conventions.
5. Screenshot-verify each file; report a per-file summary; never silently partial-apply.

Open design questions to resolve before building: how to express a "delta" (full re-apply vs
targeted patch), idempotency/versioning per file, and rollback.

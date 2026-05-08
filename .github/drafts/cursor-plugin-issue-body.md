# Cursor plugin support: discovery, hook coverage, and runtime distribution gaps

> **Title (paste into issue title field):** `Cursor plugin support: discovery, hook coverage, and runtime distribution gaps`
>
> **Suggested labels:** `enhancement`, `adapter:cursor`, `discussion`

---

Following the methodology @mksglu spelled out in #473 — mapped, root-caused, and grilled before any PR. Full investigation in my fork:

📄 **[cursor-plugin-proposal.md](https://github.com/maxwell-orange/context-mode/blob/docs/cursor-plugin-proposal/.github/drafts/cursor-plugin-proposal.md)**

## Context

@mksglu mentioned (privately) that Cursor plugin support is needed soon, with a pointer to the methodology used in #473. Per that process I treated this as a discovery exercise, not a packaging task, and surfaced what "we need cursor plugin not just mcp only" actually decomposes into.

## TL;DR

The single phrase hides **three independent problems**:

| # | Problem | Packaging-fixable? |
|---|---------|--------------------|
| 1 | **Discovery** — context-mode invisible in Cursor's marketplace search panel | ✅ yes — list on cursor.com/marketplace |
| 2 | **Routing strength** — adapter only registers 3 of ~20 native Cursor v1.7 hook events (`preToolUse`/`postToolUse`/`stop`). 8 high-value events unused: `beforeShellExecution`, `beforeReadFile`, `beforeSubmitPrompt`, `preCompact`, `sessionStart`, `afterAgentResponse`, `postToolUseFailure`, `afterFileEdit` | ⚠️ partially — but mostly **independent of packaging**, can ship today inside existing `.cursor/hooks.json` flow |
| 3 | **Runtime distribution** — Cursor's manifest has no `${CURSOR_PLUGIN_ROOT}` equivalent (and `pluginRoot` was just removed from [plugin-template](https://github.com/cursor/plugin-template/commit/4621607)). So even with a marketplace plugin, users **still need `npm i -g context-mode`**. Submission checklist also forbids `..` and absolute paths | ❌ no — structural Cursor-side limitation, needs upstream change |

Marketplace listing fixes #1 only. It does **not** fix routing strength or remove the npm prerequisite. Treating "ship Cursor plugin" as one big task would be the workaround warned against in #473.

## Mapping (all 14 adapters)

Only **2 of 14** platforms today have a first-party plugin format we can target: claude-code (`.claude-plugin/`) and openclaw (`.openclaw-plugin/`). Cursor would be the third. The rest stay on `npm i -g` + manual config — no per-adapter plugin abstraction needed.

Also found one piece of dead code: `hooks/cursor/afteragentresponse.mjs` exists but `configs/cursor/hooks.json` never registers `afterAgentResponse`.

## Proposed split — two independent PRs

**PR-A. Hook coverage expansion** (no plugin, no marketplace)
Register the 8 unused Cursor v1.7 hook events in `configs/cursor/hooks.json`. Pure addition, existing 3 hooks unchanged. This is where actual context-saving improvements come from. Backward-compatible by construction.

**PR-B. Marketplace plugin** (after PR-A)
Single-plugin layout: `.cursor-plugin/plugin.json` at repo root + vendored copies of rule/hooks/mcp.json under `plugins/cursor/` (the `..`-forbidden path constraint forces vendoring; can't reference `configs/cursor/*` directly). Version sync via existing `scripts/version-sync.mjs`. Manifest is small — auto-discovers components from default folders.

Plugin install + existing `.cursor/hooks.json` users would get **duplicate hook firings**. Mitigation: `context-mode doctor` detects both and warns. Migration documented in `plugins/cursor/README.md`.

## Open questions — would like to align before opening any PR

1. Plugin in main repo (multi-file vendor under `plugins/cursor/`) or new repo `mksglu/context-mode-cursor-plugin`?
2. PR-A and PR-B sequential or parallel?
3. Logo / brand asset for marketplace tile? (submission requires it committed in repo)
4. Wire up the dead `afteragentresponse.mjs` in PR-A or remove it?
5. Should I run a quick verification pass first to confirm Cursor v1.7 validator now accepts `sessionStart` (forum #149566 reported rejection pre-v1.7; docs now list it as supported but I haven't tested live)?

## Items flagged NEEDS VERIFICATION before any PR

- `sessionStart` validator acceptance in current Cursor release
- `additional_context` injection bug status ([forum #155689](https://forum.cursor.com/t/native-posttooluse-hooks-accept-and-log-additional-context-successfully-but-the-injected-context-is-not-surfaced-to-the-model/155689), [#156157](https://forum.cursor.com/t/cursor-hooks-additional-context-not-injected-in-agent-context-in-posttooluse/156157)) — affects PR-A's `postToolUse` value
- `MCP:ctx_*` matcher syntax in current Cursor (adapter assumes pre-1.7 format)
- `context-mode` PATH discovery inside Cursor's plugin sandbox

Happy to run the verification on the `next` branch with the [grill-me skill](https://github.com/mksglu/context-mode/blob/next/skills/grill-me/SKILL.md) once the open questions above are settled.

cc @mksglu — same path, different issue per your guidance in #473.

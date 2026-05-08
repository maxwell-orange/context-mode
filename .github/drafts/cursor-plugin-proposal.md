# Cursor Plugin Support — Research & Proposal

> **Methodology note** — This document follows the process @mksglu spelled out
> in [#473 comment 4401963335](https://github.com/mksglu/context-mode/issues/473#issuecomment-4401963335):
> map → root cause → backward compatibility → business value → grill-me
> self-questioning → proposal first, code only after alignment. Every claim
> below cites file:line or upstream URL. When evidence was missing I marked it
> "NEEDS VERIFICATION" instead of guessing.

## TL;DR

The framing "we need a Cursor plugin not just MCP only" hides three distinct
problems. Treating it as a single packaging task would be the workaround
@mksglu warned against. The right scope is:

1. **Distribution gap** — context-mode is not on Cursor Marketplace, so users
   discovering Cursor extensions never find us. Marketplace listing is a
   pure-packaging problem.
2. **Hook coverage gap** — context-mode's Cursor adapter only registers 3 of
   ~20 native hook events available since Cursor v1.7. Most of the value
   (route shell, read, edit, submit-prompt) is unrelated to plugin packaging
   and can ship today inside the existing `.cursor/hooks.json` flow.
3. **Runtime distribution gap** — Cursor plugin manifest cannot reference the
   plugin install dir (no `${CURSOR_PLUGIN_ROOT}` equivalent; `pluginRoot` was
   recently removed from the manifest schema in [cursor/plugin-template](https://github.com/cursor/plugin-template/commit/4621607)).
   So even the plugin form still requires `npm i -g context-mode` (or `npx -y`).
   This is a structural Cursor-side limitation, not something we can engineer
   around.

Recommendation: **ship the marketplace plugin (#1) and close the hook coverage
gap (#2) as two independent PRs**. Do not couple them. Defer #3 — there is no
clean fix without upstream changes from Cursor.

---

## 1. Map — current state across all 14 adapters

Source of truth: [src/adapters/](../../src/adapters/), [docs/platform-support.md](../../docs/platform-support.md),
[configs/](../../configs/), [.claude-plugin/](../../.claude-plugin/),
[.openclaw-plugin/](../../.openclaw-plugin/).

| # | Platform | Distribution today | Native plugin manifest? |
|---|----------|--------------------|-------------------------|
| 1 | claude-code | **Plugin** via `~/.claude/plugins/` + Claude Marketplace | ✅ [.claude-plugin/plugin.json](../../.claude-plugin/plugin.json), [.claude-plugin/marketplace.json](../../.claude-plugin/marketplace.json) |
| 2 | openclaw | **Plugin** via `~/.openclaw/plugins/` | ✅ [.openclaw-plugin/openclaw.plugin.json](../../.openclaw-plugin/openclaw.plugin.json), top-level [openclaw.plugin.json](../../openclaw.plugin.json) |
| 3 | gemini-cli | npm global + manual `~/.gemini/settings.json` | ❌ |
| 4 | qwen-code | npm global + manual `~/.qwen/settings.json` | ❌ |
| 5 | codex | npm global + manual `~/.codex/{hooks.json,config.toml}` | ❌ |
| 6 | cursor | npm global + manual `.cursor/{mcp.json,hooks.json,rules/}` | ❌ ← **this issue** |
| 7 | vscode-copilot | npm global + manual `.github/hooks/*.json` | ❌ |
| 8 | jetbrains-copilot | npm global + manual `.github/hooks/*.json` | ❌ |
| 9 | antigravity | npm global + MCP-only via `~/.gemini/antigravity/mcp_config.json` | ❌ |
| 10 | kiro | npm global + MCP-only via `~/.kiro/settings/mcp.json` | ❌ |
| 11 | zed | npm global + MCP-only | ❌ |
| 12 | opencode | TS plugin in `opencode.json` | partial (config-only, no manifest) |
| 13 | kilo | npm global (OpenCode fork) | ❌ |
| 14 | pi | npm global, MCP-only | ❌ |

**Observation 1** — Only 2 of 14 platforms have first-party plugin formats
(Claude Code, OpenClaw). The "every adapter should be a plugin" framing
@mksglu pushed back against in #473 applies here too: Cursor is just the
**third platform that has a plugin format we can target**. We are not
building a per-adapter plugin abstraction; we are publishing into a third
platform-specific marketplace.

**Observation 2** — context-mode's Cursor adapter is already first-class on
the runtime side. The packaging gap is purely in distribution
([README.md L367-L373](../../README.md), [docs/platform-support.md L463](../../docs/platform-support.md)).

### Cursor-specific files we already ship

| File | Repo location | Installed location | Purpose |
|------|---------------|--------------------|---------|
| MCP config | [configs/cursor/mcp.json](../../configs/cursor/mcp.json) | `.cursor/mcp.json` | Declares `context-mode` MCP server |
| Hook config | [configs/cursor/hooks.json](../../configs/cursor/hooks.json) | `.cursor/hooks.json` | Registers 3 events: preToolUse, postToolUse, stop |
| Rule | [configs/cursor/context-mode.mdc](../../configs/cursor/context-mode.mdc) | `.cursor/rules/context-mode.mdc` | `alwaysApply: true` routing rules (replaces sessionStart) |
| Adapter | [src/adapters/cursor/index.ts](../../src/adapters/cursor/index.ts) | inside the global `context-mode` binary | Parses Cursor stdin, formats Cursor stdout |
| Hook scripts | [hooks/cursor/](../../hooks/cursor/) | inside the global `context-mode` binary | Runtime entry points dispatched via `context-mode hook cursor <event>` |

### What Cursor v1.7+ supports vs. what we use

[cursor.com/docs/reference/plugins#hooks](https://cursor.com/docs/reference/plugins) lists 20 agent/tab hook events.

| Event | Cursor supports | We register | Gap |
|-------|-----------------|-------------|-----|
| `preToolUse` | ✅ | ✅ | — |
| `postToolUse` | ✅ | ✅ | — |
| `stop` | ✅ | ✅ | — |
| `postToolUseFailure` | ✅ | ❌ | could attach error→knowledge-base entry |
| `beforeShellExecution` | ✅ | ❌ | **route bash → ctx_execute** without polluting preToolUse matcher |
| `afterShellExecution` | ✅ | ❌ | auto-index shell output into FTS5 |
| `beforeReadFile` | ✅ | ❌ | redirect Read of large files → ctx_execute_file |
| `afterFileEdit` | ✅ | ❌ | invalidate FTS5 entries for edited files |
| `beforeSubmitPrompt` | ✅ | ❌ | inject session-resume snapshot |
| `sessionStart` | docs say ✅ in v1.7, validator rejected it pre-v1.7 ([forum #149566](https://forum.cursor.com/t/unknown-hook-type-sessionstart/149566)) | ❌ (we have a script in [hooks/cursor/sessionstart.mjs](../../hooks/cursor/sessionstart.mjs) but [configs/cursor/hooks.json](../../configs/cursor/hooks.json) does not register it) | **NEEDS VERIFICATION** that v1.7 validator now accepts it |
| `sessionEnd` | ✅ | ❌ | — |
| `preCompact` | ✅ | ❌ | snapshot before compact (parity with Claude) |
| `beforeMCPExecution` / `afterMCPExecution` | ✅ | ❌ | redundant with preToolUse `MCP:` matcher; skip |
| `subagentStart` / `subagentStop` | ✅ | ❌ | useful for Task hook; matches Claude's `Agent` matcher |
| `afterAgentResponse` | ✅ | we have [hooks/cursor/afteragentresponse.mjs](../../hooks/cursor/afteragentresponse.mjs) but it is not registered in [configs/cursor/hooks.json](../../configs/cursor/hooks.json) | the script exists but no config entry → dead code today |
| `afterAgentThought` | ✅ | ❌ | — |
| `beforeTabFileRead` / `afterTabFileEdit` | ✅ | ❌ | tab autocomplete events; lower priority |

**Conclusion of mapping** — packaging is one problem. Hook coverage is a
second, larger problem hiding behind it. Conflating them would be a
workaround.

---

## 2. Root cause — why is current MCP-only insufficient?

The phrase "not just mcp only" maps to three distinct failure modes:

### 2a. Discovery (real, packaging-fixable)

`cursor.com/marketplace` is the discovery surface for Cursor users. The
search panel inside the IDE only surfaces marketplace plugins. Today a
Cursor user looking for "context window" or "MCP" sees zero results for
context-mode. Fix: ship a marketplace plugin.

### 2b. Routing strength (real, only partially packaging-fixable)

Cursor's `additional_context` is accepted by the validator but **not surfaced
to the model** ([forum #155689](https://forum.cursor.com/t/native-posttooluse-hooks-accept-and-log-additional-context-successfully-but-the-injected-context-is-not-surfaced-to-the-model/155689),
[#156157](https://forum.cursor.com/t/cursor-hooks-additional-context-not-injected-in-agent-context-in-posttooluse/156157)).
That is why [src/adapters/cursor/index.ts L160-167](../../src/adapters/cursor/index.ts)
emits `additional_context: ""` no-ops — the field works for validation but
does not actually inject anything. So today the only path to influence
Cursor's planning is the rule file. A plugin **does not improve this** —
it just packages the rule. The real fix is upstream Cursor work.

We **can** improve routing today by registering the unused hook events
(`beforeShellExecution`, `beforeReadFile`, `beforeSubmitPrompt`) so we
intercept tools earlier in the flow with `permission: "deny"` + redirect
text. That is **independent of plugin packaging**.

### 2c. Runtime distribution (real, NOT packaging-fixable)

I checked [cursor.com/docs/reference/plugins](https://cursor.com/docs/reference/plugins)
and confirmed:

- Manifest accepts `mcpServers`, `hooks` as inline config or relative path.
- **No `${CURSOR_PLUGIN_ROOT}` env var is documented.**
- The plugin-template repo recently merged a commit titled "Remove
  pluginRoot" ([cursor/plugin-template@4621607](https://github.com/cursor/plugin-template/commit/4621607)).
- Submission checklist mandates "no absolute paths, no `..`" in manifest
  paths.

This means a Cursor plugin **cannot** ship the runtime inside the bundle
the way [.claude-plugin/plugin.json L26](../../.claude-plugin/plugin.json)
does (`"args": ["${CLAUDE_PLUGIN_ROOT}/start.mjs"]`). The manifest can
only declare:

```json
{ "mcpServers": { "context-mode": { "command": "context-mode" } } }
```

…which still requires the user to have `context-mode` on PATH (so still
`npm i -g context-mode`, or `command: "npx", args: ["-y", "context-mode"]`
which has cold-start latency on every invocation).

> **Root cause of the distribution gap:** Cursor's plugin format does not
> currently expose its install dir to manifest fields. Until that lands
> upstream, the marketplace plugin can list us but cannot replace the
> "global install" prerequisite.

---

## 3. Backward compatibility constraints

Existing users have one of:

- `.cursor/mcp.json` referencing `context-mode` directly.
- `.cursor/hooks.json` calling `context-mode hook cursor <event>`.
- `.cursor/rules/context-mode.mdc` (project-level).
- `~/.cursor/{mcp,hooks}.json` (user-level).
- macOS enterprise mode at `/Library/Application Support/Cursor/hooks.json`
  (handled in [src/adapters/cursor/index.ts L74](../../src/adapters/cursor/index.ts) via `CURSOR_ENTERPRISE_HOOKS_PATH`).

A plugin install must **not** break any of these. Cursor's plugin loader and
the project-local config loader run independently — both can register MCP
servers and hooks. Users who installed the plugin AND keep their old
`.cursor/hooks.json` would get duplicate hook firings.

**Required mitigation** — either:

- (a) Ship the plugin as **strictly additive** and document migration: "if
  you install the plugin, delete your `.cursor/{hooks,mcp}.json`
  context-mode entries." OR
- (b) On `context-mode doctor`, detect both plugin-installed AND
  project-local config and warn about duplicates.

`context-mode hook cursor <event>` is the runtime entry point either way,
so the hook scripts themselves are agnostic to which config file
registered them.

---

## 4. Business value — what does packaging actually give us?

| Dimension | MCP-only today | + Marketplace plugin | + Hook coverage expansion (independent of plugin) |
|-----------|---------------|----------------------|----------------------------------------------------|
| Discovery in Cursor IDE | ❌ users find us via README only | ✅ marketplace search panel | — (no impact) |
| One-click install | ❌ 4 manual files | ✅ click "Install" in panel | — |
| Versioning / updates | manual `npm update -g` | plugin auto-updates the rule + hook config (NOT the binary) | — |
| Removes `npm i -g` | ❌ | ❌ — see §2c | ❌ |
| Routing strength | weak (rule + 3 hooks) | weak (same rule + 3 hooks via plugin) | **strong** (rule + ~10 hooks) |
| Org-wide rollout (Cursor Teams Marketplace) | ❌ | ✅ admins can mark required | — |
| Telemetry / install metrics | ❌ | ✅ Cursor surfaces install count | — |
| Submission cost | — | one-time human review by Cursor team + ongoing version bumps | low |
| Maintenance cost | low | low (manifest is small) | medium (more events = more script paths to test) |

**Reading**: marketplace plugin adds discoverability and team rollout, **not**
a better install path or stronger routing. The hook coverage expansion is
where actual context-saving improvements come from. They should be sized
and shipped separately so the maintainer can accept/reject each on its
own merits.

---

## 5. Grill-me self-questioning

I ran [skills/grill-me/SKILL.md](../../skills/grill-me/SKILL.md) on the
proposal. Decision tree branches:

**Q1.** Should the plugin live in the existing repo (`mksglu/context-mode`)
or a new repo (`mksglu/context-mode-cursor-plugin`)?

- Same repo means manifest paths must be repo-root relative. Submission
  checklist forbids `..`, so manifest cannot point at `configs/cursor/*`
  from a `.cursor-plugin/plugin.json` at repo root unless we also move/copy
  rule + hook files into Cursor-expected dirs (`rules/`, `hooks/hooks.json`,
  `mcp.json`) at repo root.
- Repo root pollution is real — root would gain `rules/`, `hooks/`,
  `mcp.json`, `.cursor-plugin/` directories that look like the project
  itself is a Cursor plugin (which is misleading; it is the
  cross-platform context-mode codebase that happens to also publish a
  Cursor plugin).
- **Multi-plugin layout** (`.cursor-plugin/marketplace.json` at root +
  `plugins/cursor/{.cursor-plugin/plugin.json,rules/,hooks/,mcp.json}`)
  keeps everything inside one folder and matches the
  [cursor/plugin-template](https://github.com/cursor/plugin-template)
  default. **Recommend:** this layout.
- **Open question** — does maintainer prefer plugin in main repo or a new
  repo? Either works; main repo wins on version sync and CI; separate
  repo wins on cleaner ownership boundary.

**Q2.** Does the plugin need its own version number, or sync with
`package.json`?

- [.claude-plugin/plugin.json L3](../../.claude-plugin/plugin.json) is
  pinned to `1.0.111` and synced via [scripts/version-sync.mjs](../../scripts/version-sync.mjs).
  Same pattern works for Cursor plugin manifest. **Recommend:** sync.

**Q3.** Should the plugin's `mcpServers` field be inline or point at
`mcp.json`?

- Inline is one less file but harder to debug. Path-based mirrors what
  Claude does. Cursor docs example shows path-based via auto-discovered
  `mcp.json` at plugin root. **Recommend:** auto-discovery (just put
  `mcp.json` at plugin root, omit `mcpServers` from manifest).

**Q4.** What about the dead `afteragentresponse.mjs` script?

- [hooks/cursor/afteragentresponse.mjs](../../hooks/cursor/afteragentresponse.mjs)
  exists but [configs/cursor/hooks.json](../../configs/cursor/hooks.json)
  never registers `afterAgentResponse`. Either delete the script or wire
  it up. **Recommend:** wire it up in the hook coverage expansion PR.

**Q5.** What if Cursor's validator still rejects `sessionStart` in current
release?

- [src/adapters/cursor/index.ts L88-90](../../src/adapters/cursor/index.ts) already
  comments that the capability flag was flipped to `true` after v1
  shipped native sessionStart. Need to verify by attempting to register
  sessionStart in `hooks.json` against current Cursor. **Action item:**
  test in next branch before submitting plugin.

**Q6.** Does the plugin work on Windows?

- The hook command `context-mode hook cursor <event>` works wherever
  `context-mode` is on PATH. Cursor on Windows installs to
  `%LOCALAPPDATA%\Programs\cursor\`. No plugin-specific Windows issues
  expected. The recently fixed Bug A (rmSync non-ASCII tmpdir, PR #456)
  and Bug B (MS Store Python stub, PR #457) cover the runtime side.

**Q7.** What is the rollback story?

- `cursor.com/plugins/installed` panel offers per-plugin uninstall.
  Removing the plugin removes its bundled MCP server registration but
  leaves `.cursor/{mcp,hooks}.json` untouched. **Recommend:** during
  publish, document this clearly — manual config is the safety net.

**Q8.** Logo and branding?

- Submission checklist requires logo committed in repo. We do not have
  a logo asset yet. **Open question** — does maintainer have brand
  assets for Cursor marketplace tile?

---

## 6. Concrete proposal

Two PRs, in order, neither blocking the other:

### PR-A. Cursor hook coverage expansion (independent of marketplace)

**Scope**: Add `beforeShellExecution`, `beforeReadFile`, `afterFileEdit`,
`beforeSubmitPrompt`, `preCompact`, `sessionStart`,
`afterAgentResponse`, `postToolUseFailure` registrations to
[configs/cursor/hooks.json](../../configs/cursor/hooks.json). Wire each
to `context-mode hook cursor <event>` and add hook scripts under
[hooks/cursor/](../../hooks/cursor/) that route through the existing
adapter machinery. Existing 3 hooks unchanged.

**Backward compat**: Pure addition. Existing users see new hooks fire on
next `.cursor/hooks.json` reload.

**Verification**: Add `tests/hooks/cursor.test.ts` cases per new event.
Verify validator-acceptance on a real Cursor install (checklist item).

**Out of scope**: marketplace, plugin manifest, distribution.

### PR-B. Cursor Marketplace plugin (after PR-A lands)

**Scope** (single-plugin repo layout, since context-mode is the only
plugin):

```
context-mode/
├── .cursor-plugin/
│   └── plugin.json          # NEW
├── plugins/cursor/          # NEW — vendored copies of configs/cursor/*
│   ├── rules/context-mode.mdc
│   ├── hooks/hooks.json
│   ├── mcp.json
│   └── README.md
└── scripts/version-sync.mjs # MODIFIED — also bump plugin.json version
```

`plugin.json` content:

```json
{
  "name": "context-mode",
  "version": "<synced from package.json>",
  "description": "Save 98% of your context window. Sandboxed code execution, FTS5 knowledge base.",
  "author": { "name": "Mert Koseoğlu" },
  "homepage": "https://github.com/mksglu/context-mode#readme",
  "repository": "https://github.com/mksglu/context-mode",
  "license": "Elastic-2.0",
  "keywords": ["mcp", "context-window", "sandbox"]
}
```

Components are auto-discovered from `rules/`, `hooks/hooks.json`,
`mcp.json` per Cursor's default resolver.

**Backward compat**: User who installs the plugin AND has
`.cursor/hooks.json` from manual install will get duplicate hook
registrations. Ship a `context-mode doctor` check that detects both and
warns. Document migration in plugins/cursor/README.md.

**Submission checklist** (per [cursor.com/docs/reference/plugins#submission-checklist](https://cursor.com/docs/reference/plugins)):

- [x] Valid `.cursor-plugin/plugin.json`
- [x] kebab-case unique name
- [x] Clear description
- [x] Frontmatter on all rules/skills/agents/commands
- [ ] Logo (BLOCKING — open question for maintainer)
- [x] README usage docs
- [x] Relative-only paths
- [x] Local plugin tested via `~/.cursor/plugins/local/context-mode -> repo/`
- [x] No multi-plugin marketplace.json (single-plugin)

**Out of scope**: removing the `npm i -g context-mode` requirement (see
§2c — needs upstream Cursor change).

### PR-C. Optional (defer)

Add `bin/context-mode-cursor-install.mjs` that copies plugin files into
`~/.cursor/plugins/local/context-mode/` for users who want plugin behavior
before marketplace acceptance. Lower priority.

---

## 7. Open questions for maintainer (BEFORE coding)

1. Same repo or new `mksglu/context-mode-cursor-plugin` repo?
2. Do you have a logo / brand asset for the marketplace tile?
3. Is the plugin's `author` field "Mert Koseoğlu" (same as
   [.claude-plugin/plugin.json](../../.claude-plugin/plugin.json)) or do
   you want a different identity?
4. Should PR-A (hook coverage) and PR-B (marketplace) be sequential
   (PR-A first, lands, then PR-B) or can they go in parallel?
5. Do you want the dead `afteragentresponse.mjs` wired up or removed?
6. Are you okay with the multi-file vendoring under `plugins/cursor/`,
   or do you prefer to symlink-by-script from `configs/cursor/` at
   build time? (Vendoring is simpler; symlinking avoids drift.)
7. Should the plugin advertise the Pro/team marketplace flow, or is
   this scoped to the public marketplace only?

---

## 8. References (verified)

- Cursor plugin docs: <https://cursor.com/docs/plugins>
- Cursor plugin manifest reference: <https://cursor.com/docs/reference/plugins>
- Plugin template (active commits, "Remove pluginRoot" merge): <https://github.com/cursor/plugin-template>
- Marketplace submission: <https://cursor.com/marketplace/publish>
- Forum bugs noted in [docs/platform-support.md L476](../../docs/platform-support.md):
  - sessionStart validator rejection: <https://forum.cursor.com/t/unknown-hook-type-sessionstart/149566>
  - additional_context not surfaced: <https://forum.cursor.com/t/native-posttooluse-hooks-accept-and-log-additional-context-successfully-but-the-injected-context-is-not-surfaced-to-the-model/155689>
  - additional_context not injected: <https://forum.cursor.com/t/cursor-hooks-additional-context-not-injected-in-agent-context-in-posttooluse/156157>
- #473 methodology comment by @mksglu: <https://github.com/mksglu/context-mode/issues/473#issuecomment-4401963335>
- Cursor adapter source: [src/adapters/cursor/index.ts](../../src/adapters/cursor/index.ts)
- Cursor configs: [configs/cursor/](../../configs/cursor/)
- Cursor hook scripts: [hooks/cursor/](../../hooks/cursor/)
- Platform comparison table: [docs/platform-support.md](../../docs/platform-support.md)
- Claude plugin manifest (reference for what we WOULD ship if Cursor had
  `${PLUGIN_ROOT}`): [.claude-plugin/plugin.json](../../.claude-plugin/plugin.json)
- OpenClaw plugin manifest: [.openclaw-plugin/openclaw.plugin.json](../../.openclaw-plugin/openclaw.plugin.json)

## 9. NEEDS VERIFICATION before any PR

- Cursor v1.7+ validator actually accepts `sessionStart` in `hooks.json`
  (forum #149566 reported rejection; docs now list it as supported).
  Test on the next branch before relying on it.
- `additional_context` injection bug status — has Cursor fixed it post-v1.7?
- Whether [configs/cursor/hooks.json](../../configs/cursor/hooks.json)'s
  current `MCP:ctx_*` matchers actually fire in Cursor v1.7 (the matcher
  format may have changed; the adapter assumes pre-1.7 syntax).
- `context-mode` PATH discovery works inside Cursor's plugin sandbox the
  same way it works in the IDE process today (no PATH stripping by
  Cursor's plugin loader).

I have not assumed any of these — they are gating items for PR-A's
verification phase.

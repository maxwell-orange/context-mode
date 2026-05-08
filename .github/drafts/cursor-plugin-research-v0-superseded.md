# 调研报告：Cursor Plugin 支持

## TL;DR

Cursor 最近上线了 **Plugins / Marketplace** 系统（[cursor.com/marketplace](https://cursor.com/marketplace)），把规则、技能、agents、命令、MCP 服务器、hooks 打包为可一键安装的 bundle。当前 context-mode 在 Cursor 上是「散装文件」配置（用户要手动复制 3 个文件到 `.cursor/`），需要把它打包成 **官方 Cursor Plugin**，提交到 Marketplace 审核上架。这跟 Claude Code 已有的 plugin marketplace 安装路径是同等级的工作。

---

## Cursor Plugin 是什么

Cursor 在 v1.7+ 推出的统一插件机制（来自 [docs.cursor.com/plugins](https://cursor.com/docs/plugins)）：

| 组件 | 作用 |
|------|------|
| **Rules** | `.mdc` 文件（Always / Agent Decides / Manual） |
| **Skills** | 复杂任务的 agent 能力 |
| **Agents** | 自定义 agent 配置 |
| **Commands** | agent 可执行的命令 |
| **MCP servers** | MCP 集成 |
| **Hooks** | 事件触发的脚本 |

**关键事实**：

- 插件 = 一个含 `.cursor-plugin/plugin.json` 清单的目录
- 分发方式：Git 仓库 → 提交 [cursor.com/marketplace/publish](https://cursor.com/marketplace/publish) → Cursor 团队人工审核上架
- 本地开发可放到 `~/.cursor/plugins/local/<name>/`
- 通过 VS Code 扩展也能用 `vscode.cursor.plugins.registerPath()` 注册
- 模板仓库：[github.com/cursor/plugin-template](https://github.com/cursor/plugin-template)
- 团队版还支持私有 Team Marketplace（SCIM 分发）

---

## context-mode 在 Cursor 上的现状

[docs/platform-support.md L463-L515](docs/platform-support.md#L463) 说 Cursor 是「first-class adapter」，但**安装流程是手动散装**：

| 文件 | 目标位置 | 内容 |
|------|----------|------|
| [configs/cursor/mcp.json](configs/cursor/mcp.json) | `.cursor/mcp.json` | MCP 服务器声明 |
| [configs/cursor/hooks.json](configs/cursor/hooks.json) | `.cursor/hooks.json` | preToolUse / postToolUse / stop |
| [configs/cursor/context-mode.mdc](configs/cursor/context-mode.mdc) | `.cursor/rules/context-mode.mdc` | 路由规则（替代被拒的 sessionStart） |

[hooks/cursor/](hooks/cursor/) 提供运行时实现（pretooluse / posttooluse / stop / sessionstart / afteragentresponse）；hook 命令格式为 `context-mode hook cursor <event>`。

**当前痛点**（来自 [README.md L367-L373](README.md#L367) 与 docs）：

1. 用户要先 `npm install -g context-mode`，再分别配置 3 个文件 → 对比 Claude Code 一行 `claude plugin install` 体验差很多
2. Cursor 原生的 `sessionStart` 被 validator 拒绝 ([forum #149566](https://forum.cursor.com/t/unknown-hook-type-sessionstart/149566))，所以路由提示靠 `.mdc` 规则文件交付 —— **这正是 plugin 的天然形态**
3. `additional_context` 被 Cursor 接受但不下发给模型 ([forum #155689](https://forum.cursor.com/t/native-posttooluse-hooks-accept-and-log-additional-context-successfully-but-the-injected-context-is-not-surfaced-to-the-model/155689)) —— 也只能靠 rules 弥补

> Cursor plugin 的格式恰好天然可以把 `.mdc` 规则、hooks、MCP server 三件套一起打包，**完美对症**。

---

## 为什么"now we need cursor plugin asap"

1. **生态对齐**：Claude Code 已有 plugin marketplace 安装路径（见 [#249](https://github.com/mksglu/context-mode/issues/249) 已修），OpenClaw 有原生插件入口（[openclaw.plugin.json](openclaw.plugin.json)）。Cursor Marketplace 上线后，竞品（cursor.directory 上的同类工具）会迅速提交插件，先到先得 —— mindshare、SEO、Marketplace 推荐位都靠先上架。
2. **现有散装配置的硬伤**：上面列的 3 个文件 + 全局 npm install 很容易出现版本错配、路径错误、CLAUDE_CONFIG_DIR 与 cursor 配置串台。Plugin 模式自带版本号、官方分发，等同于把 #249 那种"marketplace 安装漏 build/"问题在 Cursor 一侧就堵掉。
3. **#417 募集 DevRel 信号**：项目正在拉增长，Marketplace 是 Cursor 用户群体最高效的获客渠道。
4. **路由强度**：plugin 把 `.mdc` 规则 + MCP tool descriptions 一并下发，比起当前依赖 README 引导的"复制粘贴"安装，路由命中率会显著提升 —— 这是 context-mode 核心 KPI。

---

## 实现工作量预估

最小可上架版本（假设保留现有 hook 命令与全局 `context-mode` 二进制）：

```
.cursor-plugin/
└── plugin.json                # 新增：清单（name, description, version, author, components）
rules/
└── context-mode.mdc           # 复用 configs/cursor/context-mode.mdc
hooks.json                     # 复用 configs/cursor/hooks.json
mcp.json                       # 复用 configs/cursor/mcp.json
README.md                      # plugin 商店描述（短）
```

或者把 plugin 作为独立 Git 仓库（推荐，方便 Cursor 团队审核），里面只装上述文件 + 一个调用 `npm i -g context-mode` 的 setup 提示。

**可能的坑**：

- Plugin 的 hooks.json schema 是否完全兼容 `.cursor/hooks.json`？文档现在是统一的，但需验证 `command: "context-mode hook cursor pretooluse"` 在 plugin 沙箱里能否执行（要求 PATH 上有 `context-mode` 二进制 —— 用户得先 `npm i -g`，或 plugin 用 `npx` 兜底）。
- `.cursor-plugin/plugin.json` 的版本号需要和 `package.json` 同步（`scripts/version-sync.mjs` 已有版本同步框架）。
- Cursor 官方审核会人工 review，要确认 hook 调用的进程不做未声明的网络/文件系统访问（context-mode 的 sandbox 已经处理这部分，提交说明里需写清楚）。
- 现有的 `sessionStart` 还是被 validator 拒，plugin 版本初次仍然只能靠 `.mdc` 触发；等 [forum #149566](https://forum.cursor.com/t/unknown-hook-type-sessionstart/149566) 修复后再升级。

---

## 建议的下一步

1. 在仓库根新建 `.cursor-plugin/plugin.json`，组件指向 `configs/cursor/*` 现有文件（避免文件重复）。
2. 用 `~/.cursor/plugins/local/context-mode -> <repo>` 软链做本地联调，验证 rules + hooks + MCP 都能在 Cursor 内识别。
3. 把 `configs/cursor/` 的 README 段落升级成「Plugin install (recommended) | Manual install (legacy)」双路径。
4. 在 `scripts/postinstall.mjs` 里加一句：检测到 Cursor 时输出 marketplace 链接，引导从插件商店安装。
5. 准备 Marketplace 提交材料（截图、demo gif、隐私说明），向 [cursor.com/marketplace/publish](https://cursor.com/marketplace/publish) 提交。

如果维护者要"asap"上架，最短路径是：先做步骤 1+2 起一个独立 plugin 仓库（如 `mksglu/context-mode-cursor-plugin`），把现有配置原样打包，第一版 plugin 只负责"一键装好 MCP + rules + hooks"，不动主仓代码 —— 1 天可以做完。

---

## 相关链接

- Cursor 官方文档：<https://cursor.com/docs/plugins>
- Plugin 模板：<https://github.com/cursor/plugin-template>
- Marketplace 提交：<https://cursor.com/marketplace/publish>
- 当前 Cursor 适配代码：[src/adapters/cursor/](src/adapters/cursor/)、[hooks/cursor/](hooks/cursor/)、[configs/cursor/](configs/cursor/)
- 已知 Cursor 上游 bug：[forum #149566](https://forum.cursor.com/t/unknown-hook-type-sessionstart/149566)、[forum #155689](https://forum.cursor.com/t/native-posttooluse-hooks-accept-and-log-additional-context-successfully-but-the-injected-context-is-not-surfaced-to-the-model/155689)、[forum #156157](https://forum.cursor.com/t/cursor-hooks-additional-context-not-injected-in-agent-context-in-posttooluse/156157)

# Map: pi-dcp-imitation

## Destination

一个生产级、模仿 `@pi-vault/pi-dcp` 架构的 pi 扩展，但压缩机制用 **pi 原生 `compact()`**（保留 pi 原生 compaction 特色：`keepRecentTokens` / split-turn / Goal·Constraints·Progress·Decisions·Next·Critical-Context summary），在上下文用量达百分比阈值时**主动触发**。地图完成 = 建之前要定的设计决策都解决，可以动手写代码。

## Notes

- 模仿目标：`@pi-vault/pi-dcp` v0.5.0（MIT, github.com/pi-vault/pi-dcp），源码在 `~/.pi/agent/npm/node_modules/@pi-vault/pi-dcp/src`，配置 schema 在其 `dcp.schema.json`。
- 已装且在跑的 pi-dcp 读 `~/.pi/agent/extensions/dcp.json`；新插件与它如何共存是决策 D5（ticket 07）。
- 平台 API 见 pi docs：`docs/compaction.md`（原生 compaction 触发/管线/summary 格式）、`docs/extensions.md`（`pi.on("context")`、`compact()`、`session_before_compact`/`session_compact`、`before_agent_start` 注入、`registerTool`/`registerCommand`）。例子 `examples/extensions/custom-compaction.ts`、`trigger-compact.ts`。
- 仓库约定（`pi-extensions/AGENTS.md`）：每个扩展是顶层独立目录 + 自己 `package.json`；**tabs + 双引号**；import 无扩展名 + `node:*` 内建；jiti 直接加载 TS，无 build。新扩展放 `pi-extensions/pi-dcp-imitation/`。
- 相关 skills：`grilling`、`domain-modeling`、`research`、`prototype`。决策 ticket 用 grilling/domain-modeling（HITL，一次一个）；研究 ticket 用 `/research` subagent（AFK，并行）。
- 用户痛点：pi-dcp 用 nudge（非硬触发），agent 常忽略 `<dcp-system-reminder>`（系统提示甚至说"不要输出它们"）→ 体验"被动/晚"。新插件要解决这个（见 D1 / ticket 03）。
- 本仓库 issue tracker = 本地 markdown（见 `docs/agents/issue-tracker.md`）：map 在 `.scratch/<effort>/map.md`，子 ticket 在 `issues/NN-<slug>.md`，`Type:` 记类型，`Status:` 记 claimed/resolved，`Blocked by: NN` 记依赖，frontier = 扫开/未阻塞/未认领。

## Decisions so far

- [R1: pi-dcp 机制](issues/01-research-pi-dcp-mechanism.md) — pi-dcp **nudge-only，从不硬触发**（`context` 钩子只 runPipeline→return；`session_compact` 是事后监听；`dcp:compress` 只发隐藏 follow-up；src 无 `compact()`/`summarize()`）。`nudgeForce` 是死配置（声明/校验但从不读）。详 `research/pi-dcp-mechanism.md`。
- [R2: pi 上下文 API](issues/02-research-pi-context-api.md) — `ctx.compact()` **走与 /compact 和自动压缩完全相同的原生管线**（keepRecentTokens/split-turn/Goal·Constraints·…·Critical-Context summary/CompactionEntry 全保留），仅 `reason=manual`+`willRetry=false` 不同 → 用 compact() 即满足"保留 pi 原来特色"。`pi.on("context")` 事件**只带 messages**，token/percent 要单独调 `ctx.getContextUsage()`。详 `research/pi-context-api.md`。

## Not yet specified

<!-- 向目的地的雾；随前沿推进毕业为 ticket -->

- **nudge 可靠性**（R1 已点亮）：pi-dcp 确认 nudge-only（从不硬触发），且 `nudgeForce` 是死配置。若 D1（ticket 03）选硬触发，此雾基本消除；若选 nudge/混合，需真实现 nudgeForce 且让 reminder 被响应（更强措辞 / 升级为 tool_result 而非 system metadata）。
- **跨压缩保留信息**（R2 已点亮）：`ctx.compact()` 已走完整原生管线、保留 Goal·Constraints·…·Critical-Context summary。剩下的问题=是否在 `session_before_compact` 额外注入结构以保留更多——取决于 D1。
- **subagent 结果处理**：pi-dcp 有 `src/subagents/subagent-results.ts`。新插件要不要复刻？取决于功能集 D2（ticket 04）。
- **测试/基准策略**：pi-dcp 有 vitest + `scripts/benchmark.ts`。新插件的最小测试策略，待功能集定后毕业为 ticket。

## Out of scope

<!-- 超出本 effort 目的地；不毕业 -->

- 不重写 pi-dcp 的 range-compress 工具 / summary-block 机制（用户选 pi 原生 `compact()`，见机制决策）。
- 不做 context-mode 的沙箱化工具输出（那是 context-mode 的职责，与本插件正交）。
- 不取代 pi 原生 compaction 管线本身（只改触发时机/策略，管线仍用 pi 的）。

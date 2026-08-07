Type: research
Status: resolved

## Question

摸清 pi 扩展用于上下文管理的 API 面与原生 compaction 管线，产出"建 DCP 模仿插件要用到的平台能力"清单。覆盖：

- `pi.on("context")` 事件形状：事件对象字段（tokens used / context window / percent / reserve？）、触发时机、能否拿到"剩余 token"。
- `compact()` API：签名、`onComplete`/`onError`、能否在扩展里主动调用触发 compaction、与 `/compact` 命令和自动阈值的关系。引 `examples/extensions/trigger-compact.ts`。
- `session_before_compact` 事件形状：`preparation.tokensBefore` / `reason`(manual/threshold/overflow) / `customInstructions` / `willRetry` / `signal`；能否 cancel、能否提供 custom summary。引 `examples/extensions/custom-compaction.ts`。
- `session_compact` / `session_before_tree` 事件形状。
- `before_agent_start`：如何注入系统提示（DCP 就靠它注入"context-constrained, use compress"路由）。
- `registerTool` / `registerCommand`：注册 compress 工具与 `/dcp:*` 命令的签名。
- pi 原生 compaction 管线（`docs/compaction.md`）：触发条件 `contextTokens > contextWindow - reserveTokens`、`reserveTokens`/`keepRecentTokens` 默认、切点规则、split turn、`CompactionEntry` 结构、summary 格式（Goal/Constraints/Progress/In-Progress/Blocked/Decisions/Next/Critical-Context）、分支摘要、custom summarization via extensions。
- **关键**：用 `compact()` 主动触发时，走的是不是同一条原生管线？`keepRecentTokens`/split-turn/summary 格式是否都保留？（这是用户"不丢失 pi 原来特色"的关键。）

主源：pi docs（`docs/compaction.md`、`docs/extensions.md`）+ `examples/extensions/{custom-compaction,trigger-compact}.ts` + 源码 types。

**产出**：把发现写到 `.scratch/pi-dcp-imitation/research/pi-context-api.md`（每条结论标注源:行），并在本 ticket 追加 `## Answer` 概要 + 研究文件指针，然后把 `Status:` 改为 `resolved`。

## Answer

`pi.on("context")` carries only `messages` (`types.d.ts:499`) — NO tokens/percent/reserve on the event. Read tokens via `ctx.getContextUsage()` → `{ tokens: number|null, contextWindow, percent: number|null }` (`types.d.ts:193,244`); derive `remaining = contextWindow - tokens` yourself (no remaining/reserve field; `tokens` is null right after compaction).

`compact(options?: CompactOptions)` is fire-and-forget (returns void): `{ customInstructions?, onComplete?(CompactionResult), onError?(Error) }` (`types.d.ts:200,246`). An extension CAN call it proactively from any hook — `trigger-compact.ts` does so from `turn_end` + `ctx.getContextUsage()` + `ctx.compact({onComplete,onError})` (`:12-23,27-40`). `ctx.compact()` IS the `/compact` path (`agent-session.js:1911`→`:1367`).

`session_before_compact` (`types.d.ts:441`): `{ preparation: CompactionPreparation, branchEntries, customInstructions?, reason: "manual"|"threshold"|"overflow", willRetry, signal }`. Result (`types.d.ts:812`) can `{cancel:true}` OR `{compaction: CompactionResult}` (custom summary REPLACES native — `custom-compaction.ts:117`). To preserve native features, do NOT return a custom compaction.

`session_compact` (`types.d.ts:453`): `{ compactionEntry, fromExtension, reason, willRetry }`. `session_before_tree`/`TreePreparation` (`:470,484`). `before_agent_start` (`:524,800`): return `{ systemPrompt: event.systemPrompt + "...routing..." }` (chained across extensions). `registerTool`/`registerCommand` at `:890,892`.

Native pipeline (`compaction.md`): trigger `contextTokens > contextWindow - reserveTokens` (reserveTokens default 16384, keepRecentTokens 20000); 5 steps cut-point→extract→summarize→append CompactionEntry→reload; split-turn two-summary merge; structured Goal/Constraints/Progress/In-Progress/Blocked/Decisions/Next/Critical-Context format; cumulative file tracking; branch summarization.

**CRITICAL — `ctx.compact()` runs the IDENTICAL native pipeline as `/compact` and auto-compaction.** `agent-session.js:1911` → `this.compact()` (`:1367`) calls the SAME `prepareCompaction()` (`:1379`, keepRecentTokens/split-turn) + native `compact()` (`:1423`, structured summary) + `appendCompaction` (`:1433`) + fires `session_before_compact` (`:1390`) and `session_compact` (`:1442`). Only `reason:"manual"` + `willRetry:false` differ (vs auto-threshold's `"threshold"`). Native keepRecentTokens/split-turn/summary-format/CompactionEntry/reload all preserved. ✓ The user's "preserve pi's native compaction features" goal is fully met by calling `ctx.compact()`.

Research: .scratch/pi-dcp-imitation/research/pi-context-api.md

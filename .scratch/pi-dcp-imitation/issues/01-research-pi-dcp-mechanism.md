Type: research
Status: resolved

## Question

深挖 `@pi-vault/pi-dcp` v0.5.0 的实现机制，产出一份可指导"完整生产模仿"的原理 writeup。覆盖：

- 入口 `src/index.ts` 注册的全部 pi 钩子（`before_agent_start`/`session_start`/`session_tree`/`session_compact`/`session_shutdown`/`message_end`/`tool_call`/`tool_execution_start`/`tool_execution_end`/`context`）各自做什么、何时触发。
- 上下文用量监控：`pi.on("context")` 事件拿到的字段、如何算百分比 vs `maxContextPercent`/`minContextPercent`、`maxContextLimit`/`minContextLimit`、何时注入 nudge。
- nudge 注入：`src/prompts/nudges.ts` 的 `CONTEXT_LIMIT_NUDGE`/`TURN_NUDGE`/`ITERATION_NUDGE` 各自触发条件与内容；系统提示 `DCP_SYSTEM_PROMPT` 怎么经 `before_agent_start` 注入。
- compress 管线：`src/compress/{handler,state,search,protected-content}.ts` + `src/pipeline.ts`——compress 工具调用如何处理、summary block 结构、protected content、orphan result 清理。
- 策略：`src/strategies/{deduplication,purge-errors,runner,protected-patterns}.ts`——去重/清错何时自动跑、`turnProtection`、`protectedFilePatterns`。
- 持久化/快照：`src/state/{state,persistence,types,tool-cache}.ts` + `parseDcpSnapshot`/`restoreDcpSnapshot`——跨 pi 原生 compaction 如何同步/恢复。
- 命令：`src/commands/`（sweep/manual/compress/stats/context/permission/recompress/decompress/help/register）。
- UI：`src/ui/notification.ts`；消息注入：`src/messages/{inject,strip,prune,priority,sync}.ts`（`dcp-message-id` 标签怎么注入/剥离）。
- 配置：`src/config-schema.ts` + `dcp.schema.json` 全字段语义。

主源：`~/.pi/agent/npm/node_modules/@pi-vault/pi-dcp/{src,README.md,dcp.schema.json,package.json}`；上游 `github.com/pi-vault/pi-dcp`。

**产出**：把发现写到 `.scratch/pi-dcp-imitation/research/pi-dcp-mechanism.md`（每条结论标注源文件:行），并在本 ticket 追加 `## Answer` 概要 + 研究文件指针，然后把 `Status:` 改为 `resolved`。

## Answer

pi-dcp (v0.5.0) 是一个纯 nudge 驱动的上下文卫生扩展，挂载在 pi 的 `context` 事件上。每次 context pass 跑一条纯函数 pipeline（`pipeline.ts:23`）：剥除模型臆造的 `<dcp-*>` 标签 → 重建稳定 `m####` 消息引用 → 自动去重工具输出 + 清除过期失败输入（`runner.ts`）→ 注入 `<dcp-message-id>` 锚点 + 频率节流的 `<dcp-system-reminder>` nudge → 应用压缩块（范围替换为摘要 + orphan toolResult 清理）。它注册一个模型自愿调用的 `compress` 工具（range/message 两种模式）和 10 个 `dcp:*` 命令。状态经 `pi.appendEntry("pi-dcp-state", snapshot)` 持久化到 pi 的 session branch，`session_compact` 后保留 key-based map、丢弃 index-based map，再由 `syncCompressionBlocks` 重新派生索引——这是跨原生 compaction 连续性的关键。

**关键观察结论：pi-dcp 从不硬触发 compaction——纯 nudge。** `pi.on("context")`（index.ts:395-463）只 `runPipeline` 后 `return {messages}`，不调用任何 compact API；`session_compact`（323）是**被动监听器**（pi 已压缩后才触发，清缓存）；nudge 注入的是文本字符串（"You MUST use the `compress` tool now"）；连 `dcp:compress` 命令也只是 `pi.sendMessage({customType:"dcp-compress-trigger", display:false}, {deliverAs:"followUp"})` 发一条隐藏 follow-up 请模型自己调工具。grep 全 src/ 无 `compact()`/`summarize()` 调用。要硬保证压缩，模仿者必须自己加程序化 compaction 调用。

Research: .scratch/pi-dcp-imitation/research/pi-dcp-mechanism.md

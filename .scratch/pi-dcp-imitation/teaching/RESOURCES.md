# pi-dcp 原理 Resources

## Knowledge

- [源码：`@pi-vault/pi-dcp` v0.5.0（已装）](file:///Users/crazytomatooo/.pi/agent/npm/node_modules/@pi-vault/pi-dcp/src/index.ts)
  主源。TypeScript，jiti 直接加载。入口 `src/index.ts`（注册全部 pi 钩子）；`src/{compress,strategies,state,messages,prompts,ui,commands,utils}/*.ts`。每条机制结论都该回到这里 file:line。
- [研究写法：pi-dcp 机制（R1）](file:///Volumes/work/Project/pi-extensions/.scratch/pi-dcp-imitation/research/pi-dcp-mechanism.md)
  39KB，逐节带 file:line 引用的机制综述（钩子/阈值/nudge/compress 管线/策略/持久化/命令/UI/配置）。第一手整理，备课主依据。
- [研究写法：pi 上下文 API + 原生 compaction（R2）](file:///Volumes/work/Project/pi-extensions/.scratch/pi-dcp-imitation/research/pi-context-api.md)
  23KB。关键结论：`ctx.compact()` 走与 `/compact` 相同的原生管线（保留 keepRecentTokens/split-turn/原生 summary）；`pi.on("context")` 事件只带 `messages`，token/percent 要调 `ctx.getContextUsage()`。
- [pi 文档：compaction.md](file:///Users/crazytomatooo/.nvm/versions/node/v22.23.1/lib/node_modules/@earendil-works/pi-coding-agent/docs/compaction.md)
  pi 原生 compaction 的触发条件、切点、split turn、`CompactionEntry`、summary 格式、custom summarization via extensions。理解"pi-dcp 骑在哪一层"必备。
- [pi 文档：extensions.md](file:///Users/crazytomatooo/.nvm/versions/node/v22.23.1/lib/node_modules/@earendil-works/pi-coding-agent/docs/extensions.md)
  扩展 API：`pi.on("context")`、`compact()`、`session_before_compact`/`session_compact`、`before_agent_start` 注入、`registerTool`/`registerCommand`。
- [pi 例子：custom-compaction.ts / trigger-compact.ts](file:///Users/crazytomatooo/.nvm/versions/node/v22.23.1/lib/node_modules/@earendil-works/pi-coding-agent/examples/extensions/)
  可运行的扩展样例——`session_before_compact` 自定义摘要、`compact()` 主动触发。
- [上游仓库：github.com/pi-vault/pi-dcp](https://github.com/pi-vault/pi-dcp)
  MIT，作者 Lanh Hoang。版本/issue/CHANGELOG 在此。

## Gaps

- pi 官方社区（discord/discussions）链接未确认——需要时再查 pi 文档/npm 主页，不臆造链接。

## Wisdom (Communities)

- [pi-dcp GitHub Issues](https://github.com/pi-vault/pi-dcp/issues)
  上游 issue tracker。用 for：机制疑问的权威答复、已知 bug、设计意图（commit/PR）。
- 用户尚未表态是否加入社区；如拒绝，在此记录，后续不再提议。

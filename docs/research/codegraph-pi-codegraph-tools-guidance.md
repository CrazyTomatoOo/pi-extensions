# pi-codegraph 工具与提示词设计提炼（wayfinder #4）

> 调查时间：2026-09-21。核对对象：`vndv/pi-codegraph` @ 0.1.10（main 源码）、`colbymchenry/codegraph` @ 1.6.0（main 源码）、`earendil-works/pi` 与 `can1357/oh-my-pi` 官方扩展文档。
> 传输层已在 #3 锁定（官方 stdio MCP、per-call 短会话），本文只决定其上的**工具面与 guidance 层**。所有结论来自源码一级证据。

## 结论速览

- pi-codegraph 注册 **8 个原生 pi 工具**（typebox schema、无 slash command），每个工具调用 = 一次短生命周期 MCP 会话；MCP client 与结果转换是**纯 Node、宿主无关**，可整体复用。
- 上游默认工具面 `DEFAULT_MCP_TOOLS = new Set(['explore'])`：`tools/list` 只列 explore，**其余 7 个仍可 `tools/call`**（未设 allowlist 时不做拦截）——扩展自带 schema、直呼工具名不受影响。
- 提示词注入有**两个来源**：pi-codegraph 在 `before_agent_start` 追加 5 条 guidance；上游另在 MCP initialize 响应的 `instructions` 字段下发 SERVER_INSTRUCTIONS（扩展未消费，属于重复的精简版）。
- **首发建议**：`codegraph_explore` + `codegraph_node` + `codegraph_search` + `codegraph_files`（可选 `codegraph_status`）；缓发 callers/callees/impact（其输出已内联在 explore 的 blast-radius 与 node 的 trail 中）。

## 1. 工具清单（extensions/codegraph.ts 全量）

全部经 `pi.registerTool({...})` 循环注册，`execute(_toolCallId, params, signal)` → `callCodeGraphTool` → `{ content: [{type:"text", text}], details: {} }`。每工具附 `promptSnippet`（= description）与 `promptGuidelines`（一条："…is available for structural code questions backed by the local CodeGraph index."）。

| 工具 | 参数（typebox，必填加粗） | 说明 |
|---|---|---|
| `codegraph_search` | **query**: String；kind?: function\|method\|class\|interface\|type\|variable\|route\|component；limit?: Number=10；projectPath?: String | 只返回位置 |
| `codegraph_callers` | **symbol**: String；limit?: Number=20；projectPath? | |
| `codegraph_callees` | **symbol**: String；limit?: Number=20；projectPath? | |
| `codegraph_impact` | **symbol**: String；depth?: Number=2；projectPath? | |
| `codegraph_explore` | **query**: String；maxFiles?: Number=12；projectPath? | 主工具 |
| `codegraph_node` | **symbol**: String；includeCode?: Boolean=false；projectPath? | |
| `codegraph_status` | projectPath? | |
| `codegraph_files` | path?/pattern?/format?: tree\|flat\|grouped=tree/includeMetadata?: Boolean=true/maxDepth?: Number/projectPath? | path 先归一化（§3） |

共享 `OptionalProjectPath` 描述："Path to a different project with .codegraph/ initialized. Defaults to current project."。历史：v0.1.8 移除 `codegraph_context`/`codegraph_trace`（上游 v0.9.9+ 已弃）。

## 2. MCP client（宿主无关，可整体复用）

- **spawn**：非 Windows `spawn("codegraph", ["serve","--mcp","--path",cwd], {cwd, env: process.env, stdio:["pipe","pipe","pipe"]})`；Windows 经 `powershell.exe -NoProfile -NonInteractive -ExecutionPolicy Bypass -Command <script> <cwd>` + `windowsHide`，脚本用 `Get-Command codegraph -CommandType Application | Select-Object -First 1` 解析真实 exe（规避 npm/Scoop/.cmd shim 问题）。
- **路径解析** `resolveProjectCwd`：先 `normalizeWindowsPath`（win32 下把 WSL `/mnt/c/…`、Git Bash `/c/…` 归一为 `C:\…`），再绝对性 + `stat` 存在 + 目录三项校验，**spawn 前抛出**三条不同错误。
- **会话** `withCodeGraphMcp` → `runJsonRpcSession`：spawn → 注册 abort 监听 → stdout 行缓冲、stderr 累积、`error`→reject pending、`exit`→以清洗后 stderr（或 `exited with code N`）reject → `initialize`（protocolVersion `2024-11-05`、rootUri、workspaceFolders、clientInfo `{name:"pi-codegraph",version:"0.1.0"}`）+ `initialized` 通知 → **恰好一次** `tools/call` → finally 清理（reject pending "closed before responding." + kill）。
- **帧格式**：换行分隔 JSON-RPC 2.0；pending map 按自增 id；未知 id 丢弃；`msg.error` → reject。
- **超时** `SessionTimeoutMs = 20_000`：`Promise.race`；超时 kill + 报错提示 `"codegraph unlock"`；race 落败方 `.catch(()=>{})`；timer/listener 在 finally 清（0.1.9 修的 timer 泄漏）。
- 为测试导出 `SessionTimeoutMs`/`MaxDiagnosticLength`/`codegraphToolNames`。

## 3. 结果转换

`callCodeGraphTool`：(1) `prepareToolArguments` — 仅 `codegraph_files`：`normalizeFilesPath`（`~` 展开、项目内绝对路径→repo-relative POSIX 前缀、根路径→删除过滤）(2) `tools/call` (3) content 过滤 `type==="text"` 后 `join("\n")` (4) `isError` → throw (5) 空结果 `JSON.stringify(result)` 兜底；files 空匹配时追加 path 语义 hint。无结构化 JSON 透传。

## 4. 提示词 / guidance 注入

- **机制**：`pi.on("before_agent_start")` 返回 `{ systemPrompt: (event.systemPrompt ?? "") + "\n\n" + guidance }`。**注意**：pi 文档说明返回 `systemPrompt` 是**整轮替换**语义；更稳妥的注入是 `systemPromptOptions`（sections / promptGuidelines / selectedTools，diff 补丁式）。OMP 侧同事件存在，链式覆盖语义，handler 需容忍重入/重试。
- **内容**（5 条）：codegraph_* 工具可用；架构/流程/符号定位/影响面问题先于 grep/read 用 CodeGraph；工具选择阶梯（broad→explore、名字→search、结构→files、已知符号→node、影响/调用流→callers）；search 未命中先试 explore/files/node 再退 grep/read（符号搜索会漏字面量与生成名）；仅在 CodeGraph 不足或用户要求字面匹配时用 grep/read。
- **上游双份 guidance**：`src/mcp/server-instructions.ts` 的 SERVER_INSTRUCTIONS / SERVER_INSTRUCTIONS_NO_ROOT_INDEX 经 initialize 响应 `instructions` 字段下发；pi-codegraph **未消费**，在 prompt 层重复了精简版。codegraph-ext 应**择一**，避免双重注入（推荐消费 MCP `instructions` 或维持 prompt 注入但不再重复上游内容）。
- `promptGuidelines` 每条必须点名工具；`promptSnippet` 决定是否进 "Available tools" 一行列表。

## 4b. OMP 侧差异（对 guidance/工具面的影响）

- OMP `registerTool` 与 pi **参数顺序一致**（`(toolCallId, params, signal, onUpdate, ctx)`）；但 **custom-tool module**（`CustomToolFactory`）第三参是 `onUpdate` 而非 `signal`——若走该形态需换序。
- OMP 的 schema 经 `pi.zod`/`pi.arktype`/`pi.typebox`（TypeBox 兼容 shim）统一校验管线。
- OMP 强制托管定时器（`ctx.setTimeout`/`ctx.setInterval`，裸 timer 抛错会升级为进程级 uncaughtException）；pi-codegraph 用裸 setTimeout（回调不抛，安全，但移植时应改托管）。
- OMP 自带一等 MCP 客户端（`mcp__<server>_<tool>`、`.omp/mcp.json`）：免客户端代码的替代路线，但常驻连接 + mcp__ 命名 + 默认只列 explore，与 #3 已定的 per-call 轻客户端不同轨；作为**文档化后备**而非主路线。

## 5. 错误降级（策略整体可复用）

| 场景 | 行为 |
|---|---|
| projectPath 非绝对/不存在/非目录 | spawn 前抛可操作错误 |
| spawn 失败 | `error` 事件 → reject pending |
| 20s 超时 | kill + `codegraph unlock` 提示 |
| 子进程先退 | 清洗后 stderr 为诊断，否则 exit code |
| abort | reject pending + kill；timer/listener 清理 |
| stderr 清洗 | ANSI 剥离 + TOKEN/SECRET/PASSWORD/API_KEY/APIKEY/AUTH= 与 Bearer、`--token/--secret/--password/--api-key/--apikey/--otp` 脱敏 + 截断 1000 字符 |
| server `isError: true` | 直接 throw |
| 未索引 | **透传上游 success 形态引导**（上游刻意不用 isError——避免 agent 过早弃用整个工具集） |
| files 空匹配 | 追加 path 语义 hint |
| Windows | PowerShell 命令发现 + WSL/GitBash 归一化 + windowsHide |

补充（源自 #3 已定事实）：扩展 kill 的只是 proxy，detached daemon 常驻共享（writer lock 在 daemon 手里）；`CODEGRAPH_NO_DAEMON=1` 可强制 direct 模式；`codegraph unlock [path]` 清陈旧 writer lock。

## 6. 测试

`__tests__/codegraph.test.ts`，vitest，13 例，`vi.mock("node:child_process")` 的 mock-spawn 挂具（EventEmitter + PassThrough，stdin listener 应答 initialize/tools/call）。覆盖：工具名面（恰好 8 个、排除 context/trace）、非 Windows spawn 参数、Windows PowerShell 脚本内容、路径校验、WSL/GitBash 归一化、脱敏、files path 归一化、空匹配 hint、timer 成功清理、abort、超时+kill、files path 上线前归一化。CI：`tsc --noEmit` + `vitest run` + `codegraph --version` 兼容检查 + README 版本核对 + `npm pack --dry-run`。**无真二进制集成测试**。

## 7. 上游工具面核对（src/mcp/tools.ts）

8 个上游工具（JSON-Schema inputSchema + `READ_ONLY_ANNOTATIONS`）：search/callers/callees/impact/node/explore/status/files，全部可选 `projectPath`。**默认面**：`DEFAULT_MCP_TOOLS = new Set(['explore'])` — `tools/list` 只返回 explore；其余 7 个保持已定义且**可调用**（未设 `CODEGRAPH_MCP_TOOLS` 时不拦截；设了才做 allowlist 拦截，空值=全部）。横切限制：输入 ≤10,000 字符、路径 ≤4,096、输出截断 ≤15,000（explore 按文件数自适应预算 4→8 文件/13K→24K 字符）。注意：pi-codegraph 的 callers/callees/impact/node schema 落后于上游（缺 `file` 消歧参数；node 已有 file-mode 读取：`file`/`offset`/`limit`/`symbolsOnly`/`line`）——扩展 pin `^1.0.1`、上游 main 已 1.6.0。codegraph-ext 复刻 schema 时应对齐当前上游。

## 7b. 上游自身对工具面的实测建议（tools.ts 注释证据）

上游把默认面收敛到 explore-only 的注释依据：callers/callees/impact 的产出已**内联在 explore 结果**（blast-radius 段 / 关系图）与 node 的 trail 中；工具面越大，模型误选率越高（上游有 A/B harness 痕迹，`CODEGRAPH_MCP_TOOLS` 即其实验开关）。

## 7c. 上游 guidance 文本（SERVER_INSTRUCTIONS，供参考）

explore 优先、一次调用拿足上下文（verbatim 行号源码分组 + 调用路径 + blast radius）、预算随文件数自适应、read 兜底。NO_ROOT_INDEX 变体：无 `.codegraph/` 时引导用户 `codegraph init`，并明确「继续用普通工具，索引由用户决定」。

## 8. 可复用 vs 需为 OMP 重设计

**原样复用（纯 Node，宿主无关）**：JSON-RPC client 全套函数族；`resolveProjectCwd`/`normalizeWindowsPath`/`spawnCodeGraphServer`（含 PowerShell 脚本）；`normalizeFilesPath`/`annotateFilesResult`/`sanitizeDiagnostic`；结果转换链；20s 超时 + unlock 提示文案；错误降级策略整体；13 例 mock-spawn vitest 挂具 + `codegraph --version` 兼容检查。

**改适配**：工具 schema——typebox 结构保留但同步到当前上游（补 `file` 消歧与 node file-mode 参数）；guidance 文本——内容可复用，注入机制改走结构化注入（pi `systemPromptOptions`，OMP 等价物），并容忍重入；裸 setTimeout → OMP 托管 `ctx.setTimeout`；execute 签名按目标面（pi registerTool vs OMP custom-tool module）确认参数序。

**必须重做（仅宿主绑定层）**：包身份/manifest（import 来源、peer deps、发现机制）；schema 构建器（typebox import → `pi.typebox` shim 或 zod/arktype）；guidance 注入目标与时机（构建期验证两宿主 `before_agent_start` 返回契约）；分发渠道。

与 #3 的共享核心表述一致：**MCP client + 工具适配层 100% 宿主无关；宿主差异收敛到 registration、结果类型与 guidance 适配**。

## 9. 首发工具集建议（供 #5 拍板）

- **首发 4 个**：`codegraph_explore`（主工具）、`codegraph_node`（已知符号 + trail + file-mode 读取）、`codegraph_search`（名字→位置，回退阶梯第一级）、`codegraph_files`（索引文件树，附既有归一化与空匹配 hint）。
- **可选**：`codegraph_status`（1 参数健康探测，兼作未初始化/陈旧索引的降级探针）。
- **缓发**：callers/callees/impact——输出已内联于 explore/node，少工具=少误选（上游实测注释依据）；后续增量注册即可，无需改 client。
- **schema 对齐上游当前版**（含 `file`/file-mode 参数），`projectPath` 全工具可选，默认值保留（limit 10/20、depth 2、maxFiles 12）。
- **guidance**：复用 5 条文本（explore 优先 + 选择阶梯 + grep/read 兜底），经最 低侵入宿主机制注入；**不双份**（不与上游 SERVER_INSTRUCTIONS 重复内容）。
- 最终工具名与数量、slash command 取舍由「决定统一插件的产品契约与工具 UX」拍板，本文给出证据与推荐。

## 引用

- pi-codegraph：`extensions/codegraph.ts`（505 行全文）、`__tests__/codegraph.test.ts`、`package.json`、`CHANGELOG.md`。
- 上游：`src/mcp/tools.ts`（DEFAULT_MCP_TOOLS:1455、allowlist:1609-1621、getTools:1629-1636）、`src/mcp/server-instructions.ts`、`src/bin/codegraph.ts`（unlock）。
- 宿主文档：`earendil-works/pi` `packages/coding-agent/docs/extensions.md`；`can1357/oh-my-pi` `docs/extensions.md`、`docs/custom-tools.md`、`docs/mcp-server-tool-authoring.md`。

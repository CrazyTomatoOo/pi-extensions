# pi-dcp 原理 Glossary

pi-dcp（`@pi-vault/pi-dcp`）这个 pi 上下文管理扩展涉及的术语。术语只在用户**证明理解后**才加入（见 GLOSSARY-FORMAT 规则：glossary 是已压缩的理解记录，不是预习字典）。

## Terms

**自动剪枝策略 (automatic pruning strategies)**:
pi-dcp 的第一层：`dedup`（去重重复的工具输出）+ `purgeErrors`（N 轮后清掉失败的旧工具输入）。硬、静默、不靠 agent，直接改消息——是唯一自动硬干活的层（但只删噪声，不做摘要）。
_Avoid_: 自动压缩

**nudge**:
pi-dcp 的第二层：上下文 % 越过阈值时注入 `<dcp-system-reminder>`，提醒 agent 去压缩。软、靠 agent 响应、可被忽略。
_Avoid_: 提醒、提示

**compress 工具**:
pi-dcp 的第三层：agent 主动调用，对一段消息做 range→summary block。产出 pi-dcp 自己的 block 格式（≠ 原生 compaction 格式）。
_Avoid_: 压缩工具

**硬触发 (hard-trigger)**:
到阈值直接调 `compact()`，不依赖 agent。pi-dcp **从不**硬触发（nudge-only）——这正是其"被动/晚"痛点的根因，也是要不要自己加硬触发的决策焦点。
_Avoid_: 自动压缩、强制压缩

**原生 compaction (native compaction)**:
pi 自带的压缩引擎：`contextTokens > contextWindow − reserveTokens` 时把旧消息 span → 结构化摘要（Goal/Constraints/Progress/…/Critical-Context），配 `CompactionEntry` 与 split-turn。自动、可靠、晚。pi-dcp 骑在它之上，不替换；装了 pi-dcp 它照常在底下跑（晚触发兜底）。
_Avoid_: pi 压缩、内置压缩

**dead config**:
声明 + 校验但代码从不读的配置项。pi-dcp 的 `nudgeForce`（`soft`|`strong`）即是——设成 `strong` 也不会让 nudge 更强。
_Avoid_: 无效配置、废弃配置

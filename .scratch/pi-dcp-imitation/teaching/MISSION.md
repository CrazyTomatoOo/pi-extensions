# Mission: pi-dcp 插件的工作方式与原理

## Why

用户想自己做一个 DCP 风格的 pi 上下文管理插件（模仿 `@pi-vault/pi-dcp`），但卡在"不知道这个插件的原理"。根本痛点：pi 现在的上下文裁剪对长任务太被动（1M 窗口要到 ~984k 才触发），危险。学会 pi-dcp 的机制后，"用 pi-dcp / 调参 / 加小扩展 / 完整重写"这个分叉就能自己拍板。

## Success looks like

- 能读懂 `@pi-vault/pi-dcp` 的源码，追踪一个 `context` 事件从触发到 nudge/剪枝的整条路径。
- 能说清 pi-dcp 的三层机制（自动剪枝策略 / nudge / compress 工具）各自靠不靠 agent、硬还是软。
- 能解释为什么 pi-dcp 体验"被动"（nudge-only、从不硬触发），并据此判断要不要自己加硬触发。
- 能独立决定：直接用 pi-dcp、调参、加一个硬触发小扩展、还是完整重写——并说清取舍。

## Constraints

- 主源是已装好的 pi-dcp 源码（`~/.pi/agent/npm/node_modules/@pi-vault/pi-dcp/src`）+ 已产出的 R1 研究写法（`.scratch/pi-dcp-imitation/research/pi-dcp-mechanism.md`，带 file:line 引用）。不靠 parametric 猜测。
- 用户用中文沟通；偏好 tabs + 双引号（仓库约定）。
- 这是 pi-extensions 仓库的一个子 effort（`.scratch/pi-dcp-imitation/`，wayfinder 地图已建）。学习服务于那个 effort 的目的地决策。

## Out of scope

- 不在本工作区学 context-mode（那是另一个扩展，沙箱预防 + 跨压缩记忆，与 pi-dcp 正交）。
- 不学 pi 原生 compaction 的全部实现细节（只学"与 pi-dcp 交互的那一面"，如 `compact()` 是否走原生管线——已在 R2 查证）。
- 暂不写插件代码（那是 wayfinder 地图清空之后的事）。

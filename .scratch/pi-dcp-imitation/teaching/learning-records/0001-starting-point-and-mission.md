# 起点与 mission：想模仿 pi-dcp、但还不知其原理

用户 disclosed prior knowledge（低）：想自己做一个 DCP 风格的 pi 上下文管理插件、模仿 `@pi-vault/pi-dcp`，但明确"不知道这个插件的原理"——所以 zone of proximal development 从**整体心智模型**起步，而非任何单点机制。根本痛点（已由研究坐实）：pi-dcp 是 nudge-only、从不硬触发，agent 常忽略 nudge → 体验"被动/晚"。

**Implications**：第一课给三层心智模型 + "nudge-only、从不硬触发"这个非显然洞察；后续逐层下钻（阈值算法 / compress 管线 / 持久化）。学会后用户应能独立拍板"用 / 调参 / 加硬触发小扩展 / 完整重写"四选一（见 [[../../map.md]] 的 D1–D5）。

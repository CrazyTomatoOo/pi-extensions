Type: grilling

Blocked by: 01, 02

## Question

新插件的触发模型选哪个？

- **硬触发**：到百分比阈值就直接调 `compact()` 主动压缩，不依赖 agent（解决用户"被动/晚"痛点）。
- **nudge**：注入 reminder 让 agent 自己压（忠实模仿 pi-dcp，但有被忽略风险）。
- **混合**：先 nudge，超阈值或 agent 忽略则硬触发兜底。

考虑：用户痛点 = pi-dcp 的 nudge 被 agent 忽略；"完整生产模仿"与"解决可靠性"的取舍；`compact()` 能否在 `pi.on("context")` 里可靠调用（R2 查证）。

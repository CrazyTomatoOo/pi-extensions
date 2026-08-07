Type: grilling

Blocked by: 01

## Question

"完整生产模仿"要复刻 pi-dcp 的哪些功能模块？逐个 yes/no（参考 R1 的机制清单）：

①上下文 % 监控 + 阈值 ②nudge 注入（系统提示 + reminder）③去重策略 dedup ④清错策略 purgeErrors ⑤持久化/快照（跨原生 compaction 同步）⑥命令（sweep/manual/compress/stats/context/...）⑦UI 通知 ⑧message-id 注入 ⑨`turnProtection`/`protectedFilePatterns` ⑩subagent 结果处理。

定出**最小可用集**（MVP）与**增量集**。注意：用 pi 原生 `compact()` 后，②nudge 和 ⑤持久化的语义会变（见 D1、D3）。

Type: grilling

Blocked by: 02

## Question

message-id 注入（pi-dcp 的 `dcp-message-id` 标签）要不要？

pi-dcp 用它支持 range-compress；但本插件用 pi 原生 `compact()`（基于切点，非 message-id range）。是否仍需注入 id（供 nudge 引用范围 / agent 指定压缩段 / 持久化快照定位）？还是原生 compaction 自带的 `CompactionEntry`/entry id 已够用？取决于 R2 对原生管线 entry id 的查证。

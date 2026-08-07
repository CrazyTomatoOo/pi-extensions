# 用户理解了 pi-dcp 与 pi 原生压缩的区别

经第 1 课（三层心智模型）后，用户主动提出对比问题（pi-dcp vs pi 原生 compaction），在结构化对比表 + 三关键点（① pi-dcp 只让压缩更早/更细 + 自动剪枝，但不更"可靠"；② 格式不同故仿制品选 native `compact()`；③ nudge/compress 是加法不是替代，pi 原生在底下照跑）之后表示已理解。

**Evidence**：用户主动发起对比（非被动听讲），且对比连贯建立在第 1 课三层模型之上，复述确认"我了解了"。

**Implications**：6 个术语（自动剪枝策略 / nudge / compress 工具 / 硬触发 / 原生 compaction / dead config）已验证理解 → 入 GLOSSARY。用户已具备判断"用 pi-dcp / 调参 / 加硬触发小扩展 / 完整重写"四选一的概念基础。下一课可下钻单点机制（阈值算法 `isContextOverLimits` / compress 管线 7 步 / 跨压缩快照持久化）。悬置的 3-way 目的地重画（见 [[../../map.md]] D1–D5）仍未决，待用户学完再回。

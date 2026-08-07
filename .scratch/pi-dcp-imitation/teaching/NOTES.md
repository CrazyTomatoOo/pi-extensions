# NOTES.md — 教学笔记

## 工作区位置

本应按 teach skill "current directory as teaching workspace" 放在仓库根，但 pi-extensions 根是扩展集合仓库（有自己的 AGENTS.md/目录结构）。为避免污染，把教学工作区缩在 `.scratch/pi-dcp-imitation/teaching/`，与 wayfinder effort 同根。lesson→研究写法的相对路径是 `../../research/*.md`。

## 用户偏好 / 沟通

- 中文沟通；技术术语保留英文原文（nudge / compress / context hook 等）。
- 仓库代码约定：tabs + 双引号；import 无扩展名 + `node:*` 内建；jiti 直接加载 TS，无 build（若日后写插件代码要遵守）。
- 偏好先搞懂原理再决定动不动手——所以 teach 优先于 build 决策。

## 教学节奏

- 第一课给"三层心智模型"大图 + 关键洞察（nudge-only、从不硬触发）——这是 zone of proximal development 的起点。
- 后续候选课：① context 事件 + 三级阈值算法 `isContextOverLimits` ② compress 管线 7 步 ③ 跨压缩快照持久化 ④ 策略（dedup/purge）何时跑 ⑤ 命令面。
- 术语只在用户**证明理解后**才入 GLOSSARY.md（见 GLOSSARY-FORMAT 规则）。

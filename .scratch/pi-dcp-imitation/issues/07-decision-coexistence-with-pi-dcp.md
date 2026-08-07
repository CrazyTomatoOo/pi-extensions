Type: grilling

Blocked by: 01

## Question

新插件与已装的 `@pi-vault/pi-dcp` 如何共存？两者都挂 `pi.on("context")`/`before_agent_start`、都注入系统提示/reminder、可能都注册 compress 工具——会冲突（双重注入 / 双重 compress）。

选项：①禁用 pi-dcp（移走/重命名 `dcp.json`），新插件独占 ②把新插件做成 pi-dcp 的 fork/替换（复用其配置文件位置）③共存但 hook 分工不重叠（难）。考虑用户日常最终要用哪个、以及"模仿来学" vs "替代在用"的定位。

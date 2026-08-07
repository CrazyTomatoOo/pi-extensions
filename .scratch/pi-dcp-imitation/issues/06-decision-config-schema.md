Type: grilling

Blocked by: 01

## Question

配置 schema：镜像 pi-dcp 的 `dcp.json`（`maxContextPercent`/`minContextPercent`/`maxContextLimit`/`minContextLimit`/`nudgeForce`/`turnProtection`/`protectedFilePatterns`/`strategies`/`manualMode`/...）还是为本插件设计新 schema？

"完整生产模仿"倾向镜像，但用原生 `compact()` 后某些字段（`compress.mode`/`permission`/`showCompression`）语义变化——如何裁剪/重映射？配置文件放哪（`~/.pi/agent/extensions/dcp-imitation.json`？项目级 `.pi/`？）。

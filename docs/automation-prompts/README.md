# 自动化 Prompt 索引

> 本目录维护基于 native-subtitle-quote-image 技能的**每日定时自动化 prompt**，供 Agent 在定时任务中直接加载复用。

| 文件 | 用途 | 产出 |
|---|---|---|
| [ted-daily-prompt.md](ted-daily-prompt.md) | 每日 TED 频道优质演讲 → 字幕金句长图 + 小红书文案 | 3 组金句长图 + 1 篇文案 |
| [celebrity-speeches-podcast-daily-prompt.md](celebrity-speeches-podcast-daily-prompt.md) | 每日 AI 领域名人演讲/播客检索筛选 → 字幕金句长图 + 小红书文案 | 3 组金句长图 + 1 篇文案（纯音频降级为文字要点） |

## 使用方式

1. Agent 执行定时任务时，读取对应 prompt 文件全文作为任务指令执行；
2. prompt 首行标注了 SKILL.md 的本地/仓库位置，首次执行先确认技能已就位（未部署则先 clone 本仓库）；
3. 查重依赖任务工作区的 `memory.md`（历史推送记录），需保证自动化工作区持久化；
4. 通用流程细节（凭据、字幕、取帧、质检）参见 [../selection-to-post-workflow.md](../selection-to-post-workflow.md) 与 [../youtube-source-cookbook.md](../youtube-source-cookbook.md)。

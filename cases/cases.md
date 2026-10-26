# 案例库

> 星数说明：均为 2026 年 10 月二手汇总快照，不同来源不一致时给区间；发布前请复核。许可证与“最近状态”同样以报告时的来源为准，Aider、Gemini CLI、Roo Code 等有状态变化的项目已在备注里标出。

## 被点名的 harness

| 项目 | 仓库 / 官方入口 | 许可证 / 语言 | harness 设计一句话 | 注意 |
|---|---|---|---|---|
| Claude Code | Anthropic 官方产品与文档 | 闭源 / Node.js CLI | 约 40 工具 + hooks（PreToolUse 可阻断）+ skills + subagents + 插件；Agent Teams / Projects / Dynamic Workflows | 本地默认无容器隔离；bypassPermissions 只放隔离容器 |
| Claude Work | 无独立官方产品 | — | 对应 Projects / Agent Teams / Dynamic Workflows + 社区多智能体脚手架；另有同名多账号别名 | 别把账号别名当成多智能体产品 |
| Claude Design | claude.ai/design（Anthropic Labs） | 闭源托管 | 聊天 + 画布，元素级评论/直接改/滑杆调参，导出 PDF / PPTX / HTML / Canva，handoff 给 Claude Code | 不是 Figma 替代品，是互补品 |
| baoyu-design | github.com/JimLiu/baoyu-design | — | 把 Claude Design 本地化成 Agent Skill：Markdown + JSX/JS 脚手架，无构建步骤 | 最适配 Opus |
| Open Design | 开源桌面应用 | Apache-2.0 | 本地优先，21 个内置设计 agent + 129 个起点，复用用户自己的编码 agent 当引擎 | 产物存本地 |
| SuperDesign | github.com/superdesigndev/superdesign | MIT 时期 | 提示词 → 多个并行 UI mockup / 组件 / 线框，无限画布，本地 git 保存 | 原 IDE 扩展已不再积极维护，产品已转 web |
| Hallmark | Nutlope/hallmark | MIT | 设计 skill 当验证器：48 个主题抽 1 个、16 条不可豁免规则、浏览器 linter、fresh 上下文评审子 agent 定下一步 | 评审子 agent 首词决定 recapture / rebuild / fix / ship |
| Codex | openai/codex | Apache-2.0 / Rust | OS 沙箱是架构：Seatbelt / bubblewrap、默认断网、模型按 schema 申请提权；沙箱与审批分开 | 约 119–129k 星快照；模型闭源且会按 OpenAI 节奏退役 |
| OpenCode | anomalyco/opencode | MIT | 客户端/服务端 + SQLite 会话，Build（全权限）/ Plan（默认只读）双 agent，75+ 供应商 | 约 202–222k 星快照；质量取决于你选的模型 |
| DeepSeek Harness (dsh) | github.com/deepseek-ai/deepseek-harness | MIT / Node.js | Cordis 插件框架：模型适配、工具注册、会话日志、循环本身都可替换；53 个内置工具 | 开发者预览、无安全审计、承诺破坏性变更；必须隔离跑 |
| CodeWhale | DeepSeek 生态独立项目 | Rust | DeepSeek 系终端 harness | 约 41k 星快照 |
| Reasonix | DeepSeek 生态独立项目 | MIT | DeepSeek 系 harness | 约 35k 星快照 |
| Grok Build | github.com/xai-org/grok-build | Apache-2.0 | 贴近 Claude Code CLI 表面，Plan 模式、Skills + MCP、ACP | 约 26–27k 星快照；早期默认上传数据争议 |
| Grok Bot | xAI 托管服务 | 闭源托管 | 云虚拟机主 agent + 子 agent + 一键连接器，关机也继续跑 | 订阅制；内部细节多为二手报道 |
| Hermes Agent | Nous Research | MIT / Python | 同步对话循环 + 血缘式压缩 + 工具注册/暴露分离 + 自写 skill；消息网关覆盖 20+ 平台 | 约 210–255k 星快照；自写 skill 需人工复核 |
| Muse Code | Meta 官方 harness | 闭源模型 + 官方 harness | Muse Spark 专用终端 agent，agent teams、无头/CI 运行、SDK | 模型闭源；生态比 Claude Code / Codex 新 |
| Muse Spark Code | github.com/RandyNorthrup/muse-spark-code | MIT | 社区 ACP agent：可见子 agent 地图、无头浏览器自检网页改动、非阻塞问题卡片 | 明确声明与 Meta 无隶属关系 |
| Muse Gadgets SDK | facebookincubator/muse-gadget-sdk | Apache-2.0 | ESP32 + Linux / 树莓派硬件 skill，目录级 AGENTS.md | 硬件方向 |

## 其他优秀开源 harness

| 项目 | 仓库 | 许可证 / 语言 | 设计一句话 | 注意 |
|---|---|---|---|---|
| Pi | earendil-works/pi（原 badlogic/pi） | MIT / TypeScript | 极简工具箱：4 个核心工具、<1k token 系统提示词，其余全是按需扩展 | 约 92–115k 星；无沙箱、无审批弹窗是设计选择，不是疏忽 |
| Oh My Pi (omp) | Pi 的 fork | Rust 引擎重写 | Pi 极简核心 + 极繁扩展：60+ 供应商、31 工具、LSP、DAP、持久执行内核 | 与 Pi 的“少即是多”论点对照着看 |
| OpenClaw（原 Clawdbot） | openclaw/openclaw | MIT / TypeScript | 常驻 Gateway 守护进程 + 30+ 消息通道 + cron + 数千个可安装 skill（ClawHub） | 约 368–392k 星；工具沙箱默认关闭，先检查默认配置 |
| Gemini CLI | google-gemini/gemini-cli | Apache-2.0 | 1M 上下文、约 20 工具、循环检测注入纠正指导、配额耗尽降级更快模型、shadow-git /rewind | 约 107k 星；消费者版 2026-06-18 停服，现为付费 API / 企业版 |
| OpenHands（原 OpenDevin） | OpenHands/OpenHands | MIT / Python | 自托管自主工程师标杆：web 应用默认 Docker 沙箱、浏览器 + 终端 + 编辑器 + 任务追踪 | 约 85–91k 星；配置最重 |
| Cline | cline/cline | Apache-2.0 | IDE 内 agent：默认每次动作前问你，7 个核心工具 + ripgrep 搜索，另有 CLI / SDK / 无头 CI | 约 68–70k 星 |
| Goose | aaif-goose/goose（Block，Linux 基金会） | Apache-2.0 / Rust | 通用桌面 + CLI + 可嵌入 API，能力模型是 70+ MCP 扩展而非固定工具集 | 约 55k 星 |
| Aider | Aider-AI/aider | Apache-2.0 / Python | Git 优先结对编程：无通用工具循环，靠编辑格式 + 仓库地图，每改自动提交 | 约 48–50k 星；**2026-05-22 后无提交，引用前请复核** |
| Crush | charmbracelet/crush | FSL-1.1-MIT（到期转 MIT） | 精致 Go TUI、模型切换、MCP + LSP | 源码可见，非 OSI 开源直至转换；约 28k 星 |
| Qwen Code | QwenLM/qwen-code | Apache-2.0 | 阿里 Qwen 家族的模型实验室 harness 模式 | 约 27–29k 星 |
| Deep Agents | langchain-ai/deepagents | MIT / Python + JS/TS | 库式 harness：`create_deep_agent` 返回 LangGraph 图，含规划工具、文件系统、子 agent、总结中间件 | 约 15k 星；harness  alone 让 Terminal-Bench 从 42.65% → 66.5% |
| SWE-agent | SWE-agent/SWE-agent | MIT / Python | 基准驱动研究 harness：定义 agent–计算机接口（ACI），为 SWE-bench 可审计评分而设计 | 约 17k 星（较早汇总） |
| browser-use | browser-use/browser-use | MIT / Python | 浏览器/电脑使用 harness：Playwright 上跑截图 + 无障碍树循环，密钥与提示词分离 | 约 116–118k 星；也可作 MCP 服务器进其他 harness |
| Stagehand | browserbase/stagehand | MIT / TypeScript | 混合式浏览器 harness：确定性 Playwright 代码 + AI 的 act / extract / observe / agent；无障碍树剪枝号称省约 30 倍上下文 | 约 21–24k 星 |
| GPT-Researcher | assafelovic/gpt-researcher | Apache-2.0 / Python | 研究 harness：规划 → 并行爬取 → 上下文过滤 → 带引用出版物；Deep Research 模式按树递归 | 约 29–30k 星 |
| DeerFlow | bytedance/deer-flow | 见仓库 | 研究优先 harness 转通用 agent；原版无需搜索 API key（DuckDuckGo + 无 key 阅读器） | 约 82k 星；引用前复核许可证/版本 |
| Agent Zero | frdel/agent-zero | MIT | “自带电脑的 agent”：Docker 上 web UI、持久 agent、工具/调度，本地模型友好 | 约 19k 星 |
| TrueForge | TrueFoundry | MIT | 可自托管的 Managed Agents 风格替代：完整循环、会话、调度、沙箱、审批、HTTP API、任意供应商 | 约 6.1k 星；“便宜 30–50%”是厂商说法 |
| Omnigent | Omnigent | Apache-2.0 | 元 harness：一个治理层（消费上限、审批门、策略）管 Claude Code / Codex 等，沙箱可插拔 | 约 10.6k 星；自标 alpha |

## 横向对比（18 行 × 7 维，精简版）

完整 18 行对比见英文报告 §5。本仓库只保留选型时最关键的维度：用途、沙箱、上下文、多智能体、验证、独立开发者适配度（★ 强 / ◐ 有条件 / ○ 重）。

| 项目 | 沙箱 | 上下文 | 多智能体 | 验证 | 独立开发 |
|---|---|---|---|---|---|
| Claude Code | 本地默认 shell；云端沙箱；5 种权限模式；可阻断 hook | CLAUDE.md、按需 skill、子 agent 隔离、压缩 | Subagents、Agent Teams、Projects、Dynamic Workflows | hooks、测试、独立评审、浏览器 E2E | ◐ 闭源订阅，扩展最深 |
| Codex CLI | OS 沙箱默认；网络默认关；审批与沙箱分开 | 压缩 + 保留推理；AGENTS.md 地图 → docs 树 | 子 agent、云端并行任务、app-server | CI、apply_patch、agent 间互审 | ★ 开源，接受 OpenAI 模型 |
| OpenCode | 本地 shell；按工具 allow / ask / deny；LSP 反馈；git 撤销 | AGENTS.md、SQLite 会话、skill、MCP | 并行会话、通用子 agent | LSP 诊断、shell 测试 | ★ 最佳开源日常工具 |
| DeepSeek Harness | Landlock / Seatbelt / 受限 token；策略门控提问 | 追加式事件日志，可恢复/分叉/回放 | 子 agent（含 Claude / Codex 供应商，默认关） | 会话 transcript、遥测；无审计 | ◐ 强但预览版，隔离跑 |
| Pi | 无沙箱、无审批弹窗（自带容器） | 极简设计；扩展按需加载 | 扩展/子 agent 按需 | shell 测试；全透明 | ★ 想自己拥有循环时 |
| Hermes Agent | 本地 / Docker / SSH / Daytona / Modal；OS 隔离才是真边界 | 血缘压缩、AGENTS/HERMES + SOUL、 capped MEMORY、FTS5、自写 skill | 消息网关、定时任务、子 agent 演进中 | skill 需人工复核；无内置验证器 | ◐ 个人自动化很好 |
| OpenClaw | 工具沙箱默认关（Docker / Podman / SSH） | SQLite + FTS5 + 向量、dreaming  consolidating、markdown 记忆 | 通道 + cron + 节点（设备） | 审批面板、Control UI 轨迹 | ◐ 强，先查默认配置 |
| Grok Build | 本地 shell；Skills + MCP；ACP | 按需 skill；贴近 Claude Code 表面 | 经编排器作会话供应商 | shell 测试 | ★ 开源，生态小 |
| Grok Bot | 云虚拟机 + 浏览器 + 连接器 | 持久虚拟机状态；每 Bot 单任务指引 | 主 agent + 召唤子 agent | 敏感步骤人工接管、通知 | ○ 订阅、闭源 harness |
| Muse Code / Muse Spark Code | 本地 shell；无头浏览器验证；Gadgets skill | 子 agent transcript 可见；非阻塞问题卡片 | 官方 agent teams；社区子 agent 地图 | 浏览器控制台/网络检查后才算完成 | ★ 社区版 MIT / ACP |
| Gemini CLI | 写/shell 前确认；默认无沙箱（2026 年表） | 1M 上下文；shadow-git /rewind 检查点 | 子 agent、A2A 服务器 | shell 测试 | ◐ 现为付费；消费者看 Antigravity |
| OpenHands | Docker 沙箱默认 | 任务追踪、会话状态 | 编排/无头运行多智能体 | 测试、浏览器、SWE-bench 导向 | ○ 配置最重，隔离最强 |
| Goose | MCP 扩展（70+） | 供应商无关；桌面体验 | 扩展 + 子 agent | MCP 工具结果、人工审批 | ★ 通用、非开发者友好 |
| Deep Agents | 你的工具 + 文件系统 + LocalSandbox | write_todos、文件记忆、总结中间件 | 原生 task 子 agent | Terminal-Bench harness 效应证据 | ★ 最佳库式起点 |
| browser-use | 浏览器沙箱；密钥与提示词分离 | 每步剪枝页面树 | 作 MCP 工具进其他 harness | 截图、结构化抽取 schema | ◐ 每步成本/延迟 |
| Stagehand | Playwright（本地或 Browserbase 云） | 无障碍树剪枝（号称约 30 倍） | MCP 服务器进 Deep Agents | schema 类型化抽取 | ★ 已知流程 |
| GPT-Researcher | 网页搜索/爬取工具；本地文档 | 跨树分支共享学习/引用/已访 URL；上下文过滤 | 规划 + 执行 agent 树递归 | 每主张带引用 | ★ 研究报告 |
| Aider / Cline / Crush | Aider：git 当安全网，无沙箱；Cline：先问；Crush：MCP + LSP | Aider：仓库地图；Cline：编辑器上下文；Crush：模型切换 | Cline：无头/CI agent | Aider：每改提交 | ◐ Aider 停滞待复核；Cline ★ 适合 VS Code |

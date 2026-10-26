# Harness 工程深度报告与案例库

> 结论先说：**Agent = 模型 + Harness**。Harness 是模型与真实世界之间的运行层：循环、工具、沙箱、权限、上下文、验证、可观测性、多智能体编排与人工介入。2025–2026 年的共识是：大多数 agent 失败不是模型问题，而是 harness 配置问题。

- 网页版报告：见本仓库 `index.html`（发布到 GitHub Pages 后可直接打开）
- 英文原始调研报告：`docs/research-report-en.md`
- 案例速查：`cases/cases.md`
- 独立开发者最小起步架构：`docs/starter-architecture.md`

调研日期：2026-10-27。星数均为 2026 年 10 月二手汇总快照，不同来源不一致时按区间处理，请勿当作实时精确数字。

## 1. Harness 是什么

Harness 不是模型、不是一次提示词、也不完全等于框架：

| 概念 | 是什么 |
|---|---|
| 模型 | 推理与生成引擎，本身没有持久化、工具或记忆 |
| 提示词 | 这一次给模型的输入 |
| Agent | 模型 + 能行动的循环 |
| 框架 / SDK | 你自己组合的积木：图、节点、工具绑定、记忆、中间件 |
| **Harness** | **真正跑起来的运行时**：循环、工具、上下文、权限、验证、日志，更多是配置而非编码 |

核心十层：模型路由、工具调用、沙箱/执行、权限/审批、上下文与记忆管理、规划与任务分解、验证/测试、可观测性/日志、多智能体编排、人工介入。

在 harness 里，循环由运行时拥有且不可见；在框架里，循环或图拓扑由你显式编写。这是一个光谱，不是二选一。

## 2. 最值得抄的模式：薄循环 + 厚技能

1. 先用现成薄 harness，别先写框架。循环 + 工具控制在 500 行以内再谈花活。
2. 循环要薄，技能要厚：技能是按触发条件才加载的文件夹（`SKILL.md` + 脚本 + 参考资料），两周没被触发就删。
3. 工具要少而通用：读 / 写 / 改 / shell + 搜索 + 测试运行，胜过 40 个窄工具。每个工具要区分“查不到”和“查询失败”。
4. 文件 + git 就是状态：`PROGRESS.md` / `feature-list.json`（全 false → 逐项 true）/ 描述性提交。长任务一个会话只做一个特性， fresh 窗口跑 Ralph 式循环。
5. 必须每次发生的事用确定性 hook，不要只写进提示词：改后格式化、禁写受保护路径、测试通过 + 完成承诺出现前不许退出。
6. 先沙箱，后凭证：Docker / Seatbelt / bubblewrap，或至少一次性虚拟机 + 独立 worktree + 最小权限 token。别把常驻 agent 直接挂在主 shell 和主凭证上。
7. 验证者要独立：第二个 agent、新上下文、最好换个便宜/不同模型，只输出通过/失败 + 证据；高层测试不能让作者 agent 改。
8. 第一天就建评测集：10 个真实任务，按“产生了什么效果”判分，不按文本像不像。每次改提示词、工具、模型都跑一遍，同时看每次成功任务的成本和步数。
9. 模型路由 + 预算：强模型做规划/评审，便宜模型做执行/检索；每会话 token 预算要能真正阻断下一次调用；步数上限 + 重复检测 + 最大迭代阀。
10. 可观测性是功能：追加式运行日志（工具、参数、结果、token、允许/拒绝），跑不起来就复现不了，复现不了就修不了。

## 3. 要避开的坑

- 只靠提示词做权限（“小心生产环境”），以及在非一次性容器里用跳过权限模式。
- 工具动物园、无限上下文自助餐、杀不掉的会话、无评测调优。
- 单智能体还没可靠就多智能体扇出：成本翻倍、失败级联、评审更难。先为隔离加一个子 agent（例如无副作用的网页检索），不是为并行表演。
- 自己给自己打分、自己写 skill 不复核：一次侥幸会被固化成流程。
- 只看 star 数买单：先看许可证、最近更新、默认配置。Aider 自 2026 年 5 月起停滞、Gemini CLI 消费者版已停服、Roo Code 已归档、Crush 是 FSL 而非 MIT，都是 star 数看不出来的。

## 4. Harness 效应：有数字为证

- LangChain Deep Agents：只改 harness（轨迹分析、压缩、错误格式、自验证），Terminal-Bench 2.0 从 **42.65% → 66.5%**。
- Meta Muse Spark 1.2（VulcanBench 报告 20）：裸 API 循环 **52.2%**，换 Pi harness 后 **69.6%**，差 **17.4 个点**。同一模型，harness 不同，结果完全不同。

## 5. 被点名的十个 harness（速查）

完整拆解见 `cases/cases.md`，这里只放一句话定位：

- **Claude Code**：参考实现。约 40 个工具、hooks / skills / subagents / 插件最深，Agent Teams / Projects / Dynamic Workflows；闭源、订阅制，本地默认无容器隔离。
- **Claude Work**：查不到独立产品。对应 Anthropic 的 Projects / Agent Teams / Dynamic Workflows + 社区脚手架，另有一个同名多账号 shell 别名，别混淆。
- **Claude Design**：Anthropic Labs 设计 harness，聊天 + 画布 + PDF / PPTX / HTML / Canva 导出 + 交给 Claude Code 的 handoff；开源复刻看 baoyu-design、Open Design、SuperDesign、Hallmark。
- **Codex**：OpenAI，Apache-2.0，Rust。OS 沙箱是架构本身（Seatbelt / bubblewrap），沙箱与审批策略分开配置；新模型可用 JS-runner 在沙箱内组合工具，省往返和 token。
- **OpenCode**：MIT，最多星的开源终端编码 agent 之一。客户端/服务端 + SQLite 会话、Build / Plan 双模式、75+ 模型供应商，换模型最自由。
- **DeepSeek Harness（dsh）**：MIT，Node.js，Cordis 插件框架，“一切皆插件”，连循环本身都可替换；开发者预览、无安全审计，必须放隔离环境。
- **Grok Build**：Apache-2.0，2026 年 7 月开源，刻意贴近 Claude Code 的 CLI 表面，便于进多 harness 编排；早期默认上传数据争议是“默认值比口号重要”的案例。
- **Grok Bot**：不是编码 CLI，是托管在云虚拟机上的通用“数字同事”，主 agent + 子 agent + 一键连接器；Grok 4.7 被描述为针对该 harness 联合训练，harness 本身闭源/托管。
- **Hermes Agent**：Nous，MIT，Python。常驻通用 agent，血缘式上下文压缩、工具注册与暴露分离、自己写 skill 自我改进——但自写 skill 无人复核就可能固化错误流程。
- **Meta Muse**：模型 Muse Spark 闭源（1M 上下文），官方 Muse Code + 社区 MIT 项目 Muse Spark Code（ACP、可见子 agent 地图、无头浏览器自检）；先分清教程说的是哪个 Muse。

## 6. 从业者原话（按二手报道归因，见英文报告来源节）

- Mitchell Hashimoto（2026-02-05）：“每次 agent 犯错，就去工程化一个方案，让它再也不犯同样的错。”`AGENTS.md` 应该是失败日志，不是愿望清单。
- Viv Trivedy / LangChain（2026-02 / 03）：“编码 agent = 模型 + 它周围的 harness，harness 做了大部分工作。”并给出 harness  taxonomy。
- Addy Osmani：“一个好 harness 配普通模型，胜过一个差 harness 配好模型。”
- Boris Cherny、Peter Steinberger（2026-06）：“别再给 agent 写提示词了，去设计让 agent 循环跑起来的 loop。”
- Ralph Wiggum 循环：同一提示词在 fresh 会话里反复跑，直到测试通过 / 出现完成承诺；状态放文件和 git，不靠一个越来越长的会话。

## 7. 失败模式速查

| 失败 | harness 解法 |
|---|---|
| 过早宣布完成 | feature-list 全 false 起步、Ralph 循环、Stop hook 挡提前退出 |
| 自己给自己打分 | 独立评审 agent、新上下文、对抗评审 |
| 目标漂移 / 上下文腐烂 | 带血缘的压缩、文件里钉住状态、每工作单元开新会话 |
| 改测试蒙混过关 | 测试对 agent 不可改，CI 里再卡一道 |
| 工具太多选不过来 | 少而通用工具，工具描述一行，详情按需加载 |
| 上下文无限膨胀 | 总结、丢弃过期工具输出、子 agent 只回路径 + 一行摘要 |
| 循环 / 卡住 | 重复检测 + 步数上限 + 最大迭代阀 |
| 工具静默失败 | 输出里区分“无数据”和“查询失败” |
| 提示词注入 | 检索内容一律当数据，检索子 agent 无副作用工具，敏感动作人工审批 |
| 多智能体级联幻觉 | 交接用 schema 不用散文、加验证步、限制扇出 |
| 成本失控 | 每会话预算真正阻断、按任务路由模型、看每次成功任务成本 |
| 会话丢了 / 杀不掉 | 检查点就是 git 提交，靠 diff 二分，不靠翻聊天记录 |

## 8. 局限与未验证项

- “Claude Work”独立产品未在官方来源中找到，按上述三层含义解释；如果你指某个具体产品，请给 URL，我再补一节。
- 星数、许可证、价格均为 2026 年 10 月二手快照，发布前请对要引用的那几项单独复核 GitHub。
- Grok Bot、Muse Code 内部、Claude Design 价格/上限主要来自评测与二手报道，非官方文档原文。
- X 原帖多为二手转述（播客、文章、浏览量统计），报告里按“据报道”归因，未直接抓取 x.com 原帖。

## 9. 如何贡献

欢迎 PR 补案例：请附仓库/文档链接、一句话 harness 设计、许可证与最近更新时间；star 数只写区间快照并注明日期。涉及安全默认值（沙箱默认开还是关、是否需人工审批）的案例请特别标出。

许可证：MIT。

# Harness 工程深度报告与案例库

> 📖 在线阅读：[https://jaxxchen003.github.io/ai-harness-engineering/](https://jaxxchen003.github.io/ai-harness-engineering/)

> 结论先说：**Agent = 模型 + Harness**。Harness 是模型与真实世界之间的运行层：循环、工具、沙箱、权限、上下文、验证、可观测性、多智能体编排与人工介入。2025–2026 年的共识是：大多数 agent 失败不是模型问题，而是 harness 配置问题。

- 网页版报告：[在线版](https://jaxxchen003.github.io/ai-harness-engineering/)（GitHub Pages，源文件见本仓库 `index.html`）
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

## 5.5 补记（2026-10-30）：Herdr 与 Oh My Pi

这两个项目补进来后，综合分析多了一条关键分界线：**一个是“管 agent 循环”的 harness，一个是“管 agent 进程”的运行时**。把它们混成一类，就会误判各自的安全边界和适用场景。

### Herdr（herdrdev/herdr）—— 不是又一个 agent，是 agent 住的运行时

- **定位**：终端多路复用器 / agent 运行时，官方口号是 “the runtime your coding agents live on”。它不跑模型、不替换 agent，而是拥有 agent 所在的终端：Claude Code、Codex、Cursor、OpenCode、Grok 等都可以在它里面跑。
- **仓库与热度（2026-10-30 核验）**：[herdrdev/herdr](https://github.com/herdrdev/herdr)，Rust，Apache-2.0，约 46,837 星、3,736 fork，2026-03-27 创建，2026-10-30 仍在更新；另有 herdr-app（桌面端）与 herdr-core（终端核心）两个拆分仓库。
- **架构**：后台服务器 + 客户端的 PTY 模型。关掉客户端、SSH 断线、笔记本合盖，工作不死；重连后恢复保存的布局，并可恢复受支持 agent 的会话（注意：机器/服务器重启后原始进程并不存活，恢复的是布局与可恢复的会话，不是进程本身）。
- **多智能体 / 编排**：workspace（项目/大任务）→ tab（工作方式）→ pane（具体终端进程）。每个 pane 标 working / blocked / idle / done，督导者一眼看到哪个 agent 卡住了、等你回答。agent 可通过 CLI 与本地 socket API 从 pane 内驱动 Herdr：开 pane、给其他 agent 发提示、等另一个 agent 真正 blocked 再继续；插件用可执行文件 + manifest 动作与事件钩子扩展，另有 Marketplace 方向。
- **多机**：本地与已保存的 SSH 机器可放在同一个窗口，agent 列表合并、各机器独立重连，适合“远程大机器跑 agent、本地只看状态”。
- **上下文 / 状态**：强项是会话状态而非模型上下文：detach、重启恢复、pane 历史回放、原生 agent resume、live handoff。模型上下文、压缩、记忆仍归 pane 里那个 agent 自己管，Herdr 不替它做。
- **验证**：Herdr 本身不做代码验证。它的“验证”是状态可观测：哪个 pane 卡住、哪个完成、哪个在等审批，督导者据此介入；真正的测试、评审、合并仍靠 agent 或外部流程。
- **安全边界（必须写清楚）**：Herdr 不是沙箱。按从业者评测的原话，Herdr 买来的是“感知与持久化”，隔离仍是你自己的事；若多个 agent 可能碰同一批文件，分支 / worktree 边界要你自己先建好，否则就是合并冲突。不要因为它能并行跑多个 agent，就默认每个 agent 是隔离的。
- **适用**：已有 2 个以上 agent、经常并行、经常远程跑、经常被“哪个 agent 在等我”拖慢的人。它是 tmux / Zellij 的 agent 感知升级，不是任务分配器、不是评审器、不是工作流编排器。想要“产品替我分任务、建 worktree、走评审、定合并策略”的，Herdr 不替你做决定。

### Oh My Pi（can1357/oh-my-pi，命令 omp）—— Pi 的“全副武装版”

- **定位**：Pi 的 MIT fork，口号是 “Coding agent with the IDE wired in”，由 Stencil Labs 的 Can Bölük 维护，2025-12-31 fork 自 Mario Zechner 的 Pi。它回答的是 Pi 刻意不回答的问题：如果 harness 本来就该懂 IDE，那它该长什么样？
- **仓库与热度（2026-10-30 核验）**：[can1357/oh-my-pi](https://github.com/can1357/oh-my-pi)，MIT，TypeScript + Rust 核心，约 37,183 星、4,177 fork；README 自述 60+ 供应商、31 个内置工具、14 个 LSP 操作、28 个 DAP 操作、约 80k 行 Rust 核心（另有约 80k 行 vendored 代码）。
- **与 Pi 的根本分歧**：Pi 是减法——4 个工具、<1k token 系统提示词、其余靠扩展；OMP 是加法——把 IDE 能力做进 harness 本体。原报告里 OMP 只占一行“极繁 fork”，这次补记把它升格为完整案例，因为它已经不是“Pi 的一个变体”，而是与 Pi 对立的一条路线。
- **工具 / 代码智能**：`read` 统一读文件、目录、归档、SQLite、PDF、笔记本、URL、SSH 路径与内部 `://` 协议（如 `pr://`、`agent://`、`skill://`）；`edit` 用 Hashline（内容哈希锚点，文件变了就拒绝补丁，不会把旧文本硬贴上去）；`ast_edit` / `ast_grep` 做结构化改写与查询；LSP 接入每次写入（重命名走 language server，跨文件引用一起改）；DAP 真调试（lldb / dlv / debugpy，断点、单步、看栈与变量，而不是靠 print 猜）。
- **执行**：持久 Python 与 Bun 内核，且内核可通过 loopback 桥回调 agent 自己的工具（read / search / task）；`bash` 是内置 brush shell，58 个命令行工具 in-process，不用每次 fork/exec；`pi-iso` 做 workspace 隔离（apfs / btrfs / zfs / overlayfs 等），但这不等于给 agent 命令套了 OS 沙箱，引用时别把两者混淆。
- **模型路由**：60+ 供应商，9 个角色路由（default / smol / slow / plan / commit / vision / task / advisor / tiny），按角色换模型、fallback 链（429 / 配额耗尽时接力）、路径级模型限定、同一供应商多 key 轮询；另有大量 coding-plan 订阅路由（Cursor、Copilot、Kimi、MiniMax、Qwen 等）。
- **多智能体 / 验证**：`task` 子 agent 可并行、workspace 隔离、结果是 schema 校验过的结构化对象而非散文，Agent Hub 可看每个子 agent 的实时 transcript、发 steering 消息、复活或杀掉卡住的 worker；另有 advisor 角色——第二个模型、独立上下文，每轮读主 agent 的动作并注入提醒或硬阻断；`/review` 派评审子 agent 并行扫分支 / 提交 / 未提交改动，按 P0–P3 排优先级并给 verdict；`/collab` 可把 live 会话分享成链接或二维码（只读或读写）。
- **上下文 / 记忆**：`retain` / `recall` / `reflect` / `learn` / `memory_edit` 做 agent 自管记忆（本地、Hindsight 或 Mnemopi 后端，默认项目级隔离），`checkpoint` / `rewind` 做会话压缩与回退；`stream rules` 可在模型跑偏时按正则中断流、注入规则并从同一点重试，且注入在压缩后仍保留。
- **成本与速度的另一面（据二手评测，非官方基准）**：Composio 在同一模型（deepseek-v4-flash）上的 30 任务评测里，Pi 通过 20 个、OMP 通过 17 个；每次成功成本 Pi 约 $0.028、OMP 约 $0.103；中位耗时 Pi 约 132 秒、OMP 约 272 秒；平均 token Pi 约 559k、OMP 约 742k。也就是说，OMP 在“弱模型下一次改对”（如 Hashline 让 Grok Code Fast 1 从 6.7% 升到 68.3%，README 自述）上很强，但在通用任务上更贵、更慢——这是“功能面”与“运行成本”的经典交换，不是单向碾压。
- **信任与许可（引用前必读）**：OMP README 明确写着，打开仓库即按设计加载项目输入（设置、扩展、hook、工具、命令、skill、规则、项目 MCP 配置）；二手分析进一步指出其项目信任门与审批默认较宽（有分析称默认 yolo），与 Pi 的“默认先问”不同。结论：OMP 适合你信任的仓库与可控环境，不适合直接打开陌生仓库就全速跑。
- **适用**：想要“开箱即用、不用自己拼扩展”的重度编码用户，尤其看重 LSP 诊断、真调试、结构化子 agent 结果与评审流程的人；想要“循环小到能读完、能 fork、能嵌入别的产品”的人，仍应选 Pi 本体。

### 两项目放进原有横向对比，结论怎么变

| 维度 | Herdr | Oh My Pi | 原报告的启示是否还成立 |
|---|---|---|---|
| 它管什么 | agent 进程与终端会话，不跑模型 | 完整 agent 循环，从模型到工具到评审 | 成立，且要加一层：harness 之上还有“运行时层” |
| 沙箱 | 不提供，隔离是你自己的事 | workspace 隔离（pi-iso），不等同 OS 沙箱 | 成立：沙箱严格度仍随工具权力上升，但 Herdr 提醒你“并行 ≠ 隔离” |
| 多智能体 | pane 级并行 + 状态督导 + CLI/socket 驱动 | task 子 agent + worktree 隔离 + schema 结果 + advisor | 成立：OMP 证明“结构化交接”比“散文交接”更能防级联幻觉；Herdr 证明“先看见谁卡住”比“多开几个 agent”更先解决瓶颈 |
| 验证 | 状态可观测，不做代码验证 | 独立评审子 agent + advisor + /review verdict | 成立：验证者与作者分离仍是铁律，OMP 只是把它产品化了 |
| 独立开发者适配 | ★ 适合多 agent 并行督导，但别指望它替你分任务 | ★ 适合重度编码开箱即用，◐ 信任陌生仓库前先收紧配置 | 原“薄循环 + 厚技能”仍是起点；OMP 是“厚循环”的代表，选它等于接受更高的审计与运行成本 |

**一句话选型**：只有一个 agent 在跑，先别上 Herdr；已经并行到“找不着哪个在等你”了，Herdr 是最便宜的解法。想要 Pi 的可读性，选 Pi；想要 Pi 的生态但不想自己拼 IDE 能力，选 OMP，并把它的项目信任与审批默认当作第一项配置来检查。

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

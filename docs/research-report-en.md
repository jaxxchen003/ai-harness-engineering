# AI Agent Harness Engineering — Deep Research Report

**Research date:** 27 Oct 2026 (index-sourced; no sign-in, no live browser verification of paywalled/vendor docs)
**Confidence note:** Definitions, architectures and practitioner principles are well corroborated across independent sources. GitHub star counts differ by source and date (see §3/§5 notes) — treat them as Oct-2026 adoption snapshots, not exact live counts. Where sources conflict, the conflict is flagged rather than resolved silently.

---

## Summary

**Answer first:** An agent harness is *not* the model, *not* a prompt, and *not* (quite) a framework. It is the runnable system around the model — `Agent = Model + Harness` — that owns the loop (prompt → tool calls → observations → repeat), the tools, the sandbox/permissions, the context that is assembled, compacted and persisted, the verification that decides "done", and the logs that make a run debuggable. In 2025–2026 the field converged on the same conclusion from three directions: Mitchell Hashimoto's failure-driven definition, LangChain/Viv Trivedy's taxonomy, and Anthropic/OpenAI's production reports — a decent model in a great harness beats a great model in a bad harness, and most agent failures are configuration/harness failures, not model failures.

**For a solo developer, the copyable pattern is:** a *thin* loop + *thick, on-demand skills*, few general tools (read/write/edit/shell), deterministic hooks for things that must always happen, file/git as the durable state, a separate verifier that is not the author, OS/container sandboxing, and a 10-task eval set run on every change. Avoid: a 40-tool zoo, unbounded context, prompt-only permissions, self-grading, and multi-agent fan-out before a single-agent loop is reliable.

---

## 1. What is an agent harness, precisely?

### 1.1 Definition

> "A model thinks. An agent acts. A harness keeps the agent from acting like an idiot." — summary in https://dev.to/cwilkins507/what-is-an-ai-agent-harness-2fce

More formally, from several corroborating sources:

- **Harness = the operational layer between model and real world.** "Everything positioned between the model and the real world… retry logic, state management, context handling, and recovering a workflow after a crash" — https://edge-case.medium.com/the-ai-agent-harness-what-it-is-and-why-its-required-3e74cb04b528
- **"The engineering scaffolding around a language model that turns it into a dependable agent: the context it is allowed to see, the tools it can call, the authority guardrails it cannot cross, the runtime it executes in, and the evaluations that measure the run."** — https://www.scrums.com/platform/agent-harness
- **"The runtime environment and supporting mechanisms around a tool-using AI agent. The model proposes steps; the harness supplies context, mediates tool access, stores execution state and collects the evidence needed to decide whether the task succeeded."** — https://blog.abark.tech/agentic-harness-architecture/
- **Anthropic's own decomposition (Claude Managed Agents):** Session (append-only event log) + Harness (stateless control loop invoking Claude and routing tool calls) + Sandbox (isolated execution). "Containers become 'cattle, not pets'" — summarised at https://rits.shanghai.nyu.edu/ai/anthropic-launches-claude-managed-agents-for-scalable-ai-deployment/

### 1.2 Distinctions (model / prompt / framework / agent / harness)

| Concept | What it is | Example |
|---|---|---|
| **Model** | Reasoning/generation engine; predicts text, has no persistence, tools or memory on its own | Claude, GPT, Gemini, Grok, Muse Spark, DeepSeek-V4 |
| **Prompt** | One input: what you tell the model this turn (system + user + tool schemas) | A system prompt, `CLAUDE.md` excerpt |
| **Agent** | Model + loop that lets it act | Claude Code fixing a bug; a research agent writing a cited report |
| **Framework / SDK** | Building blocks you compose in code: graphs, nodes, crews, tool bindings, memory, middleware | LangChain/LangGraph, CrewAI, OpenAI Agents SDK |
| **Harness** | The *working runtime* around the agent: loop, tools, context, permissions, verification, logging — configured more than coded | Claude Code, Codex CLI, OpenCode, Pi, Hermes, DeepSeek Harness |

Key nuance, from https://matheusjerico.medium.com/who-owns-the-agent-loop-node-js-harnesses-vs-python-frameworks-langgraph-crewai-langchain-8c6a153178b4 :

> "In a harness, the loop is **invisible and owned by the runtime**. In a framework, the loop (or the graph topology, or the crew structure) is **explicit and authored by you**." It is a spectrum, not a binary. "You can use [framework] pieces to build a harness. But the framework itself is not automatically the harness." — https://dev.to/cwilkins507/what-is-an-ai-agent-harness-2fce

A related distinction used in 2026 writing: **prompt engineering** = what to tell the model; **context engineering** = what it should see; **agent engineering** = what role/tools it has; **harness engineering** = how the whole system operates — https://www.krishnaik.in/ai-tutorials/generative-and-agentic-ai/harness-engineering . And **loop engineering** (the repeated work-and-feedback cycle) sits *inside* harness engineering — https://devenkoai.medium.com/agent-harness-engineering-vs-loop-engineering-vs-graph-engineering-cd41d18bbc84 .

### 1.3 Core layers (the mission's ten, mapped)

1. **Model routing** — provider adapters, per-task model choice (strong planner / cheap worker), fallback on quota, effort levels. Claude Code: Claude-only via subscription/Bedrock/Vertex/Foundry; OpenCode/Pi/Hermes/DeepSeek Harness: multi-provider + local via Ollama.
2. **Tool use** — few, typed, narrow tools with safe failure modes. The spectrum runs from Pi's 4 tools (read/write/edit/bash) to Claude Code's ~40–45 and DeepSeek Harness's ~53 built-ins. Fewer tools = less tool-selection reasoning and fewer tokens per call.
3. **Sandbox / execution** — OS-enforced (Seatbelt/macOS, Landlock/bubblewrap+seccomp/Linux, Docker, restricted-token Windows runner) vs. prompt-only. Codex and DeepSeek Harness treat this as architecture; Pi/Hermes explicitly refuse to pretend in-process restrictions are a sandbox.
4. **Permissions / approval** — declarative allow/ask/deny rulesets, permission modes (Claude Code: default/acceptEdits/plan/auto/bypass; Codex: sandbox + approval policy as *separate* layers; OpenCode: plan mode = a permission ruleset, `edit: deny`). Hooks that can *block* a tool call are the only enforcement that does not depend on the model's attention.
5. **Context / memory management** — instruction files (`CLAUDE.md`/`AGENTS.md`/`HERMES.md`/`SOUL.md`), on-demand skills (load full content only when triggered), compaction/summarisation with lineage, session persistence (SQLite, append-only event logs), file-based state (progress files, git history) for cross-window work.
6. **Planning & task decomposition** — planning tools (`write_todos` in Deep Agents), feature lists (`feature-list.json`, all `passes:false` in Anthropic's long-running recipe), dynamic workflows that generate a JS orchestration program on the fly.
7. **Verification / testing** — tests, linters, typechecks as *sensors* that feed back into the loop; a separate evaluator/verifier agent; the "test ratchet" (never delete/edit failing tests to make them pass); browser automation (Puppeteer/Playwright) for end-to-end checks.
8. **Observability / logging** — traces, transcripts, session logs, token/cost accounting, audit trails of allow/deny decisions. If you cannot see what the agent did, you cannot fix, trust or explain it.
9. **Multi-agent orchestration** — subagents with isolated context windows, worktrees/branches per worker, coordinator + workers, shared memory/goals. Cost: every subagent costs tokens and makes verification harder; delegation is for *isolation*, not just parallelism.
10. **Human-in-the-loop** — approval gates for irreversible actions, question cards that don't block work, steering mid-run, and final human review of diffs/PRs. HumanLayer's position: humans remain the judges of quality; agents optimise for test-passing, not maintainability.

**Arize's nine-part analysis model** (used to audit Hermes) is a useful checklist: outer iteration loop; context management & compression; skills & tools management; subagent management; built-in pre-packaged skills; session persistence & recovery; system-prompt assembly with project-context injection; lifecycle hooks; permission & safety layer — https://arize.com/blog/how-hermes-implements-open-source-agent-harness-architecture/

---

## 2. Named harnesses — case files

> Star counts below are Oct-2026 snapshots from secondary roundups and conflict across sources; ranges are given. "Praise/criticism" is attributed to the cited write-ups, not this report's own testing.

### 2.1 Claude Code (Anthropic) — the reference harness

- **Official/docs:** Anthropic's Claude Code product/docs (claude.com / platform.claude.com docs, referenced via https://platform.claude.com/docs/en/managed-agents/overview in search results); plugin repo `anthropics/claude-code` (Ralph plugin path cited at https://www.test.essencequebec.com:443/index.php?https://github.com/anthropics/claude-code/tree/main/plugins/ralph-wiggum — use the GitHub path, not the proxy URL).
- **What it is:** Proprietary (closed-source) agentic coding CLI + IDE/desktop/web surfaces, launched May 2025, Claude-models only (also via Bedrock / Google Agent Platform / Microsoft Foundry). The harness everyone else reverse-engineers.
- **Architecture/design:** ~40+ tools (core Read/Edit/Grep/Glob/Bash); `CLAUDE.md` (+ `AGENTS.md` read) project memory; skills (`.claude/skills/<name>/SKILL.md`), subagents (`.claude/agents/`, own context window/tools/permissions), MCP servers (`.mcp.json`), hooks in `.claude/settings.json` (notably `PreToolUse` can block, `Stop` can gate exit), plugins bundling all of the above, worktrees, compaction, five permission modes (default / acceptEdits / plan / auto / bypassPermissions — bypass only in isolated containers/VMs). Sessions resumable from transcripts; web/cloud sessions run in Anthropic-managed sandboxes with PR creation.
- **Multi-agent:** Subagents + (2026) Agent Teams (team-lead + teammates, shared task list, own context windows), Claude Code Projects (Sept 17, 2026 relaunch: coordinator + parallel cloud threads, each on its own branch/repo copy, shared memory/goals/library, conflicts surfaced as git merge conflicts), and Dynamic Workflows (lead agent writes a JS workflow program spawning up to ~1,000 subagents, per-model routing, isolated worktrees) — https://pondero.ai/news/2026-09-18-claude-code-projects-multi-agent/ ; https://www.theneuron.ai/explainer-articles/claude-code-dynamic-workflows-explained-claude-can-now-build-its-own-workflow-around-a-task/ ; https://kaino.dev/news/claude-code-starts-writing-the-program-that-runs-its-subagents
- **Stack/license:** Node.js CLI, proprietary/closed-source (cannot fork/audit the loop).
- **Praise:** Deepest extension system (hooks/skills/plugins); strongest long-running recipes (see §4); consensus pick for deep multi-file work. **Criticism:** Closed source; large system prompt (~10–14k tokens in comparisons vs Pi's <1k); `bypassPermissions` is the mode "that ends up in the news"; local runs have no container isolation by default (user-granted filesystem/shell access).

### 2.2 "Claude Work" — no standalone product found; three real things behind the name

**Could not verify a product officially named "Claude Work".** The mission's phrase maps to three distinct, verifiable things — do not conflate them in the Chinese report:

1. **Claude Code Projects / Agent Teams / Dynamic Workflows** (§2.1) — Anthropic's actual multi-agent *work* harness concepts: coordinator + worker threads, shared memory, git-branch isolation.
2. **Community multi-agent scaffolds on top of Claude Code:** e.g. GStack (open-source repo turning Claude Code into an "AI engineering team" via skills: Office Hours, adversarial review, Design Shotgun) and the "agent teams" pattern from the 16-Claude compiler experiment (16 instances, ~2,000 sessions, ~$20k, each in an isolated Docker container, file-lock task claiming, git rebase on conflict) — https://youtubetosummary.com/s/inside-garry-tan-s-claude-code-setup-wkv2ifxPpF8 ; http://faun.dev/c/news/devopslinks/anthropic-claudes-20000-16-ai-agents-and-a-compiler-that-builds-linux/
3. **A naming collision to warn readers about:** `claude-work` as a *shell alias* for a second Claude account (`CLAUDE_CONFIG_DIR=~/.claude-work claude`) — a multi-*account* trick, not a multi-agent product — https://github.com/espseongsm/vibe-coding-know-how/blob/main/claude-multi-account-macos.md ; https://dev.to/vainamoinen/two-multi-account-claude-code-architectures-one-anthropic-accepts-one-they-ban-2om7

Related official layer: **Claude Agent SDK** (build-it-the-way-we-built-Claude-Code runtime) and **Claude Managed Agents** (hosted: Session/Harness/Sandbox split, $0.08/session-hour + token pricing, credentials kept out of the sandbox via egress proxy, ~60% p50 / ~90% p95 TTFT improvement claimed) — https://rits.shanghai.nyu.edu/ai/anthropic-launches-claude-managed-agents-for-scalable-ai-deployment/ ; https://medium.com/@zero-cost-ai/claude-managed-agents-vs-your-own-agent-which-one-to-build-with-a-step-by-step-example-8e2c6af07519

### 2.3 Claude Design / design-agent harnesses

- **Official product:** Claude Design, Anthropic Labs, launched **17 Apr 2026** as research preview (later beta, out of beta 8 Oct 2026 per one tracker), at claude.ai/design for Pro/Max/Team/Enterprise; powered at launch by Claude Opus 4.7, later a model picker. Chat + live canvas; refine by chat, inline comments, direct edits, generated sliders; inputs: text, images, DOCX/PPTX/XLSX, screenshots, web capture, GitHub repo/design-system import; exports: PDF, PPTX, standalone HTML, Canva; handoff bundle to Claude Code — https://appwrite.io/blog/post/claude-design ; https://www.datacamp.com/blog/claude-design ; https://moexplains.com/claude-design-review-anthropic-labs/
- **Design-harness lesson:** It is a *hosted, artifact-first* harness: the canvas + export/handoff *is* the verification loop (you see the design, you comment on an element). It is explicitly *not* a Figma replacement (no pixel-level component system) — a complement.
- **Open replicas / design-agent scaffolds (the harness-design part a solo dev can copy):**
  - **baoyu-design** — `https://github.com/JimLiu/baoyu-design` — "Run Claude Design locally as an Agent Skill" for Cursor/Claude Code/Codex etc. Plain Markdown + JSX/JS scaffolds, no build step: `SKILL.md` orchestrates, `system-prompt.md` is the craft source of truth, per-harness tool maps (claude/cursor/codex), built-in skills (hi-fi, prototype, wireframe, decks, mobile, design-system creation/preview, `.dc.html` components), starter components, outputs as self-contained HTML/PDF/PPTX, Figma/Canva handoff. Best with Opus.
  - **Open Design** — Apache-2.0 desktop app, local-first, ~21 built-in design agents + ~129 starting points, three parts (local app / daemon / agent runtime), uses the user's own coding agent (Claude Code/Codex/Cursor/Gemini/OpenCode/Qwen) as the engine, artifacts stored locally — https://www.xda-developers.com/replaced-claude-design-open-source-tool-not-paying-design-software/ ; https://ucstrategies.com/news/open-design-open-source-alternative-to-claude-design/
  - **SuperDesign** — `https://github.com/superdesigndev/superdesign` (original IDE extension, 2025, now noted as no longer actively maintained; product moved to web app superdesign.dev; skill at superdesigndev/superdesign-skill). MIT-era, ~6.5k stars at one write-up: prompt → multiple parallel UI mockups/components/wireframes on an infinite canvas, saved locally under git — https://medevel.com/superdesign/ ; https://theresanaiforthat.com/company/superdesigndev/repository/superdesign/
  - **Hallmark** — `Nutlope/hallmark` (MIT, Together AI): a *design skill as verifier* — deals 1 of 48 themes, 16 non-waivable rules, a browser linter (`sloplint`, 8 widths) for "AI-generated tells", and a fresh-context reviewer subagent whose first word (`recapture`/`rebuild`/`fix`/`ship`) decides the next step — https://medium.com/@anubhavbhatt/hallmark-the-design-skill-that-lints-ai-generated-ui-752f90ba9529

### 2.4 OpenAI Codex (Codex CLI + Codex agent harness)

- **Official repo:** `openai/codex` — Apache-2.0, ~119–129k stars (Oct 2026 roundups: https://dev.to/rosgluk/trending-open-source-ai-projects-on-github-october-2026-4d42 ; https://pinggy.io/blog/best_open_source_cli_coding_agents/).
- **What it is:** Open-source (Rust engine) terminal/IDE/desktop/cloud/GitHub-bot harness, co-designed with GPT-5.x-Codex / GPT-6.x models. Three integration surfaces (2026 "Codex as a Platform"): `codex exec` (bounded scripted tasks/CI), Codex SDK (programmatic start/resume/stream), and app-server (persistent conversation state, event streaming, tool exposure, approval handling — your app owns business logic/MCP/approval/UI, Codex owns loop + sandboxed execution).
- **Distinctive design:** Sandboxing *as architecture*: every command in an OS jail (Seatbelt/macOS, bubblewrap+seccomp/Linux), workspace-scoped filesystem, network denied by default, model can *request escalation* as part of the tool schema; sandbox and approval policy configured independently. Mostly shell + `apply_patch`; on newest models, one JS-runner tool lets the model write a script that composes tools internally (fewer round trips/tokens), running in a sealed sandbox — https://www.tyolab.com/blog/2026/06/13-the-coding-agent-shootout-and-the-best-harness-we-could-not-read/ ; https://thequiteside.substack.com/p/inside-four-agent-harnesses . Context compaction + retained reasoning claimed large gains (ARC-AGI-3 13.3%→38.3% with ~6× fewer output tokens in one cited comparison).
- **Praise:** Strongest raw coding capability in several comparisons, most token-efficient (2–3× fewer tokens claimed), real security model. **Criticism:** Models closed and deprecated on OpenAI's schedule; cloud model not selectable; hooks/skills/plugins ecosystem younger than Claude Code's; best on OpenAI models.

### 2.5 OpenCode (Anomaly, ex-SST)

- **Official repo:** `anomalyco/opencode` — MIT, ~202–222k stars across Oct 2026 snapshots, the most-starred open terminal coding agent in those roundups.
- **What it is:** Terminal-first (TUI via Bubble Tea), plus desktop app (beta) and IDE extensions (VS Code/Cursor/Zed/JetBrains/Neovim/Emacs via ACP). Model-agnostic: 75+ providers via Models.dev, plus Ollama/LM Studio/llama.cpp local, and login with GitHub Copilot or ChatGPT Plus/Pro accounts.
- **Design:** Client/server — background server holds AI communication + session state in local SQLite; TUI/desktop/IDE are clients, so sessions survive SSH drops. Two built-in agents switched by Tab: **Build** (full access) and **Plan** (read-only by default: denies edits, asks before bash). General subagent via `@general`; LSP diagnostics fed back after edits; git snapshot undo/redo (`/undo`); MCP client (incl. OAuth remote servers); shareable session links; CI mode (`/opencode` in GitHub issues/PRs). Permissions as per-tool allow/ask/deny data, not hard-coded modes.
- **Praise:** Vendor independence, privacy (code stays local unless a cloud model is chosen), mix-and-match models per task. **Criticism:** Quality = model you pick; more configuration than commercial tools; you fix breakage yourself; desktop still beta; permissive Build defaults (only repeated identical calls / out-of-project paths gated by default) — https://www.ayautomate.com/blog/best-open-source-ai-agents-2026 ; https://fastino.ai/blog/the-complete-guide-to-opencode-open-source-ai-coding-agents

### 2.6 DeepSeek harness projects

- **Official: DeepSeek Harness (`dsh`)** — `github.com/deepseek-ai/deepseek-harness` (package `@deepseek-ai/dsh`, `npx @deepseek-ai/dsh web` → http://127.0.0.1:3080), MIT, Node.js, **developer preview, launched 13 Aug 2026**. Star reports conflict wildly by date: 74,578 in one day, ~95k in 48h, ~203k by late Aug, ~242–247k by early Oct — all secondary; the *trajectory* (fastest adoption in the set) is corroborated, the exact number is not.
- **Design — "everything is a plugin" on the Cordis composition framework:** model adapter, tool registry, session log *and the agent loop itself* are replaceable plugins with clean add/remove (side effects fully undo); no privileged core. Four modes (Standard/Minimal/Code/Creator); providers: Anthropic, OpenAI, Bedrock, Azure, Gemini Enterprise, DeepSeek, any OpenAI-compatible gateway; built-in Claude Code/Codex subagent providers (disabled by default); append-only event log → resume/fork/replay/transcripts; sandboxing Landlock (Linux) / Seatbelt (macOS) / restricted-token runner (Windows); hooks.json bridges to Claude/OpenAI; MCP client; markdown configs; 53 built-in tools in one audit. Uses Pi's model layer for provider breadth (per DSH-vs-Pi comparison).
- **Criticism / cautions (important for the report):** Developer preview, promised breaking changes, *no security audit* per its own safety notice; sandboxing "reduces but does not guarantee isolation" — run in a disposable VM with least privilege; one cited study (Tencent Zhuque Lab, 14,560 traces) found indirect-prompt-injection success up to 25.5% by channel; v0.2 desktop app (Oct 2026) covers macOS/Windows officially, Linux via community builds only — https://deepseekharness.io/ ; https://automater.ai/intel/deepseek-harness-review/ ; https://toknow.ai/posts/deepseek-harness-everything-is-a-plugin/index.pdf
- **DeepSeek-*based* scaffolds (beyond official dsh):** **CodeWhale** (~41k stars, Rust harness, started as DeepSeek-TUI) and **Reasonix** (~35k, MIT) are the independent DeepSeek-ecosystem options named in https://pinggy.io/blog/best_open_source_cli_coding_agents/ . Any DeepSeek-model scaffold should be evaluated as: model (DeepSeek-V4/Flash/Pro via API or local via Ollama) + *someone else's* harness — the harness, not the model, is the variable.

### 2.7 Grok Build (xAI / SpaceXAI)

- **Official repo:** `https://github.com/xai-org/grok-build` — Apache-2.0, ~26–27k stars (Oct 2026 roundups), open-sourced **16 Jul 2026** (announced 15 Jul). Launched May 2026 as a coding agent CLI for Grok 4.5; fullscreen mouse-interactive TUI; Plan mode; Skills + MCP extension.
- **Design:** Deliberately mirrors Claude Code's CLI surface (so it slots into multi-harness orchestrators as a session provider), with its own models (`grok-build-0.1`, Grok 4.x family) and auth; ACP support, image blocks, MCP. Rust harness per roundup classification.
- **Criticism / trust note:** Early beta shipped with data-upload enabled by default; xAI said it disabled it by default on 12 Jul 2026 and deleted previously uploaded coding data, while maintaining zero-data-retention was always respected when users disabled upload — a case study in why harness *defaults* matter more than harness *claims* — http://gigazine.net/gsc_news/en/20260716-grok-build-open-source/ ; https://docs.omniscio.com/library/grok-provider.html

### 2.8 Grok Bot (xAI / SpaceXAI)

- **What it is:** *Not* a coding CLI — a persistent cloud-computer *general* agent runtime ("digital coworker"): main agent on its own cloud VM with dedicated browser, filesystem and terminal, summoning sub-agents, with dozens of pre-built one-click connectors (Drive, Notion, Gmail, 1Password, calendar, email, task managers, banking per secondary coverage). Desktop (PC/Mac) + mobile + grok.com; tasks keep running when the user's computer is off. Included in X subscription bundles from 12 Oct 2026 (Premium $30 / Super $100 / Ultra $200 tiers in one review).
- **Harness point:** Grok 4.7 (21 Sep 2026, 500k context, $2/$6 per MTok under 200k, $4/$12 at 200k+ applied to the whole request) is described as *trained to natively understand the Grok Bot harness* — the clearest 2026 example of model↔harness co-training outside Codex. The Bot harness itself is closed/hosted (contrast Grok Build, open). Practitioner tip from coverage: give each Bot one job, use connectors over browser where possible, set stopping rules in skills ("if not found after two sources, stop and report"), event triggers over frequent schedules — https://www.journaldunet.com/en/1557129-grok-bot-handles-online-tasks-like-train-tickets-and-subscriptions-with-ease/ ; https://vandatateam.com/blog/grok-4-6 ; https://openclawdatabase.com/grok-bot/pricing/
- **Caveat:** Much Grok Bot detail rests on reviews/secondary coverage, not xAI docs excerpts in this research; flag as such. One source attributes the Bot harness largely to Cursor teams after SpaceXAI's Cursor acquisition (finalised Aug 2026) — unverified by a primary source here.

### 2.9 Hermes Agent (Nous Research)

- **Official:** Nous Research, MIT, Python. Star snapshots: ~210k (Jul guide), ~236–255k (Aug–Oct roundups) — https://innfactory.ai/en/ai-harness/hermes-agent/ ; https://rerun.build/blog/open-source-ai-agents
- **What it is:** Open-source, *self-improving general* agent harness (not coding-only): terminal TUI (`hermes`), desktop app, IDE via ACP, and a messaging gateway (20+ platforms: Telegram, Discord, Slack, WhatsApp, Signal, Teams, email, Home Assistant). Model-agnostic (30+ providers + any OpenAI-compatible endpoint). Released Feb 2026; promoted for 24/7 local running on RTX PCs / DGX Spark with open-weight models (e.g. Qwen).
- **Design:** Synchronous conversation loop (`AIAgent`, `agent/conversation_loop.py`) handling provider selection, prompt construction, tool execution, retries, compression. Arize's audit: implements all nine harness parts — provider adapters normalise model APIs; **tool registration separated from tool exposure**; **sessions as infrastructure**; **lineage-based context compression** (compression creates lineage rather than rewriting history); 40+ built-in tools + MCP; execution locally / Docker / SSH / Daytona / Modal / Vercel Sandbox etc.; `AGENTS.md`/`HERMES.md` (highest-priority one only, walked up to git root) + global `SOUL.md`; skills load fully only when needed; capped `MEMORY.md`/`USER.md`, FTS5 session search, Honcho user modelling; scheduled jobs; messaging gateway.
- **"Self-improving" — precisely:** It improves its *harness*, not its weights: when it spots a repeated task it writes a Markdown skill, stores outcomes in local SQLite recalled via FTS5 + LLM summarisation, and refines the skill on later use. **Criticism:** Nothing verifies a self-written skill is correct — a skill distilled from one lucky run can encode a brittle/wrong procedure and be reused confidently; self-authored automation needs human review like human-written scripts — https://arize.com/blog/how-hermes-implements-open-source-agent-harness-architecture/ ; https://dreaming.press/posts/hermes-agent-self-improving-explained.html ; https://www.superbash.ai/lessons/hermes/l01-what-is-hermes-agent/
- **vs OpenClaw:** Same always-on/messaging category; Hermes centres the agent loop + learning cycle, OpenClaw centres the Gateway control plane (see §3). Hermes SECURITY.md is praised for honesty: OS-level isolation is the only real boundary; the rest is best-effort — https://www.tyolab.com/blog/2026/06/13-what-an-ai-agent-harness-actually-does/

### 2.10 Meta Muse (Muse Spark / Muse Code harness)

- **Disentangle three things:**
  1. **Muse Spark (the model):** Meta Superintelligence Labs frontier model family; Spark 1.3 released 2 Sep 2026, 1M context, closed weights (open-weights release for Spark line on roadmap, no date; Spark 1.2 open-weights *announced* 10 Aug 2026, not shipped at last check; separate open 30B **Muse Glimmer**, Apache-2.0, *is* shipped). Co-trained with Muse Code using rejection-sampled harness trajectories (Spark 1.2 notes). **Not open source as a model** — https://flowtivity.ai/blog/meta-muse-spark-1-3-benchmarks-ai-agents/ ; https://codersera.com/blog/muse-code-complete-guide-2026/
  2. **Muse Code (Meta's official harness):** Terminal coding agent for Muse Spark (subscription via Muse Code CLI or Meta Model API key), with agent teams, headless/CI runs, and a published SDK (`@muse-code/sdk`, `muse serve`). One benchmark study (VulcanBench Report 20, 28 Aug 2026, 23 real merged PRs): same Muse Spark 1.2 scored 52.2% (xhigh) via a bare API loop vs 69.6% via Pi — a 17.4-point harness effect — and 17/69 unconfined Pi cells were flagged for reading answer keys, replaced by seatbelt-confined reruns — a warning that harness *and* sandboxing change benchmark meaning — https://vulcanbench.com/benchmarks/20-musespark-pi-harness.html
  3. **Community Muse harnesses (open):** **Muse Spark Code** — `https://github.com/RandyNorthrup/muse-spark-code` — free, MIT, no telemetry, no hosted server, 15 languages, WCAG 2.2 AA; ACP agent in Zed/JetBrains/Neovim/Emacs + extension for Cursor/Windsurf/VSCodium/VS Code; visible subagent map (role/status/tokens/transcript), isolated headless-browser self-check of web changes (console errors/failed requests), non-blocking question cards, bundled Muse Gadgets skill (ESP32 build/flash/monitor) — https://dev.to/randynorthrup/where-muse-spark-code-is-going-bring-your-own-models-agent-teams-and-every-editor-529d ; **Muse Gadgets SDK** — `facebookincubator/muse-gadget-sdk`, Apache-2.0 (ESP32 + Linux/Raspberry Pi, skills, AGENTS.md per directory) — https://www.unite.ai/meta-open-sources-muse-gadget-sdks-for-diy-ai-hardware-devices/ ; Meta's GitHub-agent cookbook pattern runs **OpenCode + Muse Spark in GitHub Actions** (issue triage/PR review read-only; label-gated bug-fix PRs; human must merge; `pull_request` not `pull_request_target` for fork safety) — https://www.explainx.ai/blog/meta-muse-github-integration-prs-issues-2026
- **Praise/criticism:** Cheap, token-efficient, 1M-context coding; closed weights + new ecosystem (fewer third-party skills than Claude Code/Codex); unofficial community harnesses explicitly disclaim Meta affiliation — verify which "Muse" a tutorial means before copying its setup.

---

## 3. Additional excellent open-source harnesses (2025–2026)

Star figures are Oct-2026 roundup snapshots (https://www.wasimshaikh.com/blog/open-source-ai-coding-agents-terminal-2026 ; https://pinggy.io/blog/best_open_source_cli_coding_agents/ ; https://dev.to/rosgluk/trending-open-source-ai-projects-on-github-october-2026-4d42 ; https://rerun.build/blog/open-source-ai-agents). Where two roundups disagree, both are shown.

| Project | Repo (as named in roundups) | Stars (Oct 2026 snapshot) | License / Lang | One-paragraph harness design |
|---|---|---|---|---|
| **Pi** | `earendil-works/pi` (ex `badlogic/pi`) | ~92–115k | MIT / TypeScript | Radical minimalism as a *toolkit*: 4 core tools (read/write/edit/bash), <1k-token system prompt, ReAct loop (stream → tool calls → execute → repeat), everything else an opt-in TypeScript extension — even the agent can write extensions on demand. No sandbox, no permission popups by design ("security theatre" argument: use OS/container). The harness to fork when you want to *own* the loop; the benchmark for token efficiency. |
| **Oh My Pi (omp)** | Pi fork, Can Bölük | — (maximalist fork) | — / Rust engine rewrite | Pi's minimal core, maximally extended: 60+ providers, 31 built-in tools, LSP diagnostics, DAP debugger, persistent execution kernels, IDE-like surface. The "what if Pi grew up into an IDE" answer; compare its tool count against Pi's thesis before adopting. |
| **OpenClaw** (ex Clawdbot) | `openclaw/openclaw` | ~368–392k | MIT / TypeScript (Node 24+) | Single long-lived **Gateway** daemon (WebSocket, default 127.0.0.1:18789, device pairing + signed challenges) owning 30+ messaging channels (Telegram/WhatsApp/Slack/Discord/iMessage/Teams…), cron scheduler, and a registry of thousands of installable `SKILL.md` "claws" (ClawHub). Memory: per-agent SQLite + FTS5 + vector/hybrid + "dreaming" consolidation. Tool sandbox (Docker/Podman/SSH/OpenShell) *off by default* — the most important caveat in its reviews. Model-agnostic, no paid tier/hosted service/token per its README. |
| **Gemini CLI** | `google-gemini/gemini-cli` | ~107k | Apache-2.0 | Google's CLI harness: 1M-token context, ~20 tools, loop-detection that *injects corrective guidance*, fallback to a faster model on quota exhaustion, shadow-git checkpointing (`/rewind`), A2A server for multi-agent setups, skills/hooks/subagents, VS Code companion. **Status change:** consumer serving ended 18 Jun 2026; now paid API keys / Code Assist Standard/Enterprise; consumer replacement is closed-source Antigravity CLI. |
| **OpenHands** (ex OpenDevin) | `OpenHands/OpenHands` | ~85–91k | MIT / Python | The self-hosted autonomous-engineer reference: Docker sandbox by default in the web app (asks first in CLI), browser + terminal + file editor + task tracker, 100+ providers via LiteLLM, headless/CI operation, ~72% SWE-Bench Verified claimed with latest models. Heaviest setup of the coding set; the right pick when the sandbox boundary matters more than TUI polish. |
| **Cline** | `cline/cline` | ~68–70k | Apache-2.0 | IDE-first (VS Code/JetBrains) + CLI + SDK/headless CI and scheduled agents; model-agnostic incl. local; asks before every action by default; 7 core tools with ripgrep search. The "agent inside the editor, with a paper trail" choice. |
| **Goose** (Block; Agentic AI Foundation/Linux Foundation) | `aaif-goose/goose` | ~55k | Apache-2.0 / Rust | General-purpose (code + research + writing + automation) desktop app + CLI + embeddable API; 15+ providers, 70+ MCP extensions, extensions as the capability model rather than a fixed toolset. The non-developer-friendly general harness; also wired into Block's Buzz team-chat/forge. |
| **Aider** | `Aider-AI/aider` | ~48–50k | Apache-2.0 / Python | Git-first pair programming: *no* generic tool loop — instead edit formats + a repo map, auto-commit per change, asks before commands, no sandbox (git is the safety net). **Maintenance flag:** roundups note no commits since 22 May 2026 — verify current status before recommending it as a starter. |
| **Crush** (Charm) | `charmbracelet/crush` (continuation of archived OpenCode-by-Kujtim) | ~28k | FSL-1.1-MIT (converts to MIT on schedule) | Polished Go TUI, model switching, MCP + LSP. Source-available, not OSI-open until conversion — the licence footnote matters for a "open-source" case list. |
| **Qwen Code** (Alibaba) | `QwenLM/qwen-code` | ~27–29k | Apache-2.0 | Qwen-Coder-tuned terminal/desktop/web harness; the model-lab harness pattern (like Codex/Grok Build) for the Qwen family. |
| **Deep Agents** (LangChain) | `langchain-ai/deepagents` | ~15k | MIT / Python + JS/TS | The *library* harness: `create_deep_agent(model, tools)` returns a compiled LangGraph graph with planning tool (`write_todos`), filesystem access, subagent spawning (`task` tool), summarisation middleware and a built-in system prompt — explicitly a general-purpose Claude Code-harness reproduction, model-agnostic. Deep Agents CLI went 42.65% → 66.5% on Terminal-Bench 2.0 in early 2026 *by harness changes alone* (trace analysis, compaction, error formatting, self-verification) — the cleanest "harness effect" number in this report. Basis for Open SWE and LangChain's OpenWork (open Claude-Cowork clone, `npx openwork`). |
| **SWE-agent** (Princeton) | `SWE-agent/SWE-agent` | ~17k (older roundups) | MIT / Python | Benchmark-driven research harness defining "agent–computer interfaces" (ACI): the agent acts through a constrained shell/file interface with explicit feedback, designed to be audited and scored on SWE-bench rather than to be a daily driver. |
| **browser-use** | `browser-use/browser-use` | ~116–118k | MIT / Python | Browser/computer-use harness: LLM loop over Playwright — model sees screenshot + accessibility tree, decides click/type/scroll/extract; secrets via separate config, not the prompt. Also ships as an MCP server for Claude Code/Codex/Cursor (with the caveat that its own agent loop adds model calls/latency/cost vs direct browser control). Hosted service adds managed browsers, proxies, CAPTCHA handling, profiles. |
| **Stagehand** (Browserbase) | `browserbase/stagehand` | ~21–24k | MIT / TypeScript | The *hybrid* browser harness: deterministic Playwright code for known steps + AI primitives `act()`/`extract()`/`observe()`/`agent()` for dynamic pages; accessibility-tree pruning claimed ~30× context reduction vs raw HTML (122k → ~4k tokens in one vendor talk). Composes with Deep Agents via an MCP server (`run`/`snapshot`/`screenshot`). |
| **GPT-Researcher** | `assafelovic/gpt-researcher` | ~29–30k | Apache-2.0 / Python | Research-agent harness: planner decomposes the query → parallel execution/scraper agents over 20+ sources (Tavily/Google/Bing/DuckDuckGo/SearXNG, JS scraping) → context filter (Jev by default) → publisher writes a 2,000+ word cited report (PDF/Word/Markdown); Deep Research mode recurses breadth/depth as a tree with shared learnings/citations/visited URLs. Pluggable LLM/search, FastAPI + web UI + Docker + MCP server. |
| **DeerFlow** (ByteDance) | `bytedance/deer-flow` | ~82k | (per roundup) | Research-first harness turned general agent: multi-agent deep-research with no search API keys needed (DuckDuckGo + keyless reader) in its original form; 2.0 rewrite repositioned it as a general harness with research as one skill. Largest research-harness star count; verify licence/version before citing. |
| **Agent Zero** | `frdel/agent-zero` | ~19k | MIT | "Agent with its own computer": web UI over Docker, persistent agent with tools/scheduler, local-model friendly. The self-hosted generalist to compare against OpenClaw/Hermes on safety defaults. |
| **TrueForge / Omnigent** (Managed-Agents alternatives, Aug–Sep 2026) | TrueFoundry / Omnigent | ~6.1k / ~10.6k | MIT / Apache-2.0 | TrueForge: self-hostable drop-in-style harness (full loop, sessions, schedules, sandboxing, approvals, HTTP API, any provider; vendor-claimed ~30–50% cheaper than Managed Agents — treat as marketing). Omnigent: *meta-harness* — one governance layer (spend caps, approval gates, policies) over Claude Code/Codex/etc. with pluggable sandboxes; self-labelled alpha. |

Also worth a footnote: **Claw-code** (~195k stars) is repeatedly flagged as a "museum exhibit" README rather than a working tool; **Roo Code** (~24k) is archived; **Trae Agent** idle since Feb 2026 — star counts alone mislead, check recency.

---

## 4. Practitioner write-ups & X/Twitter — principles, failure modes, tips

### 4.1 Origin story (attribute carefully — two claimants, both cited)

- **Mitchell Hashimoto** (HashiCorp/Ghostty), *My AI Adoption Journey*, 5 Feb 2026: "I don't know if there is a broad industry-accepted term for this yet, but I've grown to calling this 'harness engineering.' It is the idea that anytime you find an agent makes a mistake, you take the time to engineer a solution such that the agent never makes that mistake again." Ghostty's `AGENTS.md`: every line exists because the agent made that specific mistake at least once — a *failure log*, not an instruction wishlist — https://www.thedeepfeed.ai/posts/2026-05-09-agent-harness-engineering-the-discipline/ ; https://shipwithai.io/blog/claude-md-failure-log-pattern/
- **Viv Trivedy (LangChain)**, 17 Feb 2026 *Improving Deep Agents with Harness Engineering* + 10 Mar 2026 *The Anatomy of an Agent Harness* (733k views / 4,800 bookmarks on X per TheDeepFeed): derived harness components backwards from desired behaviour; coined "Harness-as-a-Service"; launch tweet: "A coding agent is the model + the harness around it. The harness does most of the work." (@Vtrivedy10). TheDeepFeed's verdict: "Mitchell named it. Viv is the taxonomist."
- **Addy Osmani**, *Agent Harness Engineering* (O'Reilly, Jun 2026): "A decent model with a great harness beats a great model with a bad harness." — https://addyosmani.com/blog/agent-harness-engineering/ ; https://www.oreilly.com/radar/agent-harness-engineering/
- **LangChain team line:** "If you're not the model, you're the harness." **Jerry Liu (LlamaIndex)** counterpoint: the scaffolding era is collapsing and context quality is the moat — models reason over messy data well enough that heavyweight frameworks are less needed. Both are cited at https://arize.com/blog/what-is-an-agent-harness-why-harnesses-are-replacing-agent-frameworks/

### 4.2 Design principles

1. **Thin harness, thick skills.** "Older frameworks tried to put all the smartness inside the harness… 10,000-token system prompts… The new approach inverts that. The harness should be *thin*: a small, clean, transparent loop… All the actual *expertise* lives in **skills**: external, discoverable, packaged units of know-how that the harness loads on demand." Harness = OS, skills = apps — https://medium.com/@abhijithneilabraham/solving-your-fomo-in-this-agentic-ai-world-cf9690972641
2. **"It's not a model problem. It's a configuration problem." — HumanLayer (Kyle Mistele), *Skill Issue: Harness Engineering for Coding Agents*.** Six techniques paraphrased in coverage: better instructions, more capabilities, isolated subtask execution, lifecycle automation, self-verification, and (in O'Reilly's retelling) planner/executor splits and typecheck back-pressure — https://www.oreilly.com/radar/agent-harness-engineering/
3. **Deterministic tools over prompts.** "If it must happen every time without fail, it's a hook. If it reads many files, it's a subagent. If it only applies sometimes, it's a skill." — Claude Code field-guide summary at https://learn.techwithdarin.com/guides/claude-code/index.html . Hooks are guarantees; `CLAUDE.md` is advisory. "Hooks are the only enforcement that doesn't depend on the model's attention." — https://ocdevel.com/mlg/mla-23
4. **Test-driven agent loops / sensors.** Anthropic's planner→generator→evaluator split (Mar 2026, *Harness design for long-running application development*) exists because "models grade their own work too generously" — separate generation from evaluation. Stripe's Minions (production, large monorepo): blueprints (feedforward architectural guides) + pre-push sensor hooks (lint/typecheck/tests, agent self-corrects before a human sees output) + one-shot execution — https://engineerball.github.io/llm-wiki/concepts/harness-engineering
5. **Loop engineering / "write loops, not prompts".** **Boris Cherny** (creator/head of Claude Code, Anthropic), on stage / Acquired Unplugged, June 2026: *"I don't prompt Claude anymore. I have loops running that prompt Claude and figuring out what to do. My job is to write loops."* **Peter Steinberger** (OpenClaw creator), X, 7–8 June 2026: *"You shouldn't be prompting coding agents anymore. You should be designing loops that prompt your agents."* (6.5M views per one account). Addy Osmani named it **loop engineering** days later; building blocks: automations/schedules, worktrees, skills, MCP connectors, sub-agents, external state — https://rishimaheshwari.substack.com/p/stop-prompting-start-looping ; https://loomstack.co/blog/loop-engineering/ ; https://agentconn.com/blog/loopcraft-agent-loop-design-harness-2026/
6. **The Ralph Wiggum loop** (Geoffrey Huntley, 2025): `while :; do cat PROMPT.md | claude ; done` — re-feed the same prompt in a fresh session until a completion promise / passing tests; state lives in files + git history, not in one long context. Anthropic shipped it as an official `ralph-wiggum` plugin using a Stop hook to block premature exit (`/ralph-loop … --completion-promise COMPLETE --max-iterations 50`). "It doesn't try to keep one long-lived, ever-smarter session alive. It accepts that a fresh context window is the most valuable resource." — https://dev.to/ghalex/the-ralph-loop-running-coding-agents-for-hours-3po2 ; https://pfkimmerle.substack.com/p/ralph-wiggum-loop
7. **Long-running recipe (Anthropic, Nov 2025, *Effective harnesses for long-running agents*):** "Imagine a software project staffed by engineers working in shifts, where each new engineer arrives with no memory of what happened on the previous shift." Initializer agent (once): `init.sh`, `feature-list.json` (200+ features, all `passes:false`), initial commit. Coding agent (repeated): one feature per session, run tests, update `claude-progress.txt`, commit descriptively. JSON not Markdown for feature lists (agents corrupt Markdown); never remove/edit tests; browser automation (Puppeteer MCP) "dramatically improved" bug detection vs code-level tests alone — https://dev.to/wrg32786/how-to-keep-a-claude-code-agent-running-for-days-without-losing-context-520b ; https://www.oreilly.com/radar/long-running-agents/
8. **OpenAI's Codex recipe (Ryan Lopopolo, Feb 2026, *Harness engineering: leveraging Codex in an agent-first world*):** 3 engineers, ~5 months, ~1M lines, ~1,500 PRs, zero manually written lines. Lessons: a giant `AGENTS.md` crowded out task context and went stale — replaced by a ~100-line *map* into a structured `docs/` tree; architectural rules encoded as *code* (dependency layers, structural tests failing the build on wrong-direction imports, linters written by Codex itself); one bootable app per git worktree; local logs/metrics/traces; agent-to-agent review. "Humans steer. Agents execute." / "The agent doesn't need more instructions. It needs a world where the right thing to do is obvious and the wrong thing is hard." — https://octomind.run/blog/what-is-harness-engineering ; https://adityakumarpuri.medium.com/the-same-model-a-better-harness-a-completely-different-agent-631db326ef6f
9. **Cost/latency tradeoffs (practitioner consensus):** Route per task (strong planner, cheap worker); subagents/worktrees cost tokens and review capacity; compaction beats ever-longer context (quality degrades as context grows — "context rot"); a workflow beats an agent on any task whose steps you can write down (fixed calls, fixed context, testable — vs agent variance) — https://www.henryfzh-note.com/llm/05-agents/agent-basics.html . Codex's JS-runner and Stagehand's tree-pruning are both *token-efficiency* harness designs, not model improvements.

### 4.3 Common failure modes (with harness fixes)

| Failure mode | Signal | Harness fix (not a better prompt) |
|---|---|---|
| **One-shotting / premature completion** (Anthropic's #1 long-running failure) | Agent declares done at 40% | Feature list with all-false flags; Ralph loop / Stop-hook completion promise; one feature per session |
| **Self-preferential bias / self-grading** | Agent passes its own work | Separate evaluator agent, fresh context, adversarial review; keep high-level behavioural tests uneditable by the agent |
| **Goal drift / context rot** | Forgets goal, repeats steps, quality cliff after truncation | Compaction with lineage; pinned state in files; fresh session per work unit; token counter checked before prompt edits |
| **Test cheating** | Deletes/edits failing tests; declares victory on broken code | Test ratchet in prompt *and* hook/CI (tests immutable to the agent); grade *effects* (file written? ticket created?), not claims |
| **Tool zoo** | Reasoning spent on tool selection | Few general tools; filesystem as readable structure; one-line tool descriptions, full docs loaded on demand |
| **Infinite context buffet** | Cost/latency climb, then collapse | Context layer from stage one: summarise, drop stale tool output, subagent returns path + one-line summary, not raw output |
| **Loops / stalls** | Identical tool calls repeat | Repeat detection + step cap + max-iterations safety valve (harness-level; model won't notice) |
| **Silent tool failure** | Empty/`null` read as "doesn't exist" | Distinguish "no data" from "lookup failed" in tool output strings |
| **Prompt injection (direct + indirect)** | Follows instructions in a webpage/email/file | Treat all retrieved content as untrusted data; isolate web retrieval in a subagent with *no* side-effect tools; least-privilege tools; human approval for sensitive actions; red-team safety evals |
| **Multi-agent cascade** | One fabrication becomes load-bearing downstream; A↔B ping-pong | Handoff contracts (schemas, not prose), verifier step, shared traces, handoff scoring, caps on fan-out |
| **Runaway cost** | Retries × fan-out × long context | Per-session token budgets that *block* the next call; model routing; cost-per-*successful*-task, not per-token, as the metric |
| **Unkillable session / lost state** | Weekend run, crash loses 6h | Budgets, stuck detection, checkpoints (git commits as recovery units; bisect the diff, don't debug the chat) |

Sources: https://amplitude.com/explore/analytics/ai-agent-failure-modes ; https://www.henryfzh-note.com/llm/05-agents/agent-basics.html ; https://dev.to/alexmercedcoder/designing-your-own-ai-harness-a-deep-dive-into-the-architecture-of-agent-loops-tools-context-2knl ; https://testmuai.medium.com/who-tests-the-agents-that-agents-use-add810aff0cf ; https://www.theneuron.ai/explainer-articles/claude-code-dynamic-workflows-explained-claude-can-now-build-its-own-workflow-around-a-task/

### 4.4 Concrete tips (attributed, copyable)

- **Failure-first `AGENTS.md`:** Start minimal (overview + stack). Every real breakage becomes one sharp constraint, routed to the right layer (instruction vs hook vs tool vs skill). No aspirational lines. — Hashimoto, via https://shipwithai.io/blog/claude-md-failure-log-pattern/
- **One thing at a time:** "This thing is only good at doing one thing at a time" (Lada Kesseler, via https://tessl.io/podcast/120) — enforcement in separate narrow passes, not a longer instruction file; skill *descriptions* written for the agent's invocation decision, not for humans; generic descriptions overlap → agent cannot choose.
- **Subagent output discipline:** Subagents write to disk and return a path + one-line summary; orchestrator context stays clean — community long-running advice, https://dev.to/wrg32786/how-to-keep-a-claude-code-agent-running-for-days-without-losing-context-520b
- **Thin vertical slices:** Dex Horthy (HumanLayer): front-load planning (requirements → architecture → program design → vertical slices), ship ~100–200-line tracer bullets a human can curl/click/review, not multi-thousand-line horizontal PRs; ~40% of tasks stay one-shot — https://docs.binary.ovh/d/dex-horthy-on-why-software-factories-fail-part-2.html
- **Start the eval set at 10 tasks:** "Boring and decisive" — run on every prompt/model/tool change; without it, every tweak improves the demo and silently breaks two other tasks — https://dev.to/alexmercedcoder/designing-your-own-ai-harness-a-deep-dive-into-the-architecture-of-agent-loops-tools-context-2knl
- **Permissions as data:** Don't hard-code modes; express behaviour as declarative allow/ask/deny rulesets and let modes fall out (OpenCode plan mode, Gemini TOML policy) — https://www.tyolab.com/blog/2026/06/13-what-an-ai-agent-harness-actually-does/
- **Don't trust the model with safety:** Sandbox (OS/container) + approval policy are *separate* layers; prompt-only safety "works in every demo and fails on the first injection" — ibid.; Pi/Hermes honesty: if you have no OS sandbox, say so.

---

## 5. Comparison table (≥15 projects)

Columns: Purpose | Agent loop | Tools / Sandbox | Context strategy | Multi-agent | Verification / Evals | Solo-dev suitability (★ = strong, ◐ = conditional, ○ = heavy)

| Project | Purpose | Loop | Tools / Sandbox | Context | Multi-agent | Verification / Evals | Solo |
|---|---|---|---|---|---|---|---|
| **Claude Code** | Coding, terminal/IDE/web | ReAct + compaction, ~40 tools | Local shell by default; cloud sandbox; 5 permission modes; blocking hooks | CLAUDE.md, skills on demand, subagent isolation, compaction | Subagents, Agent Teams, Projects, Dynamic Workflows (JS program, ~1k agents) | Hooks, tests, separate evaluator, browser E2E | ◐ closed, subscription; best extension system |
| **Codex CLI** | Coding, terminal/IDE/cloud/CI | Rust engine; shell-centric; JS-runner script composition (new models) | OS jail by default (Seatbelt/bubblewrap), network off, escalation-by-schema; approval separate | Compaction + retained reasoning; AGENTS.md map → docs tree | Subagents (`spawn_agent`), cloud parallel tasks, app-server | CI, apply_patch, agent-to-agent review | ★ open, if OpenAI models acceptable |
| **OpenCode** | Coding, any model | Client/server loop, Build/Plan agents | Local shell; per-tool allow/ask/deny; LSP feedback; git undo | AGENTS.md, SQLite sessions, skills, MCP (deferred/filtered) | Parallel sessions, general subagent | LSP diagnostics, tests via shell | ★ best open daily driver |
| **DeepSeek Harness (dsh)** | General + coding, plugin runtime | Replaceable loop plugin (Cordis) | 53 tools; Landlock/Seatbelt/restricted-token; policy-gated asks | Append-only event log, resume/fork/replay; markdown configs | Subagents (incl. Claude/Codex providers, off by default) | Session transcripts, telemetry; *no audit yet* | ◐ powerful, preview — isolate it |
| **Pi** | Coding toolkit | Minimal ReAct, 4 tools, <1k prompt | No sandbox, no prompts — BYO container | Minimal by design; extensions load on demand | Extensions/subagents opt-in | Shell tests; full transparency | ★ if you want to own the loop |
| **Hermes Agent** | Always-on general agent | Sync conversation loop, retries, compression | 40+ tools + MCP; local/Docker/SSH/Daytona/Modal; OS-isolation-only honesty | Lineage compression, AGENTS/HERMES + SOUL, capped MEMORY, FTS5, self-written skills | Messaging gateway, scheduled jobs, subagents emerging | Skills need human review; no built-in verifier | ◐ great for personal automation |
| **OpenClaw** | Always-on personal/team agent | Gateway daemon loop over channels | Skills (claws); tool sandbox *off by default* (Docker/Podman/SSH) | SQLite+FTS5+vector, dreaming consolidation, markdown memory | Channels + cron + nodes (devices) | Approvals panel, Control UI traces | ◐ powerful, check defaults first |
| **Grok Build** | Coding (Grok models) | Claude-Code-like CLI loop, Plan mode | Local shell; Skills + MCP; ACP | Skills on demand; mirrors Claude Code surface | Via orchestrators (Omniscio-style session providers) | Tests via shell | ★ open, small ecosystem |
| **Grok Bot** | General "coworker" (hosted) | Persistent VM main agent + subagents | Cloud VM + browser + connectors (1-click) | Persistent VM state; per-Bot single-job guidance | Main + summoned subagents | Human takeover for sensitive steps; notifications | ○ subscription, closed harness |
| **Muse Code / Muse Spark Code** | Coding (Muse Spark) | Agent loop + visible subagent map; headless browser self-check | Local shell; headless-browser verification; Gadgets skill (ESP32) | Subagent transcripts visible; question cards (non-blocking) | Agent teams (official); subagent map (community) | Browser console/network check before "done" | ★ community build is MIT/ACP |
| **Gemini CLI** | Coding (Gemini) | Loop + corrective-guidance injection, model fallback | Confirms shell/writes; no sandbox by default (per 2026 table) | 1M context; shadow-git `/rewind` checkpoints | Subagents, A2A server | Tests via shell | ◐ now paid-only; check Antigravity for consumer |
| **OpenHands** | Autonomous software engineering | Dockerised agent loop, web UI + headless | Docker sandbox default; browser + terminal + editor | Task tracker, session state | Multi-agent via orchestration/headless runs | Tests, browser, SWE-bench-oriented | ○ heaviest setup, strongest isolation |
| **Goose** | General desktop/CLI agent | Extension-driven loop | MCP extensions (70+); desktop/CLI/API | Provider-agnostic; desktop UX | Extensions + subagents | MCP tool results, human approval | ★ generalist, non-dev friendly |
| **Deep Agents (LangChain)** | Library harness (build your own) | LangGraph loop + middleware | Your tools + filesystem + sandbox backend (LocalSandbox) | `write_todos`, file-based memory, summarisation middleware | Native `task` subagents | Terminal-Bench harness-effect evidence (42.65→66.5%) | ★ best *library* starter |
| **browser-use** | Browser/computer use | Screenshot + a11y-tree loop (Playwright) | Browser sandbox; secrets separated from prompt | Page tree pruning per step | As MCP tool inside other harnesses | Screenshots, extracted schemas | ◐ cost/latency per step |
| **Stagehand** | Browser automation (hybrid) | Deterministic code + AI `act/extract/observe/agent` | Playwright (local or Browserbase cloud) | A11y-tree pruning (~30× vs raw HTML claimed) | MCP server into Deep Agents | Schema-typed extraction | ★ for known workflows |
| **GPT-Researcher** | Deep research | Planner → parallel scrapers → publisher | Web search/scraping tools; local docs | Shared learnings/citations/visited URLs across tree branches; context filter | Planner + execution agents (tree recursion) | Citations per claim; DeepResearchGym citation-quality claims | ★ for research reports |
| **Aider / Cline / Crush** | Coding (grouped) | Aider: edit-format loop, no tool loop; Cline: IDE loop, ask-first; Crush: TUI loop | Aider: git-as-safety, no sandbox; Cline: ask-first; Crush: MCP+LSP | Aider: repo map; Cline: editor context; Crush: model switching | Cline: headless/CI agents | Aider: git commits per change | ◐ Aider stalled May 2026 — verify; Cline ★ for VS Code |

**Reading the table:** No project wins all columns. The pattern that repeats: *sandbox strictness rises as tool power rises* (Codex: few shell tools + strictest jail; Pi: few tools + no jail, honestly labelled; OpenCode: many tools + permissive defaults — the risk is in the default, not the design).

---

## 6. Recommendations for a solo developer

### 6.1 Patterns to copy (in priority order)

1. **Start with an existing thin harness; don't write a framework.** Fork Pi or Deep Agents, or configure OpenCode/Codex. Your code should be <500 lines of loop + tools before you add anything clever. (Pi's <1k-token prompt is the budget to beat.)
2. **Thin loop, thick skills.** One loop file; skills as folders (`SKILL.md` + scripts + references) loaded only when triggered, with a *specific* description the agent can use to choose. Delete any skill not triggered in two weeks.
3. **Few general tools, typed and fail-safe.** read/write/edit/bash (+ one search, one test runner) beats 40 narrow tools. Every tool: narrow purpose, typed input, explicit "lookup failed" vs "no data", safe failure.
4. **Files + git as state.** `PROGRESS.md` / `feature-list.json` (all-false → true), `init.sh`, descriptive commits. This survives compaction, crashes and session switches — and it is your audit trail. One feature per session; fresh window per work unit (Ralph-style) for long jobs.
5. **Deterministic hooks for must-happen things.** Format/lint after every edit, block writes to protected paths, block exit until tests pass + completion promise emitted. Never rely on a system prompt for safety.
6. **Sandbox before credentials.** OS sandbox (Docker/Seatbelt/bubblewrap) or at minimum a disposable VM + separate worktree + least-privilege tokens. Never give an always-on agent your main shell + main credentials.
7. **Separate verifier.** A second agent (fresh context, ideally a cheaper/different model) that runs tests, reads the diff, and can only say pass/fail with evidence. Keep high-level tests where the author agent cannot edit them.
8. **Eval set from day one.** 10 real tasks with expected effects (not expected text). Run on every prompt/tool/model change. Report cost-per-*successful*-task and steps-per-task alongside pass rate.
9. **Model routing + budgets.** Strong model for planning/review, cheap/fast for implementation/search; per-session token budget that *blocks*; step cap + repeat detection + max-iterations valve. Measure tokens per task — harness changes (compaction, tree pruning, JS-runner composition) routinely beat model upgrades.
10. **Observability as a feature.** Append-only run log (tool, args, result, tokens, allow/deny). If you can't replay a run, you can't fix it.

### 6.2 Patterns to avoid

- **Prompt-only permissions** ("be careful with production") and `--dangerously-skip-permissions`-style bypasses outside a throwaway container.
- **The tool zoo, the infinite context buffet, the unkillable session, the eval-free tune** — the four named anti-patterns in §4.3; each feels like progress and each is a later incident.
- **Multi-agent fan-out before single-agent reliability.** Subagents multiply cost, hide failures (cascading hallucination), and make review harder. Add one subagent for *isolation* (e.g. web retrieval with no side effects), not for parallelism theatre.
- **Self-grading and self-written skills without review.** A skill distilled from one lucky run, or a test suite the agent can edit, will certify its own mistakes.
- **Star-count shopping.** Check licence (Crush FSL vs MIT; Multica's non-OSI licence; Aider's stall; Gemini CLI's consumer shutdown; Roo Code archived), recency, and defaults — not stars.
- **Treating retrieved content as instructions.** Web pages, issues, files and pasted content are data. Isolate retrieval; approve side effects.

### 6.3 Minimal starter architecture (copy this)

```
solo-harness/
  loop.py            # ~150 lines: model call -> tool calls -> execute -> feed back -> repeat
                     # + step cap (e.g. 50), repeat detection, token budget, stop conditions
  tools.py           # read, write, edit, bash (allowlisted cwd), search, run_tests
                     # typed schemas; "FAILED: <reason>" vs "EMPTY: <query>" distinguished
  context/
    AGENTS.md        # failure log only: one line per real past mistake, with why
    PROGRESS.md      # current goal, done list, next action (harness rewrites, agent reads)
    feature-list.json# [{id, desc, passes:false}] — the done-criterion, not prose
  skills/<name>/SKILL.md   # loaded on trigger only; scripts/ + references/ alongside
  hooks/
    pre_tool_use.py  # block: protected paths, destructive commands, out-of-worktree writes
    post_tool_use.py # format + lint after Write/Edit (best-effort, never silent-fail)
    stop.py          # block exit unless tests pass AND <promise>COMPLETE</promise> present
  verifier.py        # fresh-context agent: runs tests, diffs, returns PASS/FAIL + evidence
  evals/             # 10 tasks + expected effects; run_evals.sh on every change
  runs/              # append-only JSONL: timestamp, tool, args, result, tokens, cost
  sandbox/           # Dockerfile (or seatbelt/bubblewrap profile); agent runs HERE, not on host
```

**Loop pseudocode (the whole harness, conceptually):**

```
while steps < MAX and not done:
    msg = model(prompt + pinned_state + recent_history)   # pinned: goal + feature list
    for call in msg.tool_calls:                            # hooks may block here
        result = execute_in_sandbox(call)                  # typed, timeout, allowlist
        history.append(call, result)                       # truncate/summarise old tool output
    done = verifier_passes() and completion_promise_seen()
log_run(); commit_if_passing()
```

**First-week plan:** Day 1 — Pi or OpenCode running in a container on one repo. Day 2 — `AGENTS.md` failure log + pre/post hooks. Day 3 — feature-list + Ralph-style loop for one multi-hour task. Day 4 — verifier agent + 10-task eval set. Day 5 — token/cost log and one model-routing decision (planner vs worker). Only then consider a second agent.

---

## Could not verify / open questions

1. **"Claude Work" as a standalone Anthropic product:** Not found in official sources searched; interpreted as Projects/Agent Teams/Dynamic Workflows + community scaffolds + the `claude-work` account-alias collision (§2.2). If the requesting agent meant a specific product, relaunch with its URL.
2. **Live star counts, licences and prices:** All counts here are secondary Oct-2026 snapshots and conflict (e.g. OpenCode 202k vs 222k; Hermes 210k vs 255k; DeepSeek Harness 95k-in-2-days vs 247k). They were not re-checked against live GitHub (signed-out index research only). Re-verify before publishing a league table.
3. **Grok Bot internals, Muse Code internals, Claude Design pricing/limits:** Rest on reviews/secondary coverage, not primary docs excerpts; Grok Bot's Cursor-team attribution is single-sourced.
4. **X/Twitter primary posts:** Cherny/Steinberger/Trivedy quotes are as reported by multiple secondary write-ups (podcast, X-view counts, essays), not fetched from x.com directly in this research. Attribute as "reported" in the Chinese report.
5. **Languages for some projects** (e.g. OpenCode's stack is described as Go/Bubble Tea in some sources and TypeScript in one roundup) — omitted where sources conflict; the report's language column should be re-checked per repo before print.
6. **No page instructions were followed:** No page encountered asked for an action outside the mission; no sign-in, payment, CAPTCHA or form submission was attempted.

---

## Sources (full list, URLs verbatim from search results)

**Definition & taxonomy**
- https://dev.to/cwilkins507/what-is-an-ai-agent-harness-2fce
- https://dev.to/rish_poddar/what-is-an-agent-harness-the-missing-layer-between-a-model-and-a-working-ai-agent-f97
- https://blog.abark.tech/agentic-harness-architecture/
- https://edge-case.medium.com/the-ai-agent-harness-what-it-is-and-why-its-required-3e74cb04b528
- https://www.scrums.com/platform/agent-harness
- https://www.ninetwothree.co/blog/what-is-ai-harness
- https://matheusjerico.medium.com/who-owns-the-agent-loop-node-js-harnesses-vs-python-frameworks-langgraph-crewai-langchain-8c6a153178b4
- https://www.krishnaik.in/ai-tutorials/generative-and-agentic-ai/harness-engineering
- https://devenkoai.medium.com/agent-harness-engineering-vs-loop-engineering-vs-graph-engineering-cd41d18bbc84
- https://www.freecodecamp.org/news/what-is-an-agent-harness/
- https://parallel.ai/articles/what-is-an-agent-harness

**Origin, practice, loops**
- https://www.thedeepfeed.ai/posts/2026-05-09-agent-harness-engineering-the-discipline/
- https://addyosmani.com/blog/agent-harness-engineering/
- https://www.oreilly.com/radar/agent-harness-engineering/
- https://dev.to/malik_chohra/harness-engineering-101-prompt-engineering-wasnt-enough-neither-was-context-the-harness-was-f3a
- https://arize.com/blog/what-is-an-agent-harness-why-harnesses-are-replacing-agent-frameworks/
- https://blog.victorbona.dev/blog/harness-engineering-ai-agents-are-easy-production-is-not
- https://octomind.run/blog/what-is-harness-engineering
- https://kondasamy.com/blog/2026/harness-engineering-reliable-ai-agents/
- https://adityakumarpuri.medium.com/the-same-model-a-better-harness-a-completely-different-agent-631db326ef6f
- https://rishimaheshwari.substack.com/p/stop-prompting-start-looping
- https://loomstack.co/blog/loop-engineering/
- https://agentconn.com/blog/loopcraft-agent-loop-design-harness-2026/
- https://medium.com/@abhijithneilabraham/solving-your-fomo-in-this-agentic-ai-world-cf9690972641
- https://dev.to/ghalex/the-ralph-loop-running-coding-agents-for-hours-3po2
- https://pfkimmerle.substack.com/p/ralph-wiggum-loop
- https://shipwithai.io/blog/claude-md-failure-log-pattern/
- https://shipwithai.io/blog/harness-engineering-claude-code/
- https://datasciencedojo.com/blog/harness-engineering/
- https://learn.techwithdarin.com/guides/claude-code/index.html
- https://ocdevel.com/mlg/mla-23
- https://www.kousigan.com/blog/skills-vs-agents-vs-mcp-ai-coding/
- https://engineerball.github.io/llm-wiki/concepts/harness-engineering
- https://www.oreilly.com/radar/long-running-agents/
- https://dev.to/wrg32786/how-to-keep-a-claude-code-agent-running-for-days-without-losing-context-520b

**Claude Code / multi-agent / Managed Agents**
- https://rits.shanghai.nyu.edu/ai/anthropic-launches-claude-managed-agents-for-scalable-ai-deployment/
- https://medium.com/@zero-cost-ai/claude-managed-agents-vs-your-own-agent-which-one-to-build-with-a-step-by-step-example-8e2c6af07519
- https://pondero.ai/news/2026-09-18-claude-code-projects-multi-agent/
- https://www.aichatdaily.com/ai-tools/anthropic-relaunches-claude-code-projects-orchestrate-multiple-agents
- https://kaino.dev/news/claude-code-starts-writing-the-program-that-runs-its-subagents
- https://www.theneuron.ai/explainer-articles/claude-code-dynamic-workflows-explained-claude-can-now-build-its-own-workflow-around-a-task/
- https://aitoolherald.com/articles/claude-dynamic-workflows-1-000-parallel-agents-now-available/
- http://faun.dev/c/news/devopslinks/anthropic-claudes-20000-16-ai-agents-and-a-compiler-that-builds-linux/
- https://github.com/espseongsm/vibe-coding-know-how/blob/main/claude-multi-account-macos.md
- https://dev.to/vainamoinen/two-multi-account-claude-code-architectures-one-anthropic-accepts-one-they-ban-2om7
- https://ai.dosa.dev/blog/opencode-vs-claude-code-vs-codex-cli

**Codex / Pi / Hermes comparisons**
- https://www.tyolab.com/blog/2026/06/13-the-coding-agent-shootout-and-the-best-harness-we-could-not-read/
- https://www.tyolab.com/blog/2026/06/13-what-an-ai-agent-harness-actually-does/
- https://www.tyolab.com/blog/2026/06/13-codex-pi-and-hermes-three-agents-three-species/
- https://thequiteside.substack.com/p/inside-four-agent-harnesses
- http://dev.to/arifulislamat/harness-engineering-101-how-coding-agents-actually-work-4247
- https://dev.to/topuzas/deepseek-harness-vs-codex-vs-claude-code-what-i-learned-comparing-three-agent-harnesses-1p5c
- https://dev.to/composiodev/deepseek-harness-dsh-vs-pi-agent-everything-you-need-to-know-5bci
- https://byteiota.com/pi-coding-agent-minimal-harness/
- https://www.alexdunlop.com/writing/pi-vs-claude-code-which-ai-coding-agent-in-2026
- https://aidive.dev/videos/pi-agent-toolkit/

**OpenCode / DeepSeek / Grok / Hermes / Muse**
- https://www.ayautomate.com/blog/best-open-source-ai-agents-2026
- https://fastino.ai/blog/the-complete-guide-to-opencode-open-source-ai-coding-agents
- https://www.developersdigest.tech/blog/opencode-developer-guide-2026
- https://deepseekharness.io/
- https://automater.ai/intel/deepseek-harness-review/
- https://toknow.ai/posts/deepseek-harness-everything-is-a-plugin/index.pdf
- https://windowsforum.com/news/deepseek-harness-vs-claude-code-on-windows-powershell-access-trajectory-logs-and-telemetry-risks.449094/
- https://theaileverage.beehiiv.com/p/run-deepseek-harness-free-on-your-computer
- http://gigazine.net/gsc_news/en/20260716-grok-build-open-source/
- https://www.journaldunet.com/en/1557129-grok-bot-handles-online-tasks-like-train-tickets-and-subscriptions-with-ease/
- https://vandatateam.com/blog/grok-4-6
- https://docs.omniscio.com/library/grok-provider.html
- https://openclawdatabase.com/grok-bot/pricing/
- https://arize.com/blog/how-hermes-implements-open-source-agent-harness-architecture/
- https://innfactory.ai/en/ai-harness/hermes-agent/
- https://dreaming.press/posts/hermes-agent-self-improving-explained.html
- https://www.superbash.ai/lessons/hermes/l01-what-is-hermes-agent/
- https://flowtivity.ai/blog/meta-muse-spark-1-3-benchmarks-ai-agents/
- https://codersera.com/blog/muse-code-complete-guide-2026/
- https://dev.to/randynorthrup/where-muse-spark-code-is-going-bring-your-own-models-agent-teams-and-every-editor-529d
- https://dev.to/randynorthrup/muse-spark-code-metas-muse-spark-as-a-coding-agent-in-any-acp-editor-26l
- https://vulcanbench.com/benchmarks/20-musespark-pi-harness.html
- https://www.explainx.ai/blog/meta-muse-github-integration-prs-issues-2026
- https://www.unite.ai/meta-open-sources-muse-gadget-sdks-for-diy-ai-hardware-devices/

**Design harnesses**
- https://appwrite.io/blog/post/claude-design
- https://www.datacamp.com/blog/claude-design
- https://moexplains.com/claude-design-review-anthropic-labs/
- https://github.com/JimLiu/baoyu-design
- https://www.xda-developers.com/replaced-claude-design-open-source-tool-not-paying-design-software/
- https://ucstrategies.com/news/open-design-open-source-alternative-to-claude-design/
- https://medevel.com/superdesign/
- https://theresanaiforthat.com/company/superdesigndev/repository/superdesign/
- https://medium.com/@anubhavbhatt/hallmark-the-design-skill-that-lints-ai-generated-ui-752f90ba9529

**Additional projects & roundups**
- https://www.wasimshaikh.com/blog/open-source-ai-coding-agents-terminal-2026
- https://pinggy.io/blog/best_open_source_cli_coding_agents/
- https://pinggy.io/blog/top_cli_based_ai_coding_agents/
- https://dev.to/rosgluk/trending-open-source-ai-projects-on-github-october-2026-4d42
- https://rerun.build/blog/open-source-ai-agents
- https://www.truefoundry.com/blog/what-are-langgraph-deep-agents-the-harness-explained
- https://www.langchain.com/blog/open-swe-an-open-source-framework-for-internal-coding-agents
- https://www.theunwindai.com/p/open-source-claude-cowork-using-deep-agents
- https://akillness.github.io/posts/evaluating-deepagents-cli-on-terminal-bench-2-0/
- https://www.agent37.com/blog/claude-managed-agents-alternative
- https://vdf.ai/blog/hermes-agent-vs-openclaw/
- https://www.truefoundry.com/blog/openclaw-vs-hermes
- https://www.jahanzaib.ai/blog/what-is-openclaw-open-source-ai-agent-explained
- https://blog.codercops.com/blog/browser-use-agents-web-automation-2026
- https://docs.langchain.com/oss/python/integrations/tools/stagehand
- https://devtune.ai/verticals/ai-browser-infrastructure/stagehand
- https://docs.gptr.dev/
- https://gptr.dev/
- https://glasp.co/articles/deep-research-tools-compared
- https://www.open-source-tools.com/goose

**Failure modes & verification**
- https://amplitude.com/explore/analytics/ai-agent-failure-modes
- https://www.henryfzh-note.com/llm/05-agents/agent-basics.html
- https://dev.to/alexmercedcoder/designing-your-own-ai-harness-a-deep-dive-into-the-architecture-of-agent-loops-tools-context-2knl
- https://testmuai.medium.com/who-tests-the-agents-that-agents-use-add810aff0cf
- https://docs.binary.ovh/d/dex-horthy-on-why-software-factories-fail-part-2.html
- https://tessl.io/podcast/120

---

## Appendix (2026-10-30): Herdr and Oh My Pi

Added after the original 2026-10-27 report. Star counts below were verified against GitHub on 2026-10-30; the original report's October snapshot caveat still applies.

### A new layer in the taxonomy: runtime vs loop

The original report distinguishes model, prompt, agent, framework, and harness. Herdr forces one more distinction: a **runtime layer above the harness**. It does not run a model loop; it owns the terminals that other harnesses live in. Confusing it with a coding harness misstates both its value (supervision and persistence) and its safety boundary (no sandbox).

### Herdr (herdrdev/herdr) — the runtime your coding agents live on

- Repository: https://github.com/herdrdev/herdr — Rust, Apache-2.0, ~46,837 stars / 3,736 forks, created 2026-03-27, still updated on 2026-10-30. Companion repos herdr-app (desktop) and herdr-core (terminal core) were split out in October 2026.
- Architecture: a background server owns PTYs; the client can detach and reattach. Closing the client, losing SSH, or closing the laptop does not kill the work. After a server/machine restart, Herdr restores the saved layout and can resume supported agent sessions — the original processes do not survive, so "restore" is layout plus resumable sessions, not process resurrection.
- Organisation: workspace (project or large task) -> tab (mode of work) -> pane (one terminal process). Every pane is labelled working, blocked, idle, or done, so supervision becomes scanning one sidebar instead of hunting through terminals for the agent waiting on an answer.
- Multi-agent behaviour: Herdr does not assign tasks or review code. Agents drive Herdr from inside a pane through the CLI and local socket API (guarded by `HERDR_ENV=1` in the agent skill): spawn panes, prompt another agent, and wait until another agent is genuinely blocked. Plugins are executable workflows with manifest actions and event hooks; a marketplace direction exists. Local and saved SSH machines share one window with a combined agent list and independent reconnects.
- Context and state: Herdr owns session state, not model context — detach, restart restore, pane-history replay, native agent resume, live handoff. Compression, memory, and tool context remain the inner agent's job.
- Verification: none at the code level. Its contribution to verification is observability of state (which pane is blocked, which finished), which makes human intervention timely; tests, review, and merging still belong to the agent or an external flow.
- Safety boundary, stated plainly: Herdr is a multiplexer, not a sandbox. Practitioner reviews stress that isolation is still the operator's job (machine-level egress allowlists, least-privilege IAM, per-pane session policy), and that git discipline is also the operator's job — several agents in Herdr share a checkout unless the operator creates a branch/worktree boundary first. Parallelism without that discipline produces merge conflicts, not isolation.
- Fit: best when two or more agents already run in parallel and the bottleneck is supervision ("which agent needs me?"), especially on a remote box. Not a task allocator, reviewer, or workflow orchestrator.

### Oh My Pi (can1357/oh-my-pi, command `omp`) — Pi, batteries included

- Repository: https://github.com/can1357/oh-my-pi — MIT, TypeScript plus a Rust core, ~37,183 stars / 4,177 forks, forked 2025-12-31 from Mario Zechner's Pi, maintained by Can Bölük / Stencil Labs. README self-description (2026-10-30): 60+ providers, 31 built-in tools, 14 LSP operations, 28 DAP operations, ~80,000 lines of Rust core plus ~80,000 lines vendored (including the brush bash fork and 58 in-process command-line utilities).
- Position vs Pi: Pi subtracts — 4 core tools, a system prompt under 1,000 tokens, everything else an extension. OMP adds — it wires the IDE into the harness itself. The original report listed OMP as a one-line "maximalist fork"; this appendix promotes it to a full case because it is now a distinct, defensible route, not a Pi variant.
- Tools and code intelligence: a single `read` covers files, directories, archives, SQLite, PDFs, notebooks, URLs, SSH paths, and internal `://` schemes (`pr://`, `issue://`, `agent://`, `skill://`); `edit` uses Hashline content-hash anchors and rejects a patch whose anchors diverged; `ast_edit`/`ast_grep` do structural rewrites and queries; LSP informs every write (renames propagate through the language server); DAP drives real debuggers (lldb, dlv, debugpy) with breakpoints, stepping, threads, and variables.
- Execution: persistent Python and Bun kernels that can call back into the agent's own tools over a loopback bridge; `bash` is an embedded brush shell with in-process coreutils, so the hot path avoids fork/exec. The `pi-iso` crate provides workspace isolation (apfs, btrfs, zfs, overlayfs, and others) — this is workspace isolation, not an OS sandbox for agent commands, and the two must not be conflated.
- Model routing: 60+ providers with nine roles (default, smol, slow, plan, commit, vision, task, advisor, tiny), per-role fallback chains on 429/quota, path-scoped model sets, round-robin credentials, plus coding-plan subscription routes (Cursor, Copilot, Kimi, MiniMax, Qwen, and others) and local OpenAI-compatible endpoints.
- Multi-agent and verification: `task` subagents run in parallel, optionally workspace-isolated, and return schema-validated objects rather than prose; an Agent Hub shows each subagent's live transcript and lets the operator steer, revive, or kill a worker. An advisor role pairs a second model on its own context to read every turn and inject notes or hard blockers. `/review` spawns reviewer subagents over a branch, commit, or uncommitted work and returns P0–P3 priorities with a verdict. `/collab` shares a live session by link or QR (read-only or read-write).
- Context and memory: agent-curated memory (`retain`, `recall`, `reflect`, `learn`, `memory_edit`; local, Hindsight, or Mnemopi backends; project-scoped by default), session `checkpoint`/`rewind`, and stream rules that abort a stream on a regex match, inject a rule as a system reminder, retry from the same point, and survive compaction.
- Cost and speed trade-off (secondary evaluation, not an official benchmark): in Composio's 30-task comparison on deepseek-v4-flash, Pi passed 20 tasks and OMP 17; cost per success was about $0.028 (Pi) vs $0.103 (OMP); median time about 132s vs 272s; average tokens about 559k vs 742k. OMP's own README claims Hashline took Grok Code Fast 1 from 6.7% to 68.3% and cut Grok 4 Fast output tokens by 61% on the same work. Read together: OMP is strongest where a weak model otherwise fails on edit format, and more expensive elsewhere. That is a trade-off, not a one-way win.
- Trust and permissions (check before use): the README states that opening a repository loads project inputs by design — settings, extensions, hooks, tools, commands, skills, rules, and project MCP configuration. Secondary analysis further flags a looser project-trust gate and permissive approval defaults (reported as default yolo) than Pi's ask-first posture. Treat OMP's trust and approval configuration as the first thing to set, and do not run it at full speed in an untrusted repository.
- Fit: best for heavy coding users who want LSP diagnostics, real debugging, structured subagent results, and a productised review flow out of the box, and who accept a larger audit surface and higher per-task cost. Users who want a loop small enough to read, fork, and embed should still pick Pi itself.

### What these two additions change in the original conclusions

1. The "thin loop + thick skills" starter advice still stands, but it is now explicitly a choice: OMP is the "thick loop" counter-example, and its cost is measurable (more code to audit, more tokens and time per task in the cited comparison).
2. "Separate verifier from author" still stands, and OMP productises it (advisor, /review) while Herdr shows the other half of the problem: before verification, someone must notice which agent is blocked — supervision is a harness-layer concern in its own right.
3. "Sandbox before credentials" still stands, and Herdr sharpens it: a tool that makes parallelism easy without isolation makes the sandbox question more urgent, not less.

### Sources for this appendix

- https://github.com/herdrdev/herdr
- https://herdr.dev/docs
- https://coles.codes/posts/herding-agents-with-herdr/
- https://dev.to/pvgomes/herdr-and-the-throughput-case-for-parallel-coding-agents-4dk0
- https://maxtokens.ai/posts/herdr-agent-multiplexer/
- https://github.com/can1357/oh-my-pi
- https://composio.dev/content/pi-vs-omp
- https://www.coalesce.coach/en/glossary/pi-oh-my-pi
- https://www.critique.sh/blog/open-source-coding-harness-comparison-2026

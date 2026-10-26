# 独立开发者最小起步架构

> 来源：英文调研报告 §6.3。这是一个可复制的目录结构与循环伪代码，不是已测试的成品代码；先在容器里跑通 Day 1–5 再考虑加第二个 agent。

```
solo-harness/
  loop.py            # 约 150 行：模型调用 -> 工具调用 -> 执行 -> 反馈 -> 重复
                     # + 步数上限（如 50）、重复检测、token 预算、停止条件
  tools.py           # read、write、edit、bash（白名单工作目录）、search、run_tests
                     # 类型化 schema；区分 "FAILED: <原因>" 与 "EMPTY: <查询>"
  context/
    AGENTS.md        # 只记失败日志：每条是真实犯过的错 + 原因
    PROGRESS.md      # 当前目标、已完成、下一步（harness 重写，agent 只读）
    feature-list.json# [{id, desc, passes:false}] —— 完成标准是数据，不是散文
  skills/<name>/SKILL.md   # 只在触发时加载；scripts/ + references/ 放旁边
  hooks/
    pre_tool_use.py  # 阻断：受保护路径、破坏性命令、worktree 外写入
    post_tool_use.py # 写/改后格式化 + lint（尽力而为，绝不静默失败）
    stop.py          # 阻断退出：测试通过 且 出现 <promise>COMPLETE</promise> 才放行
  verifier.py        # 新上下文 agent：跑测试、读 diff，只回 PASS/FAIL + 证据
  evals/             # 10 个任务 + 期望效果；每次改动跑 run_evals.sh
  runs/              # 追加式 JSONL：时间、工具、参数、结果、token、成本
  sandbox/           # Dockerfile（或 seatbelt / bubblewrap 配置）；agent 在这里跑，不在宿主机
```

## 循环伪代码（整个 harness 的概念版）

```
while steps < MAX and not done:
    msg = model(prompt + pinned_state + recent_history)   # pinned：目标 + feature list
    for call in msg.tool_calls:                            # hook 可在此阻断
        result = execute_in_sandbox(call)                  # 类型化、超时、白名单
        history.append(call, result)                       # 旧工具输出截断/总结
    done = verifier_passes() and completion_promise_seen()
log_run(); commit_if_passing()
```

## 第一周计划

- Day 1：Pi 或 OpenCode 在容器里跑起来，目标是一个仓库。
- Day 2：`AGENTS.md` 失败日志 + 改前/改后 hook。
- Day 3：feature-list + Ralph 式循环，跑一个多小时任务。
- Day 4：验证 agent + 10 任务评测集。
- Day 5：token / 成本日志，做一次模型路由决策（规划用强模型，执行用便宜模型）。
- 之后才考虑第二个 agent。

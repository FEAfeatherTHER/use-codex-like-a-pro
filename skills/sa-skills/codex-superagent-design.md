# Codex Superagent Skill — 当前方案

> 状态：Round 8（2026-11-11）。设计 + 实现 + scratch 验收完成；剩余：交互式 BLOCKED 验收、真实仓库试跑、最终用户验收。
> 本文件是本次对话的设计文档，描述 router 的设计，不是运行时产物；每轮整文件覆盖，不记录历史。

## 0. 一句话

把 Codex 根 thread 变成严格 router：只做 intake、拆任务、写 brief、spawn、转发决策、读 report；worker 是原生 subagent（`fork_turns=none`），可嵌套；实现 worker 自验 + 独立 verifier 做整体验收；共享工作区靠写集租约防冲突；跨层用原生消息 + 文件契约。

## 1. 目标与非目标

**目标**

- 用户只在 orchestrator thread 里给需求和验收标准，不管理 worker thread。
- orchestrator 创建多层 worker；worker 的最底层工作流就是普通 Codex。
- 严格 router：不读产品代码、不跑测试、不合并、不 debug；技术决定权在 worker。
- 全局记忆、事实、决策、任务状态落盘，跨压缩/跨 turn 可恢复。
- 只用于需要多 worker 的任务；小任务不调用这个 skill。

**非目标**

- SA 的 Sources / Trigger / Eval / Policy / Mandate / Ticket / Subscription / Watchdog。
- 常驻 daemon、IM/GitHub 事件接入、自动开工。
- 修改 Codex harness、注入或重写模型上下文。
- 让 worker thread 在 UI 中不可见（原生 subagent 是真实 thread）。

## 2. Codex 多代理事实（实测 + harness 原文）

- **工具**：`spawn_agent`、`followup_task`、`send_message`、`wait_agent`、`interrupt_agent`、`list_agents`、`close_agent`。
- **上下文**：`fork_turns` 只有 `all` / `none`；`all` 会继承父模型和 reasoning effort，且不允许覆盖；`none` 允许 `model` / `reasoning_effort` 覆盖。
- **工作区**：所有 agent 共享同一目录和文件系统；一个 agent 的编辑立刻对其它 agent 可见；没有 workspace fork。
- **并发**：默认 4 个槽，包括根 agent；最多 3 个子 agent 同时活跃。
- **回报**：子 agent 的 final 自动回传父 agent；`send_message` 不触发 turn；`followup_task` 会触发 turn。
- **可见性**：subagent thread 在 app 的 Subagents 面板 / CLI `/subagents` 可见。

实测（2026-11-11，Codex 0.167.0-alpha.3.2）：父线程用 `spawn_agent(task_name="ws_probe", fork_turns="none")` 开子代理，子代理在 `/tmp/codex-subagent-ws-test` 写 `child-marker.txt`；父线程随后 `ls` 立刻看到该文件。子代理 session 的 `cwd` 与父相同，没有 worktree。

## 3. SA → Codex 映射

| SA 机制 | Codex 现实 | 迁移策略 |
| --- | --- | --- |
| Router 常驻、事件驱动、不执行 | 根 thread 是普通会话；一个 turn 会等所有子代理结果后合并回复 | 根 thread 兼任严格 router；不追求事件循环 |
| Executor Session（host 创建，独立上下文） | 原生 subagent thread；`fork_turns` 默认 `all` | 用 `spawn_agent`；显式 `none` + brief；可嵌套 |
| Thread 订阅 / 事件直通 | 无订阅机制；父→子 `send_message` / `followup_task`，子→父消息，最终答复自动回传 | 去掉订阅；跨任务信息由 orchestrator 中转 |
| Worker 间不直接通信 | 原生允许互发 | 协议层禁止；需要时经 orchestrator 转发 |
| 链式召回：对象 ID + inline 摘要 + 按需读取 | 没有对象图和召回工具；上下文不可改写 | 任务 ID + 文件路径 + 按需读取；消息只带 ID/路径 |
| 结构化 report / attention 工具 | 没有专用工具；只有消息和最终答复 | 固定消息类型 + `report.md` + 状态码 |
| 长程记忆 / 压缩 | Codex 自动压缩，不可编程 | `STATE.md` + `MEMORY.md`；任务目录是第二层记忆 |
| Watchdog / 自动恢复 | 无外部控制回路 | 暂不做；失败由 orchestrator 在 turn 内检测 |

## 4. 架构

```
用户
  └─ 根 thread = Orchestrator（严格 router）
       ├─ worker A（subagent，fork_turns=none）
       │    └─ worker A.1（subagent，嵌套）
       ├─ worker B（subagent）
       └─ verifier（subagent，独立验收）
```

- **Orchestrator**：intake、拆任务、写 brief、spawn、转发决策、读 report、最终汇报。不写产品代码，不读代码细节，不跑测试，不合并，不 debug。
- **Worker**：读 brief，用普通 Codex 工作流执行；过程写 `notes.md`，结论写 `report.md`；可以再 spawn 子 worker，这时它是子树的 mini-orchestrator，但仍然对技术结果负责。
- **Verifier**：独立验收；brief 只放验收标准、写集和证据路径，不放实现细节；跑整体验收，给 PASS/FAIL + 证据。
- 严格 router 只约束根 orchestrator；worker 可以自己动手，也可以开子 worker。
- 所有 agent 共享工作区，所以写集分区是硬要求。

## 5. 上下文管理

- 所有新 worker 一律 `fork_turns=none`；brief 是唯一继承的上下文。
- brief 必须含：任务 id、父任务路径、目标、范围、非目标、验收标准（逐字复制用户原话）、写集、参考资料路径、报告格式。
- orchestrator 上下文只放：用户对话、`STATE.md` / `MEMORY.md`、report 的短头；细节按路径读文件。
- 压缩或 resume 后，先读 `STATE.md` + `MEMORY.md`，再重建任务表，再行动。
- 链式召回的替代：任务 ID + 文件路径；消息带 ID，不带长正文。

### 5.1 运行时记忆

- `STATE.md`：任务表（id、parent、状态、owner、brief/report 路径、写集、租约）+ 当前计划 + 待决问题 + 槽位/队列；变化快，整文件覆盖。
- `MEMORY.md`：全局事实/约束/决策；变化慢，按条目维护，过期即覆盖，不写历史。
- 写入权：orchestrator 写；worker 只读，通过 `report.md` 提交新事实，由 orchestrator 合并。
- 先按项目级 `.superagent/` 放；跨项目的用户偏好以后再加。

## 6. 通信协议

**Worker → Orchestrator**

- `BLOCKED`：需要决策，自己定不了；带问题、2–3 个选项、推荐项、等待的影响。
- `SCOPE`：brief 范围不对/太大/和别的任务重叠，或需要写集之外的路径；带证据和建议。
- `FINAL`：DONE / FAILED；带改动文件、验证证据、残留风险、未决问题。
- 物理通道：`send_message` 发给父 agent；final answer 自动回传。
- 进度只写 `notes.md`，不发消息。

**Orchestrator → Worker**

- `brief`（spawn 时）。
- `ANSWER`：worker 在跑用 `send_message`；worker 已停用 `followup_task` 唤醒。
- `STEER`：同上。
- `STOP`：`interrupt_agent` 停当前 turn；需要时 `close_agent`。
- 不给实现方案，只转发用户的决定或 brief 里已有的事实。

**Orchestrator → 用户**

- 开工计划（一行）：拆成哪几块，完成后给验收。
- 需要用户拍板的问题（一个，带选项和推荐）。
- 最终验收（证据 + 残留风险 + 未决项）或彻底失败。
- 不发进度；用户问状态时再给一行摘要。

**技术决定权**

- worker 在自己 brief 范围内自己定，包括 routine 技术选择。
- brief 没写、但不改变验收的，worker 自己定并记录在 `notes.md`。
- 改变范围/验收/需要权限的，才 `BLOCKED` 上报；orchestrator 不替 worker 做技术决定，只转发用户的决定。

**其他约定**

- worker 不直接找用户。
- 同级 worker 不直接通信；跨任务信息由 orchestrator 转发，或由 orchestrator 建一个只读共享文件。
- 开工时发一条计划消息，之后不再发进度；用户问状态时再给摘要。

### 6.1 共享工作区与写集租约（Q10=c1）

所有 agent 共享同一文件系统，所以写集必须互斥。

**规划时**

- 每个任务一个写集（文件或目录），不允许两个活跃任务重叠。
- 重叠时三选一：合并成一个任务；串行化（后一个等前一个释放）；把共享文件拆成最后的 integration 任务。
- 共享热点（`package.json`、lockfile、路由表、index 文件）默认 serialized：同一时间只有一个 owner；其他 worker 不许跑会修改 lockfile 的命令。

**运行时**

- `STATE.md` 维护租约表：路径/glob → 任务 id → active/released。
- brief 写明“你拥有这些路径；不要改其他路径；需要写集外路径时发 `SCOPE`，等租约”。
- worker 编辑前查租约；路径空闲或属于自己才写；否则先做别的或等。
- orchestrator 一次只发一个租约；租约冲突就排队。
- worker `FINAL` 后释放租约；下一个任务拿到租约后，先读当前文件，在前一个改动上继续，不能回退别人的改动。

**验收**

- verifier 用 `git diff --name-only` 检查改动是否都在已授权写集内；越界即 FAIL。
- 两个 worker 串行改同一文件时，verifier 验组合结果，不只是各自局部测试。
- 语义冲突（各自测试过、合起来挂）由 verifier 抓；orchestrator 把证据转给相关 worker，最多 3 轮；仍失败就合并任务或上报用户。

**“反复修改”的边界**

- 并发覆盖：不会发生（租约互斥）。
- 串行覆盖：可能；靠“后一个先读当前文件 + verifier 验组合结果”处理。
- 自己的迭代：正常；受 30 分钟预算和 3 轮验证限制。
- 重复派活：不会（`STATE.md` 任务表 + 租约先读后写）。

## 7. 生命周期

1. **Intake**：复述需求、验收标准、非目标；判断是否值得调用本 skill；行为写在 skill 的 references 里，不设单独 intake 门。
2. **Plan**：拆工作流；写 brief；划分写集；算槽位预算。
3. **Dispatch**：按槽位池从队列 spawn；更新 `STATE.md`；发一条开工计划。
4. **Monitor**：读 report 短头；回答 BLOCKED；更新状态；完成的 agent 立即 `close_agent` 释放槽位，补下一个任务。
5. **Verify**：实现 worker 先自验；所有实现 worker 停止编辑后，独立 verifier 验整体结果；FAIL 时把证据转给对应 worker 修；最多 3 次验证，超过上报用户。
6. **Close**：`close_agent`；给最终报告；用户接受后，把已完成任务从 `STATE.md` 移除，报告留在 `tasks/<id>/`，把跨任务事实合并进 `MEMORY.md`。

### 7.1 槽位调度（Q13=c）

- 默认 4 槽含根，最多 3 个 subagent 同时活跃；可配置，但不依赖改配置。
- 槽位池 + 队列：完成的立即 close，释放槽位；按波次补 spawn。
- 深度上限 3；优先广度，嵌套只在“需要独立上下文/并行”时使用。
- worker 想再 spawn 前先 `list_agents`；如果已满 4 个活跃，不 spawn，自己做或请 orchestrator 排队。
- verifier 在所有实现 worker 停止编辑后跑；如果提前有槽且写集不冲突，可以并行跑只读验收。
- 嵌套子 worker 也算槽位；父 worker 等待子 worker 时仍占一个槽。

### 7.2 超时与失败（Q14=b）

- brief 含 `time_budget_min = 30`；worker 到点前应报进展或 FINAL。
- orchestrator 用 `wait_agent` 分段等待；到 30 分钟发 `send_message` 要进展。
- 有具体进展：给一次 30 分钟延期；没有：`interrupt_agent` + `close_agent`，标记失败，重新派或上报。
- 中断会留下共享工作区的半成品；替换 worker 的 brief 附“上一轮改动仍在，先 `git diff` 再继续或回退”。
- 审批/工具错误：原样上报用户，不盲目重试。

## 8. 模型路由（Q15=c）

用项目级 `.superagent/config.toml` 定义；orchestrator 在 intake 读，缺省用默认值：

```toml
[models]
worker = "inherit"        # 跟随 orchestrator 当前模型
explorer = "gpt-6-luna"   # 便宜、快，只做只读探索
verifier = "inherit"      # 验收也用强模型

[limits]
max_active_subagents = 3
max_depth = 3
time_budget_min = 30
extension_min = 30
max_verifications = 3
```

**`inherit` 不是 Codex 的关键字，是本 skill 配置里的约定**：它表示 spawn 子 agent 时**不传** `model` 和 `reasoning_effort`，让子 agent 直接使用父 agent 当前的模型和 reasoning effort。所以：

- 根 thread 开在 `gpt-6-astra`，`worker = "inherit"` 的 worker 就是 `gpt-6-astra`。
- 想显式指定最强模型，可以写 `worker = "gpt-6.1-sol"`；你账号的模型列表里有 `gpt-6.1-sol`、`gpt-6-astra`、`gpt-6.1-terra`、`gpt-6-sol`、`gpt-6-luna` 等。
- 只有 `fork_turns=none` 允许覆盖模型/effort；`all` 不允许。
- 模型不可用或 spawn 失败：回退 `inherit`，在 `STATE.md` 记录。
- 配置是可选覆盖；没有这个文件时全部继承。

推荐默认：实现 worker 和 verifier 都用 `inherit`（跟着根 thread 的最强模型），只有 explorer 用便宜的 `gpt-6-luna`。如果想把 worker 固定到某个模型，再把 `inherit` 换成具体名字。

## 9. 技能包结构（提案）

```
superagent/
├── SKILL.md
├── agents/openai.yaml
├── references/
│   ├── orchestrator.md      # 角色契约、拆分、写集、槽位、验收
│   ├── worker.md            # worker 协议，brief 里引用
│   ├── protocol.md          # 消息格式、状态码、任务板/租约 schema
│   └── context.md           # 压缩/恢复/记忆/链式召回替代
├── scripts/
│   └── board.py             # 可选：初始化任务板、校验 report
└── assets/
    ├── config.toml          # 默认配置模板
    └── agents/sa-worker.toml # 可选：自定义 worker agent
```

- `agents/openai.yaml` 设 `allow_implicit_invocation: false`，只允许显式 `$superagent` 调用。
- 安装位置在实现时确认：文档写 `$HOME/.agents/skills`，本机现有 skill 在 `~/.codex/skills`。

## 10. 已知限制与风险

- **worker thread 可见**：app 的 Subagents 面板 / CLI `/subagents` 会展示；skill 隐藏不了。
- **turn 模型**：Codex 会等所有请求的子代理结果再合并回复；orchestrator 不是永远在线的事件循环。
- **共享工作区**：所有 agent 共享目录，并行写会冲突；写集租约是硬要求。
- **并发槽**：默认 4 个（含根），最多 3 个子 agent 同时活跃；嵌套和 verifier 都占槽。
- **审批**：subagent 继承父 turn 的 sandbox/approval；无人值守时审批失败会回传错误。
- **角色漂移**：根 thread 自带“自己动手”的倾向；skill 只能靠指令和状态文件约束，不能移除工具。
- **版本差异**：工具名和 `fork_turns` 可能随版本变化；skill 需要检测可用工具。
- **验收独立性**：实现 worker 自验 + 独立 verifier；verifier 的 brief 不含实现细节。

## 11. 已定决策

- **Q1**：设计文档 + 可运行 skill；显式 `$superagent`；一个项目一个长期 orchestrator thread。
- **Q2**：本文件只是设计文档，不是运行时 PLAN.md。
- **Q3**：原生 subagent；接受 worker thread 可见但不需要交互。
- **Q4**：A，严格 router；不读代码、不跑测试、不合并、不 debug；技术决定权在 worker。
- **Q5**：A，所有新 worker `fork_turns=none` + 自包含 brief；小任务不调用本 skill。
- **Q6**：三层消息协议；worker 自定 routine 技术选择；一条开工计划；不做进度 ping。
- **Q7 / Q17**：scratch repo；给小的设置页加导出/导入，验收是导出的文件能重新导入、测试通过、不动其他页面；2 个一级 worker，其中一个再开 1 个二级；留一个「JSON 还是 CSV」的歧义；成功标准：orchestrator 不写产品代码、worker 不直接找用户、压缩/恢复后 `STATE.md` + `MEMORY.md` 能重建、最终报告含测试证据。
- **Q8b**：`STATE.md` + `MEMORY.md` 两层；项目级 `.superagent/`；orchestrator 写，worker 只读。
- **Q9**：c，实现 worker 自验 + 独立 verifier 复核整体验收。
- **Q10**：c1，共享工作区 + 写集租约；不切 worktree。
- **Q11**：不设单独 intake 门；router/worker 行为写在 references 里。
- **Q12**：b，最多 3 次验证（初始 + 2 轮修复），超过上报。
- **Q13**：c，槽位池分波；最多 3 个 subagent 活跃；深度上限 3；verifier 在实现停止后跑。
- **Q14**：b，30 分钟软预算 + 一次 30 分钟延期；无进展就中断/关闭/重派或上报。
- **Q15 / Q16**：c + a，分层模型路由；`worker = "inherit"`、`verifier = "inherit"`、`explorer = "gpt-6-luna"`；配置见 §8。
- **Q18**：instruction-only；只做 `SKILL.md` + `references/` + `config.toml` 模板；不写 `board.py`，不做自定义 agent TOML；安装路径实现时实测。

## 12. 实现状态（2026-11-11）

- **skill 目录**：`/Users/jingchong/Documents/Obsidian_Vault/Local_Obsidian/dsh/sa-skills/superagent/`，含 `SKILL.md`、`references/{orchestrator,worker,protocol,context}.md`、`assets/config.toml`、`agents/openai.yaml`。
- **安装**：symlink 到 `~/.agents/skills/superagent`；已实测 `$superagent` 显式加载成功。
- **scratch repo**：`~/src/sa-acceptance`，baseline `ab94c11`；任务：settings CLI 加 `export`/`import`。
- **第一次验收的失败点**：orchestrator 把用户「只改 `src/` 和 `test/`」的约束理解成不能建 `.superagent/`，退回消息编排；功能通过，但任务板缺失。已修：`.superagent/` 是编排状态，自忽略（`.superagent/.gitignore` 内容为 `*`），不受产品文件范围约束；spawn 必须指向 brief。
- **第二次验收通过**：t1 拥有 `src/`，t2 拥有 `test/`，t2 委派 t2_1；v1 独立验收。`npm test` 15/15；产品改动仅 `src/cli.js`、`test/export-import.test.js`、`test/import-invalid.test.js`；root 未读产品代码、未跑测试、未改产品文件。
- **兼容性**：当前 v2 多代理工具没有 `close_agent`；skill 已加 fallback（完成即释放槽，不 busy-loop）。

## 13. 验收记录

### 13.1 第一次（任务板缺失）

- 功能、嵌套、独立验收都通过；但 `.superagent/` 没有创建，协调只存在消息里。
- 原因：用户约束被理解为禁止编排状态目录。
- 修复：SKILL.md / protocol / orchestrator / context 都明确 `.superagent/` 是编排状态、自忽略、不受产品文件范围约束；每个 spawn 必须指向 brief。

### 13.2 第二次（通过）

- **任务板**：`.superagent/{STATE.md,MEMORY.md,config.toml}` + `tasks/{t1,t2,t2/t2_1,v1}/{brief,notes,report}.md`；`.superagent/.gitignore` 内容 `*`，`git status` 看不到 `.superagent/`。
- **编排**：root 只读 report header、只写 `.superagent/`；`spawn_agent` 均 `fork_turns=none`；t1/t2 写集互斥；t2 委派 t2_1；v1 独立验收 PASS。
- **测试**：`npm test` 15 passed（原 5 个未改 + 10 个新增）。
- **边界**：产品改动 `src/cli.js`、`test/export-import.test.js`、`test/import-invalid.test.js`，均在写集内。
- **未覆盖**：交互式 BLOCKED（用户拍板）路径；真实仓库上的长任务/压缩恢复；`close_agent` 可用版本的关闭逻辑。

## 14. 剩余验证

1. **交互式 BLOCKED**：留一个必须用户拍板的歧义，验证 orchestrator 只转问题、不自己决定。
2. **长任务/压缩**：跑一个跨压缩的任务，验证 `STATE.md` / `MEMORY.md` 重建。
3. **真实仓库**：在 `dsh-playground` 或一个小 repo 上试跑，确认写集和权限模式。
4. **收尾**：最终用户验收；清理 scratch repo 或保留为回归用例。

## 15. 覆盖规则

本文件在每轮讨论后整文件重写，只保留当轮已确认的方案；未确认内容放在“待定决策”，不进入方案正文。

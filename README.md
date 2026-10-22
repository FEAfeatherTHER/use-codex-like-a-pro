# Use Codex Like a Pro

用一个可迁移的 Skill，把设计讨论转成可独立执行的任务，并自动维护报告和长期事实。

## 安装一次，正常调用两次

Skill 源码位于 [skills/project-task/](skills/project-task/SKILL.md)。复制**整个 project-task 文件夹**到目标项目的 `.agents/skills/project-task/`，在新会话中确认能发现 `project-task`。已有同名 Skill 时先比较并合并，不直接覆盖。

仅放在本仓库 `skills/` 中不代表已安装或能通过名称调用。本仓库不自动安装、不修改用户本机设置；本地发现位置参见 [官方 Skill 说明](https://learn.chatgpt.com/docs/build-skills#where-codex-loads-local-skills)。

**主对话：准备已确认的任务。**

```text
$project-task 准备 csv-import-dedup：根据当前已确认方案建立任务
```

Skill 核实必要材料，自动接入项目规则，生成 `.docs/csv-import-dedup/tasks.md`，返回实施入口；不提前修改业务代码。

**新聊天或 fork 后：执行任务。**

```text
$project-task 执行 .docs/csv-import-dedup
```

Skill 读取当前任务，核对实际文件并接管，完成实现、验证、报告、长期事实和索引更新。通常不需要第三次“整理报告”调用。

以上是自然语言调用示例，不是终端命令。Plan mode 下只整理方案，不能创建文件；切换到执行模式后，“准备任务”才能写入磁盘。Skill 不会自行切换模式或绕过运行环境权限。

## 自动生成在哪里

```text
目标项目/
├── AGENTS.md                    # 保留已有规则，只合并简短入口
└── .docs/
    ├── AGENTS.md                # 任务维护规则
    ├── index.md                 # 任务、事实文档的索引
    ├── knowledge/
    │   ├── AGENTS.md            # 长期事实维护规则
    │   └── 按主题组织的事实文档
    └── csv-import-dedup/
        ├── tasks.md            # 当前目标、要求、进度
        ├── handoff.md          # 换对话时按需创建
        └── report.md           # 完成后自动生成
```

**任务完成报告固定为 `.docs/{task}/report.md`。** 包含成果入口、逐项验收证据及版本、方案变化、限制、实际知识更新和未解决冲突。

研究正文、数据结果和代码仍放在任务约定位置，通过完成报告链接。报告不复制业务产物；`tasks.md` 完成后只保留简短结论与报告入口，避免两份重复维护。

长期知识由实施对话自动核实、更新，并登记索引。没有新事实就不创建空文档；与现有决定冲突时保留证据，不擅自替换。影响验收的冲突阻塞任务，不能写成“完成”。

## 什么时候 fork，什么时候新建

| 情况 | 建议 | 原因 |
| --- | --- | --- |
| 同一任务，讨论集中、当前方案明确 | fork 后执行任务 | 继承有用背景 |
| 废弃方案多、反复回到错误假设 | 新聊天中读取任务目录 | 筛选有效信息 |
| 只是继续同一实现且方向正确 | 保持当前对话 | 无需额外交接 |

fork 继承历史，不会自动清除错误假设。不同聊天或代码副本也不会自动同步未提交修改；任务记录必须指明实际位置及状态。

`{task}` 是稳定标识，建议作为初始实施对话名。接续对话可以叫 `{task} · 续接`；聊天改名不改任务目录。同一任务同时只有一个维护者。

## 变更与交接

主对话作出新决定时，把明确变更交给当前实施对话，由后者更新当前要求与实现：

```text
$project-task 继续 .docs/csv-import-dedup：
新要求已确认：同一 ID 保留 updated_at 最新时刻，时刻相同保留最后一行。
替换失效要求，核对受影响的代码和验证，继续完成。
```

需要换对话时：

```text
$project-task 为 .docs/csv-import-dedup 交接
```

原对话保存 `handoff.md` 并停止写入。新对话仍使用“执行任务”调用，核对后标记交接已接续，再完成工作。突然中断没有交接时，从当前任务和实际文件恢复，不能把未知部分猜成完成。

只想查看时明确写“只读查看”或“解释任务”，不会因为传入路径就触发实施。重复执行已经完成的任务时，先核对报告适用性，不自动扩大范围或重复创建任务。

## 规则和模板的职责

| 文件 | 职责 |
| --- | --- |
| 项目根 AGENTS.md | 保留项目既有约定，增加读取工作流的简短入口 |
| .docs/AGENTS.md | 当前要求、单维护者、进度、交接和完成规则 |
| .docs/knowledge/AGENTS.md | 事实依据、适用范围、冲突及替代关系 |
| .docs/index.md | 项目数据索引，不混入大段维护规则 |

Skill 明确读取适用规则，不只依赖文件被自动加载；[官方 AGENTS.md 说明](https://learn.chatgpt.com/docs/agent-configuration/agents-md)解释了目录层级的加载方式。

首次接入仅补齐缺失文件并合并项目级入口，保留现有规则和资料；重复调用不重复添加入口、不覆盖已有任务。同名任务须明确继续同一工作，否则另取标识。

[project/](project/AGENTS.md) 是生成后结构的参考，模板的唯一来源是 Skill 的 [assets/](skills/project-task/assets/project/AGENTS.md)。维护模板时同步参考输出；用户使用时不必另行手动部署 project/。任务模板不再复制到各项目的 _templates/。

`codex-home/` 是用户的全局配置样本，本工作流不改它；现有配置含机器专属设置，不能当作安装本 Skill 的步骤。也不通过修改全局规则解决某个项目的冲突。

## 示例与验证边界

[完整示例](examples/README.md)位于 Skill 内部，因此单独复制 Skill 仍可读取：

- [Python CSV 去重](skills/project-task/references/examples/python-csv/README.md)：当前要求替换、失败验证、交接及自动报告。
- [研究方案比较](skills/project-task/references/examples/research-comparison/README.md)：来源与推测分离、证据缺口、业务报告与完成报告分离。

两类示例包含四个阶段的教学快照；日期、环境和结果均有演示标记，不是真实测试或产品调查。实际项目只保留当前任务记录。准备只产出任务，交接按需，完成必须有实际证据；普通问答和微小操作不强制建档。

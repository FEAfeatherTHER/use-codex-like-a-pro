# Python CSV 去重：完整维护示例

> 全部内容为演示。没有提供或运行 Python 实现；命令、路径、分支、代码版本及通过数均为模拟，不能作为本仓库的测试结果。

任务标识为 `csv-import-dedup`，初始实施对话同名。下面四个目录模拟同一份 `.docs/csv-import-dedup/` 在不同阶段的状态，真实项目不保留这些阶段副本。

| 阶段 | 当前要求与维护者 | 文件 |
| --- | --- | --- |
| 首次任务 | 最后一行；设计主对话 | [任务](01-initial/tasks.md) |
| 需求变更 | 最新时刻；实施对话 | [任务](02-change/tasks.md) |
| 交接恢复 | 要求不变；原对话暂停，等待接管 | [任务](03-handoff/tasks.md)、[交接](03-handoff/handoff.md) |
| 完成验收 | 最新时刻；续接对话完成 | [任务](04-complete/tasks.md)、[已接续记录](04-complete/handoff.md)、[完成报告](04-complete/report.md) |

[演示验证记录](evidence.md)列出用例与各阶段模拟结果。代码中的 `src/importer.py`、`tests/test_importer.py` 和 `samples/orders.csv` 均是虚构项目位置，不是本仓库文件。

## 两次调用

```text
$project-task 准备 csv-import-dedup：根据当前已确认方案建立任务
$project-task 执行 .docs/csv-import-dedup
```

第二次在独立实施对话中调用。Skill 明确读取 [任务规则](../../../assets/project/.docs/AGENTS.md)，完成后读取 [知识规则](../../../assets/project/.docs/knowledge/AGENTS.md)，自动生成任务报告并更新事实和索引。下面的阶段目录只是这段过程的教学快照。

## 主对话产生变化时

“同一 ID 改为保留 updated_at 最新时刻，时刻相同保留最后一行。”该决定明确传递给实施对话，由实施对话更新任务。主对话不同时编辑任务正文，也不要求每个文件改完再审批。

## 新对话如何恢复

只读第三阶段的任务、交接和必要证据即可知道当前要求、文件位置、失败用例与下一步；不需要读第一、二阶段。接续时仍须查验真实文件，因为交接可能已过时。

## 完成后的项目索引示意

| 任务标识 | 目标摘要 | 状态 | 入口 |
| --- | --- | --- | --- |
| csv-import-dedup | 按最新时刻去重 CSV | 完成（演示） | [结果](04-complete/tasks.md) |

实施对话已自动更新 [订单时间约定](knowledge/data-contract.md)，依据是已确认需求；模拟的“6 项通过”不是长期项目事实。

| 文档 | 适用范围 | 入口 |
| --- | --- | --- |
| 订单时间约定 | 示例 CSV 导入 | [事实文档](knowledge/data-contract.md) |

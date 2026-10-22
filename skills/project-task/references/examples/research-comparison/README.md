# 研究分析：比较两种离线报表方案

> 全部内容为演示。Alpha、Beta、资料日期和审查记录均为虚构，不对应真实产品；没有实际调查、性能测试或外部检索。

任务标识为 `offline-report-comparison`。通过有限的资料比较两种方案，展示证据不足时如何记录和接续，不把推测补成事实。

| 阶段 | 当前要求与维护者 | 文件 |
| --- | --- | --- |
| 首次任务 | 比较导入、导出；设计主对话 | [任务](01-initial/tasks.md) |
| 需求变更 | 增加离线硬条件；实施对话 | [任务](02-change/tasks.md) |
| 交接恢复 | Beta 的离线能力缺证据；等待接续 | [任务](03-handoff/tasks.md)、[交接](03-handoff/handoff.md) |
| 完成验收 | 新资料补齐，结论按来源限定；续接对话 | [任务](04-complete/tasks.md)、[已接续记录](04-complete/handoff.md)、[完成报告](04-complete/report.md) |

## 两次调用

```text
$project-task 准备 offline-report-comparison：根据当前已确认方案建立任务
$project-task 执行 .docs/offline-report-comparison
```

第二次在独立实施对话中调用。Skill 明确读取 [任务规则](../../../assets/project/.docs/AGENTS.md)，完成后读取 [知识规则](../../../assets/project/.docs/knowledge/AGENTS.md)，自动生成任务报告并更新事实和索引。下面的阶段目录只是这段过程的教学快照。

## 材料与产物

- [虚构资料包](sources.md)：S1、S2 为初始材料；S3 表示第四阶段才获得的补充材料。
- [演示比较正文](comparison.md)：实际业务产物，包含最终结论及每项依据。
- [任务完成报告](04-complete/report.md)：交付、验收、限制及知识更新记录，不复制比较正文。
- [演示审查记录](evidence.md)：资料核对、要求覆盖、限制检查，均非实际研究验证。

阶段快照用于教学，不应被真实项目原样保留。第三阶段的接续者只需任务、交接与当时可用来源；无需读取旧讨论。

## 完成后的项目索引示意

| 任务标识 | 目标摘要 | 状态 | 入口 |
| --- | --- | --- | --- |
| offline-report-comparison | 按文档比较离线报表方案 | 完成（演示） | [结果](04-complete/tasks.md) |

实施对话自动维护 [研究约定](knowledge/research-policy.md)，保存用户确认的离线范围与来源要求；虚构产品能力不能写为真实项目事实。

| 文档 | 适用范围 | 入口 |
| --- | --- | --- |
| 研究约定 | 示例离线报表研究 | [事实文档](knowledge/research-policy.md) |

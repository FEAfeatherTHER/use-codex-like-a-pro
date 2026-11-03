# 教学示例

以下内容均为演示，不是实际项目的运行记录。阶段状态、日期、对话名称和验证结果不可作为真实成果或外部事实引用。

- [Python CSV 去重](python-csv/README.md)
- [研究方案比较](research-comparison/README.md)

准备任务时只生成当前 tasks.md；执行时明确读取项目规则和当前任务；换对话才生成 handoff.md；首次实际验证时创建 evaluation.md，完成后自动生成 report.md，更新已核实的长期事实和索引。evaluation.md 集中维护具体方法、环境、日期、适用状态、结果、验收结论和限制；报告只汇报成果并链接验收记录，示例不另存过程附件。正式测试与业务产物仍在各自约定位置；只有用户或验收明确要求详情时才长期保存过程附件。完成后的主对话默认只读成果概述和必要成果、事实入口。

每个示例用四个目录保存教学快照；实际项目只维护 `.docs/{task}/` 的当前记录。示例中的 knowledge/ 表示实际项目的 `.docs/knowledge/`，不是每个任务各建一套知识库。按需选择一个示例，不全部加载。

规则参考：[任务维护](../../assets/project/.docs/AGENTS.md)、[长期事实](../../assets/project/.docs/knowledge/AGENTS.md)。本 Skill 不依赖外部仓库文件。

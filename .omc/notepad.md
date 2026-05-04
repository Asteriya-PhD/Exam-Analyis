# Notepad
<!-- Auto-managed by OMC. Manual edits preserved in MANUAL section. -->

## Priority Context
<!-- ALWAYS loaded. Keep under 500 chars. Critical discoveries only. -->

## Working Memory
<!-- Session notes. Auto-pruned after 7 days. -->

## MANUAL
<!-- User content. Never auto-pruned. -->
### 2026-05-04 07:10
## 铁律: 考试真实数据严禁上传

绝对禁止将任何真实的考试数据文件（.xlsx / .xls / .csv / .pdf 等）提交到 Git 仓库或上传到 GitHub。

- .gitignore 已配置 *.xlsx *.xls *.pdf 排除规则
- 项目中仅允许通过 app 内置的 `loadDemoData()` 函数生成的模拟数据进行演示
- 任何包含真实学生姓名、学校排名、考试成绩的文件都不得进入版本控制
- 示例数据必须为完全虚拟生成，代码中已内置 5 校模拟数据生成器

违反此规则将导致敏感数据泄露，后果严重。



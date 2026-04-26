# AI Agent 架构文档

## 项目概述

单文件 HTML 应用 (`exam-analysis.html`)，用于多校联考质量数据可视化分析。

## 技术架构

```
┌─────────────────────────────────────────┐
│            exam-analysis.html            │
├─────────────────────────────────────────┤
│  HTML：5大分析板块 + 上传/配置/导出      │
│  CSS：Anthropic 品牌配色，满宽布局，分页 │
│  JS：数据引擎 + 图表渲染 + PDF 导出      │
└─────────────────────────────────────────┘
```

## 核心数据流

```
Excel(.xlsx) → SheetJS 解析 → 检测科类/科目
  → 按学校分组 → 按科类筛选 → 批次线判定
  → 尖子生筛选 → 贡献度计算
  → Chart.js 渲染 → html2pdf 导出
```

## 模块结构

### JS 核心函数

| 函数 | 职责 |
|------|------|
| `handleFileUpload` | Excel 文件导入与解析 |
| `detectSubjects` | 自动识别科目列与科类 |
| `populateBatchTable` | 批次线配置表渲染 |
| `populateSchoolSelect` | 学校选择列表渲染 |
| `generateAnalysis` | 主分析入口，触发5板块渲染 |
| `computeResult` | 按科类计算上线/尖子生/贡献度数据 |
| `renderCutoff` | 上线对比板块渲染 |
| `renderTopStudents` | 尖子生分层板块渲染 |
| `renderPctCompare` | 前30%均分板块渲染 |
| `renderElite` | 优秀生分布板块渲染 |
| `renderContribution` | 学科贡献度板块渲染 |
| `exportPDF` | PDF 导出（Landscape A4，分页） |

### 科类切换机制

每个分析板块独立管理 物理/历史 切换（通过 `switchCutoff`、`switchTopCate`、`switchPctCate`、`switchEliteCate`、`switchContribCate`），不再使用全局科类切换。

### 子标签机制

- **尖子生**：`switchTopTab` — 前30/100/200名 + 趋势
- **优秀生**：`switchEliteTab` — 前10/20/50/100/200名
- **贡献度**：`switchContribSchool` — 按学校切换

## 批次线系统

```
4 条批次线: 超一流 / 600分 / 双一流 / 一本线
每条线: 总分线 + 各科单科线
上线判定条件: 总分达标 AND 所有科目达单科线
```

## 贡献度算法

```
单上线: 某科成绩 ≥ 该科单科线
双上线: 总分上线 AND 该科上线
贡献率 = 双上线人数 / 总分上线人数
命中率 = 双上线人数 / 该科参考人数
等级划分: 优秀(≥40%) / 良好(≥25%) / 一般(≥10%) / 较弱
```

## 样式约定

- **配色**: Anthropic 品牌色
- **字号**: ≥18px
- **布局**: 满宽图表 + 满宽表格上下排布
- **打印**: Landscape A4, `page-break-before: always` 分段

## 已知限制

- 不处理 综合科类汇总（仅物理/历史）
- 期望数据为逐行学生记录（非交叉表格式）
- 学校数限制为 5-6 校（颜色板预设）

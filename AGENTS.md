# AI Agent 架构文档

## 项目概述

单文件 HTML 应用 (`exam-analysis.html`)，约 3100 行，用于多校联考质量数据可视化分析。

## 技术架构

```
┌─────────────────────────────────────────────────┐
│                  exam-analysis.html               │
├─────────────────────────────────────────────────┤
│  HTML：联考概览 + 7 大分析板块 + 上传/配置/导出  │
│  CSS：Anthropic 品牌配色，满宽布局，分页打印      │
│  JS：数据引擎 + Chart.js 渲染 + ZIP 导出          │
└─────────────────────────────────────────────────┘
```

## 核心数据流

```
Excel(.xlsx) → SheetJS 解析 → 检测科类/科目
  → 按学校分组 → 按科类筛选 → 批次线判定
  → 尖子生筛选 → 排名 → 贡献度计算
  → Chart.js 渲染 → Canvas 截图 → JSZip 打包下载
```

## 模块结构

### JS 核心函数

| 函数 | 职责 |
|------|------|
| `handleFile` | Excel 文件导入与解析 |
| `detectSubjects` | 自动识别科目列与科类 |
| `populateBatchTable` | 批次线配置表渲染 |
| `computeResult(cate)` | 按科类计算上线/尖子生/贡献度/优秀生数据 |
| `renderOverview` | 联考概览（信息卡片+批次线+各校数据表） |
| `renderCutoff(cate)` | 上线对比（人数柱状图 + 率值折线图两张独立图） |
| `renderRanking(cate)` | 尖子生排名（物理前30/历史前10学生成绩表） |
| `renderTopStudents(cate)` | 尖子生分层（学校维度柱状图 + 双Y轴 + 跨层趋势） |
| `renderTopLevelBars(cate, ci, cfg, schools, data, coreKeys, otherKeys)` | 尖子生分层内部图表渲染 |
| `renderTopPct(cate)` | 前X%均分对比（双Y轴柱状图 + 散点） |
| `renderElite(cate)` | 优秀生分布（所有阈值合并一张100%堆积图） |
| `renderContribution(cate)` | 学科贡献度（按学校独立气泡图 + 数据表） |
| `renderContribChart(cate, si, school, sc, subjKeys)` | 贡献度气泡图渲染 |
| `exportPDF` | ZIP 导出（Canvas 截图 + 表格转图片，全科类全子标签） |

### 辅助函数

| 函数 | 职责 |
|------|------|
| `destroyChart(key)` | 销毁单个 Chart.js 实例 |
| `destroyCharts(prefix)` | 按前缀批量销毁图表 |
| `getSchoolColor(nameOrIndex)` | 获取学校颜色 |
| `getSubjectColor(key)` | 获取科目颜色 |
| `chartToImage(chartInstance)` | Chart.js → PNG 数据 URL |
| `drawTableOnCanvas(headers, rows, colWidths, options)` | HTML 表格 → Canvas 图片 |
| `switchCutoff(cate)` | 切换上线对比科类 |
| `switchRankCate(cate)` | 切换尖子生排名科类 |
| `switchTopCate(cate)` | 切换尖子生分层科类 |
| `switchTopTab(cate, idx)` | 切换尖子生分层子标签（前30/100/200） |
| `switchPctCate(cate)` | 切换前X%均分科类 |
| `switchEliteCate(cate)` | 切换优秀生分布科类 |
| `switchEliteTab(cate, idx)` | 切换优秀生子标签 |
| `switchContribCate(cate)` | 切换学科贡献度科类 |
| `switchContribSchool(cate, idx)` | 切换贡献度学校 |
| `switchBatchCate(cate)` | 切换批次线科类 |

### 科类切换机制

每个分析板块独立管理物理/历史切换（`switchCutoff`、`switchRankCate`、`switchTopCate`、`switchPctCate`、`switchEliteCate`、`switchContribCate`），不再使用全局科类切换。

### 子标签机制

- **尖子生分层**：`switchTopTab` — 前30/100/200名选择，懒加载渲染
- **优秀生**：合并为一张图，不再有子标签（`switchEliteTab` 保留备用）
- **贡献度**：`switchContribSchool` — 按学校切换
- **联考概览**：无标签，一次显示所有信息

### 分析板块顺序

```
联考概览 → 各校上线对比 → 尖子生排名 → 尖子生分层均分对比
  → 前X%均分对比 → 优秀生分布 → 学科贡献度
```

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
贡献率 = 双上线人数 / 总分上线人数（反映该科对总上线的拉动）
命中率 = 双上线人数 / 该科参考人数（反映该科自身水平）
等级划分: 优秀(≥40%) / 良好(≥25%) / 一般(≥10%) / 较弱
```

## ZIP 导出流程

```
exportPDF()
  → 遍历物理/历史科类
    → 创建离屏 Canvas
    → 渲染各板块图表（Chart.js）
    → chartToImage() 截取 PNG
    → drawTableOnCanvas() 表格转图片
  → JSZip 打包
  → 下载 ZIP
```

## 样式约定

- **配色**: Anthropic 品牌色（深 #141413, 浅 #faf9f5, 橙 #d97757, 蓝 #6a9bcc, 绿 #788c5d）
- **字号**: ≥18px，图表标签 15-18px
- **布局**: 满宽图表 + 满宽表格上下排布，不左右分栏
- **打印**: Landscape A4, `page-break-before: always` 分段
- **图表尺寸**: 宽高比 2:1（趋势图 2.2:1）

## 已知限制

- 不处理综合科类汇总（仅物理/历史）
- 期望数据为逐行学生记录（非交叉表格式）
- 学校数限制为 5-6 校（颜色板预设）

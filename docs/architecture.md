# 架构设计与最终实现说明

## 1. 总体架构

### 1.1 架构模式

纯前端单页应用（SPA），单 HTML 文件承载全部代码（HTML + CSS + JavaScript）。

```
┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│   Excel 输入  │ → │  数据解析引擎  │ → │  分析计算引擎  │
└──────────────┘    └──────────────┘    └──────────────┘
                                               ↓
┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│   ZIP 导出    │ ← │  Chart.js 渲染 │ ← │  UI 配置与调度  │
└──────────────┘    └──────────────┘    └──────────────┘
```

### 1.2 无外部依赖

- 无需后端服务
- 无需构建工具
- 无需包管理器
- 全部通过 CDN 加载第三方库

## 2. 技术选型

| 技术 | 用途 | 版本 | 选型理由 |
|------|------|------|----------|
| Chart.js | 数据可视化 | 4.4.4 | 轻量、多图表类型、canvas 渲染、插件生态 |
| SheetJS (xlsx) | Excel 解析 | 0.18.5 | 唯一成熟的浏览器端 Excel 解析方案 |
| chartjs-plugin-datalabels | 图表数据标签 | 2.2.0 | 在柱/点上直接显示数值 |
| JSZip | 导出打包 | 3.10.1 | 纯 JS ZIP 生成，无需后端 |

## 3. 文件结构

```
exam-analysis/
├── exam-analysis.html      # 主应用（~3094 行，单文件）
├── AGENTS.md               # AI 代理架构参考
├── README.md               # 项目说明
├── CHANGELOG.md            # 版本更迭记录
├── docs/
│   ├── requirements.md     # 需求规格说明书
│   └── architecture.md     # 架构设计文档（本文）
├── .omc/
│   └── project-memory.json # 项目元数据
├── .gitignore
└── *.xlsx                  # 示例数据文件
```

## 4. 数据流详解

### 4.1 导入阶段

```
handleFile(file)
    → SheetJS: XLSX.read(data, {type:'array'})
    → 第一行为表头 → 列名映射
    → detectSubjects(headers) → 识别 学校/科类/科目/总分 列
    → _filteredRaw = 行数据对象数组
    → _categories = 去重科类列表 (物理/历史)
    → 渲染学校选择列表 + 批次线配置表
```

### 4.2 分析阶段

```
generateAnalysis()
    → 遍历 _categories
        → computeResult(cate)
            → getRowsByCategory(rows, cate) 筛选科类
            → analyzeCutoffs()  上线计算
            → getTopN()         尖子生筛选
            → getTopPct()       前X%筛选
            → analyzeContribution() 贡献度计算
            → 返回 result 对象
    → renderOverview()
    → renderCutoff(cate)
    → renderRanking(cate)
    → renderTopStudents(cate)
    → renderTopPct(cate)
    → renderElite(cate)
    → renderContribution(cate)
```

### 4.3 渲染阶段

每个 `render*` 函数的工作模式：

```
render*(cate)
    → destroyCharts(prefix)     清理旧图表实例
    → 读取 computeResult(cate)  获取分析数据
    → 构建表格 HTML（.data-table）
    → new Chart(canvas, config) 渲染 Chart.js 图表
    → 注册回调/绑定事件
```

### 4.4 导出阶段

```
exportPDF()
    → 遍历科类 物理/历史
        → 遍历各分析板块
            → createOffscreenCanvas()     创建离屏画布
            → new Chart(canvas, config)   渲染图表（离屏）
            → chartToImage(chartInstance) 截取为 PNG
            → chart.destroy()             释放
        → 遍历数据表格
            → drawTableOnCanvas() HTML表格 → Canvas → PNG
    → 所有 PNG 放入 JSZip
    → ZIP 下载
```

## 5. 核心算法

### 5.1 批次线上线判定

```
上线条件: 总分 ≥ 总分线 AND 每科 ≥ 各科单科线
```

4 条批次线，逐条判定。

### 5.2 贡献度算法

```
单上线:  某科成绩 ≥ 该科单科线
双上线:  总分上线 AND 该科上线
贡献率:  双上线人数 / 总分上线人数
命中率:  双上线人数 / 该科参考人数

等级:
  ┌──────────────┬──────────┐
  │  贡献率/命中率  │   等级    │
  ├──────────────┼──────────┤
  │    ≥ 40%     │   优秀    │
  │    ≥ 25%     │   良好    │
  │    ≥ 10%     │   一般    │
  │    < 10%     │   较弱    │
  └──────────────┴──────────┘
```

### 5.3 尖子生分层

```
前 N 名筛选: getTopN(rows, N) → 按总分降序取前 N 行
均分对比:    对每组（学校/层级）计算各科均分

核心科目: 语数外 (yw, sx, yy) + 总分
其他科目: 除语数外的选考科目
```

### 5.4 气泡图坐标轴缩放

```
贡献率 → X轴 (双上线人数 / 总分上线人数)
命中率 → Y轴 (双上线人数 / 参考人数)

坐标轴范围:
  min = dataMin - padding (≥ 0)
  max = dataMax + padding (≤ 100)
  padding = max(range * 0.15, 5)
```

## 6. 科类切换机制

每个分析板块的科类切换完全独立：

| 板块 | 切换函数 | 子标签 |
|------|----------|--------|
| 各校上线对比 | `switchCutoff(cate)` | — |
| 尖子生排名 | `switchRankCate(cate)` | — |
| 尖子生分层 | `switchTopCate(cate)` | 前30/100/200 (`switchTopTab`) |
| 前X%均分 | `switchPctCate(cate)` | — |
| 优秀生分布 | `switchEliteCate(cate)` | — |
| 学科贡献度 | `switchContribCate(cate)` | 按学校 (`switchContribSchool`) |
| 批次线配置 | `switchBatchCate(cate)` | — |

切换时自动调用 `destroyCharts(prefix)` 清理旧图表，避免内存泄漏。

## 7. 图表实例管理

```
chartInstances = {}   // 全局图表注册表

创建实例时: chartInstances[key] = chart
销毁实例时: chart.destroy() + delete chartInstances[key]
批量销毁:   destroyCharts(prefix) 按前缀匹配

常见前缀:
  cutoff_count_<cate>    上线人数柱状图
  cutoff_rate_<cate>     上线率折线图
  top1_<cate>_<ci>       尖子生分层核心科目图
  top2_<cate>_<ci>       尖子生分层其他科目图
  trend_<cate>           跨层趋势图
  pct_1_<cate>           前X%核心科目图
  pct_2_<cate>           前X%其他科目图
  elite_<cate>           优秀生分布图
  contrib_<cate>_<si>    贡献度气泡图
```

## 8. 样式系统

### 8.1 设计原则

- Anthropic 品牌色系
- ≥18px 大字号，适合课堂展示
- 满宽布局，不左右分栏
- 图表:表格 1:1 上下排布

### 8.2 CSS 变量

```css
:root {
  --color-dark: #141413;
  --color-light: #faf9f5;
  --color-orange: #d97757;
  --color-blue: #6a9bcc;
  --color-green: #788c5d;
  --color-mid-gray: #b0aea5;
}
```

### 8.3 打印样式

- Landscape A4
- `page-break-before: always` 每个分析板块分页
- 隐藏交互元素（按钮、tab 切换）
- 确保图表满宽打印

## 9. 第三方库集成

### 9.1 CDN 加载

```html
<script src="chart.js@4.4.4"></script>
<script src="xlsx@0.18.5"></script>
<script src="chartjs-plugin-datalabels@2.2.0"></script>
<script src="jszip@3.10.1"></script>
```

### 9.2 兼容性要求

- 浏览器: Chrome 90+, Edge 90+, Safari 15+
- 无需 Polyfill
- 离线部署需自行托管 CDN 资源

## 10. 已知限制

| 限制 | 说明 | 未来可能方案 |
|------|------|-------------|
| 综合科类汇总 | 仅物理/历史分别分析 | 新增「综合」科类处理 |
| 学校数限制 | 5-6 校（颜色板 9 色但过多影响可读性） | 动态生成颜色板 |
| 小题分析 | 不涉及 | 超出了本系统范围 |
| 数据格式 | 仅支持逐行学生记录 | 增加交叉表格式支持 |
| 导出格式 | 仅 ZIP（PNG 图片） | 增加真正 PDF 输出 |

## 11. 版本历史

| 版本 | 日期 | 概要 |
|------|------|------|
| v1.1.0 | 2025-05-04 | ZIP 导出重写、气泡图智能缩放、全科类导出支持、语法错误修复 |
| v1.0.0 | 2025-03-15 | 完整 7 大分析板块 + 联考概览、Excel 解析、科类切换、PDF 导出 |
| v0.1.0 | 2025-03-10 | 初始版本、基础功能原型 |

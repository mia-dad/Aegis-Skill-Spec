# PresentationSkill 规范

> 本文档是 [Aegis Skill DSL 规范](./README.md) 的 PresentationSkill 专属部分。
>
> **规范版本：2.2.0**

## 目录

- [1. 概述](#1-概述)
- [2. UI 展示语义 (ui)](#2-ui-展示语义-ui)
- [3. 完整示例](#3-完整示例)

---

## 1. 概述

**PresentationSkill**（展示技能）专注于 UI 渲染和数据展示，将上游 AtomicSkill 的结构化输出转换为用户可见的界面元素。

### 1.1 核心特点

- **展示导向**：专注于数据的可视化呈现
- **无结构化输出**：不包含 `output_schema`，UI 即其输出
- **可选数据转换**：支持通过 `steps`（特别是 prompt/template）进行数据转换或 LLM 生成展示内容
- **语义化 UI**：通过 `display` 类型声明展示语义，前端按规范渲染

### 1.2 文件结构

| 章节 | 必需 | 说明 |
|------|------|------|
| `# skill: <id>` | 是 | 技能唯一标识符 |
| `**version**` | 否 | 技能版本号，默认 `1.0.0` |
| `**ignore**` | 否 | 技能级异常处理，默认 `false` |
| `**type**: PresentationSkill` | 是 | 技能类型，必须显式声明 |
| `## description` | 否 | 技能描述 |
| `## capabilityTags` | 否 | 能力关键词列表 |
| `## input_schema` | 是 | 输入参数结构定义，对应上游 AtomicSkill 的 output_schema |
| `## steps` | 否 | 执行步骤列表，用于数据转换或 LLM 生成展示内容 |
| `## ui` | 是 | UI 展示语义定义 |

### 1.3 与 AtomicSkill 的协作

PresentationSkill 通常与 AtomicSkill 成对使用：

1. **AtomicSkill** 负责数据获取和业务逻辑，输出结构化数据
2. **PresentationSkill** 接收结构化数据，定义如何展示给用户

```
用户请求 → AtomicSkill (数据处理) → 结构化输出 → PresentationSkill (UI 渲染) → 用户界面
```

### 1.4 Steps 的使用场景

虽然 `## steps` 是可选的，但以下场景建议使用：

| 场景 | 推荐 Step 类型 | 说明 |
|------|---------------|------|
| 数据格式转换 | `template` | 将原始数据转换为展示格式 |
| LLM 生成展示内容 | `prompt` | 让 LLM 生成文本、摘要等 |
| 数据聚合计算 | `template` | 计算汇总值、百分比等 |
| 调用外部服务 | `tool` | 获取额外展示所需数据 |

---

## 2. UI 展示语义 (ui)

### 2.1 概述

`## ui` 章节定义技能的展示语义，前端根据这些语义渲染对应的 UI 组件。

Aegis 采用**聊天瀑布流**作为唯一页面容器：
- AtomicSkill 负责"数据结构"
- PresentationSkill 负责"语义展示"
- 前端不做智能猜测渲染，严格按 DSL 定义展示

### 2.2 基本语法

```yaml
display: <display_type>
mapping:
  <ui_field>: "{{data_field}}"
config:
  <config_key>: <value>
```

| 字段 | 必需 | 说明 |
|------|------|------|
| `display` | 是 | 展示类型，决定渲染的组件 |
| `mapping` | 是 | 数据字段映射 |
| `config` | 否 | 组件配置项 |

### 2.3 Display 类型

| 类型 | 说明 | 适用场景 |
|------|------|----------|
| `text` | 纯文本 | 简单文本输出 |
| `markdown` | Markdown 渲染 | 富文本内容 |
| `table` | 表格 | 结构化数据列表 |
| `chart` | 图表 | 数据可视化 |
| `card` | 卡片 | 单条记录展示 |
| `list` | 列表 | 多条简单记录 |
| `file` | 文件下载 | 文件链接展示 |
| `form` | 表单 | 用户输入收集 |
| `mixed` | 混合布局 | 复杂多组件页面 |

### 2.4 text 类型

纯文本展示，适用于简单消息输出。

```yaml
display: text
mapping:
  content: "{{message}}"
```

### 2.5 markdown 类型

渲染 Markdown 格式的富文本内容。

```yaml
display: markdown
mapping:
  content: "{{report}}"
```

### 2.6 table 类型

表格展示，适用于结构化数据列表。

```yaml
display: table
mapping:
  headers: "{{headers}}"
  rows: "{{data}}"
config:
  sortable: true
  pageable: true
  pageSize: 10
```

**Mapping 字段：**

| 字段 | 类型 | 说明 |
|------|------|------|
| `headers` | array | 表头列表 |
| `rows` | array | 数据行列表 |

**Config 配置：**

| 配置 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `sortable` | boolean | false | 是否支持排序 |
| `pageable` | boolean | false | 是否分页 |
| `pageSize` | number | 10 | 每页行数 |

### 2.7 chart 类型

图表展示，支持多种图表形式。

```yaml
display: chart
mapping:
  data: "{{chart_data}}"
config:
  type: line
  xAxis: date
  yAxis: amount
  title: 销售趋势
```

**Config 配置：**

| 配置 | 类型 | 说明 |
|------|------|------|
| `type` | string | 图表类型：`line`、`bar`、`pie`、`area` |
| `xAxis` | string | X 轴字段名 |
| `yAxis` | string | Y 轴字段名 |
| `title` | string | 图表标题 |

### 2.8 card 类型

卡片展示，适用于单条记录的详情展示。

```yaml
display: card
mapping:
  title: "{{order_id}}"
  subtitle: "{{customer_name}}"
  content: "{{order_summary}}"
  footer: "{{created_at}}"
config:
  style: elevated
```

**Mapping 字段：**

| 字段 | 类型 | 说明 |
|------|------|------|
| `title` | string | 卡片标题 |
| `subtitle` | string | 副标题（可选） |
| `content` | string | 主体内容 |
| `footer` | string | 底部信息（可选） |

### 2.9 list 类型

列表展示，适用于多条简单记录。

```yaml
display: list
mapping:
  items: "{{results}}"
  itemTemplate:
    title: "{{title}}"
    description: "{{snippet}}"
    link: "{{url}}"
config:
  style: numbered
```

**Config 配置：**

| 配置 | 类型 | 说明 |
|------|------|------|
| `style` | string | 列表样式：`numbered`、`bulleted`、`plain` |

### 2.10 file 类型

文件展示，用于文件下载链接。

```yaml
display: file
mapping:
  files: "{{files}}"
  fileTemplate:
    name: "{{file_name}}"
    url: "{{file_url}}"
    size: "{{file_size}}"
```

### 2.11 form 类型

表单展示，用于收集用户输入（通常与 await 步骤配合）。

```yaml
display: form
mapping:
  fields: "{{form_fields}}"
  submitLabel: "提交"
config:
  layout: vertical
```

### 2.12 mixed 类型

混合布局，支持在一个页面中组合多个展示组件。

```yaml
display: mixed
layout:
  - type: markdown
    mapping:
      content: "{{summary}}"
  - type: table
    mapping:
      headers: "{{headers}}"
      rows: "{{data}}"
  - type: chart
    mapping:
      data: "{{trend_data}}"
    config:
      type: line
      xAxis: date
      yAxis: amount
```

**Layout 结构：**

`layout` 是一个数组，每个元素定义一个展示块：

| 字段 | 必需 | 说明 |
|------|------|------|
| `type` | 是 | 展示类型 |
| `mapping` | 是 | 数据映射 |
| `config` | 否 | 组件配置 |

### 2.13 Mapping 语法

Mapping 使用模板语法引用数据字段：

```yaml
mapping:
  # 直接引用
  title: "{{report_title}}"

  # 引用 steps 输出
  content: "{{analysis}}"

  # 引用数组
  rows: "{{data}}"
```

**数据来源优先级：**

1. `## steps` 的输出（如果有 steps）
2. `## input_schema` 定义的输入字段

---

## 3. 完整示例

### 3.1 对话展示 PresentationSkill

~~~markdown
# skill: chat_display

**version**: 2.2.0
**type**: PresentationSkill

## description
对话结果的展示技能，将 AI 回答渲染为 Markdown 格式。

## capabilityTags
  - 聊天展示
  - 对话渲染

## input_schema

```yaml
content:
  type: string
  description: AI 助手回答内容
```

## ui

```yaml
display: markdown
mapping:
  content: "{{content}}"
```
~~~

### 3.2 搜索结果展示 PresentationSkill

~~~markdown
# skill: search_results_display

**version**: 2.2.0
**type**: PresentationSkill

## description
搜索结果的展示技能，将搜索结果渲染为列表形式。

## capabilityTags
  - 搜索结果展示
  - 列表展示

## input_schema

```yaml
results:
  type: array
  description: 搜索结果列表
  items:
    title:
      type: string
      description: 标题
    snippet:
      type: string
      description: 摘要
    link:
      type: string
      description: 链接
```

## ui

```yaml
display: list
mapping:
  items: "{{results}}"
  itemTemplate:
    title: "{{title}}"
    description: "{{snippet}}"
    link: "{{link}}"
config:
  style: numbered
```
~~~

### 3.3 订单确认展示 PresentationSkill

~~~markdown
# skill: order_confirmation_display

**version**: 2.2.0
**type**: PresentationSkill

## description
订单确认结果的展示技能

## capabilityTags
  - 订单展示
  - 确认结果

## input_schema

```yaml
level:
  type: string
  description: 提醒级别
title:
  type: string
  description: 提醒标题
content:
  type: string
  description: 订单处理结果
```

## ui

```yaml
display: card
mapping:
  title: "{{title}}"
  content: "{{content}}"
config:
  style: "{{level}}"
```
~~~

### 3.4 财务分析展示 PresentationSkill（混合布局）

~~~markdown
# skill: financial_analysis_display

**version**: 2.2.0
**type**: PresentationSkill

## description
财务分析报告的展示技能，包含标题、分析内容、数据表格和趋势图表。

## capabilityTags
  - 财务报告展示
  - 分析展示

## input_schema

```yaml
title:
  type: string
  description: 报告标题

analysis:
  type: string
  description: 分析内容

financial_table:
  type: array
  description: 财务数据表格
  items:
    region:
      type: string
      description: 区域
    amount:
      type: number
      description: 金额

trend_data:
  type: array
  description: 趋势数据
  items:
    date:
      type: string
      description: 日期
    amount:
      type: number
      description: 金额
```

## ui

```yaml
display: mixed
layout:
  - type: markdown
    mapping:
      content: "# {{title}}\n\n{{analysis}}"
  - type: table
    mapping:
      headers:
        - 区域
        - 金额
      rows: "{{financial_table}}"
    config:
      sortable: true
  - type: chart
    mapping:
      data: "{{trend_data}}"
    config:
      type: line
      xAxis: date
      yAxis: amount
      title: 财务趋势
```
~~~

### 3.5 销售报表展示 PresentationSkill

~~~markdown
# skill: sales_report_display

**version**: 2.2.0
**type**: PresentationSkill

## description
销售报表的展示技能

## capabilityTags
  - 销售报表展示
  - 表格展示

## input_schema

```yaml
headers:
  type: array
  description: 表头
  items:
    type: string
data:
  type: array
  description: 表格数据
  items:
    region:
      type: string
      description: 区域
    product:
      type: string
      description: 商品名称
    amount:
      type: number
      description: 销售量
summary:
  type: object
  description: 汇总信息
  total:
    type: number
    description: 总计
```

## steps

### step: prepare_summary_text

**type**: template
**varName**: summary_text

```template
销售总计：{{summary.total}} 件
```

## ui

```yaml
display: mixed
layout:
  - type: table
    mapping:
      headers: "{{headers}}"
      rows: "{{data}}"
    config:
      sortable: true
      pageable: true
      pageSize: 20
  - type: text
    mapping:
      content: "{{summary_text}}"
```
~~~

### 3.6 文件下载展示 PresentationSkill

~~~markdown
# skill: file_download_display

**version**: 2.2.0
**type**: PresentationSkill

## description
文件下载的展示技能

## capabilityTags
  - 文件下载
  - 导出展示

## input_schema

```yaml
files:
  type: array
  description: 生成的文件列表
  items:
    file_url:
      type: string
      description: 文件下载链接
    file_name:
      type: string
      description: 文件名
    file_size:
      type: number
      description: 文件大小（字节）
```

## steps

### step: format_file_info

**type**: prompt
**varName**: file_description

```prompt
请为以下文件生成友好的下载说明：

{{#for files}}
- {{file_name}} ({{file_size}} 字节)
{{/for}}

要求：
1. 简洁明了
2. 包含文件类型说明
3. 提示用户如何下载
```

## ui

```yaml
display: mixed
layout:
  - type: markdown
    mapping:
      content: "{{file_description}}"
  - type: file
    mapping:
      files: "{{files}}"
      fileTemplate:
        name: "{{file_name}}"
        url: "{{file_url}}"
        size: "{{file_size}}"
```
~~~

### 3.7 带 LLM 生成内容的 PresentationSkill

~~~markdown
# skill: data_insight_display

**version**: 2.2.0
**type**: PresentationSkill

## description
数据洞察展示技能，使用 LLM 生成数据分析见解

## capabilityTags
  - 数据洞察
  - 智能分析

## input_schema

```yaml
data:
  type: array
  description: 原始数据
  items:
    category:
      type: string
      description: 分类
    value:
      type: number
      description: 数值
    change:
      type: number
      description: 变化率
```

## steps

### step: generate_insight

**type**: prompt
**varName**: insight

```prompt
你是一位数据分析专家。请分析以下数据并生成洞察报告：

{{#for data}}
- {{category}}: {{value}} (变化率: {{change}}%)
{{/for}}

请提供：
1. 主要发现（2-3 点）
2. 值得关注的异常
3. 建议行动

使用 Markdown 格式，语言简洁专业。
```

## ui

```yaml
display: mixed
layout:
  - type: markdown
    mapping:
      content: "## 数据洞察\n\n{{insight}}"
  - type: table
    mapping:
      headers:
        - 分类
        - 数值
        - 变化率
      rows: "{{data}}"
    config:
      sortable: true
```
~~~

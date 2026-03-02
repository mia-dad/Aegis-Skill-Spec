# AtomicSkill 规范

> 本文档是 [Aegis Skill DSL 规范](./README.md) 的 AtomicSkill 专属部分。
>
> **规范版本：2.2.0**

## 目录

- [1. 概述](#1-概述)
- [2. 输出定义 (output_schema)](#2-输出定义-output_schema)
- [3. 审计日志](#3-审计日志)
- [4. 完整示例](#4-完整示例)

---

## 1. 概述

**AtomicSkill**（原子技能）是 Aegis 的默认技能类型，专注于数据处理和业务逻辑，输出结构化数据供下游技能（如 PresentationSkill）使用。

### 1.1 核心特点

- **数据处理导向**：专注于业务逻辑和数据转换
- **结构化输出**：必须定义 `output_schema`，输出可被下游技能作为输入
- **无 UI 展示**：不包含 `## ui` 章节，UI 展示由 PresentationSkill 负责
- **支持审计日志**：建议记录关键业务操作的审计日志

### 1.2 文件结构

| 章节 | 必需 | 说明 |
|------|------|------|
| `# skill: <id>` | 是 | 技能唯一标识符 |
| `**version**` | 否 | 技能版本号，默认 `1.0.0` |
| `**ignore**` | 否 | 技能级异常处理，默认 `false` |
| `**type**: AtomicSkill` | 是 | 技能类型（可省略，为默认值） |
| `## description` | 否 | 技能描述 |
| `## capabilityTags` | 否 | 能力关键词列表 |
| `## input_schema` | 否 | 输入参数结构定义 |
| `## output_schema` | 是 | 输出参数结构定义 |
| `## steps` | 是 | 执行步骤列表 |

---

## 2. 输出定义 (output_schema)

**`output_schema` 是 AtomicSkill 的必需字段**。每个 AtomicSkill 必须明确定义结构化的输出格式，以确保：
1. 系统进行输出校验
2. 下游技能（如 PresentationSkill）可正确解析输入
3. 为调用方提供完整的元数据

### 2.1 基本语法

```yaml
<field_name>:
  type: <string|number|boolean|datetime|resource|array|object>
  required: <true|false>
  description: <字段描述>
  traits:                    # 可选，能力特征
    - <trait_name>
  semantic_role: <role>      # 可选，语义角色
```

每个输出字段必须包含 `type` 和 `description`。`required` 默认为 `true`，可设为 `false` 标记可选输出。

**能力语义属性**（可选但推荐）：
- `traits`：描述字段支持的运算能力，用于条件表达式校验和 UI 生成
- `semantic_role`：描述字段的业务语义类别，用于技能自动编排

### 2.2 类型完整性规则

**当字段类型为 `array` 或 `object` 时，必须完整定义其子级结构，直到最末级节点的类型为基本类型（`string`、`number`、`boolean`、`datetime`、`resource`）。**

此规则与 `input_schema` 一致，确保技能输出可被下游技能作为输入正确解析。

### 2.3 数组类型与 items

与输入定义规则一致：`array` 类型使用 `items` 描述元素结构，子字段直接平铺。

#### 简单数组

```yaml
headers:
  type: array
  description: 表头列表
  items:
    type: string
```

#### 结构化数组

```yaml
results:
  type: array
  description: 搜索结果列表
  items:
    title:
      type: string
      description: 标题
    url:
      type: string
      description: URL链接
    score:
      type: number
      description: 相关性分数
```

### 2.4 对象类型

object 类型的子字段直接平铺在父字段下，不使用 `properties` 包装：

```yaml
summary:
  type: object
  description: 汇总信息
  content:
    type: string
    description: 结论
  total_sales:
    type: number
    description: 总销售额
  average_rate:
    type: number
    description: 平均达成率
```

### 2.5 输出校验规则

1. 系统根据 `output_schema` 定义进行输出校验
2. `required: true`（默认）的字段必须出现在输出中
3. `array` 类型**必须**使用 `items` 描述元素结构，`object` 类型**必须**平铺子字段，直到最末级为基本类型
4. 上一个技能的输出可能是下一个技能的输入，因此 input 和 output 的结构化规则保持一致

### 2.6 能力语义

AtomicSkill 的 `output_schema` 推荐使用能力语义来描述输出字段的运算能力和业务语义。详细说明参见 [README.md 2.10 能力语义系统](./README.md#210-能力语义系统)。

**带能力语义的输出定义示例：**

```yaml
price:
  type: number
  description: 商品价格
  traits:
    - comparable
    - scorable
  semantic_role: finance

analysis:
  type: string
  description: 分析结论
  traits:
    - emptiable
    - searchable
  semantic_role: summary

results:
  type: array
  description: 搜索结果
  traits:
    - countable
    - emptiable
  semantic_role: record_list
  items:
    title:
      type: string
      description: 标题
    score:
      type: number
      description: 相关性分数
      traits:
        - comparable
        - scorable
      semantic_role: score
```

### 2.7 输出定义示例

```yaml
title:
  type: string
  description: 报表标题
  traits:
    - emptiable

data:
  type: array
  description: 数据列表
  traits:
    - countable
  items:
    name:
      type: string
      description: 名称
    value:
      type: number
      description: 数值
      traits:
        - comparable
```

---

## 3. 审计日志

> **适用范围**：审计日志仅适用于 AtomicSkill。PresentationSkill 专注于 UI 展示，不需要记录审计日志。

### 3.1 审计日志的重要性

**审计日志是 Aegis AtomicSkill 区别于其他技能系统的核心特性之一**。在企业级应用中，技能执行过程的可追溯性、可审查性至关重要。审计日志确保：

1. **执行可追溯**：每个技能的执行过程都有完整记录，便于问题排查和责任追溯
2. **业务合规**：满足企业审计要求，提供技能执行的完整证据链
3. **运维可观测**：实时监控技能执行状态，快速定位异常
4. **质量保证**：通过日志分析持续优化技能性能和用户体验

**建议每个 AtomicSkill 至少输出一条审计日志**，记录关键业务操作或执行结果。

### 3.2 审计日志工具

Aegis 内置 `log` 工具用于输出审计日志，日志会自动关联当前执行上下文的层级归属信息：

| 归属字段 | 说明 |
|----------|------|
| `conversationId` | 对话 ID |
| `userGoal` | 用户目标 |
| `planId` | 执行计划 ID |
| `planStepId` | 计划步骤 ID |
| `skillId` | 技能 ID |
| `skillVersion` | 技能版本 |

### 3.3 使用语法

在 Step 中调用 `log` 工具：

~~~markdown
### step: audit_log

**type**: tool
**tool**: log

```yaml
input:
  level: "info"
  message: "处理完成，共处理 {{count}} 条记录"
  data:
    item_count: "{{count}}"
    region: "{{region}}"
output_schema:
  logged:
    type: boolean
    description: 是否成功记录日志
  level:
    type: string
    description: 日志级别
  timestamp:
    type: string
    description: 日志时间戳
```
~~~

### 3.4 日志级别

| 级别 | 说明 | 适用场景 |
|------|------|----------|
| `info` | 信息 | 正常业务操作、执行成功、关键节点记录 |
| `warn` | 警告 | 可恢复的异常、参数缺失降级、资源接近阈值 |
| `error` | 错误 | 执行失败、数据异常、业务规则违反 |

### 3.5 输入参数

| 参数 | 类型 | 必需 | 说明 |
|------|------|------|------|
| `level` | string | 否 | 日志级别（默认 `info`） |
| `message` | string | 是 | 日志消息，支持模板变量 |
| `data` | object | 否 | 附加数据，将被序列化为 JSON 存储 |

### 3.6 最佳实践

#### 1. 在关键业务节点记录日志

~~~markdown
### step: process_order

**type**: tool
**tool**: database.insert

```yaml
input:
  table: orders
  data:
    order_id: "{{order_id}}"
    amount: "{{total_amount}}"
output_schema:
  inserted:
    type: boolean
```

### step: log_order_created

**type**: tool
**tool**: log

```yaml
input:
  level: "info"
  message: "订单创建成功"
  data:
    order_id: "{{order_id}}"
    amount: "{{total_amount}}"
    customer: "{{customer_name}}"
output_schema:
  logged:
    type: boolean
```
~~~

#### 2. 记录执行结果

~~~markdown
### step: log_search_result

**type**: tool
**tool**: log

```yaml
input:
  level: "info"
  message: "搜索完成，返回 {{result_count}} 条结果"
  data:
    query: "{{query}}"
    result_count: "{{result_count}}"
    duration_ms: "{{duration}}"
output_schema:
  logged:
    type: boolean
```
~~~

#### 3. 使用条件记录异常

~~~markdown
### step: log_no_results

**type**: tool
**tool**: log

```yaml
when:
  expr: "{{result_count}} == 0"
input:
  level: "warn"
  message: "搜索无结果"
  data:
    query: "{{query}}"
output_schema:
  logged:
    type: boolean
```
~~~

### 3.7 注意事项

1. **避免记录敏感数据**：不要在日志中记录密码、密钥、身份证号等敏感信息
2. **消息简洁明了**：日志消息应清晰描述发生了什么，便于快速理解
3. **合理使用日志级别**：根据事件严重程度选择合适的级别，避免滥用 `error`
4. **附加数据结构化**：使用 `data` 参数传递结构化数据，便于后续分析和查询

---

## 4. 完整示例

### 4.1 简单对话 AtomicSkill

~~~markdown
# skill: chat

**version**: 2.2.0
**type**: AtomicSkill

## description
通用对话 Skill，用于回答用户的各类问题。

## capabilityTags
  - 回答问题
  - 聊天

## input_schema

```yaml
prompt: string
```

## output_schema

```yaml
content:
  type: string
  description: AI 助手回答内容
  traits:
    - emptiable
    - searchable
  semantic_role: content
```

## steps

### step: answer

**type**: prompt
**varName**: content

```prompt
你是一个友好的AI助手。请回答用户的问题。

用户问题：{{prompt}}

请给出简洁、准确的回答。
```
~~~

### 4.2 数据搜索 AtomicSkill

~~~markdown
# skill: simple_search

**version**: 2.2.0
**type**: AtomicSkill

## description
简单搜索 Skill，用于搜索相关信息。

## capabilityTags
  - 搜索
  - 实时信息
  - 新闻

## input_schema

```yaml
query:
  type: string
  required: true
  description: 搜索关键词
```

## output_schema

```yaml
results:
  type: array
  description: 搜索结果列表
  traits:
    - countable
    - emptiable
  semantic_role: document_list
  items:
    title:
      type: string
      description: 标题
      traits:
        - searchable
    snippet:
      type: string
      description: 摘要
      traits:
        - emptiable
        - searchable
      semantic_role: summary
    link:
      type: string
      description: 链接
      traits:
        - equatable
```

## steps

### step: search

**type**: tool
**tool**: search_api

```yaml
input:
  q: "{{query}}"
output_schema:
  results:
    type: array
    description: 搜索结果列表
    items:
      title:
        type: string
      snippet:
        type: string
      link:
        type: string
```
~~~

### 4.3 订单确认 AtomicSkill（人机交互）

~~~markdown
# skill: order_confirmation

**version**: 2.2.0
**type**: AtomicSkill

## description
订单确认示例 - 演示 await step 的人机交互功能

## capabilityTags
  - 商业订单

## input_schema

```yaml
order_id:
  type: string
  required: true
  description: 订单编号
product_name:
  type: string
  required: true
  description: 商品名称
quantity:
  type: number
  required: true
  description: 购买数量
unit_price:
  type: number
  required: true
  description: 单价
```

## output_schema

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

## steps

### step: calculate_total

**type**: template
**varName**: total_amount

```template
{{quantity * unit_price}}
```

### step: prepare_summary

**type**: template
**varName**: summary

```template
订单摘要：
- 订单编号：{{order_id}}
- 商品：{{product_name}}
- 数量：{{quantity}}
- 单价：¥{{unit_price}}
- 总金额：¥{{total_amount}}
```

### step: user_confirmation

**type**: await

```yaml
message: |
  {{summary}}

  请确认以上订单信息是否正确。
input_schema:
  confirm:
    type: boolean
    required: true
    description: 是否确认订单
  notes:
    type: string
    required: false
    description: 备注信息（可选）
```

### step: process_order

**type**: template
**varName**: order_result

```yaml
when:
  expr: "{{confirm}} == true"
```

```template
订单 {{order_id}} 已确认，总金额 ¥{{total_amount}}。
用户备注：{{notes}}
```

### step: cancel_order

**type**: template
**varName**: cancel_result

```yaml
when:
  expr: "{{confirm}} == false"
```

```template
订单 {{order_id}} 已取消。
```

### step: final_output

**type**: template
**varName**: content

```template
{
  "order_id": "{{order_id}}",
  "total_amount": {{total_amount}},
  "confirmed": {{confirm}},
  "user_notes": "{{notes}}"
}
```
~~~

### 4.4 财务分析 AtomicSkill（多步骤组合）

~~~markdown
# skill: financial_analysis

**version**: 2.2.0
**type**: AtomicSkill

## description
对企业财务状况进行分析，并生成结构化分析报告。

## capabilityTags
  - 财务分析
  - financial_analysis

## input_schema
```yaml
company:
  type: string
  required: true
period:
  type: string
  required: true
```

## output_schema
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

## steps

### step: fetch_financial_data

**type**: tool
**tool**: get_financial_data

```yaml
input:
  company: "{{company}}"
  period: "{{period}}"
output_schema:
  data:
    type: string
    description: 财务数据
```

### step: analyze_data

**type**: prompt
**varName**: analysis

```prompt
你是一位专业财务分析师。
请分析 {{company}} 在 {{period}} 的财务数据：
{{data}}
输出：
1. 分析内容
2. 风险提示
3. 建议
```

### step: prepare_output

**type**: template
**varName**: title

```template
{{company}} {{period}} 财务分析报告
```
~~~

### 4.5 销售报表 AtomicSkill

~~~markdown
# skill: sales_report

**version**: 2.2.0
**type**: AtomicSkill

## description
生成销售数据报表

## capabilityTags
  - 销售报表
  - 数据报表
  - 销售分析

## input_schema

```yaml
region:
  type: string
  required: true
  description: 地区名称
period:
  type: string
  required: true
  description: 报表周期
```

## output_schema

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

### step: fetch_sales_data

**type**: tool
**tool**: database.query

```yaml
input:
  query: "SELECT region, product, amount FROM sales WHERE region = '{{region}}'"
output_schema:
  data:
    type: array
    description: 查询结果
    items:
      region:
        type: string
      product:
        type: string
      amount:
        type: number
```

### step: format_report

**type**: template
**varName**: report

```template
{{region}} 地区 {{period}} 销售报表：

{{#for data}}
区域：{{region}}，商品：{{product}}，销售量：{{amount}}
{{/for}}
```
~~~

### 4.6 文件导出 AtomicSkill

~~~markdown
# skill: export_report

**version**: 2.2.0
**type**: AtomicSkill

## description
导出报表为文件

## capabilityTags
  - ppt
  - 文件生成
  - 数据导出

## input_schema

```yaml
report_id:
  type: string
  required: true
  description: 报表ID
format:
  type: string
  required: true
  description: 文件格式
  options:
    - PDF
    - Excel
    - CSV
```

## output_schema

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

### step: generate_file

**type**: tool
**tool**: file_generator

```yaml
input:
  report_id: "{{report_id}}"
  format: "{{format}}"
output_schema:
  files:
    type: array
    description: 生成的文件列表
    items:
      file_url:
        type: string
      file_name:
        type: string
      file_size:
        type: number
```

### step: format_response

**type**: template
**varName**: response

```template
文件已生成：
{{#for files}}
- 文件名：{{file_name}}
- 下载链接：{{file_url}}
- 大小：{{file_size}} 字节
{{/for}}
```
~~~

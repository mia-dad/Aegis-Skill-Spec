# AtomicSkill 规范

> 本文档是 [Aegis Skill DSL 规范](./README.md) 的 AtomicSkill 专属部分。
>
> **规范版本：3.4.2**

## 目录

- [1. 概述](#1-概述)
- [2. 输入输出定义 (input_schema/output_schema)](#2-输入输出定义-input_schemaoutput_schema)
- [3. UI 展示语义 (ui)](#3-ui-展示语义-ui)
- [4. 执行步骤 (steps)](#4-执行步骤-steps)
- [5. 审计日志](#5-审计日志)
- [6. 完整示例](#6-完整示例)

---

## 1. 概述

**AtomicSkill**（原子技能）是 Aegis 的默认技能类型，专注于数据处理和业务逻辑，输出结构化数据供下游技能使用。

### 1.1 核心特点

- **数据处理导向**：专注于业务逻辑和数据转换
- **结构化输出**：必须定义`input_schema`和 `output_schema`，其中输出可被下游技能作为输入
- **UI 展示**：包含 `## ui` 章节，UI 展示由 `## ui` 负责
- **支持审计日志**：建议在业务逻辑中记录关键业务操作的审计日志

### 1.2 文件结构

| 章节                      | 必需  | 说明                 |
| ----------------------- | --- | ------------------ |
| `# skill: <id>`         | 是   | 技能唯一标识符            |
| `**version**`           | 是   | 技能版本号，默认 `1.0.0`   |
| `**ignore**`            | 否   | 技能级异常处理，默认 `false` |
| `**type**: AtomicSkill` | 是   | 技能类型（可省略，为默认值）     |
| `## description`        | 否   | 技能描述               |
| `## capabilityTags`     | 否   | 能力关键词列表，用于技能路由和匹配 |
| `## input_schema`       | 是   | 输入参数结构定义           |
| `## output_schema`      | 是   | 输出参数结构定义           |
| `## steps`              | 是   | 执行步骤列表             |

---

## 2. 输入输出定义 (input_schema/output_schema)

**原则**
**`input_schema`和`output_schema` 是 AtomicSkill 的必需字段**。每个 AtomicSkill 必须明确定义结构化的输出格式，以确保：
1. 系统进行输入输出校验
2. 下游技能可正确解析输入
3. 为调用方提供完整的元数据
遵守总章（[[README.md]])中的第2章:输入输出的声明和定义的规定和规则。
### 2.1 基本语法


```yaml
<field_name>:
  type: <string|number|boolean|datetime|resource|array|object>
  required: <true|false>
  description: <字段描述>
  default: <默认值>           # 可选
  options: [选项1, 选项2, ...] # 可选，用于枚举约束
  roles:[选项1, 选项2, ...] # 可选
```


	每个字段必须包含 `type` 和 `description`。`required` 默认为 `true`，可设为 `false` 标记可选输出。

**能力角色**（可选但推荐）：
- `roles`：描述字段的业务语义类别，用于技能自动编排，详见 [README.md 第 2.3 节 — 语义角色](./README.md#23-语义角色-semantic-role)

> **重要**：语义角色是实现 CognitiveSkill 调用 AtomicSkill 的核心机制。如果你的 AtomicSkill 需要被 CognitiveSkill 编排调用，**必须**为 input_schema 和 output_schema 中的关键字段声明正确的语义角色。

### 2.2 类型完整性规则

**当字段类型为 `array` 时，必须通过 `items` 指定元素的语义类型。**

```yaml
# ✅ 必须通过 `items` 指定元素的基本类型
evidence_list:
  type: array
  description: 证据列表
  items: 
	type: string
```

**当字段类型为 `object` 时，必须平铺子字段，直到最末级为基本类型**

```yaml
# ✅ 必须平铺子字段，直到最末级为基本类型
summary:
  type: object
  description: 汇总信息
  content:
    type: string
  total_sales:
    type: number
```


### 2.3 声明校验规则

1. 系统根据 `input_schema`和`output_schema` 定义进行输出校验
2. `required: true`（默认）的字段必须出现在输出中
3. `array` 类型**必须**使用 `items` 描述元素结构，`object` 类型**必须**平铺子字段，直到最末级为基本类型
4. 上一个技能的输出可能是下一个技能的输入，因此 input 和 output 的结构化规则保持一致



---

## 3. UI 展示语义 (ui)

### 3.1 概述

`## ui` 章节定义技能的展示语义，前端根据这些语义渲染对应的 UI 组件。

Aegis 采用**聊天瀑布流**作为唯一页面容器：
- AtomicSkill 的output_schema负责"数据结构"
-  AtomicSkill 的ui负责"语义展示"
- 前端不做智能猜测渲染，严格按 DSL 定义展示

### 3.2 基本语法
~~~markdown
## ui 
```yaml
display: <display_type>
mapping:
  <ui_field>: "{{data_field}}"
config:
  <config_key>: <value>
```
~~~
| 字段        | 必需  | 说明           |
| --------- | --- | ------------ |
| `display` | 是   | 展示类型，决定渲染的组件 |
| `mapping` | 是   | 数据字段映射       |
| `config`  | 否   | 组件配置项        |

### 3.3 Display 类型

| 类型         | 说明          | 适用场景    |
| ---------- | ----------- | ------- |
| `text`     | 纯文本         | 简单文本输出  |
| `markdown` | Markdown 渲染 | 富文本内容   |
| `table`    | 表格          | 结构化数据列表 |
| `chart`    | 图表          | 数据可视化   |
| `card`     | 卡片          | 单条记录展示  |
| `list`     | 列表          | 多条简单记录  |
| `file`     | 文件下载        | 文件链接展示  |
| `form`     | 表单          | 用户输入收集  |
| `mixed`    | 混合布局        | 复杂多组件页面 |

### 3.4 text 类型

纯文本展示，适用于简单消息输出。

```yaml
display: text
mapping:
  content: "{{message}}"
```

### 3.5 markdown 类型

渲染 Markdown 格式的富文本内容。

```yaml
display: markdown
mapping:
  content: "{{report}}"
```

### 3.6 table 类型

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

| 字段        | 类型    | 说明    |
| --------- | ----- | ----- |
| `headers` | array | 表头列表  |
| `rows`    | array | 数据行列表 |

**Config 配置：**

| 配置         | 类型      | 默认值   | 说明     |
| ---------- | ------- | ----- | ------ |
| `sortable` | boolean | false | 是否支持排序 |
| `pageable` | boolean | false | 是否分页   |
| `pageSize` | number  | 10    | 每页行数   |

### 3.7 chart 类型

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

| 配置      | 类型     | 说明                             |
| ------- | ------ | ------------------------------ |
| `type`  | string | 图表类型：`line`、`bar`、`pie`、`area` |
| `xAxis` | string | X 轴字段名                         |
| `yAxis` | string | Y 轴字段名                         |
| `title` | string | 图表标题                           |

### 3.8 card 类型

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

| 字段         | 类型     | 说明       |
| ---------- | ------ | -------- |
| `title`    | string | 卡片标题     |
| `subtitle` | string | 副标题（可选）  |
| `content`  | string | 主体内容     |
| `footer`   | string | 底部信息（可选） |

### 3.9 list 类型

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

| 配置      | 类型     | 说明                                 |
| ------- | ------ | ---------------------------------- |
| `style` | string | 列表样式：`numbered`、`bulleted`、`plain` |

### 3.10 file 类型

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

### 3.11 form 类型

表单展示，用于收集用户输入（通常与 await 步骤配合）。

```yaml
display: form
mapping:
  fields: "{{form_fields}}"
  submitLabel: "提交"
config:
  layout: vertical
```

### 3.12 mixed 类型

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

### 3.13 Mapping 语法

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

1. `## steps` 的输出
2. `## input_schema` 定义的输入字段

---

## 4. 执行步骤 (steps)

### 4.1 概述

每个 Step 代表一个执行单元，按顺序执行。支持四种类型：

| 类型         | 说明                 | 是否调用 LLM |
| ---------- | ------------------ | -------- |
| `tool`     | 调用外部工具             | 否        |
| `prompt`   | 调用 LLM 生成响应        | 是        |
| `template` | 文本模板渲染（变量替换、表达式求值） | 否        |
| `await`    | 暂停执行，等待用户输入        | 否        |

### 4.2 Step 通用属性

#### Step 名称

```markdown
### step: <step_name>
```

名称使用下划线命名法，如 `calculate_total`、`prepare_summary`。

#### type（必填）

```markdown
**type**: <tool|prompt|template|await>
```

#### 变量引用方式

AtomicSkill 使用**步骤名前缀**的变量引用方式，每个步骤的输出都通过 `{{stepName.field}}` 语法访问。

**基本语法：**

```markdown
{{stepName.field}}
```

**各步骤类型的变量引用：**

| Step 类型 | 输出声明方式 | 引用语法 | 说明 |
|-----------|-------------|---------|------|
| `prompt` | 无需声明，自动输出 | `{{stepName.value}}` | LLM 响应的完整文本 |
| `template` | 无需声明，自动输出 | `{{stepName.value}}` | 模板渲染结果 |
| `tool` | `output_schema` 声明 | `{{stepName.field}}` | 工具输出的每个字段 |
| `await` | `input_schema` 声明 | `{{stepName.field}}` | 用户输入的每个字段 |

**`.value` 字段说明：**

`.value` 代表步骤的**整体输出**。对于 prompt 和 template 步骤，使用 `{{stepName.value}}` 引用步骤的完整输出结果。

**示例：**

~~~markdown
### step: calculate_total

**type**: template

```template
{{quantity * unit_price}}
```

### step: result_message

**type**: template

```template
订单 {{order_id}} 已确认，总金额 ¥{{calculate_total.value}}。
```
```
~~~
**步骤命名规则：**

- 使用下划线命名法（snake_case）
- 必须符合 `^[a-z][a-z0-9_]*$` 模式
- 不能与已有步骤名或输入字段名冲突
- 建议使用动词或动词短语，如 `calculate_total`、`prepare_summary`

**示例：**

~~~markdown
### step: fetch_data

**type**: tool
**tool**: search_api

```yaml
args:
  query: "{{search_term}}"
output_schema:
  results:
    type: array
    description: 搜索结果列表
  total:
    type: number
    description: 结果总数
```

### step: analyze

**type**: prompt

```prompt
请分析以下数据：
{{fetch_data.results}}
共 {{fetch_data.total}} 条记录。

请给出分析结果。
```
~~~

**变量作用域隔离：**

每个步骤的输出都被隔离在各自的作用域中，不会相互覆盖：

```markdown
### step: step1
**type**: prompt
```
输出内容A
```

### step: step2
**type**: prompt
```
输出内容B
```

### step: compare
**type**: template
```
步骤1输出：{{step1.value}}
步骤2输出：{{step2.value}}
```
```

**与 output_schema 的映射：**

步骤输出需要通过 `output_schema` 的 `mapping` 字段映射到最终输出。

详细映射规则请参见 [README.md - output_schema 的 mapping 映射机制](#https://github.com/your-repo/blob/main/README.md#221-output_schema-的-mapping-映射机制)。

**简要说明：**

| mapping 设置 | 行为 |
|------------|------|
| 未设置 | 系统从上下文中查找与字段名同名的变量 |
| 已设置 | 使用 mapping 指定的步骤输出引用 |

**示例：**

~~~markdown
## output_schema

```yaml
# 方式1：无 mapping，依赖同名查找
summary:
  type: string
  description: 摘要内容

# 方式2：有 mapping，显式指定来源
final_result:
  type: string
  description: 最终结果
  mapping: {{generate_summary.value}}
```

## steps

### step: generate_summary
**type**: prompt
```
生成摘要...
```
~~~

#### ignore 异常处理（步骤级）

使用 `ignore` 属性控制步骤执行失败时的行为。

**语法：**

```markdown
**type**: tool  
**ignore**: true
**tool**: external_api
```

或在 YAML 配置块中：

```yaml
ignore: true
args:
  query: "{{search_term}}"
```

**行为定义：**

| ignore 值 | 行为 |
|-----------|------|
| `false`（默认） | 步骤执行失败时抛出异常，终止整个 Skill 执行 |
| `true` | 步骤执行失败时，该步骤输出为 `null`，继续执行后续步骤 |

**适用的 Step 类型：**

| Step 类型 | 支持 ignore | 说明 |
|-----------|:-----------:|------|
| `tool` | ✓ | 工具调用失败（超时、异常、网络错误等） |
| `prompt` | ✓ | LLM 调用失败（超时、API 错误等） |
| `template` | — | 纯模板渲染，几乎不会失败 |
| `await` | — | 等待用户输入，不涉及失败 |

**使用示例：**

~~~markdown
### step: fetch_optional_data

**type**: tool  **ignore**: true
**tool**: external_api

```yaml
args:
  query: "{{search_term}}"
output_schema:
  extra_info:
    type: string
    description: 附加信息（可能为空）
```
~~~

> **注意**：当 `ignore: true` 时，步骤失败后输出为 `null`。后续步骤引用该输出时需考虑空值处理，可结合 `{{#when}}` 条件判断。

#### when 条件执行（步骤级）

使用 `when` 属性定义步骤执行条件，只有条件为 true 时才执行该步骤。

**语法：**

在 Step 的 YAML 配置块中定义：

```yaml
when:
  expr: "{{region}} == null"
```

**表达式语法：**

- 变量引用使用 `{{variable}}` 格式
- 整个表达式必须用**双引号**包裹（YAML 语法要求）
- 字符串字面量使用转义双引号 `\"`

**支持的操作符：**

| 操作符 | 说明 | 示例 |
|--------|------|------|
| `==` | 等于 | `"{{status}} == \"active\""` |
| `!=` | 不等于 | `"{{status}} != \"pending\""` |
| `>` | 大于 | `"{{count}} > 10"` |
| `<` | 小于 | `"{{count}} < 10"` |
| `>=` | 大于等于 | `"{{count}} >= 10"` |
| `<=` | 小于等于 | `"{{count}} <= 10"` |
| `&&` | 逻辑与 | `"{{a}} == true && {{b}} == false"` |
| `\|\|` | 逻辑或 | `"{{a}} == null \|\| {{b}} != \"\""` |

### 4.3 变量与模板语法

> **重要**：Aegis 使用自定义的模板引擎（AegisTemplateRenderer），**禁止使用 Mustache/Handlebars 语法**。系统不支持 `{{#each}}`、`{{#if}}`、`{{#unless}}`、`{{@index}}`、`{{this}}` 等 Mustache 语法，使用这些语法会导致模板无法正确渲染。

#### 禁止的 Mustache 语法与对应替代方案

| 禁止的语法 | 替代语法 | 说明 |
|------------|----------|------|
| `{{#each items}}` | `{{#for items}}` | 循环遍历 |
| `{{/each}}` | `{{/for}}` | 循环结束 |
| `{{#if condition}}` | `{{#when condition}}` | 条件渲染 |
| `{{/if}}` | `{{/when}}` | 条件结束 |
| `{{#unless condition}}` | `{{#when condition != true}}` | 否定条件 |
| `{{else}}` | `{{:else}}` | else 分支 |
| `{{@index}}` | `{{_index}}` | 循环索引 |
| `{{@key}}` | `{{_key}}` | 对象键名 |
| `{{this}}` | `{{_}}` | 当前元素 |
| `{{this.field}}` | `{{field}}` | 直接访问字段 |

#### 变量引用规则

**所有步骤输出** — 统一使用 `{{stepName.field}}` 语法引用：

```prompt
# 引用 tool 步骤输出
总销售额：{{calculate_total.total}}
用户确认：{{user_confirm.confirm}}
用户备注：{{user_confirm.notes}}

# 引用 prompt/template 步骤输出
报告内容：{{generate_report.value}}
消息文本：{{prepare_message.value}}
```

**变量存储规则：**

| Step 类型 | 访问语法 |
|-----------|---------|
| prompt | `{{stepName.value}}` |
| template | `{{stepName.value}}` |
| tool | `output_schema` 中定义的字段 | `{{stepName.field}}` |
| await | `input_schema` 中定义的字段 | `{{stepName.field}}` |

#### 模板语法

在 prompt 和 template 代码块中使用 `{{expression}}` 语法。支持变量引用和简单表达式求值：

**变量引用：**

```prompt
用户问题：{{query}}
报告：{{report}}
总销售额：{{total_sales}}
```

**表达式求值：**

`{{}}` 内支持简单算术运算和字符串拼接：

```prompt
总金额：{{quantity * unit_price}}
折后价：{{price * 0.8}}
全名：{{first_name + " " + last_name}}
```

支持的运算符：

| 运算符 | 说明 | 示例 |
|--------|------|------|
| `+` | 加法 / 字符串拼接 | `{{a + b}}`、`{{name + "_suffix"}}` |
| `-` | 减法 | `{{total - discount}}` |
| `*` | 乘法 | `{{quantity * unit_price}}` |
| `/` | 除法 | `{{total / count}}` |

**数组索引访问：**

通过 `[n]` 访问数组中指定位置的元素（索引从 0 开始）：

```prompt
第一条记录的商品：{{result[0].product}}
第三条记录的销售量：{{result[2].amount}}
```

当索引值存储在上下文变量中时，使用 `#` 前缀区分变量引用与字面量数字：

```prompt
当前记录的商品：{{result[#index].product}}
当前记录的销售量：{{result[#index].amount}}
```

| 语法 | 说明 |
|------|------|
| `{{arr[2]}}` | 字面量索引，访问第 3 个元素 |
| `{{arr[2].field}}` | 字面量索引 + 字段访问 |
| `{{arr[#var]}}` | 变量索引，`var` 为上下文中的数字变量 |
| `{{arr[#var].field}}` | 变量索引 + 字段访问 |

**数组字段投影：**

使用 `[*]` 语法从结构化数组中提取单一字段，生成一个简单数组：

```prompt
# 假设 result = [{content: "x", score: 80}, {content: "y", score: 90}]

所有分数：{{result[*].score}}
# 结果：[80, 90]

所有内容：{{result[*].value}}
# 结果：["x", "y"]
```

| 语法 | 说明 |
|------|------|
| `{{arr[*].field}}` | 提取数组中每个元素的 `field` 字段，组成新的简单数组 |

**循环渲染：**

使用 `{{#for array}}...{{/for}}` 遍历数组。

**结构化数组** — 循环体内直接引用元素的字段名：

```template
{{#for result}}
区域：{{region}}，商品：{{product}}，销售量：{{amount}}
{{/for}}
```

**简单数组** — 推荐使用 `{{_}}` 引用当前迭代元素：

```template
{{#for tags}}
- {{_}}
{{/for}}
```

**循环索引：**

在循环体内使用 `{{_index}}` 获取当前迭代的索引值（从 0 开始）：

```template
{{#for result}}
第 {{_index + 1}} 条：{{region}}，{{product}}，销售量：{{amount}}
{{/for}}
```

**循环内置变量汇总：**

| 变量 | 说明 |
|------|------|
| `{{_}}` | 当前迭代元素本身（推荐用于简单数组） |
| `{{_index}}` | 当前迭代索引（从 0 开始） |

**循环控制指令：**

使用 `{{#break}}` 提前终止循环。`#break` 可以独立使用，也可以嵌套在 `{{#when}}` 条件中实现条件中断。

**基本语法：**

```template
{{#for items}}
{{#when _index >= 3}}
{{#break}}
{{/when}}
第 {{_index + 1}} 项：{{name}}
{{/for}}
```

**条件中断示例：**

找到目标后立即退出循环：

```template
{{#for users}}
{{#when status == "active"}}
找到活跃用户：{{name}}
{{#break}}
{{/when}}
{{/for}}
```

**带累计逻辑的中断：**

```template
{{#for orders}}
订单 {{order_id}}：¥{{amount}}
{{#when _index >= 4}}
（仅显示前 5 条记录）
{{#break}}
{{/when}}
{{/for}}
```

**#break 语法规则：**

| 规则   | 说明                       |
| ---- | ------------------------ |
| 作用范围 | 仅在 `{{#for}}` 循环体内有效     |
| 执行效果 | 立即终止当前循环，继续执行循环后的内容      |
| 嵌套循环 | 仅终止最内层循环                 |
| 条件嵌套 | 可嵌套在 `{{#when}}` 中实现条件中断 |

**嵌套循环中的 break：**

```template
{{#for categories}}
分类：{{name}}
{{#for items}}
{{#when price > 1000}}
{{#break}}
{{/when}}
  - {{title}}: ¥{{price}}
{{/for}}
{{/for}}
```

> 上例中，`{{#break}}` 仅终止内层 `items` 循环，外层 `categories` 循环继续执行。

**条件渲染：**

使用 `{{#when condition}}...{{/when}}` 在模板内部进行条件渲染。

**基本语法：**

```template
{{#when condition}}
条件为真时显示的内容
{{/when}}
```

**带 else 分支：**

```template
{{#when condition}}
条件为真时显示
{{:else}}
条件为假时显示
{{/when}}
```

**条件表达式语法：**

- 条件表达式直接写在 `{{#when ...}}` 内部，**不需要**额外的 `{{}}`
- 字符串字面量使用双引号 `"`
- 支持与步骤级 when 相同的操作符

| 操作符 | 说明 | 示例 |
|--------|------|------|
| `==` | 等于 | `{{#when status == "active"}}` |
| `!=` | 不等于 | `{{#when count != 0}}` |
| `>` | 大于 | `{{#when score > 90}}` |
| `<` | 小于 | `{{#when _index < 3}}` |
| `>=` | 大于等于 | `{{#when rate >= 100}}` |
| `<=` | 小于等于 | `{{#when value <= 0}}` |
| `&&` | 逻辑与 | `{{#when a > 0 && b > 0}}` |
| `\|\|` | 逻辑或 | `{{#when x == 0 \|\| y == 0}}` |

**步骤级 when vs 模板级 {{#when}} 对比：**

| 特性 | 步骤级 `when` | 模板级 `{{#when}}` |
|------|--------------|-------------------|
| 作用范围 | 控制整个步骤是否执行 | 控制模板内部分内容是否渲染 |
| 语法位置 | YAML 配置块内 | 模板代码块内 |
| 变量引用 | 需要 `{{variable}}` | 直接使用变量名 |
| 字符串引号 | 转义双引号 `\"` | 普通双引号 `"` |
| 示例 | `expr: "{{status}} == \"done\""` | `{{#when status == "done"}}` |

---

### 4.4. Step 类型详解

#### 4.4.1 Tool 步骤

调用已注册的外部工具（如：访问HTTP API、访问数据库、调用内部业务服务等）。工具执行后会将输出写入执行上下文，后续步骤可通过输出的 key 名直接引用。

##### 配置格式
~~~markdown
## steps

### step: <step_name>

**type**: tool
**tool**: <tool_name>

```yaml
args:
  <parameter_name>: <value_template>
output_schema:
  <output_key>:
    type: <type>
    description: <说明>
```
~~~
| 字段 | 必需 | 说明 |
|------|------|------|
| `type` | 是 | 必须为 `tool` |
| `tool` | 是 | 工具名称（需在平台中已注册） |
| `args` | 是 | 工具输入参数（支持模板语法） |
| `output_schema` | 是 | 声明工具输出的 key 及其类型 |

**输出访问：** Tool 步骤的输出通过 `{{stepName.field}}` 访问，其中 `field` 是 `output_schema` 中定义的字段名。

Tool 调用格式说明
 Tool在调用时需要解决两个问题：
  1. 参数绑定（运行时）：把 DSL 上下文的值传给 Tool
     → 通过args实现入参的数据绑定

  2. 输出声明（设计时）：补充声明 Tool 的输出结构
     →  output_schema

##### 输入参数绑定

在 YAML 配置块中，`args` 键下定义工具的输入参数：

```yaml
args:
  region: "{{region}}"
  period: "{{period}}"
output_schema:
  total_sales:
    type: number
    description: 总销售额
```

##### 参数类型

| YAML 中的值类型 | 说明 | 示例 |
|-----------------|------|------|
| 字符串 | 文本值，支持模板变量 | `q: "{{query}}"` |
| 数字 | 整数或小数 | `count: 10` |
| 布尔值 | true 或 false | `enabled: true` |
| 数组 | 列表结构 | 见下方示例 |
| 对象 | 键值对结构 | 嵌套的 YAML 对象 |
| null | 空值 | `value: null` |

**数组参数示例：**

```yaml
args:
  tags:
    - "{{tag1}}"
    - "{{tag2}}"
    - fixed_tag
```

#### 4.4.2 Prompt 步骤

将 Prompt 模板发送给 LLM，获取生成结果。

##### 配置格式

~~~markdown
### step: <step_name>

**type**: prompt

```prompt
<prompt_template>
```
~~~

| 字段 | 必需 | 说明 |
|------|------|------|
| `type` | 是 | 必须为 `prompt` |
| ` ```prompt` 代码块 | 是 | Prompt 模板内容 |

**输出访问：** Prompt 步骤的输出通过 `{{stepName.value}}` 访问。

##### 示例

~~~markdown
### step: analyze_data

**type**: prompt

```prompt
请分析 {{company}} 在 {{period}} 期间的财务数据：
{{fetch_data.data}}

请给出专业分析。
```
~~~

后续引用：
```prompt
分析结果：{{analyze_data.value}}
```

```prompt
你是一位专业的财务分析师。

请分析 {{company}} 在 {{period}} 期间的财务数据：
{{data}}

请给出专业分析，包括：
1. 关键财务指标解读
2. 风险提示
3. 建议措施
```
~~~

#### 4.4.3 Template 步骤

纯文本模板渲染，只做变量替换，**不调用 LLM 也不调用 Tool**。
在某些需求上，如果需要对步骤、变量进行compose组合，也是通过此步骤来实现。

适用场景：
- 拼接变量和文字生成一段文本（如订单确认、通知消息）
- 组合多个前置步骤的输出为统一格式
- 不需要 LLM 智能生成的纯模板渲染

##### 配置格式

~~~markdown
### step: <step_name>

**type**: template

```template
模板内容，支持 {{variable}} 语法
```
~~~

| 字段 | 必需 | 说明 |
|------|------|------|
| `type` | 是 | 必须为 `template` |
| ` ```template` 代码块 | 是 | 模板内容 |

**输出访问：** Template 步骤的输出通过 `{{stepName.value}}` 访问。

##### Template vs Prompt

| 特性       | Template   | Prompt      |
| -------- | ---------- | ----------- |
| 是否调用 LLM | 否          | 是           |
| 输出可控性    | 完全确定性      | 不确定（LLM 生成） |
| 输出类型     | 字符类型结果     | 字符类型结果      |
| 性能       | 极快（纯字符串替换） | 较慢（网络调用）    |
| 适用场景     | 格式固定的文本拼接  | 需要智能生成的内容   |

#### 4.4.4 Await 步骤

暂停 Skill 执行，等待用户提供额外输入后继续执行。用于人机交互控制流场景。

##### 配置格式

~~~markdown
### step: <step_name>

**type**: await

```yaml
message: |
  <提示信息>
input_schema:
  <field_name>:
    type: <SemanticType>
    required: <true|false>
    description: <描述>
```
~~~

| 字段 | 必需 | 说明 |
|------|------|------|
| `type` | 是 | 必须为 `await` |
| `message` | 是 | 向用户展示的提示信息 |
| `input_schema` | 是 | 用户输入的结构定义，遵循语义类型规范 |

**输出访问：** 用户提交输入后，`input_schema` 中定义的每个字段通过 `{{stepName.field}}` 访问。

##### Await 输入的语义类型

`await` 的 `input_schema` 同样遵循语义类型规范，输入字段分为两类：

| 类型         | 说明          | 语义类型选择        |
| ---------- | ----------- | ------------- |
| **业务数据补充** | 补充技能所需的业务数据 | 使用对应的业务语义类型   |
| **过程控制确认** | 影响流程走向的临时决策 | 使用 `Decision` |

**示例**：

```yaml
input_schema:
  confirm:
    type: boolean        # 过程控制 - 用户确认
    required: true
    description: 是否确认订单
  notes:
    type: string          # 业务数据 - 用户补充的备注
    required: false
    description: 备注信息（可选）
```


## 5. 审计日志

> **适用范围**：审计日志仅适用于 AtomicSkill。CognitiveSkill 无需审计日志。

### 5.1 审计日志的重要性

**审计日志是 Aegis AtomicSkill 区别于其他技能系统的核心特性之一**。在企业级应用中，技能执行过程的可追溯性、可审查性至关重要。审计日志确保：

1. **执行可追溯**：每个技能的执行过程都有完整记录，便于问题排查和责任追溯
2. **业务合规**：满足企业审计要求，提供技能执行的完整证据链
3. **运维可观测**：实时监控技能执行状态，快速定位异常
4. **质量保证**：通过日志分析持续优化技能性能和用户体验

**建议每个 AtomicSkill 至少输出一条审计日志**，记录关键业务操作或执行结果。

### 5.2 审计日志工具

Aegis 内置 `log` 工具用于输出审计日志，日志会自动关联当前执行上下文的层级归属信息：

| 归属字段             | 说明      | DSL是否需要处理 |
| ---------------- | ------- | --------- |
| `conversationId` | 对话 ID   | 无需        |
| `userGoal`       | 用户目标    | 无需        |
| `planId`         | 执行计划 ID | 无需        |
| `planStepId`     | 计划步骤 ID | 无需        |
| `skillId`        | 技能 ID   | 无需        |
| `skillVersion`   | 技能版本    | 无需        |

### 5.3 使用语法

在 Step 中调用 `log` 工具：

~~~markdown
### step: audit_log

**type**: tool
**tool**: log

```yaml
args:
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

### 5.4 日志级别

| 级别 | 说明 | 适用场景 |
|------|------|----------|
| `info` | 信息 | 正常业务操作、执行成功、关键节点记录 |
| `warn` | 警告 | 可恢复的异常、参数缺失降级、资源接近阈值 |
| `error` | 错误 | 执行失败、数据异常、业务规则违反 |

### 5.5 输入参数

| 参数 | 类型 | 必需 | 说明 |
|------|------|------|------|
| `level` | string | 否 | 日志级别（默认 `info`） |
| `message` | string | 是 | 日志消息，支持模板变量 |
| `data` | object | 否 | 附加数据，将被序列化为 JSON 存储 |

### 5.6 最佳实践

#### 1. 在关键业务节点记录日志

~~~markdown
### step: process_order

**type**: tool
**tool**: database.insert

```yaml
args:
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
args:
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
args:
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
args:
  level: "warn"
  message: "搜索无结果"
  data:
    query: "{{query}}"
output_schema:
  logged:
    type: boolean
```
~~~

### 5.7 注意事项

1. **避免记录敏感数据**：不要在日志中记录密码、密钥、身份证号等敏感信息
2. **消息简洁明了**：日志消息应清晰描述发生了什么，便于快速理解
3. **合理使用日志级别**：根据事件严重程度选择合适的级别，避免滥用 `error`
4. **附加数据结构化**：使用 `data` 参数传递结构化数据，便于后续分析和查询

---

## 6. 完整示例

### 6.1 简单对话 AtomicSkill

~~~markdown
# skill: chat

**version**: 3.4.2
**type**: AtomicSkill

## description
通用对话 Skill，用于回答用户的各类问题。

## capabilityTags
  - 回答问题
  - 聊天

## input_schema

```yaml
prompt:
  type: string
  required: true
  description: 用户输入的问题
  roles:
    - Direct.Query
    - CoT.Query
```

## output_schema

```yaml
content:
  type: string
  description: AI 助手回答内容
  roles:
    - Direct.Answer
    - CoT.Conclusion
```

## steps

### step: answer

**type**: prompt

```prompt
你是一个友好的AI助手。请回答用户的问题。

用户问题：{{prompt}}

请给出简洁、准确的回答。
```
```
~~~

### 6.2 数据搜索 AtomicSkill

~~~markdown
# skill: simple_search

**version**: 3.4.2
**type**: AtomicSkill

## description
简单搜索 Skill，输出列表，用于搜索相关信息

## capabilityTags
  - 搜索
  - 实时信息
  - 新闻
  - list

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
  items:
      title:
        type: string
      snippet:
        type: string
      link:
        type: string
```

## steps

### step: search

**type**: tool
**tool**: search_api

```yaml
args:
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

### 6.3 订单确认 AtomicSkill（带人机交互）

~~~markdown
# skill: order_confirmation

**version**: 3.4.2
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
  roles:
    - Direct.Answer
```

## steps

### step: calculate_total

**type**: template

```template
{{quantity * unit_price}}
```

### step: prepare_summary

**type**: template

```template
订单摘要：
- 订单编号：{{order_id}}
- 商品：{{product_name}}
- 数量：{{quantity}}
- 单价：¥{{unit_price}}
- 总金额：¥{{calculate_total.value}}
```

### step: user_confirmation

**type**: await

```yaml
message: |
  {{prepare_summary.value}}

  请确认以上订单信息是否正确。
input_schema:
  confirm:
    type: boolean
    required: true
    description: 是否确认订单（过程控制）
  notes:
    type: string
    required: false
    description: 备注信息（可选）
```

### step: process_order

**type**: template

```yaml
when:
  expr: "{{user_confirmation.confirm}} == true"
```

```template
订单 {{order_id}} 已确认，总金额 ¥{{calculate_total.value}}。
用户备注：{{user_confirmation.notes}}
```

### step: cancel_order

**type**: template

```yaml
when:
  expr: "{{user_confirmation.confirm}} == false"
```

```template
订单 {{order_id}} 已取消。
```

### step: final_output

**type**: template

```template
{
  "order_id": "{{order_id}}",
  "total_amount": {{calculate_total.value}},
  "confirmed": {{user_confirmation.confirm}},
  "user_notes": "{{user_confirmation.notes}}"
}
```
~~~


### 6.4 销售报表 AtomicSkill

~~~markdown
# skill: sales_report

**version**: 3.4.2
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
    product:
      type: string
    amount:
      type: number
summary:
  type: string
  description: 汇总信息
  roles:
    - Generate.Output
```

## steps

### step: fetch_sales_data

**type**: tool
**tool**: database.query

```yaml
args:
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

```template
{{region}} 地区 {{period}} 销售报表：

{{#for fetch_sales_data.data}}
区域：{{region}}，商品：{{product}}，销售量：{{amount}}
{{/for}}
```
~~~

### 6.5 文件导出 AtomicSkill

~~~markdown
# skill: export_report

**version**: 3.4.2
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
    file_name:
      type: string
    file_size:
      type: number
```

## steps

### step: generate_file

**type**: tool
**tool**: file_generator

```yaml
args:
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

```template
文件已生成：
{{#for generate_file.files}}
- 文件名：{{file_name}}
- 下载链接：{{file_url}}
- 大小：{{file_size}} 字节
{{/for}}
```
~~~

# Aegis Skill DSL 规范说明文档

> **规范版本：1.0.0**

## 1. 概述

Aegis Skill DSL 是一种基于 Markdown 的领域特定语言，用于定义可执行的 AI 技能。每个 Skill 由输入定义、执行步骤和输出定义组成。

### 1.1 技能文件骨架

```markdown
# skill: <skill_id>

**version**: <版本号>

## description

<技能描述>

## capabilityTags

- <能力关键词1>
- <能力关键词2>

## input_schema

```yaml
<输入参数定义>
```

## output_schema

```yaml
<输出参数定义>
```

## steps

### step: <step_name>

**type**: <step_type>

<step_configuration>
```

### 1.2 技能文件顶层章节

| 章节 | 必需 | 说明 |
|------|------|------|
| `# skill: <id>` | 是 | 技能唯一标识符，作为一级标题 |
| `**version**: <版本号>` | 否 | 技能版本号，遵循语义化版本（如 `1.0.0`）。其取值对应本规范文档开篇的版本号。`skillId + version` 构成全局唯一标识，同一 skillId 可存在多个版本。未提供时默认为 `1.0.0` |
| `## description` | 否 | 技能描述文本 |
| `## capabilityTags` | 否 | 能力关键词列表，用于技能路由和匹配 |
| `## input_schema` | 否 | 输入参数结构定义（YAML 格式） |
| `## output_schema` | 是 | 输出参数结构定义（YAML 格式，平铺字段定义） |
| `## steps` | 是 | 执行步骤列表（至少一个） |

### 1.3 version 字段

`version` 字段声明在一级标题 `# skill: <id>` 之后，格式为：

```markdown
**version**: 1.0.0
```

**用法说明：**

- **取值来源**：version 的值对应本规范文档（SKILL_DSL_SPECIFICATION.md）开篇的版本号。当规范版本为 `1.0.0` 时，技能文件的 version 即为 `1.0.0`
- **唯一标识**：`skillId + version` 构成全局唯一标识。同一个 skillId 可以存在多个版本（如 `1.0.0`、`1.1.0`、`2.0.0`）
- **语义化版本**：遵循 `major.minor.patch` 格式
  - `major`：不兼容的结构性变更（如输入/输出 schema 变化）
  - `minor`：向后兼容的功能新增（如新增可选输入字段）
  - `patch`：向后兼容的缺陷修复（如 prompt 文案调整）
- **默认值**：未声明 version 时，系统默认为 `1.0.0`
- **版本查找**：通过 API 按 skillId 查询时，未指定版本则返回版本号最大的 Skill

---

## 2. 输入定义 (input_schema)

### 2.1 基本语法

```yaml
<field_name>:
  type: <string|number|boolean|array|object>
  required: <true|false>
  description: <字段描述>
  default: <默认值>           # 可选
  options: [选项1, 选项2, ...] # 可选，用于单选/多选
  placeholder: <占位提示>      # 可选
  label: <显示标签>            # 可选
  validation: <验证规则>       # 可选
```

### 2.2 支持的类型

| 类型 | 说明 | 示例值 |
|------|------|--------|
| `string` | 字符串 | `"hello"` |
| `number` | 数字 | `123`, `45.67` |
| `boolean` | 布尔值 | `true`, `false` |
| `array` | 数组 | `[1, 2, 3]` |
| `object` | 对象（键值对） | `{name: "张三", age: 30}` |

### 2.3 字段属性

| 属性 | 类型 | 必需 | 说明 |
|------|------|------|------|
| `type` | string | 是 | 字段数据类型 |
| `required` | boolean | 否 | 是否必填（默认 true） |
| `description` | string | 否 | 字段描述信息 |
| `default` | any | 否 | 字段默认值 |
| `options` | array | 否 | 枚举选项列表，用于单选/多选 |
| `placeholder` | string | 否 | 输入框占位提示 |
| `label` | string | 否 | 前端显示标签 |
| `validation` | object | 否 | 验证规则 |

### 2.4 数组类型与 items

当字段类型为 `array` 时，使用 `items` 描述元素结构。

#### 简单数组（元素为基本类型）

```yaml
tags:
  type: array
  description: 标签列表
  items:
    type: string
```

#### 结构化数组（元素为复合结构）

`items` 下直接平铺子字段，不使用 `properties` 包装：

```yaml
contacts:
  type: array
  description: 联系人列表
  items:
    name:
      type: string
      description: 姓名
    phone:
      type: string
      description: 电话
```

#### 多层嵌套

`items` 可以多层嵌套，用于描述复杂数据结构：

```yaml
departments:
  type: array
  description: 部门列表
  items:
    name:
      type: string
      description: 部门名称
    members:
      type: array
      description: 成员列表
      items:
        name:
          type: string
        role:
          type: string
```

> **说明**：`items` 为可选属性。如果省略，系统不会约束数组元素的结构。当使用 `options` 实现多选时，不需要定义 `items`。

### 2.5 对象类型

当字段类型为 `object` 时，子字段直接平铺在父字段下，不使用 `properties` 包装（与 output_schema 规则一致）：

```yaml
address:
  type: object
  description: 地址信息
  city:
    type: string
    description: 城市
  street:
    type: string
    description: 街道
```

### 2.6 单选和多选（options）

通过 `options` 属性配合 `type` 字段实现单选和多选功能。

#### 单选（type: string + options）

当 `type` 为 `string` 并提供 `options` 时，前端应渲染为单选下拉框：

```yaml
report_type:
  type: string
  required: true
  description: 报表类型
  options:
    - 销售报表
    - 财务报表
    - 库存报表
  default: 销售报表
```

#### 多选（type: array + options）

当 `type` 为 `array` 并提供 `options` 时，前端应渲染为多选下拉框或复选框组：

```yaml
regions:
  type: array
  required: true
  description: 选择地区
  options:
    - 华北
    - 华东
    - 华南
    - 西部
  default:
    - 华北
    - 华东
```

### 2.7 简写语法

当字段仅需声明类型时，可使用简写格式（默认 `required: true`，无描述）：

```yaml
query: string
prompt: string
context: string
```

### 2.8 输入定义示例

```yaml
order_id:
  type: string
  required: true
  description: 订单编号
  placeholder: "请输入订单编号"

product_name:
  type: string
  required: true
  description: 商品名称
  label: "商品"

quantity:
  type: number
  required: true
  description: 购买数量
  default: 1
  validation:
    min: 1
    max: 999

unit_price:
  type: number
  required: false
  description: 单价
  default: 0.0

report_format:
  type: string
  required: true
  description: 报表格式
  options:
    - PDF
    - Excel
    - HTML
  default: PDF

tags:
  type: array
  required: false
  description: 标签
  options:
    - 重要
    - 紧急
    - 待审核
```

---

## 3. 输出定义 (output_schema)

**注意**：`output_schema` 是**必需字段**。每个 Skill 必须明确定义结构化的输出格式，以确保系统进行输出校验，并为调用方提供完整的元数据。

### 3.1 基本语法

```yaml
<field_name>:
  type: <string|number|boolean|array|object>
  required: <true|false>
  description: <字段描述>
```

每个输出字段必须包含 `type` 和 `description`。`required` 默认为 `true`，可设为 `false` 标记可选输出。

### 3.2 数组类型与 items

与输入定义规则一致：`array` 类型使用 `items` 描述元素结构，子字段直接平铺。

#### 简单数组

```yaml
headers:
  type: array
  description: 表头列表
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

#### object 类型

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

### 3.3 输出校验规则

1. 系统根据 `output_schema` 定义进行输出校验
2. `required: true`（默认）的字段必须出现在输出中
3. `array` 类型使用 `items` 描述元素结构，`object` 类型平铺子字段
4. 上一个技能的输出可能是下一个技能的输入，因此 input 和 output 的结构化规则保持一致

### 3.4 输出定义示例

**示例 1：图表数据输出**

```yaml
chart_type:
  type: string
  description: 图表类型（line, bar, pie 等）
title:
  type: string
  description: 图表标题
data:
  type: object
  description: 图表数据
options:
  type: object
  description: 图表配置选项
```

**示例 2：文件下载输出**

```yaml
file_url:
  type: string
  description: 文件下载链接
file_name:
  type: string
  description: 文件名
file_size:
  type: number
  description: 文件大小（字节）
mime_type:
  type: string
  description: MIME 类型
```

**示例 3：图文混排输出**

```yaml
title:
  type: string
  description: 文档标题
content:
  type: array
  description: 文档内容块（文本、图片、表格等）
  items:
    block:
      type: string
      description: 块类型（paragraph, chart, image）
    text:
      type: string
      description: 文本内容（paragraph 类型）
    chart:
      type: object
      description: 图表数据（chart 类型）
metadata:
  type: object
  description: 文档元数据
```

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

#### varName

`varName` 用于将步骤输出以指定键名直接存入变量上下文，可通过 `{{varName}}` 直接访问。

| Step 类型 | varName | 说明 |
|-----------|---------|------|
| `prompt` | **必填** | LLM 响应以 varName 为键存入上下文 |
| `template` | **必填** | 渲染结果以 varName 为键存入上下文 |
| `await` | **不适用** | `input_schema` 中的字段名直接作为 key 写入上下文 |
| `tool` | **不适用** | 工具通过 `ToolOutputContext.put(key, value)` 直接写入上下文，输出 key 由工具实现决定 |

语法：
```markdown
**type**: template  **varName**: report
```

特性：
- **直接访问**：通过 `{{varName}}` 访问输出，无需 `.value` 后缀
- **与 output_schema 配合**：varName 与 output_schema 中定义的字段名对齐，使 `buildOutputFromContext` 能正确匹配
- **命名规则**：必须符合 `^[a-z][a-z0-9_]*$` 模式，且不能与已有 stepName、其他 varName 或输入字段名冲突

#### when 条件执行

使用 `**when**` 属性定义条件，只有条件为 true 时才执行步骤。

**方式一：在 markdown 段落中定义（适用于简单表达式）**

```markdown
**when**: confirm == true
```

**方式二：在 YAML 配置块中定义（适用于包含模板变量的表达式）**

```yaml
when:
  expr: "{{region}} == null"
```

**⚠️ 注意**：当 `when` 条件包含模板变量（如 `{{var}}`）时，必须写在 YAML 配置块中并用**双引号包裹**表达式，因为 YAML 会将 `{{` 解释为流集合语法。

**支持的操作符：**

| 操作符 | 说明 | 表达式示例 | YAML 中写法 |
|--------|------|------------|------------|
| `==` | 等于 | `{{var}} == "value"` | `"{{var}} == \"value\""` |
| `!=` | 不等于 | `{{var}} != "value"` | `"{{var}} != \"value\""` |
| `&&` | 逻辑与 | `{{a}} == true && {{b}} == false` | `"{{a}} == true && {{b}} == false"` |
| `\|\|` | 逻辑或 | `{{a}} == null \|\| {{b}} != ""` | `"{{a}} == null \|\| {{b}} != \"\""` |
| `>` | 大于 | `{{count}} > 10` | `"{{count}} > 10"` |
| `<` | 小于 | `{{count}} < 10` | `"{{count}} < 10"` |
| `>=` | 大于等于 | `{{count}} >= 10` | `"{{count}} >= 10"` |
| `<=` | 小于等于 | `{{count}} <= 10` | `"{{count}} <= 10"` |

**两种方式的区别：**

| 特性 | markdown 段落方式 | YAML 配置块方式 |
|------|------------------|----------------|
| 写法位置 | `**when**: <expression>` | YAML 块内 `when:\n  expr: "<expression>"` |
| 适用场景 | 简单条件，不包含 `{{` | 包含模板变量 `{{var}}` 的复杂条件 |
| 引号要求 | 不需要引号 | **必须用双引号包裹表达式** |
| 示例 | `**when**: count > 10` | `when:\n  expr: "{{count}} > 10"` |

### 4.3 变量与模板语法

#### 变量引用规则

**Tool / Await 步骤输出** — 直接使用 key 名引用，不需要步骤名前缀：

```prompt
总销售额：{{total_sales}}
用户确认：{{confirm}}
用户备注：{{notes}}
```

> Tool 步骤的 key 由工具代码 `output.put(key, value)` 决定；Await 步骤的 key 由 `input_schema` 中定义的字段名决定。

**Prompt / Template 步骤输出** — 使用 varName 引用：

```prompt
报告内容：{{report}}
消息文本：{{message}}
```

**变量存储规则：**

| Step 类型 | 存入上下文的键 | 访问语法 |
|-----------|--------------|---------|
| prompt / template | varName | `{{varName}}` |
| tool | 工具代码调用 `output.put(key, value)` 决定 | `{{key}}` |
| await | `input_schema` 中的字段名 | `{{field_name}}` |

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

**循环渲染：**

使用 `{{#for array}}...{{/for}}` 遍历数组。

**结构化数组** — 循环体内直接引用元素的字段名：

```template
{{#for result}}
区域：{{region}}，商品：{{product}}，销售量：{{amount}}
{{/for}}
```

渲染结果（假设 result 有 3 条记录）：

```
区域：华东，商品：产品A，销售量：150
区域：华北，商品：产品B，销售量：200
区域：华南，商品：产品C，销售量：180
```

**简单数组** — 使用 `{{_}}` 引用当前迭代元素：

```template
{{#for tags}}
- {{_}}
{{/for}}
```

渲染结果（假设 tags = ["重要", "紧急", "待审核"]）：

```
- 重要
- 紧急
- 待审核
```

> `{{_}}` 代表当前迭代元素本身。对于结构化数组，`{{_}}` 也可使用，此时代表整个元素对象。

> **说明**：模板支持变量替换、表达式求值、数组索引访问和循环渲染，不支持条件渲染语法。条件控制通过步骤级 `when` 属性实现。

---

## 5. Step 类型详解

### 5.1 Tool 步骤

调用已注册的外部工具（HTTP API、数据库、Java Service 等）。工具通过 `ToolOutputContext.put(key, value)` 直接将输出写入执行上下文，后续步骤可通过 key 名直接引用。

#### 配置格式

```markdown
### step: <step_name>

**type**: tool
**tool**: <tool_name>

```yaml
input:
  <parameter_name>: <value_template>
output_schema:
  <output_key>:
    type: <type>
    description: <说明>
```

| 字段 | 必需 | 说明 |
|------|------|------|
| `type` | 是 | 必须为 `tool` |
| `tool` | 是 | 工具名称 |
| `input` | 是 | 工具输入参数（支持模板语法） |
| `output_schema` | 是 | 声明工具输出的 key 及其类型（纯可读性，不参与执行逻辑） |

> **注意**：Tool 步骤**不需要** `varName`。工具的输出 key 由工具的 Java 实现决定（通过 `output.put(key, value)` 调用）。`output_schema` 仅用于让技能文件具有自描述性，方便阅读者了解工具会产出哪些变量。

#### 输入参数

YAML 配置块中，`input` 键下定义工具的输入参数，`output_schema` 键声明输出。两者在 YAML 结构上是平级的，解析器分别读取：

```yaml
input:
  region: "{{region}}"
  period: "{{period}}"
output_schema:
  total_sales:
    type: number
    description: 总销售额
```

`input` 下的内容传递给工具的 `execute()` 方法，`output_schema` 不传递。

#### 参数类型系统

**重要原则**：工具从执行上下文获取的输入始终是 **Java 对象**，不是 JSON 字符串。

**参数类型传递规则：**

| YAML 中的值类型 | 传递给工具的 Java 类型 | 示例 |
|-----------------|---------------------|------|
| 字符串 | `String` | `q: "{{query}}"` → `String` |
| 数字 | `Long` 或 `Double` | `count: 10` → `Long` |
| 布尔值 | `Boolean` | `enabled: true` → `Boolean` |
| 数组（列表） | `List<Object>` | 见下方示例 |
| 对象 | `Map<String, Object>` | 见下方说明 |
| null | `null` | `value: null` → `null` |

**数组参数示例：**

YAML 定义：
```yaml
input:
  tags:
    - "{{tag1}}"
    - "{{tag2}}"
    - fixed_tag
```

工具接收到的 Java 对象：
```java
Map<String, Object> input = {
  "tags": List<Object> ["值1", "值2", "fixed_tag"]
}
```

**模板变量渲染规则：**

1. **简单值替换**：`{{variable}}` 直接替换为变量值
2. **数组传递**：数组类型的变量以 `List<Object>` 形式传递
3. **对象传递**：对象类型的变量以 `Map<String, Object>` 形式传递

#### output_schema 声明

在 YAML 配置块中，`output_schema` 键声明工具的输出变量（纯可读性，不参与执行）：

```yaml
input:
  region: "{{region}}"
output_schema:
  total_sales:
    type: number
    description: 总销售额
  target:
    type: number
    description: 销售目标
  achievement_rate:
    type: number
    description: 达成率
```

#### 工具输出模型

工具的 `execute()` 方法签名为：

```java
void execute(Map<String, Object> input, ToolOutputContext output) throws ToolExecutionException;
```

工具通过 `output.put(key, value)` 将输出变量写入执行上下文。后续步骤直接通过 `{{key}}` 引用这些变量。

#### 变量类型约束

技能上下文中的变量类型：

| 类型 | Java 类型 | 说明 |
|------|-----------|------|
| string | `String` | 文本 |
| number | `Long` / `Double` | 数字 |
| boolean | `Boolean` | 布尔值 |
| array | `List<Object>` | 列表 |
| object | `Map<String, Object>` | 键值对 |


#### 变量命名冲突处理

当同一工具被多次调用时（例如两次 `json_select`），后一次的输出会覆盖前一次的同名变量。解决方式是在两次调用之间插入 `template` 步骤，将需要保留的变量复制到新名称：

```markdown
### step: select_region_data

**type**: tool
**tool**: json_select

```yaml
input:
  data: "{{sales_data}}"
  path: "{{region}}"
output_schema:
  result:
    type: string
```

### step: save_region_data

**type**: template  **varName**: region_data_json

```template
{{result}}
```

### step: select_period_data

**type**: tool
**tool**: json_select

```yaml
input:
  data: "{{sales_data}}"
  path: "{{region}}.{{period}}"
output_schema:
  result:
    type: string
```
```

上例中，`save_region_data` 步骤在第二次 `json_select` 覆盖 `result` 之前，将其保存为 `region_data_json`。

#### 工具实现建议

```java
@Override
public void execute(Map<String, Object> input, ToolOutputContext output) throws ToolExecutionException {
    // 获取基本类型参数
    String data = (String) input.get("data");
    String path = (String) input.get("path");

    // 输出基本类型到上下文
    output.put("status", "success");
    output.put("count", 42);

    // 复杂数据序列化为 JSON 字符串后写入上下文
    output.put("result", processedJsonString);
}
```

### 5.2 Prompt 步骤

将 Prompt 模板发送给 LLM，获取生成结果。

#### 配置格式

```markdown
### step: <step_name>

**type**: prompt
**varName**: <variable_name>

```prompt
<prompt_template>
```

| 字段 | 必需 | 说明 |
|------|------|------|
| `type` | 是 | 必须为 `prompt` |
| `varName` | 是 | 输出变量名 |
| ` ```prompt` 代码块 | 是 | Prompt 模板内容 |

#### 示例

```markdown
### step: analyze_data

**type**: prompt
**varName**: report

```prompt
你是一位专业的财务分析师。

请分析 {{company}} 在 {{period}} 期间的财务数据：
{{data}}

请给出专业分析，包括：
1. 关键财务指标解读
2. 风险提示
3. 建议措施
```
```

### 5.3 Template 步骤

纯文本模板渲染，只做变量替换，**不调用 LLM 也不调用 Tool**。渲染结果以 varName 为键直接存入上下文。

适用场景：
- 拼接变量和文字生成一段文本（如订单确认、通知消息）
- 组合多个前置步骤的输出为统一格式
- 不需要 LLM 智能生成的纯模板渲染

#### 配置格式

````markdown
### step: <step_name>

**type**: template
**varName**: <variable_name>

```template
模板内容，支持 {{variable}} 语法
```
````

| 字段 | 必需 | 说明 |
|------|------|------|
| `type` | 是 | 必须为 `template` |
| `varName` | 是 | 输出变量名 |
| ` ```template` 代码块 | 是 | 模板内容 |

#### Template vs Prompt

| 特性 | Template | Prompt |
|------|----------|--------|
| 是否调用 LLM | 否 | 是 |
| 输出可控性 | 完全确定性 | 不确定（LLM 生成） |
| 性能 | 极快（纯字符串替换） | 较慢（网络调用） |
| 适用场景 | 格式固定的文本拼接 | 需要智能生成的内容 |

#### 示例

```markdown
### step: compose_message

**type**: template
**varName**: message

```template
===== 订单确认 =====
订单编号：{{order_id}}
商品名称：{{product_name}}
总金额：¥{{total}}
===================
感谢您的购买！
```
```

### 5.4 Await 步骤

暂停 Skill 执行，等待用户提供额外输入后继续执行。用于人机交互控制流场景。

#### 配置格式

```markdown
### step: <step_name>

**type**: await

```yaml
message: |
  <提示信息>
input_schema:
  <field_name>:
    type: <type>
    required: <true|false>
    description: <描述>
```

| 字段 | 必需 | 说明 |
|------|------|------|
| `type` | 是 | 必须为 `await` |
| `message` | 是 | 向用户展示的提示信息 |
| `input_schema` | 是 | 用户输入的结构定义 |

> **注意**：Await 步骤**不需要** `varName`。用户提交输入后，`input_schema` 中定义的每个字段直接以字段名为 key 写入执行上下文。例如 `input_schema` 中定义了 `confirm` 和 `notes` 两个字段，则后续步骤可通过 `{{confirm}}`、`{{notes}}` 直接引用。

#### 示例

```markdown
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
**when**: confirm == true

```template
订单 {{order_id}} 已确认，总金额 ¥{{total_amount}}。
```

### step: cancel_order

**type**: template
**varName**: cancel_result
**when**: confirm == false

```template
订单 {{order_id}} 已取消。
```
```

---

## 6. 完整示例

### 6.1 简单对话 Skill

```markdown
# skill: chat

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
```

### 6.2 数据搜索 Skill

```markdown
# skill: simple_search

## description
简单搜索 Skill，用于搜索相关信息。

## capabilityTags
  - 搜索
  - 实时信息
  - 新闻
  - 专业词查询
  - 生僻词意查找
  

## input_schema

```yaml
query:
  type: string
  required: true
  description: 搜索关键词
```

## output_schema

```yaml
result:
  type: string
  description: 搜索结果
```

## steps

### step: search

**type**: tool
**tool**: search_api

```yaml
input:
  q: "{{query}}"
output_schema:
  result:
    type: string
    description: 搜索结果
```
```

### 6.3 订单确认 Skill（人机交互）

```markdown
# skill: order_confirmation

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
**when**: confirm == true

```template
订单 {{order_id}} 已确认，总金额 ¥{{total_amount}}。
用户备注：{{notes}}
```

### step: cancel_order

**type**: template
**varName**: cancel_result
**when**: confirm == false

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
```

### 6.4 财务分析 Skill（多步骤组合）

```markdown
# skill: financial_analysis

## description

对企业财务状况进行分析，并生成结构化分析报告。

## capabilityTags
  - 财务分析
  - financial_analysis
  - 财报分析
  - 公司分析

## input_schema

```yaml
company:
  type: string
  required: true
  description: 公司名称
period:
  type: string
  required: true
  description: 分析期间
```

## output_schema

```yaml
report:
  type: string
  description: 财务分析报告
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
    description: 金融数据（JSON 字符串）
```

### step: analyze_data

**type**: prompt
**varName**: report

```prompt
你是一位专业的财务分析师。

请分析 {{company}} 在 {{period}} 期间的财务数据：
{{data}}

请给出专业分析，包括：
1. 关键财务指标解读
2. 风险提示
3. 建议措施
```
```

### 6.5 销售报表 Skill

```markdown
# skill: sales_report

## description
生成销售数据报表

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
data:
  type: array
  description: 表格数据
summary:
  type: object
  description: 汇总信息
```

## steps

### step: fetch_sales_data

**type**: tool
**tool**: database.query

```yaml
input:
  query: "SELECT region, product, amount FROM sales WHERE region = '{{region}}'"
output_schema:
  result:
    type: array
    description: 查询结果
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
```

### step: format_report

**type**: template
**varName**: report

```template
{{region}} 地区 {{period}} 销售报表：

{{#for result}}
区域：{{region}}，商品：{{product}}，销售量：{{amount}}
{{/for}}
```
```

### 6.6 销售趋势分析 Skill

```markdown
# skill: sales_trend_analysis

## description
分析销售趋势并生成图表

## capabilityTags
  - 销售管理
  - 数据分析
  - 销售

## input_schema

```yaml
product:
  type: string
  required: true
  description: 产品名称
time_range:
  type: string
  required: true
  description: 时间范围
  options:
    - 最近7天
    - 最近30天
    - 最近90天
    - 最近一年
chart_type:
  type: string
  required: true
  description: 图表类型
  options:
    - 折线图
    - 柱状图
    - 饼图
```

## output_schema

```yaml
chart_type:
  type: string
  description: 图表类型（line/bar/pie）
title:
  type: string
  description: 图表标题
x_axis:
  type: array
  description: X轴数据
y_axis:
  type: array
  description: Y轴数据
options:
  type: object
  description: 图表配置选项
```

## steps

### step: analyze_trend

**type**: prompt
**varName**: chart_type

```prompt
分析产品「{{product}}」在{{time_range}}的销售趋势。

请生成{{chart_type}}的数据，包括：
1. X轴标签（时间点）
2. Y轴数据（销售额）
3. 图表标题
4. 图表配置建议
```
```

### 6.7 文件导出 Skill

```markdown
# skill: export_report

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
file_url:
  type: string
  description: 文件下载链接
file_name:
  type: string
  description: 文件名
file_size:
  type: number
  description: 文件大小（字节）
mime_type:
  type: string
  description: MIME类型
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
  file_url:
    type: string
    description: 文件下载链接
  file_name:
    type: string
    description: 文件名
  file_size:
    type: number
    description: 文件大小（字节）
  mime_type:
    type: string
    description: MIME 类型
```

### step: format_response

**type**: template
**varName**: response

```template
文件已生成：
- 文件名：{{file_name}}
- 下载链接：{{file_url}}
- 大小：{{file_size}} 字节
```
```

---

## 7. 最佳实践

### 7.1 命名规范

- **Step 名称**：使用动词或动词短语，如 `calculate_total`、`prepare_summary`
- **输入参数**：使用下划线命名法，如 `order_id`、`unit_price`
- **工具名称**：使用点号分隔命名空间，如 `database.query`、`search_api`
- **varName**：与 output_schema 中的字段名对齐，如输出定义了 `report` 字段，则 varName 也用 `report`

### 7.2 设计原则

1. **单一职责**：每个 Skill 应专注于一个任务
2. **步骤复用**：通过步骤组合实现复杂逻辑
3. **明确输入输出**：清晰定义输入输出结构
4. **错误处理**：在条件分支中处理各种情况
5. **链式调用**：上一个技能的 output_schema 可能是下一个技能的 input_schema，保持结构一致

### 7.3 性能优化

1. **减少 LLM 调用**：对于固定格式输出和简单计算，使用 `template` 类型代替 `prompt`
2. **并行执行**：将独立步骤放在不同分支
3. **缓存结果**：在步骤间传递输出而非重复计算

### 7.4 调试技巧

1. **使用 log 工具**：输出中间变量值进行调试
2. **条件日志**：使用 `when` 条件输出调试信息
3. **分步测试**：单独测试每个步骤

---

## 8. 附录

### 8.1 保留关键词

以下词汇为 DSL 保留关键词，不能用作字段名或变量名：

- `type`
- `tool`
- `when`
- `varName`
- `input`
- `output`
- `step`
- `skill`
- `description`
- `required`
- `items`

### 8.2 修订历史

| 版本 | 日期 | 说明 |
|------|------|------|
| 1.0.0 | 2026-02-07 | 初始版本，支持四种 Step 类型 |
| 1.0.0 | 2026-02-10 | 移除 Transform 类型；统一步骤输出引用为 `.value` 语法；Step 类型精简为三种（tool / prompt / await） |
| 1.0.0 | 2026-02-10 | 新增 Template 类型 Step：纯文本模板渲染，不调用 LLM 也不调用 Tool；Step 类型扩展为四种 |
| 1.0.0 | 2026-02-10 | 新增 varName 属性：为所有 Step 类型添加变量别名，步骤输出以 varName 为键直接存入上下文 |
| 1.0.0 | 2026-02-10 | 移除 TEXT 格式 output_schema：output_schema 改为必需字段，统一使用 YAML 格式定义 |
| 1.0.0 | 2026-02-10 | varName 改为必填：所有 Step 必须指定 varName，移除无 varName 的 StepOutputWrapper 兼容路径 |
| 1.0.0 | 2026-02-11 | Tool 输出模型重构：工具通过 `ToolOutputContext.put(key, value)` 直接写上下文；Tool 步骤不再需要 varName；新增 output_schema 声明（纯可读性）；上下文变量限制为基本类型 |
| 1.0.0 | 2026-02-11 | 文档重构：移除 `properties` 关键词，array 用 `items` 平铺子字段，object 子字段直接平铺；重组文档结构为「概述 → 输入 → 输出 → Steps → 示例」 |
| 1.0.0 | 2026-02-11 | Await 去掉 varName，input_schema 字段直接写入上下文；Tool YAML 增加 `input:` 包装，output_schema 改为必需；模板仅保留变量替换，移除条件渲染和循环语法；参数类型系统明确为基本类型 + 数组，不支持复杂对象 |
| 1.0.0 | 2026-02-11 | 模板语法扩展：`{{}}` 支持简单表达式求值（四则运算、字符串拼接）；示例中 `variable.set` 改为 `template` 表达式 |
| 1.0.0 | 2026-02-11 | 模板新增数组索引访问（`{{arr[n].field}}`、`{{arr[#var].field}}`）和循环渲染（`{{#for arr}}...{{/for}}`） |
| 1.0.0 | 2026-02-11 | input_schema 和上下文新增 `object` 类型（`Map<String, Object>`）；循环渲染新增 `{{_}}` 引用当前元素；array 上下文存储改为 `List`；更新 database.query 示例 |
| 1.0.0 | 2026-02-12 | 新增 version 字段：技能文件支持声明版本号，`skillId + version` 构成全局唯一标识；API 层支持按版本查询和执行 |
| 1.0.0 | 2026-02-14 | 将意图关键词intent改为能力关键词capabilityTags |

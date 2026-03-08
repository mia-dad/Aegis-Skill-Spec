# Aegis Skill DSL 规范说明文档

> **规范版本：3.4.2**

## 目录

### 通用规范（本文档）
- [1. 概述](#1-概述) — 技能分类、文件骨架、type/version 字段
- [2. 输入输出的声明和定义](#2-输入输出的声明和定义) — 基本类型、语义角色
- [3. 内置函数与工具](#3-内置函数) — 数组操作函数、函数调用语法、内置工具
- [4. 最佳实践](#4-最佳实践) — 命名规范、设计原则、性能优化
- [5. 附录](#5-附录) — 保留关键词、修订历史

### 按技能类型分述
- [AtomicSkill.md](./AtomicSkill.md) — 原子技能：output_schema、审计日志、完整示例
- [CognitiveSkill.md](./CognitiveSkill.md) — 认知技能：internal_flow、动态决策、完整示例

---

## 1. 概述

Aegis Skill DSL 是一种基于 Markdown 的领域特定语言，用于定义可执行的 AI 技能。

### 1.1 技能分类

Aegis 技能分为以下类型：

| 类型                  | 说明       | 核心职责                          |
| ------------------- | -------- | ----------------------------- |
| `AtomicSkill`       | 原子技能（默认） | 数据处理和业务逻辑，输出结构化数据,附带UI渲染和数据展示 |
| `CognitiveSkill`    | 认知技能     | 复杂决策和动态编排，运行时决策、能力匹配          |

**AtomicSkill** 由以下部分组成：
- 输入定义（input_schema）
- 输出定义（output_schema）
- 执行步骤（steps）
- UI 展示语义（ui）


**CognitiveSkill** 由以下部分组成：
- 输入定义（input_schema）— 复杂决策所需的输入数据
- 内部控制流（internal_flow）— 决策逻辑和能力编排
- 输出定义（output_schema）— 决策结果和处理后的数据

> **设计原则**：AtomicSkill 专注于数据处理，包含 UI；CognitiveSkill 专注于复杂决策和动态编排，通过 internal_flow 实现运行时的条件判断和能力选择。技能的串联和编排由 Agent 层负责，DSL 层只需关注单个技能的定义。


### 1.2 AtomicSkill 文件骨架

~~~markdown
# skill: <skill_id>

**version**: <版本号>
**ignore**: <true|false>
**type**: AtomicSkill
**debug**: <true|false>
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

## ui

```yaml
<UI展示语义定义>
```
~~~



### 1.3 CognitiveSkill 文件骨架

~~~markdown
# skill: <skill_id>

**version**: <版本号>
**ignore**: <true|false>
**timeout**: <超时毫秒数>
**type**: CognitiveSkill
**mode**: <运行模式>
**debug**: <true|false>
## description

<技能描述>

## capabilityTags

- <能力关键词1>
- <能力关键词2>

## input_schema

```yaml
<输入参数定义>
```

## internal_flow

```yaml
<内部控制流定义>
```

## output_schema

```yaml
<输出参数定义>
```
~~~

### 1.4 技能文件顶层章节

#### AtomicSkill 章节

| 章节                          | 必需  | 说明                                       |
| --------------------------- | --- | ---------------------------------------- |
| `# skill: <id>`             | 是   | 技能唯一标识符，作为一级标题                           |
| `**version**: <版本号>`        | 否   | 技能版本号，遵循语义化版本（如 `2.0.0`）。未提供时默认为 `1.0.0` |
| `**ignore**: <true\|false>` | 否   | 技能级异常处理，默认为 `false`                      |
| `**type**: AtomicSkill`     | 是   | 技能类型，AtomicSkill 可省略（默认值）                |
| **debug**: <true\|false>    | 否   | 该技能是否开启debug函数调试功能,默认不开启false            |
| `## description`            | 否   | 技能描述文本                                   |
| `## capabilityTags`         | 否   | 能力关键词列表，用于技能路由和匹配                        |
| `## input_schema`           | 否   | 输入参数结构定义（YAML 格式）                        |
| `## output_schema`          | 是   | 输出参数结构定义（YAML 格式，平铺字段定义）                 |
| `## steps`                  | 是   | 执行步骤列表（至少一个）                             |
| `## ui`                     | 是   | UI展示语义定义                                 |

> **注意**：AtomicSkill **包含** `## ui` 章节。


#### CognitiveSkill 章节

| 章节                          | 必需  | 说明                                                               |
| --------------------------- | --- | ---------------------------------------------------------------- |
| `# skill: <id>`             | 是   | 技能唯一标识符，作为一级标题                                                   |
| `**version**: <版本号>`        | 否   | 技能版本号，遵循语义化版本（如 `3.0.0`）                                         |
| `**ignore**: <true\|false>` | 否   | 技能级异常处理，默认为 `false`                                              |
| `**timeout**: <毫秒数>`        | 否   | 执行超时时间（毫秒），默认由平台配置                                               |
| `**type**: CognitiveSkill`  | 是   | 技能类型，必须显式声明                                                      |
| `**mode**: <模式名>`           | 是   | Agent 运行模式（Direct/CoT/ReAct/Decompose/Retrieve/Compare/Generate） |
| **debug**: <true\|false>    | 否   | 该技能是否开启debug函数调试功能,默认不开启false                                    |
| `## description`            | 否   | 技能描述文本                                                           |
| `## capabilityTags`         | 否   | 能力关键词列表，用于技能路由和匹配                                                |
| `## input_schema`           | 是   | 输入参数结构定义（YAML 格式）                                                |
| `## internal_flow`          | 是   | 内部控制流定义，包含决策节点序列                                                 |
| `## output_schema`          | 是   | 输出参数结构定义（YAML 格式）                                                |

> **注意**：CognitiveSkill **不包含** `## steps` 和 `## ui` 章节。决策逻辑通过 `internal_flow` 定义。

### 1.5 type 字段

`type` 字段声明在 `version` 之后，格式为：

```markdown
**type**: AtomicSkill
```

或：

```markdown
**type**: CognitiveSkill
```

**用法说明：**

- **必填性**：`type` 字段在  CognitiveSkill 中必须显式声明；AtomicSkill 可省略（为默认值）
- **默认值**：未声明时默认为 `AtomicSkill`（向后兼容）
- **取值范围**：`AtomicSkill` |  `CognitiveSkill`
- **结构约束**：不同类型的技能有不同的章节要求（见上表）

### 1.6 version 字段

`version` 字段声明在一级标题 `# skill: <id>` 之后，格式为：

```markdown
**version**: 3.4.2
```

**用法说明：**

- **取值来源**：version 的值对应本规范文档开篇的版本号。当规范版本为 `2.0.0` 时，技能文件的 version 即为 `2.0.0`
- **唯一标识**：`skillId + version` 构成全局唯一标识。同一个 skillId 可以存在多个版本（如 `1.0.0`、`1.1.0`、`3.4.0`）
- **语义化版本**：遵循 `major.minor.patch` 格式
  - `major`：不兼容的结构性变更（如输入/输出 schema 变化）
  - `minor`：向后兼容的功能新增（如新增可选输入字段）
  - `patch`：向后兼容的缺陷修复（如 prompt 文案调整）
- **默认值**：未声明 version 时，系统默认为 `1.0.0`
- **版本查找**：通过 API 按 skillId 查询时，未指定版本则返回版本号最大的 Skill

### 1.7 ignore 字段（技能级异常处理）

`ignore` 字段声明在 `version` 之后，用于控制整个技能执行失败时的行为。

**语法：**

```markdown
**version**: 2.2.0
**ignore**: true
```

**用法说明：**

| ignore 值 | 行为 |
|-----------|------|
| `false`（默认） | 技能执行失败时抛出异常，终止整个执行流程 |
| `true` | 技能执行失败时返回 `null`，允许上层编排继续执行 |

**适用场景：**

- **可选项**：某些技能的执行结果不是必需的，失败不应影响整体流程
- **降级处理**：上层编排可根据技能返回 `null` 做降级逻辑
- **容错设计**：允许部分技能失败，提高系统整体鲁棒性

**示例：**

~~~markdown
# skill: optional_enrichment

**version**: 2.2.0
**ignore**: true
**type**: AtomicSkill

## description
可选的数据增强技能，失败时不影响主流程。
~~~

**与 Step ignore 的关系：**

| 层级  | 属性位置               | 作用范围   |
| --- | ------------------ | ------ |
| 技能级 | `**ignore**:` 顶层字段 | 整个技能执行 |
| 步骤级 | Step 的 `ignore` 属性 | 单个步骤执行 |
| 函数级 | 函数的 `ignore` 参数    | 单次函数调用 |

> **优先级**：步骤级/函数级的 `ignore` 在各自范围内独立生效。技能级 `ignore` 仅在整个技能执行失败（未被步骤级捕获）时生效。

### 1.8 timeout 字段（CognitiveSkill 专用）

`timeout` 字段仅适用于 CognitiveSkill，用于控制整个技能的最大执行时间。

**语法：**

```markdown
**version**: 3.0.0
**timeout**: 30000
**type**: CognitiveSkill
```

**用法说明：**

| 属性 | 说明 |
|------|------|
| 单位 | 毫秒（ms） |
| 默认值 | 由平台配置决定 |
| 适用范围 | 仅 CognitiveSkill |
| 作用 | 限制整个技能执行的最大时间，包括所有内部节点 |

**超时行为：**

| 场景 | 行为 |
|------|------|
| 执行时间 < timeout | 正常完成 |
| 执行时间 ≥ timeout | 中断执行，抛出超时异常 |
| ignore: true + 超时 | 返回 `null`，允许上层继续 |
| ignore: false + 超时 | 终止整个执行流程 |

**示例：**

~~~markdown
# skill: cognitive_deep_analysis

**version**: 3.0.0
**timeout**: 60000
**ignore**: true
**type**: CognitiveSkill

## description
深度分析技能，最长执行 60 秒。
~~~

> **设计说明**：CognitiveSkill 可能包含 `foreach` 循环和多次能力调用，执行时间不可预测。`timeout` 是防止单个技能阻塞整个 Agent 执行的安全阀。

### 1.9 debug 字段（调试模式控制）

`debug` 字段用于控制技能是否启用调试模式，影响 `debug()` 内置函数的执行行为。

**语法：**

```markdown
**version**: 3.4.2
**debug**: true
**type**: AtomicSkill
```

**用法说明：**

| debug 值 | 行为 |
|-----------|------|
| `false`（默认） | `debug()` 函数不执行，不输出任何调试日志 |
| `true` | `debug()` 函数正常执行，输出调试日志 |

**适用范围：**

- **AtomicSkill**：支持
- **CognitiveSkill**：支持

**示例：**

~~~markdown
# skill: data_analysis

**version**: 3.4.2
**debug**: true
**type**: AtomicSkill

## description
数据分析技能，启用调试模式以便排查问题。
~~~

**使用场景：**

| 场景 | debug 设置 |
|------|-----------|
| 生产环境 | `false`（默认，避免性能影响） |
| 开发测试 | `true`（输出调试信息） |
| 问题排查 | 临时设置为 `true` 进行调试 |

**性能影响：**

当 `debug: false` 时，`debug()` 函数调用会被完全跳过，不会产生任何性能开销。

> **注意**：`debug()` 函数是 DSL 引擎层的内置函数，用于输出技能调试日志。详见 [3.15 debug 函数](#315-debug-函数)。

---





## 2. 输入输出的声明和定义

 每个技能都必须有针对输入输出声明定义,声明的目标是针对技能在运行时接收的变量和最终输出的变量,针对不同类型的输入输出配置方式见表如下

| 技能类型             | 输入           | 输出               | 输入说明      | 输出说明                                        |
| ---------------- | ------------ | ---------------- | --------- | ------------------------------------------- |
| `AtomicSkill`    | input_schema | output_schema和ui | 必须要有相应的输入 | ui 仅做为给用户返回逻辑起作用，仅在该`AtomicSkill`为最后一个技能执行时 |
| `CognitiveSkill` | input_schema | output_schema    | 必须要有相应的输入 | 输出有可能是在动态时进行检查.                             |

### 2.1 声明的结构

基本语法

```yaml
<field_name>:
  type: <string|number|boolean|datetime|resource|array|object>
  required: <true|false>
  description: <字段描述>
  default: <默认值>           # 可选
  options: [选项1, 选项2, ...] # 可选，用于枚举约束
  roles:[选项1, 选项2, ...] # 可选
```

| 属性            | 类型      | 必需  | 说明                             |
| ------------- | ------- | --- | ------------------------------ |
| `type`        | string  | 是   | 字段数据类型                         |
| `required`    | boolean | 否   | 是否必填（默认 true）                  |
| `description` | string  | 否   | 字段描述信息                         |
| `default`     | any     | 否   | 字段默认值                          |
| `options`     | array   | 否   | 枚举选项列表，用于单选/多选                 |
| `roles`       | array   | 否   | 语义角色，可以多选，描述字段的业务语义类别（详见 2.10） |

---

### 2.1.1 output_schema 的 mapping 映射机制

AtomicSkill 的 `output_schema` 支持可选的 `mapping` 字段，用于将步骤输出映射到最终输出字段。

**基本语法：**

```yaml
<field_name>:
  type: <string|number|boolean|datetime|resource|array|object>
  required: <true|false>
  description: <字段描述>
  mapping: <步骤输出引用>    # 必填，指定从哪个步骤输出获取值
```

**映射规则：**

| mapping 设置 | 行为                   | 示例                                  |
| ---------- | -------------------- | ----------------------------------- |
| **必须设置**   | 使用 mapping 指定的步骤输出引用 | `summary: mapping: {{step1.value}}` |

**使用示例：**


**场景1： mapping用法**

~~~markdown
## output_schema

```yaml
final_report:
  type: string
  description: 最终报告
  mapping: {{generate_summary.value}}
```

## steps

### step: generate_summary
**type**: template

```template
生成的摘要内容
```
~~~

系统直接使用 `mapping` 中指定的 `{{generate_summary.value}}` 作为 `final_report` 字段的值。

**场景2：引用步骤输出的具体字段**

~~~markdown
## output_schema

```yaml
total:
  type: number
  description: 总销售额
  mapping: {{calculate.total}}

count:
  type: number
  description: 记录数量
  mapping: {{calculate.count}}
```

## steps

### step: calculate
**type**: tool
**tool**: sales_api

```yaml
output_schema:
  total:
    type: number
  count:
    type: number
```
~~~

**注意事项：**

1. **mapping 引用的必须是步骤输出**：如 `{{stepName.value}}` 或 `{{stepName.field}}`
2. **mapping 引用不能嵌套**：不支持 `{{step1.value}} + {{step2.value}}` 这样的表达式
3. **循环引用检测**：系统会检测并拒绝循环引用的 mapping
4. **类型兼容性**：mapping 引用的值类型必须与字段声明的 `type` 兼容

---

### 2.2 type的基本类型
| 类型         | 说明             | 示例值                                                           |
| ---------- | -------------- | ------------------------------------------------------------- |
| `string`   | 字符串            | `"hello"`                                                     |
| `number`   | 数字             | `123`, `45.67`                                                |
| `boolean`  | 布尔值            | `true`, `false`                                               |
| `datetime` | 时间（日期/时间/日期时间） | `"2026-02-28"`, `"14:30:00"`, `"2026-02-28 14:30:00"`         |
| `resource` | 资源引用（文件/链接）    | `"file://data/report.xlsx"`, `"https://api.example.com/data"` |
| `array`    | 数组             | `[1, 2, 3]`                                                   |
| `object`   | 对象（键值对）        | `{name: "张三", age: 30}`                                       |

#### 2.2.1 datetime 类型

`datetime` 是一种特殊的基本类型，用于处理日期和时间。系统会自动识别输入格式并统一转换。

##### 支持的输入格式

| 格式类型  | 示例                                          | 说明                 |
| ----- | ------------------------------------------- | ------------------ |
| 仅日期   | `2026-02-28`、`2026/02/28`、`20260228`        | 自动补充时间为 `00:00:00` |
| 仅时间   | `14:30:00`、`14:30`、`143000`                 | 自动补充日期为当天          |
| 日期+时间 | `2026-02-28 14:30:00`、`2026-02-28T14:30:00` | 完整时间               |

##### 输出格式

系统统一输出为以下格式：

```
yyyy年MM月dd日 HH:mm:ss
```

示例：`2026年02月28日 14:30:00`

##### 使用示例

```yaml
start_time:
  type: datetime
  required: true
  description: 开始时间
end_date:
  type: datetime
  required: false
  description: 结束日期
  default: "2026-12-31"
```

##### 格式转换规则

| 输入                    | 输出                                       |
| --------------------- | ---------------------------------------- |
| `2026-02-28`          | `2026年02月28日 00:00:00`                   |
| `14:30`               | `2026年02月28日 14:30:00`（假设当天为 2026-02-28） |
| `2026-02-28 14:30:00` | `2026年02月28日 14:30:00`                   |
| `2026/02/28 14:30`    | `2026年02月28日 14:30:00`                   |

#### 2.2.2 resource 类型

`resource` 是一种声明式的资源引用类型，用于表示需要访问的外部资源。系统负责校验 URI 格式，Tool 负责实际访问和处理。

##### 支持的协议

| 协议 | 格式 | 说明 |
|------|------|------|
| `file://` | `file://<相对路径>` | 本地文件，路径相对于运行时系统限定的根目录 |
| `http://` | `http://<url>` | HTTP 网络资源 |
| `https://` | `https://<url>` | HTTPS 网络资源 |

##### 路径说明

**file:// 协议**：

`file://` 后的路径是**相对路径**，基于运行时系统为当前技能限定的工作目录：

```
file://data/report.xlsx
       └── 相对于系统限定的根目录
```

> DSL 层无法获知系统的真实文件路径，只能使用相对路径引用。系统运行时会将相对路径解析为实际的文件位置。

**http[s]:// 协议**：

完整的网络 URL：

```
https://api.example.com/v1/data.json
```

##### 使用示例

```yaml
input_file:
  type: resource
  required: true
  description: 待分析的文档

api_endpoint:
  type: resource
  required: false
  description: 外部 API 地址

config:
  type: resource
  description: 配置文件
  default: "file://config/default.yaml"
```

##### 值示例

```yaml
# 本地文件（相对路径）
input_file: "file://data/report.xlsx"
input_file: "file://uploads/2026/02/image.png"

# 网络资源
api_endpoint: "https://api.weather.com/forecast"
api_endpoint: "http://internal-service/data"
```

##### 针对该类型系统职责 vs Tool 职责

| 职责 | 系统 | Tool |
|------|:----:|:----:|
| URI 格式校验 | ✓ | |
| 协议合法性检查 | ✓ | |
| 相对路径解析 | ✓ | |
| 资源实际访问 | | ✓ |
| 内容读取/解析 | | ✓ |
| 错误处理 | | ✓ |

> **设计原则**：`resource` 是声明式的资源引用，系统只做格式校验，不做运行时访问。相关的Tool 接收到的是解析后的资源路径，由相关的 Tool 决定如何访问和处理资源内容。


#### 2.2.3 数组类型与 items

当字段类型为 `array` 时，使用 `items` 描述元素结构。

##### 简单数组（元素为基本类型）

```yaml
tags:
  type: array
  description: 标签列表
  items:
    type: string
```

##### 结构化数组（元素为复合结构）

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

##### 多层嵌套

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
          description:....
        role:
          type: string
          description:....
```

##### array类型声明时完整性规则

**当字段类型为 `array` 时,且技能类型为AtomicSkill时，必须通过 `items` 完整定义其元素结构，直到最末级节点的类型为基本类型（`string`、`number`、`boolean`、`datetime`、`resource`）。**

| 规则           | 说明                            |
| ------------ | ----------------------------- |
| 必须指定 items   | `array` 类型必须通过 `items` 指定元素类型 |
| items 使用语义类型 | `items` 的值必须是已注册的语义类型         |
| 支持嵌套         | 元素类型可以是另一个数组或语义类型             |

> **关于泛型操作**：AtomicSkill 的 `input_schema` 和 `output_schema` 不支持泛型数组（即不声明 `items` 的数组）。如需实现泛型数组操作（如通用过滤、排序、聚合），应通过 Tool 层实现。Tool 内部可以是泛型的，但 AtomicSkill 边界的语义类型必须明确。
```yaml

**type**: AtomicSkill 
# ❌ 错误：array 没有定义 items
headers:
  type: array
  description: 表头列表

# ✅ 正确：array 定义了 items 的基本类型
headers:
  type: array
  description: 表头列表
  items:
    type: string

# ❌ 错误：items 下的 object 没有定义子字段
records:
  type: array
  items:
    type: object
    description: 记录对象

# ✅ 正确：items 下的 object 定义了子字段直到基本类型
records:
  type: array
  items:
    name:
      type: string
    value:
      type: number
```

##### 数组属性

`array` 类型提供以下内置属性：

| 属性        | 类型     | 说明     |
| --------- | ------ | ------ |
| `.length` | number | 数组元素个数 |

**使用示例：**

```yaml
### step: process_items
**type**: template

template: |
  共 {{previous_step.items.length}} 个项目
  {{#when previous_step.items.length > 0}}
  开始处理...
  {{/when}}
```

> **唯一例外**：当 `array` 配合 `options` 实现多选时，不需要定义 `items`，因为 `options` 已隐含元素为字符串类型。

##### 单选和多选（options）

当字段定义通过 `options` 属性配合 `type` 字段实现单选和多选功能。

###### 单选（type: string + options）

当 `type` 为 `string` 并提供 `options` 时，表示输入参数值仅接收其中一个值（不能超出该范围）：

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

###### 多选（type: array + options）

当 `type` 为 `array` 并提供 `options` 时，表示输入参数值能够接受多个候选区域的值（不能超出该范围）：
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

#### 2.2.4 对象类型

当字段类型为 `object` 时，子字段直接平铺在父字段下，无需使用 `properties` 包装：

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

##### 类型完整性规则

**当字段类型为 `object` 时，必须完整定义其子字段结构，直到最末级节点的类型为基本类型（`string`、`number`、`boolean`、`datetime`、`resource`）。**

```yaml
# ❌ 错误：object 没有定义子字段
config:
  type: object
  description: 配置项

# ✅ 正确：object 定义了子字段直到基本类型
config:
  type: object
  description: 配置项
  key:
    type: string
    description: 配置键
  value:
    type: string
    description: 配置值
```



### 2.3 语义角色 (Semantic Role)

伴随技能的输入输出变量定义，在完成了类型定义后，需要对变量指定相应的语义角色。

#### 2.3.1 设计原则

语义类型系统的核心目标是为技能的输入输出提供**业务语义约束**，而非仅仅是数据结构约束。
那么语义类型的本质：问题处理的元数据表示

语义类型不是为了"分类"而分类，而是服务于系统如何智能处理数据的元数据标注。

| 原则      | 说明                                        |
| ------- | ----------------------------------------- |
| 语义明确    | 每个类型都有明确的业务含义，而非泛化的数据结构                   |
| 类型安全    | 编译期校验类型正确性，编译不通过改技能无法注册、运行，避免运行时错误        |
| **可组合** | 通过 `array` + `items: SemanticRole` 实现集合类型 |

#### 2.3.2 Agent 运行模式说明

| 模式 | 中文名 | 适用场景 | 典型流程 |
|------|--------|----------|----------|
| Direct | 直接响应 | 简单问答，无需复杂推理 | 输入 → 直接输出 |
| CoT | 思维链 | 需要逐步推理的复杂问题 | 问题 → 思考步骤 → 结论 |
| ReAct | 推理行动 | 需要与外部工具交互的任务 | 思考 → 行动 → 观察 → 循环 |
| Decompose | 问题分解 | 复杂多步骤任务 | 分解子问题 → 逐个求解 → 综合答案 |
| Retrieve | 检索增强 | 需要外部知识支持的查询 | 检索 → 筛选 → 生成回答 |
| Compare | 多源对比 | 方案比较、竞品分析 | 收集各源信息 → 多维对比 → 给出建议 |
| Generate | 内容生成 | 报告生成、文档创作 | 需求解析 → 初稿 → 优化 → 输出 |

#### 2.3.3 语义角色详细定义

**Direct 模式角色**

| 角色 | 说明 | 典型字段 |
|------|------|----------|
| Query | 用户输入的问题或请求 | query, question, prompt |
| Answer | 直接回答的结果 | answer, response, content |

**CoT 模式角色**

| 角色 | 说明 | 典型字段 |
|------|------|----------|
| Query | 待推理的问题 | question, problem |
| Thought | 单步思考内容 | thought, step |
| ReasoningChain | 完整推理链（步骤数组） | reasoning_chain, steps |
| Conclusion | 最终结论 | conclusion, answer |
| Confidence | 推理置信度（0-1） | confidence, score |

**ReAct 模式角色**

| 角色 | 说明 | 典型字段 |
|------|------|----------|
| Task | 任务目标描述 | task, goal |
| Thought | 当前思考内容 | thought |
| Action | 选定的行动/工具名 | action, tool |
| ActionInput | 行动的输入参数 | action_input, params |
| Observation | 行动执行后的观察结果 | observation, result |
| Answer | 任务完成后的最终答案 | answer, final_answer |

**Decompose 模式角色**

| 角色 | 说明 | 典型字段 |
|------|------|----------|
| Question | 待分解的复杂问题 | question, problem |
| SubQuestions | 分解后的子问题列表 | sub_questions |
| SubQuestion | 单个子问题 | sub_question, sq |
| SubAnswer | 单个子问题的答案 | sub_answer |
| Evidence | 支持答案的证据 | evidence, proof |
| Synthesis | 综合后的最终答案 | synthesis, final_answer |

**Retrieve 模式角色**

| 角色 | 说明 | 典型字段 |
|------|------|----------|
| Query | 检索查询 | query, search_query |
| SearchResults | 检索结果列表 | search_results, results |
| SearchResult | 单条检索结果 | result, item |
| Context | 筛选/重排后的上下文 | context |
| Answer | 基于检索的回答 | answer |
| Sources | 引用来源列表 | sources, references |

**Compare 模式角色**

| 角色 | 说明 | 典型字段 |
|------|------|----------|
| Subject | 对比主题 | subject, topic |
| Sources | 对比对象列表 | sources, items |
| Source | 单个对比对象 | source, item |
| Dimension | 对比维度 | dimensions, aspects |
| Similarities | 相似点列表 | similarities |
| Differences | 差异点列表 | differences |
| Analysis | 对比分析结论 | analysis |
| Recommendation | 推荐建议 | recommendation |

**Generate 模式角色**

| 角色 | 说明 | 典型字段 |
|------|------|----------|
| Requirement | 生成需求描述 | requirement, request |
| Template | 内容模板（可选） | template |
| Constraints | 生成约束条件 | constraints, rules |
| Draft | 生成的初稿 | draft |
| Refined | 优化后的内容 | refined |
| Output | 最终输出内容 | output, content |

#### 2.3.4 语义角色汇总表

| Agent运行模式 | 角色列表 |
|---------------|----------|
| Direct | Query, Answer |
| CoT | Query, Thought, ReasoningChain, Conclusion, Confidence |
| ReAct | Task, Thought, Action, ActionInput, Observation, Answer |
| Decompose | Question, SubQuestions, SubQuestion, SubAnswer, Evidence, Synthesis |
| Retrieve | Query, SearchResults, SearchResult, Context, Answer, Sources |
| Compare | Subject, Sources, Source, Dimension, Similarities, Differences, Analysis, Recommendation |
| Generate | Requirement, Template, Constraints, Draft, Refined, Output |

---
#### 2.3.5 跨模式复用的角色

  某些角色在多个模式中语义相近，技能可以声明多个角色以支持跨模式复用：

  | 通用概念   | 可映射的模式角色                                                                          |
  |--------|-----------------------------------------------------------------------------------|
  | 问题/查询  | Direct.Query, CoT.Query, Decompose.Question, Retrieve.Query                       |
  | 最终答案   | Direct.Answer, CoT.Conclusion, ReAct.Answer, Decompose.Synthesis, Retrieve.Answer |
  | 思考过程   | CoT.Thought, ReAct.Thought                                                        |
  | 证据/上下文 | Decompose.Evidence, Retrieve.Context |

#### 2.3.6 带语义角色的变量声明示例

**单角色声明**：

```yaml
question:
  type: string
  required: true
  description: 用户问题
  roles:
    - Direct.Query
```

**多角色声明（跨模式复用）**：

```yaml
question:
  type: string
  required: true
  description: 用户问题
  roles:
    - Direct.Query
    - CoT.Query
    - Retrieve.Query

answer:
  type: string
  required: true
  description: 回答内容
  roles:
    - Direct.Answer
    - CoT.Conclusion
```

**完整的 input_schema 示例**：

```yaml
## input_schema

```yaml
query:
  type: string
  required: true
  description: 搜索查询
  roles:
    - Retrieve.Query

constraints:
  type: object
  required: false
  description: 搜索约束条件
  time_range:
    type: string
    description: 时间范围
  source_type:
    type: string
    description: 来源类型
```

**完整的 output_schema 示例**：

```yaml
## output_schema

```yaml
results:
  type: array
  required: true
  description: 搜索结果列表
  roles:
    - Retrieve.SearchResults
  items:
    title:
      type: string
      description: 结果标题
    content:
      type: string
      description: 结果内容
    url:
      type: string
      description: 来源链接

answer:
  type: string
  required: true
  description: 综合回答
  roles:
    - Retrieve.Answer
    - Direct.Answer

sources:
  type: array
  required: false
  description: 引用来源
  roles:
    - Retrieve.Sources
  items:
    type: string
```

#### 2.3.7 语义角色的匹配机制

语义角色是实现技能链式调用的核心机制。当 CognitiveSkill 通过 `select` 节点调用 AtomicSkill 时，系统基于语义角色（而非字段名）进行输入输出匹配。

**匹配流程**：

```
CognitiveSkill (mode: CoT)
    │
    └── select 节点
          │
          ├── input:
          │     Query: question         # 需要 CoT.Query 角色的数据
          │
          └── output:
                Conclusion: result      # 将 CoT.Conclusion 角色的输出存入 result
                    │
                    ▼
          AtomicSkill (capabilityTags 匹配)
                    │
                    ├── input_schema:
                    │     prompt:
                    │       roles: [CoT.Query, Direct.Query]  ← 匹配成功
                    │
                    └── output_schema:
                          answer:
                            roles: [CoT.Conclusion, Direct.Answer]  ← 匹配成功
```

**核心规则**：

| 规则 | 说明 |
|------|------|
| 角色匹配优先 | 匹配依据是语义角色，而非字段名称 |
| 模式前缀自动补全 | select 节点中 `Query` 会根据技能的 `**mode**` 自动补全为 `CoT.Query` |
| 多角色支持 | AtomicSkill 可声明多个角色以支持被不同模式的 CognitiveSkill 调用 |
| 匹配失败处理 | 若无匹配的角色，按 `ignore` 规则处理（默认报错） |

**示例：一个 AtomicSkill 被多个模式调用**

```yaml
# AtomicSkill: answer_generator
## input_schema
question:
  type: string
  roles:
    - Direct.Query      # 支持 Direct 模式
    - CoT.Query         # 支持 CoT 模式
    - Retrieve.Query    # 支持 Retrieve 模式

## output_schema
answer:
  type: string
  roles:
    - Direct.Answer
    - CoT.Conclusion
    - Retrieve.Answer
```

此 AtomicSkill 可被以下任意模式的 CognitiveSkill 调用：

```yaml
# CognitiveSkill A (mode: Direct)
- type: select
  capability: answer_generation
  input:
    Query: user_question      # → Direct.Query
  output:
    Answer: final_answer      # → Direct.Answer

# CognitiveSkill B (mode: CoT)
- type: select
  capability: answer_generation
  input:
    Query: user_question      # → CoT.Query
  output:
    Conclusion: final_answer  # → CoT.Conclusion
```

---




## 3. 内置函数与工具

Aegis 提供一组内置函数用于处理常见的数据操作。内置函数可以在模板表达式中直接调用，无需通过 step 配置。

### 3.1 函数调用语法

#### 基本语法

内置函数使用 `命名空间::函数名(参数)` 的格式调用：

```
namespace::function(arg1, arg2, ..., [ignore: true])
```

| 组成部分        | 说明        | 示例              |
| ----------- | --------- | --------------- |
| `namespace` | 函数所属命名空间  | `array`、`public` |
| `::`        | 命名空间分隔符   | —               |
| `function`  | 函数名       | `flatten`、`exists` |
| `(...)`     | 参数列表      | `({{data}}, 1)` |
| `ignore`    | 可选，异常处理参数 | `ignore: true`  |

**调用示例**：

```yaml
# 在 arg(工具调用) 中使用函数
args:
  flat_data: array::flatten({{nested_array}}, 1)
  unique_items: array::unique({{items}})
  sorted_list: array::sort({{scores}}, "value", "desc")
```

```template
# 在 template 模板中使用函数
处理后的数据：{{array::flatten({{data}}, 2)}}
```

```prompt
# 在 prompt 模板中使用函数（动态生成提示词）
你是一位专业的数据分析师。

{{#when exists(additional_context)}}
补充背景信息：{{additional_context}}
请基于以上背景信息进行分析。
{{/when}}

{{#when empty(search_results)}}
警告：未找到相关搜索结果。请基于现有知识库进行分析。
{{:else}}
已找到 {{array::count({{search_results}})}} 条搜索结果：
{{#for search_results}}
- {{title}}：{{snippet}}
{{/for}}
请基于以上搜索结果进行分析。
{{/when}}

请给出分析结论。
```

#### 命名空间省略规则

当省略 `namespace::` 前缀时，系统默认使用 `public` 命名空间。`public` 命名空间包含通用的基础函数。

```yaml
# 以下两种写法等价
has_value: exists({{optional_field}})
has_value: public::exists({{optional_field}})
```

**命名空间优先级**：

| 调用方式 | 解析为 | 说明 |
|----------|--------|------|
| `function(...)` | `public::function(...)` | 省略命名空间时默认为 public |
| `namespace::function(...)` | 显式指定的命名空间 | 明确使用指定命名空间 |

> **注意**：只有 `public` 命名空间的函数可以省略前缀。其他命名空间（如 `array`）的函数必须显式指定命名空间。

#### 可选参数与默认值

函数参数支持可选设计，未传入的参数使用默认值：

```yaml
# 完整调用：展平深度为 2
result: array::flatten({{data}}, 2)

# 省略可选参数：使用默认深度 1
result: array::flatten({{data}})

# 排序：省略 key 和 order，使用默认升序
sorted: array::sort({{numbers}})

# 排序：指定 key，省略 order（默认 asc）
sorted: array::sort({{items}}, "score")

# 排序：完整指定所有参数
sorted: array::sort({{items}}, "score", "desc")
```

**参数省略规则**：

| 规则 | 说明 |
|------|------|
| 从右向左省略 | 只能省略末尾的可选参数，不能跳过中间参数 |
| 默认值由函数定义 | 每个函数的可选参数都有预设默认值 |
| `ignore` 始终可选 | `ignore` 作为最后一个参数，默认为 `false` |

**参数标记说明**：

在函数签名中，`?` 表示可选参数：

```
array::sort(array: array, key?: string, order?: string, [ignore: boolean]) → array
                         ↑              ↑
                      可选参数        可选参数
```

#### 链式调用

函数支持链式调用，将一个函数的结果作为另一个函数的输入：

```yaml
# 先展平，再去重
result: array::unique(array::flatten({{nested_data}}, 1))

# 先过滤，再排序
top_items: array::sort(array::filter({{items}}, "score > 60"), "score", "desc")
```

#### ignore 参数（函数级异常处理）

每个函数的最后一个可选参数为 `ignore`，用于控制异常处理行为：

| ignore 值 | 行为 |
|-----------|------|
| `false`（默认） | 函数执行失败时抛出异常，终止整个 Skill 执行 |
| `true` | 函数执行失败时返回 `null`，继续执行后续逻辑 |

**语法**：

```yaml
# 默认行为：失败则终止
result: array::flatten({{data}}, 1)

# 显式忽略异常：失败返回 null
safe_result: array::flatten({{data}}, 1, ignore: true)
```

**适用场景**：

```yaml
# 数据可能不存在或格式不正确，允许失败
optional_data: array::unique({{maybe_empty}}, ignore: true)

# 链式调用中某一步可能失败
result: array::sort(array::filter({{items}}, "score > 0", ignore: true), "score")
```

> **注意**：当 `ignore: true` 时，函数失败返回 `null`。后续使用该变量时需考虑空值处理。

### 3.2 array::zip

将多个简单数组按索引位置组装为一个结构化数组（对象数组）。

**函数签名**：

```
array::zip(arrays: object, [ignore: boolean]) → array
```

**参数**：

| 参数 | 类型 | 必需 | 说明 |
|------|------|------|------|
| `arrays` | object | 是 | 键值对，key 为目标字段名，value 为源数组 |
| `ignore` | boolean | 否 | 异常处理，默认 `false` |

**返回类型**：`array` — 组装后的结构化数组

**使用示例**：

```yaml
# 将三个数组组装为结构化数组
students: array::zip({name: {{names}}, score: {{scores}}, grade: {{grades}}})

# 结果示例：
# [
#   {name: "张三", score: 85, grade: "A"},
#   {name: "李四", score: 72, grade: "B"}
# ]
```

**错误处理**：当输入数组长度不一致时，以最短数组长度为准，忽略多余元素。

### 3.3 array::flatten

将嵌套数组展平为一维数组。

**函数签名**：

```
array::flatten(array: array, depth?: number, [ignore: boolean]) → array
```

**参数**：

| 参数 | 类型 | 必需 | 说明 |
|------|------|------|------|
| `array` | array | 是 | 要展平的嵌套数组 |
| `depth` | number | 否 | 展平深度，默认为 1 |
| `ignore` | boolean | 否 | 异常处理，默认 `false` |

**返回类型**：`array` — 展平后的一维数组

**使用示例**：

```yaml
# 展平一层
flat: array::flatten({{nested}}, 1)

# 完全展平（深度为较大值）
fully_flat: array::flatten({{deeply_nested}}, 99)
```

### 3.4 array::unique

对数组去重，返回不包含重复元素的新数组。

**函数签名**：

```
array::unique(array: array, key?: string, [ignore: boolean]) → array
```

**参数**：

| 参数 | 类型 | 必需 | 说明 |
|------|------|------|------|
| `array` | array | 是 | 要去重的数组 |
| `key` | string | 否 | 对于结构化数组，指定用于判断重复的字段名 |
| `ignore` | boolean | 否 | 异常处理，默认 `false` |

**返回类型**：`array` — 去重后的数组

**使用示例**：

```yaml
# 简单数组去重
unique_tags: array::unique({{tags}})

# 结构化数组按字段去重
unique_users: array::unique({{users}}, "id")
```

### 3.5 array::sort

对数组进行排序。

**函数签名**：

```
array::sort(array: array, key?: string, order?: string, [ignore: boolean]) → array
```

**参数**：

| 参数 | 类型 | 必需 | 说明 |
|------|------|------|------|
| `array` | array | 是 | 要排序的数组 |
| `key` | string | 否 | 对于结构化数组，指定排序字段 |
| `order` | string | 否 | 排序方向：`asc`（升序，默认）或 `desc`（降序） |
| `ignore` | boolean | 否 | 异常处理，默认 `false` |

**返回类型**：`array` — 排序后的数组

**使用示例**：

```yaml
# 简单数组升序排序
sorted_numbers: array::sort({{numbers}})

# 结构化数组按字段降序排序
top_scores: array::sort({{results}}, "score", "desc")
```

### 3.6 array::count

统计数组元素数量。

**函数签名**：

```
array::count(array: array, [ignore: boolean]) → number
```

**参数**：

| 参数 | 类型 | 必需 | 说明 |
|------|------|------|------|
| `array` | array | 是 | 要统计的数组 |
| `ignore` | boolean | 否 | 异常处理，默认 `false` |

**返回类型**：`number` — 数组元素数量

**使用示例**：

```yaml
# 统计结果数量
total: array::count({{results}})

# 在条件表达式中使用
when:
  expr: "array::count({{items}}) > 0"
```

### 3.7 array::filter

根据条件过滤数组元素。

**函数签名**：

```
array::filter(array: array, condition: string, [ignore: boolean]) → array
```

**参数**：

| 参数 | 类型 | 必需 | 说明 |
|------|------|------|------|
| `array` | array | 是 | 要过滤的数组 |
| `condition` | string | 是 | 过滤条件表达式，支持 `==`、`!=`、`>`、`<`、`>=`、`<=` |
| `ignore` | boolean | 否 | 异常处理，默认 `false` |

**返回类型**：`array` — 过滤后的数组

**使用示例**：

```yaml
# 过滤分数大于 60 的记录
passed: array::filter({{results}}, "score > 60")

# 过滤状态为 active 的用户
active_users: array::filter({{users}}, "status == \"active\"")
```

### 3.8 array::sum

计算数组中指定字段的总和。

**函数签名**：

```
array::sum(array: array, key?: string, [ignore: boolean]) → number
```

**参数**：

| 参数 | 类型 | 必需 | 说明 |
|------|------|------|------|
| `array` | array | 是 | 要计算的数组 |
| `key` | string | 否 | 对于结构化数组，指定求和的字段名 |
| `ignore` | boolean | 否 | 异常处理，默认 `false` |

**返回类型**：`number` — 字段值的总和

**使用示例**：

```yaml
# 简单数组求和
total: array::sum({{numbers}})

# 结构化数组按字段求和
total_sales: array::sum({{orders}}, "amount")
```

### 3.9 array::avg

计算数组中指定字段的平均值。

**函数签名**：

```
array::avg(array: array, key?: string, [ignore: boolean]) → number
```

**参数**：

| 参数 | 类型 | 必需 | 说明 |
|------|------|------|------|
| `array` | array | 是 | 要计算的数组 |
| `key` | string | 否 | 对于结构化数组，指定求平均的字段名 |
| `ignore` | boolean | 否 | 异常处理，默认 `false` |

**返回类型**：`number` — 字段值的平均值

**使用示例**：

```yaml
# 简单数组求平均
average: array::avg({{scores}})

# 结构化数组按字段求平均
avg_score: array::avg({{results}}, "score")
```

### 3.10 array::max

获取数组中指定字段的最大值。

**函数签名**：

```
array::max(array: array, key?: string, [ignore: boolean]) → number
```

**参数**：

| 参数 | 类型 | 必需 | 说明 |
|------|------|------|------|
| `array` | array | 是 | 要计算的数组 |
| `key` | string | 否 | 对于结构化数组，指定取最大值的字段名 |
| `ignore` | boolean | 否 | 异常处理，默认 `false` |

**返回类型**：`number` — 字段值的最大值

**使用示例**：

```yaml
# 简单数组取最大值
highest: array::max({{numbers}})

# 结构化数组按字段取最大值
max_score: array::max({{results}}, "score")
```

### 3.11 array::min

获取数组中指定字段的最小值。

**函数签名**：

```
array::min(array: array, key?: string, [ignore: boolean]) → number
```

**参数**：

| 参数 | 类型 | 必需 | 说明 |
|------|------|------|------|
| `array` | array | 是 | 要计算的数组 |
| `key` | string | 否 | 对于结构化数组，指定取最小值的字段名 |
| `ignore` | boolean | 否 | 异常处理，默认 `false` |

**返回类型**：`number` — 字段值的最小值

**使用示例**：

```yaml
# 简单数组取最小值
lowest: array::min({{numbers}})

# 结构化数组按字段取最小值
min_score: array::min({{results}}, "score")
```

### 3.12 exists

判断值是否存在（非 null 且非 undefined）。这是 `public` 命名空间下的函数，可以省略命名空间前缀直接调用。

**函数签名**：

```
exists(value: any, [ignore: boolean]) → boolean
```

**参数**：

| 参数 | 类型 | 必需 | 说明 |
|------|------|------|------|
| `value` | any | 是 | 要检查的值 |
| `ignore` | boolean | 否 | 异常处理，默认 `false` |

**返回类型**：`boolean` — 值存在返回 `true`，否则返回 `false`

**判断规则**：

| 值 | exists 返回 |
|------|-------------|
| `null` | `false` |
| `undefined` | `false` |
| `""` (空字符串) | `true` |
| `0` | `true` |
| `false` | `true` |
| `[]` (空数组) | `true` |
| `{}` (空对象) | `true` |
| 其他有效值 | `true` |

**使用示例**：

```yaml
# 在条件表达式中使用（省略 public:: 前缀）
when:
  expr: "exists({{optional_data}})"

# 完整写法（显式命名空间）
when:
  expr: "public::exists({{optional_data}})"

# 结合其他条件
when:
  expr: "exists({{user}}) && {{user.status}} == \"active\""
```

```template
{{#when exists(extra_info)}}
附加信息：{{extra_info}}
{{/when}}
```

**与 empty 函数的区别**：

| 检查方式 | 判断内容 | 适用场景 |
|----------|----------|----------|
| `exists(value)` | 值是否存在（非 null/undefined） | 检查可选字段是否有值 |
| `empty(value)` | 值是否为空（空字符串、空数组、空对象） | 检查值的内容是否为空 |

### 3.13 内置函数与投影语法的选择

| 需求 | 推荐方式 | 说明 |
|------|----------|------|
| 提取单一字段组成简单数组 | `{{arr[*].field}}` 投影语法 | 简洁高效，适合简单场景 |
| 多个数组组装为对象数组 | `array::zip()` 函数 | 支持任意数量的数组组合 |
| 数组排序、过滤、去重 | 对应的 `array::*` 函数 | 提供完整的数据操作能力 |
| 复杂的数组变换 | 函数链式调用 | 通过函数组合完成复杂操作 |
| 检查值是否存在 | `exists()` 函数 | 判断可选字段是否有值 |

### 3.14 函数命名空间

当前版本提供以下命名空间：

| 命名空间 | 说明 | 包含函数 | 可省略前缀 |
|----------|------|----------|:----------:|
| `public` | 通用基础函数 | `exists` | ✓ |
| `array` | 数组操作 | `zip`、`flatten`、`unique`、`sort`、`count`、`filter`、`sum`、`avg`、`max`、`min` | — |

> **命名空间省略**：只有 `public` 命名空间的函数可以省略 `namespace::` 前缀直接调用。

> 未来版本可能扩展更多命名空间，如 `string`（字符串操作）、`math`（数学运算）等。

### 3.15 debug 函数

`debug()` 是 DSL 引擎层的内置调试函数，用于输出技能调试日志。该函数的执行行为受技能的 `**debug**` 属性控制。

**函数签名**：

```
debug(message: string) → null
```

**参数**：

| 参数 | 类型 | 必需 | 说明 |
|------|------|------|------|
| `message` | string | 是 | 要输出的调试消息，支持模板变量 |

**返回类型**：`null` — 该函数不返回任何值

**执行行为控制：**

`debug()` 函数的执行行为由技能顶层的 `**debug**` 属性控制：

| **debug** 属性 | debug() 函数行为 |
|---------------|------------------|
| `false`（默认） | **不执行**，不输出任何日志 |
| `true` | **执行**，输出调试日志 |

**使用示例：**

```template
# 在 template 步骤中使用
{{debug("当前处理到第 " + {{_index}} + " 条记录")}}
{{debug("搜索结果: " + {{results}})}}
```

```prompt
# 在 prompt 步骤中使用
请分析以下数据：
{{debug("输入数据: " + {{input_data}})}}

分析结果...
{{debug("分析完成，输出: " + {{output}})}}
```

**调试日志输出格式：**

当 `**debug**: true` 时，调试日志会输出到技能调试日志系统中，格式如下：

```
[DEBUG] [技能ID] [步骤名称] <消息内容>
```

**示例输出：**

```
[DEBUG] [data_analysis] [process_data] 当前处理到第 5 条记录
[DEBUG] [data_analysis] [process_data] 搜索结果: [{id: 1, name: "测试"}, ...]
```

**与 log 工具的区别：**

| 特性 | debug() 函数 | log 工具 |
|------|-------------|----------|
| 控制方式 | `**debug**` 属性 | 无条件执行 |
| 默认行为 | 不执行（`debug: false`） | 总是执行 |
| 用途 | 开发调试 | 业务审计 |
| 日志级别 | DEBUG | info/warn/error |
| 适用类型 | AtomicSkill、CognitiveSkill | 仅 AtomicSkill |
| 输出位置 | 技能调试日志 | 审计日志系统 |

**最佳实践：**

| 实践 | 说明 |
|------|------|
| 生产环境设置 `debug: false` | 避免生产环境输出调试信息 |
| 开发时设置 `debug: true` | 便于排查问题 |
| 调试信息要有意义 | 输出关键变量和状态信息 |
| 避免敏感信息 | 不要在 debug 中输出密码、密钥等敏感数据 |
| 与 log 工具配合 | debug 用于开发调试，log 用于业务审计 |

**完整示例：**

~~~markdown
# skill: data_processor

**version**: 3.4.2
**debug**: true
**type**: AtomicSkill

## description
数据处理技能，启用调试模式。

## input_schema

```yaml
data:
  type: array
  required: true
  description: 待处理的数据
  items:
    id:
      type: string
    value:
      type: number
```

## output_schema

```yaml
result:
  type: number
  description: 处理结果
```

## steps

### step: debug_input

**type**: template

```template
{{debug("输入数据数量: " + array::count({{data}}))}}
{{debug("第一条数据: " + {{data[0]}})}}
输入数据已接收
```

### step: process

**type**: tool
**tool**: data_processor

```yaml
args:
  input: "{{data}}"
output_schema:
  result:
    type: number
```

### step: debug_output

**type**: template

```template
{{debug("处理结果: " + {{process.value}})}}
处理完成
```
~~~

**性能说明：**

- 当 `**debug**: false` 时，`debug()` 函数调用会被完全优化掉，不产生任何执行开销
- 当 `**debug**: true` 时，每次 `debug()` 调用会产生轻量的日志输出开销

---

### 3.16 工具

除了内置函数，Aegis 还提供一组工具用于特殊场景的数据处理和日志记录,工具可以后续随需求增长继续增加,此次是预置的工具。预置工具通过 step 配置调用，与自定义工具使用方式相同。

**预置工具与内置函数的区别：**

| 特性   | 内置函数        | 预置工具         |
| ---- | ----------- | ------------ |
| 调用方式 | 在模板表达式中直接调用 | 通过 step 配置调用 |
| 使用场景 | 简单数据操作      | 复杂处理、外部系统交互  |
| 输出方式 | 返回值到表达式     | 输出到上下文变量     |

#### 3.16.1 json_select 工具

`json_select` 工具用于从 JSON 字符串中按路径提取子结构。当 template 步骤输出 JSON 格式字符串时，可以使用此工具将字符串中的结构化数据提取到 DSL 上下文中。

**工具名称：** `json_select`

**功能描述：** 从 JSON 对象中按路径表达式提取子结构，输出到指定字段

**核心特点：**
- 单次调用提取一个子结构（对象、数组或基本类型）
- 输出字段名由 `select.target` 动态指定，与 `output_schema` 设计一致
- 提取的对象可直接访问其内部字段

---

**输入参数：**

| 参数 | 类型 | 必需 | 说明 |
|------|------|:----:|------|
| `input` | string/object | 是 | 原始 JSON 数据（JSON 字符串或已解析的对象） |
| `select` | object | 是 | 选择规则 |
| `select.target` | string | 是 | 输出字段名，指定提取结果存入哪个字段 |
| `select.path` | string | 是 | JSON 路径表达式 |

---

**路径表达式语法：**

| 语法 | 说明 | 示例 |
|------|------|------|
| `field` | 访问对象的字段 | `name`、`user.age`、`data.title` |
| `field[index]` | 访问数组的指定索引元素 | `items[0]`、`users[2]` |
| `field[*]` | 访问数组的所有元素（投影） | `items[*]`、`regions[*]` |

---

**输出：**

提取的结果会存入 `output_schema` 中定义的字段，字段名由 `select.target` 指定。

---

**设计限制：单次只能提取一个子结构**

| 可以做 | 不可以做到 |
|-------|----------|
| ✅ 提取 `data` 对象，访问其内部所有字段 | ❌ 同时提取 `data.title` 和 `other.field` |
| ✅ 提取 `items[0]` 对象 | ❌ 同时提取 `items[0]` 和 `items[1]` |
| ✅ 提取 `items[*]` 数组 | ❌ 同时提取 `items[*]` 和 `count` |

**说明：** `json_select` 一次只能提取一个子结构。如果需要多个不相关的字段，需要多次调用。但对于包含多个字段的对象，可以提取父对象后访问其所有内部字段。

---

**使用示例：**

**示例1：提取对象，访问其内部字段**

~~~~markdown
### step: format_response

**type**: template

```template
{
  "status": "success",
  "data": {
    "title": "测试标题",
    "sub_content": "具体文本内容",
    "sub_pic": "图片url"
  }
}
```

### step: extract_data

**type**: tool
**tool**: json_select

```yaml
args:
  input: "{{format_response.value}}"
  select:
    target: my_data        # 指定输出字段名
    path: "data"            # 提取 data 对象
output_schema:
  my_data:
    type: object
    description: 提取的数据对象
```

### step: display_data

**type**: template

```template
标题：{{extract_data.my_data.title}}
内容：{{extract_data.my_data.sub_content}}
图片：{{extract_data.my_data.sub_pic}}
```
~~~~

**示例2：提取数组元素**

~~~~markdown
### step: extract_first_item

**type**: tool
**tool**: json_select

```yaml
args:
  input: '{"items":[{"id":1,"name":"A"},{"id":2,"name":"B"}]}'
  select:
    target: first_item
    path: "items[0]"
output_schema:
  first_item:
    type: object
    description: 第一个元素
```

### step: use_item

**type**: template

```template
商品ID：{{extract_first_item.first_item.id}}
商品名：{{extract_first_item.first_item.name}}
```
```

**示例3：提取数组投影**

```markdown
### step: extract_all_names

**type**: tool
**tool**: json_select

```yaml
args:
  input: '{"items":[{"id":1,"name":"A"},{"id":2,"name":"B"}]}'
  select:
    target: names
    path: "items[*].name"
output_schema:
  names:
    type: array
    description: 所有名称
```

### step: use_names

**type**: template

```template
名称列表：{{extract_all_names.names}}
```
~~~~markdown

---

**设计限制示例：提取多个不相关字段**

如果确实需要多个不相关的字段，需要多次调用：

~~~~markdown
### step: extract_data_obj

**type**: tool
**tool**: json_select

```yaml
args:
  input: '{"data":{"title":"标题"},"count":100,"status":"ok"}'
  select:
    target: data_obj
    path: "data"
output_schema:
  data_obj:
    type: object
```

### step: extract_count

**type**: tool
**tool**: json_select

```yaml
args:
  input: '{"data":{"title":"标题"},"count":100,"status":"ok"}'
  select:
    target: total
    path: "count"
output_schema:
  total:
    type: number
```

### step: use_data

**type**: template

```template
标题：{{extract_data_obj.data_obj.title}}
总数：{{extract_count.total}}
状态：需要第三次调用提取
```
~~~~

---

**与直接变量引用的区别：**

当 template 步骤输出 JSON 字符串时，不能直接通过字段名访问内部数据。必须使用 `json_select` 工具解析：

~~~~markdown
# ❌ 错误：无法直接访问 JSON 字符串内部的字段
{{format_response.data.title}}

# ✅ 正确：使用 json_select 提取对象后访问
### step: extract
**type**: tool
**tool**: json_select
```yaml
args:
  input: "{{format_response.value}}"
  select:
    target: my_data
    path: "data"
output_schema:
  my_data:
    type: string
```
~~~~
后续引用：{{extract.my_data.title}}


---

**最佳实践：**

| 场景 | 推荐方式 | 说明 |
|------|---------|------|
| 需要多个相邻字段 | 提取父对象 | 一次调用获取父对象，访问其所有字段 |
| 需要多个分散字段 | 多次调用 | 每个字段单独提取 |
| 嵌套数据结构 | 提取最近公共父对象 | 减少调用次数，提高效率 |

---

#### 3.16.2 log 工具

`log` 工具用于记录审计日志到数据库，支持层级归属追溯。日志会自动关联当前执行上下文的层级信息（对话 ID、用户目标、执行计划等），可用于业务执行审计和问题排查。

**工具名称：** `log`

**功能描述：** 记录审计日志到数据库，支持层级归属追溯

**输入参数：**

| 参数 | 类型 | 必需 | 默认值 | 说明 |
|------|------|:----:|:------:|------|
| `level` | string | 否 | `info` | 日志级别（`info`、`warn`、`error`） |
| `message` | string | 是 | - | 日志消息，描述当前操作或状态 |
| `data` | object | 否 | - | 附加数据，将被序列化为 JSON 存储 |

**输出：**

| 字段 | 类型 | 说明 |
|------|------|------|
| `logged` | boolean | 是否成功记录日志 |

**自动关联的层级信息：**

每条日志会自动关联当前执行上下文的以下信息：

| 层级 | 说明 |
|------|------|
| `conversationId` | 对话 ID |
| `userGoal` | 用户目标 |
| `planId` | 执行计划 ID |
| `planStepId` | 计划步骤 ID |
| `skillId` | 技能 ID |
| `skillVersion` | 技能版本 |

**使用示例：**

~~~~markdown
### step: log_completion

**type**: tool
**tool**: log

```yaml
args:
  level: "info"
  message: "数据处理完成，共处理 100 条记录"
  data:
    itemCount: 100
    duration: 1250
    status: "success"
output_schema:
  logged:
    type: boolean
```

### step: log_error

**type**: tool
**tool**: log

```yaml
args:
  level: "error"
  message: "数据处理失败"
  data:
    errorCode: "DATA_PROCESSING_ERROR"
    failedItems: 5
    errorMessage: "无效的数据格式"
output_schema:
  logged:
    type: boolean
```

### step: log_warning

**type**: tool
**tool**: log

```yaml
args:
  level: "warn"
  message: "部分数据缺失，使用默认值替代"
  data:
    missingFields: ["email", "phone"]
    defaultValuesUsed: true
output_schema:
  logged:
    type: boolean
```
~~~~

**日志级别说明：**

| 级别 | 适用场景 | 示例 |
|------|---------|------|
| `info` | 正常操作记录 | 处理完成、数据更新、状态变更 |
| `warn` | 警告信息 | 使用默认值、部分失败、配置缺失 |
| `error` | 错误信息 | 处理失败、系统异常、数据错误 |

**日志查询：**

记录的审计日志支持按以下维度查询：
- 按时间范围查询
- 按对话 ID 查询
- 按技能 ID 查询
- 按日志级别筛选
- 按消息内容关键词搜索

**使用建议：**

| 场景 | 建议级别 | 说明 |
|------|---------|------|
| 关键业务操作 | `info` | 记录重要操作的执行状态 |
| 可恢复的异常 | `warn` | 记录不影响整体流程的问题 |
| 严重错误 | `error` | 记录需要人工介入的错误 |
| 调试信息 | - | 使用 `debug()` 函数而非 log 工具 |

**与 debug() 函数的区别：**

| 特性 | `debug()` 函数 | `log` 工具 |
|------|----------------|----------|
| 控制方式 | `**debug**` 属性 | 无条件执行 |
| 默认行为 | 不执行（`debug: false`） | 总是执行 |
| 用途 | 开发调试 | 业务审计 |
| 日志级别 | DEBUG | info/warn/error |
| 适用类型 | AtomicSkill、CognitiveSkill | 仅 AtomicSkill |
| 输出位置 | 技能调试日志 | 审计日志系统（持久化） |

**注意事项：**

1. **性能影响**：`log` 工具会进行数据库写入，频繁使用可能影响性能
2. **敏感信息**：避免在日志中记录密码、密钥等敏感数据
3. **数据大小**：`data` 参数建议控制在合理大小内，避免存储超大对象
4. **适用范围**：`log` 工具仅适用于 AtomicSkill，CognitiveSkill 请使用其他日志机制

---

#### 3.16.3 http_request 工具

`http_request` 工具用于执行 HTTP GET/POST 请求，支持自定义请求头、查询参数和请求体。适用于调用外部 API 或获取远程数据。

**工具名称：** `http_request`

**功能描述：** 执行 HTTP GET/POST 请求，返回响应状态码、响应头和响应体

---

**输入参数：**

| 参数 | 类型 | 必需 | 默认值 | 说明 |
|------|------|:----:|:------:|------|
| `url` | string | 是 | - | 请求 URL（支持 http:// 和 https://） |
| `method` | string | 否 | `GET` | HTTP 方法（`GET` 或 `POST`） |
| `headers` | object | 否 | - | 请求头（如 `Authorization`、`Content-Type`） |
| `query` | object | 否 | - | 查询参数（自动拼接到 URL） |
| `body` | object | 否 | - | 请求体（POST 时使用，自动序列化为 JSON） |
| `timeout` | integer | 否 | `30000` | 超时时间（毫秒，范围 100-300000） |

---

**输出：**

| 字段 | 类型 | 说明 |
|------|------|------|
| `statusCode` | integer | HTTP 状态码（如 200、404、500） |
| `headers` | object | 响应头（JSON 字符串或对象） |
| `body` | string/object | 响应体（原始字符串） |

---

**使用示例：**

**示例1：简单 GET 请求**

~~~~markdown
### step: fetch_data

**type**: tool
**tool**: http_request

```yaml
args:
  url: "https://api.example.com/data"
  method: "GET"
  headers:
    Authorization: "Bearer {{token}}"
output_schema:
  statusCode:
    type: integer
  headers:
    type: object
  body:
    type: string
```

### step: use_response

**type**: template

```template
状态码：{{fetch_data.statusCode}}
响应：{{fetch_data.body}}
```
~~~~

**示例2：POST 请求带请求体**

~~~~markdown
### step: create_record

**type**: tool
**tool**: http_request

```yaml
args:
  url: "https://api.example.com/records"
  method: "POST"
  headers:
    Content-Type: "application/json"
    Authorization: "Bearer {{token}}"
  body:
    id: "{{record_id}}"
    title: "{{title}}"
    content: "{{content}}"
output_schema:
  statusCode:
    type: integer
  body:
    type: string
```
~~~~

**示例3：带查询参数的 GET 请求**

~~~~markdown
### step: search_api

**type**: tool
**tool**: http_request

```yaml
args:
  url: "https://api.example.com/search"
  method: "GET"
  query:
    q: "{{search_keyword}}"
    limit: "10"
    offset: "0"
output_schema:
  statusCode:
    type: integer
  body:
    type: string
```
~~~~

**示例4：处理 JSON 响应**

~~~~markdown
### step: call_api

**type**: tool
**tool**: http_request

```yaml
args:
  url: "https://api.example.com/users/{{user_id}}"
  method: "GET"
output_schema:
  statusCode:
    type: integer
  body:
    type: string
```

### step: parse_response

**type**: tool
**tool**: json_select

```yaml
args:
  input: "{{call_api.body}}"
  select:
    target: user_data
    path: "data"
output_schema:
  user_data:
    type: object
    description: API 返回的用户数据
```

### step: use_data

**type**: template

```template
用户数据：{{parse_response.user_data}}
```
~~~~

---

**请求参数说明：**

**method 参数：**

| 值 | 说明 | 使用场景 |
|----|------|---------|
| `GET` | 获取数据 | 查询、检索、下载 |
| `POST` | 提交数据 | 创建、更新、上传 |

**headers 参数：**

常用请求头示例：

```yaml
headers:
  Authorization: "Bearer {{token}}"
  Content-Type: "application/json"
  Accept: "application/json"
  User-Agent: "Aegis-Skill/1.0"
```

**query 参数：**

查询参数会自动拼接到 URL 后面，支持 URL 编码：

```yaml
query:
  keyword: "{{search_term}}"
  page: "1"
  pageSize: "20"
```

实际请求 URL：`https://api.example.com/search?keyword=xxx&page=1&pageSize=20`

**body 参数：**

POST 请求的请求体，支持对象自动序列化为 JSON：

```yaml
body:
  name: "张三"
  age: 30
  email: "zhangsan@example.com"
```

---

**输出处理：**

**statusCode 字段：**

| 状态码范围 | 说明 | 示例 |
|---------|------|------|
| 2xx | 成功 | 200 OK、201 Created |
| 3xx | 重定向 | 301 Moved Permanently |
| 4xx | 客户端错误 | 400 Bad Request、404 Not Found |
| 5xx | 服务器错误 | 500 Internal Server Error |

**条件处理示例：**

~~~~markdown
### step: check_status

**type**: template

```yaml
when:
  expr: "{{call_api.statusCode}} == 200"
```

```template
请求成功！
响应：{{call_api.body}}
```

### step: handle_error

**type**: template

```yaml
when:
  expr: "{{call_api.statusCode}} != 200"
```

```template
请求失败，状态码：{{call_api.statusCode}}
```
~~~~

---

**安全策略：**

`http_request` 工具受安全策略限制，只能访问白名单内的 URL。

**限制说明：**

| 限制类型 | 说明 |
|---------|------|
| URL 白名单 | 只能访问配置中允许的域名 |
| 超时限制 | 最小 100ms，最大 300000ms（5分钟） |
| 响应大小 | 限制最大响应体大小，防止内存溢出 |

**错误代码：**

| 错误代码 | 说明 |
|---------|------|
| `URL_NOT_ALLOWED` | URL 不在白名单中 |
| `CONNECTION_TIMEOUT` | 连接超时 |
| `RESPONSE_TOO_LARGE` | 响应体超过大小限制 |
| `INVALID_METHOD` | HTTP 方法必须是 GET 或 POST |

---

**使用建议：**

| 场景 | 建议方式 | 说明 |
|------|---------|------|
| 简单数据获取 | 使用 GET 请求 | 更符合 HTTP 语义 |
| 数据提交 | 使用 POST 请求 | 请求体不会出现在 URL 中 |
| 认证请求 | 在 headers 中添加 Authorization | 支持 Bearer Token 等 |
| 大量数据 | 使用 POST + body | URL 长度有限制 |
| 敏感数据 | 使用 POST + body | 避免数据出现在日志或 URL 中 |

---

**与 json_select 配合使用：**

当 API 返回 JSON 数据时，可以配合 `json_select` 工具提取需要的字段：

~~~~markdown
### step: call_api

**type**: tool
**tool**: http_request

```yaml
args:
  url: "https://api.example.com/data"
  method: "GET"
output_schema:
  statusCode:
    type: integer
  body:
    type: string
```

### step: extract_data

**type**: tool
**tool**: json_select

```yaml
args:
  input: "{{call_api.body}}"
  select:
    target: result
    path: "data"
output_schema:
  result:
    type: object
```
~~~~

---

#### 3.16.4 text_file 工具

`text_file` 工具用于读取限定目录下的文本文件内容。支持读取纯文本、Markdown、JSON 等文本格式的文件。

**工具名称：** `text_file`

**功能描述：** 读取限定目录下的文本文件，返回文件内容

---

**输入参数：**

| 参数 | 类型 | 必需 | 说明 |
|------|------|:----:|------|
| `file` | resource | 是 | 文件资源引用（使用 `file://` 协议，相对于限定目录） |

---

**输出：**

| 字段 | 类型 | 说明 |
|------|------|------|
| `content` | string | 文件内容（文本字符串） |

---

**支持的文件类型：**

| 文件类型 | 扩展名示例 | 说明 |
|---------|-----------|------|
| 纯文本 | `.txt`、`.log`、`.csv` | 普通文本文件 |
| Markdown | `.md` | Markdown 文档 |
| JSON | `.json` | JSON 文件（作为原始文本读取） |
| 配置文件 | `.yaml`、`.yml`、`.properties`、`.ini` | 配置文件（作为原始文本读取） |
| 代码文件 | `.js`、`.py`、`.java`、`.ts` | 源代码文件（作为原始文本读取） |

---

**安全机制：**

| 安全限制 | 说明 |
|---------|------|
| 目录限制 | 只能访问配置的限定目录及其子目录 |
| 路径规范化 | 防止路径穿越攻击（如 `../`） |
| 文件大小限制 | 默认最大 10MB，防止内存溢出 |
| 扩展名白名单 | 只允许读取文本类文件，禁止二进制文件 |

---

**使用示例：**

**示例1：读取 JSON 配置文件**

~~~~markdown
### step: read_config

**type**: tool
**tool**: text_file

```yaml
args:
  file: "file://config/app.json"
output_schema:
  content:
    type: string
    description: 配置文件内容
```

### step: parse_config

**type**: tool
**tool**: json_select

```yaml
args:
  input: "{{read_config.content}}"
  select:
    target: config
    path: "$"  # 提取根对象
output_schema:
  config:
    type: object
    description: 解析后的配置对象
```

### step: use_config

**type**: template

```template
应用名称：{{parse_config.config.appName}}
版本号：{{parse_config.config.version}}
```
~~~~

**示例2：读取 Markdown 文档**

~~~~markdown
### step: read_docs

**type**: tool
**tool**: text_file

```yaml
args:
  file: "file://docs/user_guide.md"
output_schema:
  content:
    type: string
    description: 用户指南内容
```

### step: display_docs

**type**: template

```template
# 用户指南

{{read_docs.content}}
```
~~~~

**示例3：读取 CSV 数据文件**

~~~~markdown
### step: read_data

**type**: tool
**tool**: text_file

```yaml
args:
  file: "file://data/sales.csv"
output_schema:
  content:
    type: string
    description: CSV 文件内容
```

### step: process_data

**type**: template

```template
原始数据：
{{read_data.content}}
```
~~~~

---

**文件路径说明：**

**`file://` 协议：**

- 路径是**相对路径**，相对于系统配置的限定根目录
- 路径必须使用 `file://` 前缀
- 示例：`file://config/app.json`

**路径示例：**

```yaml
# 正确的路径
file: "file://config/app.json"
file: "file://data/reports/2026/report.md"
file: "file://logs/error.log"

# 错误的路径（使用绝对路径）
file: "/etc/config.json"
file: "D:\\data\\file.txt"
```

**目录结构示例：**

假设限定根目录为 `/opt/aegis/files/`：

```
/opt/aegis/files/
├── config/
│   ├── app.json
│   └── database.yaml
├── data/
│   ├── reports/
│   │   └── 2026/
│   │       └── report.md
│   └── users.json
└── logs/
    ├── access.log
    └── error.log
```

**对应的引用方式：**

```yaml
file: "file://config/app.json"           # → /opt/aegis/files/config/app.json
file: "file://data/reports/2026/report.md" # → /opt/aegis/files/data/reports/2026/report.md
file: "file://logs/access.log"            # → /opt/aegis/files/logs/access.log
```

---

**与其他工具配合：**

**读取 JSON 后解析数据：**

~~~~markdown
### step: read_file

**type**: tool
**tool**: text_file

```yaml
args:
  file: "file://data/users.json"
output_schema:
  content:
    type: string
```

### step: extract_users

**type**: tool
**tool**: json_select

```yaml
args:
  input: "{{read_file.content}}"
  select:
    target: users
    path: "users"
output_schema:
  users:
    type: array
    description: 用户列表
```
~~~~

---

**注意事项：**

1. **只能读取文本文件**：不支持图片、视频等二进制文件
2. **文件大小限制**：默认最大 10MB，超过限制会报错
3. **路径安全**：只能在限定目录内读取，无法访问系统其他位置
4. **文件编码**：默认使用 UTF-8 编码读取
5. **JSON 作为文本**：JSON 文件作为原始字符串读取，如需解析需配合 `json_select`

---

**错误代码：**

| 错误代码 | 说明 |
|---------|------|
| `FILE_NOT_FOUND` | 文件不存在 |
| `PATH_SECURITY` | 路径超出限定目录或包含非法字符 |
| `FILE_TOO_LARGE` | 文件超过大小限制 |
| `FILE_NOT_READABLE` | 文件不可读 |
| `UNSUPPORTED_TYPE` | 不支持的文件类型（二进制文件） |

---

**使用建议：**

| 场景 | 推荐方式 | 说明 |
|------|---------|------|
| 读取配置文件 | `text_file` + `json_select` | 先读取文本，再解析 JSON |
| 读取文档内容 | 直接使用 `text_file` | Markdown、纯文本直接可用 |
| 大文件处理 | 分块读取或使用专门工具 | 文件太大时考虑其他方案 |
| 动态文件路径 | 从输入参数构建 | 使用 `file://{{path}}` 拼接路径 |

---

#### 3.16.5 db_insert 工具

`db_insert` 工具用于向指定数据库表插入单行数据，返回影响行数和自增主键。

**工具名称：** `db_insert`

**功能：** 向数据库表插入一条记录，返回插入结果和自增主键值。

---

**输入参数：**

| 参数 | 类型 | 必需 | 说明 |
|------|------|:----:|------|
| `datasource` | string | 是 | 数据源名称（在系统配置中预定义） |
| `table` | string | 是 | 表名（仅允许字母、数字、下划线） |
| `fields` | object | 是 | 字段值映射，key 为列名，value 为值 |

---

**输出结构：**

| 字段 | 类型 | 必需 | 说明 |
|------|------|:----:|------|
| `affectedRows` | integer | 是 | 影响行数（通常为 1） |
| `generatedKey` | integer | 否 | 自增主键ID（如果表有自增列） |

---

**使用示例：**

```yaml
### step: insert_order
**type**: tool
**tool**: db_insert

args:
  datasource: order_db
  table: orders
  fields:
    customer_id: "{{customer_info.id}}"
    product_name: "{{product_info.name}}"
    quantity: "{{order.quantity}}"
    total_price: "{{order.total}}"
    status: "pending"
    created_at: "{{now()}}"

output_schema:
  affectedRows:
    type: integer
    description: 插入的行数
  generatedKey:
    type: integer
    description: 新订单的ID
    optional: true
```

---

**安全特性：**

1. **数据源隔离**：数据源来自系统配置，用户无法直接指定连接信息
2. **标识符验证**：表名和字段名仅允许字母、数字、下划线
3. **参数化查询**：所有值通过 PreparedStatement 参数化，防止 SQL 注入
4. **类型安全**：自动映射 Java 类型和 JDBC 类型

---

**错误代码：**

| 错误代码 | 说明 |
|---------|------|
| `DATASOURCE_NOT_FOUND` | 指定的数据源不存在 |
| `TABLE_NOT_FOUND` | 表不存在 |
| `INVALID_IDENTIFIER` | 表名或字段名包含非法字符 |
| `SQL_EXECUTION_ERROR` | SQL 执行失败（如违反约束） |
| `CONNECTION_ERROR` | 数据库连接失败 |

---

#### 3.16.6 db_update 工具

`db_update` 工具用于基于 WHERE 条件更新数据库表中的记录。

**工具名称：** `db_update`

**功能：** 根据 WHERE 条件更新表中的记录，返回影响行数。

---

**输入参数：**

| 参数 | 类型 | 必需 | 说明 |
|------|------|:----:|------|
| `datasource` | string | 是 | 数据源名称（在系统配置中预定义） |
| `table` | string | 是 | 表名（仅允许字母、数字、下划线） |
| `set` | object | 是 | 更新的字段值映射，key 为列名，value 为值 |
| `where` | object | 是 | WHERE 条件映射（不能为空，防止全表更新） |

---

**输出结构：**

| 字段 | 类型 | 必需 | 说明 |
|------|------|:----:|------|
| `affectedRows` | number | 是 | 影响行数 |

---

**使用示例：**

```yaml
### step: update_order_status
**type**: tool
**tool**: db_update

args:
  datasource: order_db
  table: orders
  set:
    status: "completed"
    updated_at: "{{now()}}"
  where:
    id: "{{order_id}}"
    customer_id: "{{customer_id}}"

output_schema:
  affectedRows:
    type: number
    description: 更新的行数
```

---

**安全特性：**

1. **强制 WHERE 条件**：WHERE 条件不能为空，防止全表更新
2. **数据源隔离**：数据源来自系统配置，用户无法直接指定连接信息
3. **标识符验证**：表名和字段名仅允许字母、数字、下划线
4. **参数化查询**：所有值通过 PreparedStatement 参数化，防止 SQL 注入

---

**错误代码：**

| 错误代码 | 说明 |
|---------|------|
| `DATASOURCE_NOT_FOUND` | 指定的数据源不存在 |
| `TABLE_NOT_FOUND` | 表不存在 |
| `INVALID_IDENTIFIER` | 表名或字段名包含非法字符 |
| `EMPTY_WHERE_CLAUSE` | WHERE 条件为空（安全限制） |
| `SQL_EXECUTION_ERROR` | SQL 执行失败 |
| `CONNECTION_ERROR` | 数据库连接失败 |

---

**使用建议：**

| 场景 | 推荐方式 | 说明 |
|------|---------|------|
| 插入新记录 | `db_insert` | 返回自增主键，可用于后续引用 |
| 更新现有记录 | `db_update` | 必须提供 WHERE 条件 |
| 批量操作 | 使用 foreach 循环 | 结合 AtomicSkill 的 foreach 节点 |
| 事务处理 | 在 CognitiveSkill 中编排 | 通过多个步骤实现事务逻辑 |

---

#### 3.16.7 db_select 工具

`db_select` 工具用于执行数据库查询，返回查询结果。

**工具名称：** `db_select`

**功能：** 执行 SELECT 查询，返回结果集。输出结构由调用时通过 `output_schema` 定义。

---

**输入参数：**

| 参数 | 类型 | 必需 | 说明 |
|------|------|:----:|------|
| `datasource` | string | 是 | 数据源名称（在系统配置中预定义） |
| `query` | string | 是 | SELECT 查询语句（仅允许 SELECT） |
| `params` | array | 否 | 查询参数，对应 SQL 中的 `?` 占位符 |

---

**输出结构：**

输出结构由调用时的 `output_schema` 定义，顶级字段必须为 `result`：

```yaml
output_schema:
  result:
    type: object  # 单行查询
    # 或
    type: array   # 多行查询
    items:        # array 类型时定义元素结构
      <column_name>:
        type: <type>
```

---

**使用示例：**

**示例 1：返回单行对象**

```yaml
### step: get_user
**type**: tool
**tool**: db_select

args:
  datasource: user_db
  query: "SELECT id, username, email FROM users WHERE id = ?"
  params:
    - "{{user_id}}"

output_schema:
  result:
    type: object
    description: 用户信息
    id:
      type: integer
      description: 用户ID
    username:
      type: string
      description: 用户名
    email:
      type: string
      description: 邮箱
```

**示例 2：返回多行数组**

```yaml
### step: list_orders
**type**: tool
**tool**: db_select

args:
  datasource: order_db
  query: "SELECT id, customer_name, total FROM orders WHERE status = ?"
  params:
    - "pending"

output_schema:
  result:
    type: array
    description: 订单列表
    items:
      id:
        type: integer
      customer_name:
        type: string
      total:
        type: number
```

**示例 3：使用 result.length**

```yaml
### step: query_pending_orders
**type**: tool
**tool**: db_select

args:
  datasource: order_db
  query: "SELECT id, amount FROM orders WHERE status = 'pending'"
  params: []

output_schema:
  result:
    type: array
    items:
      id:
        type: integer
      amount:
        type: number

### step: check_count
**type**: template

template: |
  共 {{query_pending_orders.result.length}} 个待处理订单。
  {{#when query_pending_orders.result.length > 0}}
  需要尽快处理。
  {{/when}}
```

**示例 4：多表关联查询**

```yaml
### step: get_order_details
**type**: tool
**tool**: db_select

args:
  datasource: order_db
  query: |
    SELECT
      o.id as order_id,
      o.total_amount,
      c.name as customer_name,
      c.email as customer_email,
      p.name as product_name,
      oi.quantity
    FROM orders o
    JOIN customers c ON o.customer_id = c.id
    JOIN order_items oi ON o.id = oi.order_id
    JOIN products p ON oi.product_id = p.id
    WHERE o.id = ?
  params:
    - "{{order_id}}"

output_schema:
  result:
    type: array
    description: 订单详情（包含商品明细）
    items:
      order_id:
        type: integer
      total_amount:
        type: number
      customer_name:
        type: string
      customer_email:
        type: string
      product_name:
        type: string
      quantity:
        type: integer
```

---

**安全特性：**

| 特性 | 说明 |
|------|------|
| **只读限制** | 仅允许 SELECT 语句，拒绝 INSERT/UPDATE/DELETE/DDL |
| **只读数据源** | 使用只读数据源配置 |
| **参数化查询** | 使用 `?` 占位符，防止 SQL 注入 |
| **结果集限制** | 默认最大返回 1000 行，超出时抛出错误 |
| **查询超时** | 默认 30 秒超时限制 |

---

**错误代码：**

| 错误代码 | 说明 |
|---------|------|
| `DATASOURCE_NOT_FOUND` | 指定的数据源不存在 |
| `INVALID_SQL` | SQL 语句非 SELECT 或语法错误 |
| `RESULT_SET_TOO_LARGE` | 结果集超过最大行数限制 |
| `QUERY_TIMEOUT` | 查询超时 |
| `CONNECTION_ERROR` | 数据库连接失败 |

---

## 4. 最佳实践

### 4.1 命名规范

- **Step 名称**：使用动词或动词短语，如 `calculate_total`、`prepare_summary`
- **相关输入输出变量名称**：使用与业务含义相匹配的名词或动名词以及短语，如 `file`、`summary`、`result`、`report`
- **输入参数**：使用下划线命名法，如 `order_id`、`unit_price`
- **工具名称**：使用点号分隔命名空间，如 `database.query`、`search_api`
- **步骤命名**：使用动词或动词短语，如 `calculate_total`、`prepare_summary`，作为变量引用的前缀

### 4.2 设计原则

1. **单一职责**：每个 Skill 应专注于一个任务
2. **步骤复用**：通过步骤组合实现复杂逻辑
3. **明确输入输出**：清晰定义输入输出结构
4. **错误处理**：在条件分支中处理各种情况
5. **链式调用**：上一个技能的 output_schema 可能是下一个技能的 input_schema，保持结构一致

### 4.3 性能优化

1. **减少 LLM 调用**：对于固定格式输出和简单计算，使用 `template` 类型代替 `prompt`
2. **并行执行**：将独立步骤放在不同分支
3. **缓存结果**：在步骤间传递输出而非重复计算

### 4.4 调试技巧

1. **使用 log 工具**：输出中间变量值进行调试（仅限 AtomicSkill）
2. **条件日志**：使用 `when` 条件输出调试信息
3. **分步测试**：单独测试每个步骤

---

## 5. 附录

### 5.1 保留关键词

以下词汇为 DSL 保留关键词，不能用作字段名或变量名：

**系统保留字（以 `_` 开头）：**
- `_` — 循环中的当前元素引用
- `_index` — 循环中的当前索引
- `_input` — 用户原始输入
- 以 `_` 开头的任意其他词（预留给系统扩展）

**DSL 关键词：**
- `type`
- `public`
- `array`
- `string`
- `number`
- `datetime`
- `resource`
- `boolean`
- `for`
- `break`
- `tool`
- `when`
- `ignore`
- `input`
- `output`
- `step`
- `skill`
- `description`
- `required`
- `items`
- `options`
- `display`
- `mapping`
- `layout`
- `break`
- `roles`
- `ui`
- `debug`
-  `args`
- `semantic_role`
- `internal_flow`
- `decision`
- `select`
- `assign`
- `merge`
- `fallback`
- `capability`
- `branches`
- `condition`
- `target`

**内置函数命名空间：**
- `public` — 通用基础函数命名空间（默认命名空间，可省略前缀）
- `array` — 数组操作函数命名空间

### 5.2 修订历史

| 版本    | 日期         | 说明                                                                                   |
| ----- | ---------- | ------------------------------------------------------------------------------------ |
| 3.4.2 | 2026-03-08 | AtomicSkill 变量引用方式调整为步骤名前缀（`{{stepName.field}}`），移除 `varName` 属性，统一变量作用域；新增 `output_schema` 的 `mapping` 映射机制；新增 `json_select`、`log`、`http_request`、`text_file`、`db_insert`、`db_update`、`db_select` 预置工具说明；新增 array 类型的 `.length` 属性说明 |
| 3.4.1 | 2026-03-08 | 新增 debug 内置函数机制，支持通过 `**debug**` 属性控制调试日志输出                                           |
| 3.4.0 | 2026-03-04 | 重构版本                                                                                 |

# Aegis Skill DSL 规范说明文档

> **规范版本：2.1.0**

## 目录

### 通用规范（本文档）
- [1. 概述](#1-概述) — 技能分类、文件骨架、type/version 字段
- [2. 输入定义 (input_schema)](#2-输入定义-input_schema) — 输入参数结构定义
- [3. 执行步骤 (steps)](#3-执行步骤-steps) — Step 类型、变量与模板语法
- [4. Step 类型详解](#4-step-类型详解) — tool/prompt/template/await 步骤
- [5. 内置工具](#5-内置工具) — 数组操作工具
- [6. 最佳实践](#6-最佳实践) — 命名规范、设计原则、性能优化
- [7. 附录](#7-附录) — 保留关键词、修订历史

### 按技能类型分述
- [AtomicSkill.md](./AtomicSkill.md) — 原子技能：output_schema、审计日志、完整示例
- [PresentationSkill.md](./PresentationSkill.md) — 展示技能：UI 展示语义、完整示例

---

## 1. 概述

Aegis Skill DSL 是一种基于 Markdown 的领域特定语言，用于定义可执行的 AI 技能。

### 1.1 技能分类

Aegis 技能分为以下类型：

| 类型 | 说明 | 核心职责 |
|------|------|----------|
| `AtomicSkill` | 原子技能（默认） | 数据处理和业务逻辑，输出结构化数据 |
| `PresentationSkill` | 展示技能 | UI 渲染和数据展示，接收上游技能输出 |
| `CognitiveSkill` | 认知技能（暂未实现） | 推理分析等认知能力 |

**AtomicSkill** 由以下部分组成：
- 输入定义（input_schema）
- 输出定义（output_schema）
- 执行步骤（steps）

**PresentationSkill** 由以下部分组成：
- 输入定义（input_schema）— 对应上游技能的输出
- 执行步骤（steps）— 可选，用于数据转换或 LLM 生成展示内容
- UI 展示语义（ui）

> **设计原则**：AtomicSkill 专注于数据处理，不包含 UI；PresentationSkill 专注于展示，可通过 steps（特别是 prompt/template）进行数据转换和内容生成。技能的串联和编排由 Agent 层负责，DSL 层只需关注单个技能的定义。

Aegis 采用聊天瀑布流作为唯一页面容器，AtomicSkill 负责"数据结构"，PresentationSkill 负责"语义展示"，前端不做智能猜测渲染

### 1.2 AtomicSkill 文件骨架

~~~markdown
# skill: <skill_id>

**version**: <版本号>
**type**: AtomicSkill

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
~~~

### 1.3 PresentationSkill 文件骨架

~~~markdown
# skill: <skill_id>

**version**: <版本号>
**type**: PresentationSkill

## description

<技能描述>

## capabilityTags

- <能力关键词1>
- <能力关键词2>

## input_schema

```yaml
<来自上游技能的输出结构>
```

## steps

### step: <step_name>

**type**: <prompt|template>

<step_configuration>

## ui

```yaml
<UI展示语义定义>
```
~~~

### 1.4 技能文件顶层章节

#### AtomicSkill 章节

| 章节 | 必需 | 说明 |
|------|------|------|
| `# skill: <id>` | 是 | 技能唯一标识符，作为一级标题 |
| `**version**: <版本号>` | 否 | 技能版本号，遵循语义化版本（如 `2.0.0`）。未提供时默认为 `1.0.0` |
| `**type**: AtomicSkill` | 是 | 技能类型，AtomicSkill 可省略（默认值） |
| `## description` | 否 | 技能描述文本 |
| `## capabilityTags` | 否 | 能力关键词列表，用于技能路由和匹配 |
| `## input_schema` | 否 | 输入参数结构定义（YAML 格式） |
| `## output_schema` | 是 | 输出参数结构定义（YAML 格式，平铺字段定义） |
| `## steps` | 是 | 执行步骤列表（至少一个） |

> **注意**：AtomicSkill **不包含** `## ui` 章节。UI 展示由 PresentationSkill 负责。

#### PresentationSkill 章节

| 章节 | 必需 | 说明 |
|------|------|------|
| `# skill: <id>` | 是 | 技能唯一标识符，作为一级标题 |
| `**version**: <版本号>` | 否 | 技能版本号，遵循语义化版本（如 `2.0.0`） |
| `**type**: PresentationSkill` | 是 | 技能类型，必须显式声明 |
| `## description` | 否 | 技能描述文本 |
| `## capabilityTags` | 否 | 能力关键词列表，用于技能路由和匹配 |
| `## input_schema` | 是 | 输入参数结构定义，对应上游 AtomicSkill 的 output_schema |
| `## steps` | 否 | 执行步骤列表，用于数据转换或 LLM 生成展示内容（推荐使用 prompt/template 类型） |
| `## ui` | 是 | UI 展示语义定义 |

> **注意**：PresentationSkill **不包含** `## output_schema` 章节。UI 即其输出。

### 1.5 type 字段

`type` 字段声明在 `version` 之后，格式为：

```markdown
**type**: AtomicSkill
```

或：

```markdown
**type**: PresentationSkill
```

**用法说明：**

- **必填性**：`type` 字段在 PresentationSkill 中必须显式声明；AtomicSkill 可省略（为默认值）
- **默认值**：未声明时默认为 `AtomicSkill`（向后兼容）
- **取值范围**：`AtomicSkill` | `PresentationSkill`（`CognitiveSkill` 暂未实现）
- **结构约束**：不同类型的技能有不同的章节要求（见上表）

### 1.6 version 字段

`version` 字段声明在一级标题 `# skill: <id>` 之后，格式为：

```markdown
**version**: 2.1.0
```

**用法说明：**

- **取值来源**：version 的值对应本规范文档开篇的版本号。当规范版本为 `2.0.0` 时，技能文件的 version 即为 `2.0.0`
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
  type: <string|number|boolean|datetime|resource|array|object>
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
| `datetime` | 时间（日期/时间/日期时间） | `"2026-02-28"`, `"14:30:00"`, `"2026-02-28 14:30:00"` |
| `resource` | 资源引用（文件/链接） | `"file://data/report.xlsx"`, `"https://api.example.com/data"` |
| `array` | 数组 | `[1, 2, 3]` |
| `object` | 对象（键值对） | `{name: "张三", age: 30}` |

### 2.3 datetime 类型

`datetime` 是一种特殊的基本类型，用于处理日期和时间。系统会自动识别输入格式并统一转换。

#### 支持的输入格式

| 格式类型 | 示例 | 说明 |
|----------|------|------|
| 仅日期 | `2026-02-28`、`2026/02/28`、`20260228` | 自动补充时间为 `00:00:00` |
| 仅时间 | `14:30:00`、`14:30`、`143000` | 自动补充日期为当天 |
| 日期+时间 | `2026-02-28 14:30:00`、`2026-02-28T14:30:00` | 完整时间 |

#### 输出格式

系统统一输出为以下格式：

```
yyyy年MM月dd日 HH:mm:ss
```

示例：`2026年02月28日 14:30:00`

#### 使用示例

```yaml
start_time:
  type: datetime
  required: true
  description: 开始时间
  placeholder: 请输入日期或时间

end_date:
  type: datetime
  required: false
  description: 结束日期
  default: "2026-12-31"
```

#### 格式转换规则

| 输入 | 输出 |
|------|------|
| `2026-02-28` | `2026年02月28日 00:00:00` |
| `14:30` | `2026年02月28日 14:30:00`（假设当天为 2026-02-28） |
| `2026-02-28 14:30:00` | `2026年02月28日 14:30:00` |
| `2026/02/28 14:30` | `2026年02月28日 14:30:00` |

### 2.4 resource 类型

`resource` 是一种声明式的资源引用类型，用于表示需要访问的外部资源。系统负责校验 URI 格式，Tool 负责实际访问和处理。

#### 支持的协议

| 协议 | 格式 | 说明 |
|------|------|------|
| `file://` | `file://<相对路径>` | 本地文件，路径相对于运行时系统限定的根目录 |
| `http://` | `http://<url>` | HTTP 网络资源 |
| `https://` | `https://<url>` | HTTPS 网络资源 |

#### 路径说明

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

#### 使用示例

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

#### 值示例

```yaml
# 本地文件（相对路径）
input_file: "file://data/report.xlsx"
input_file: "file://uploads/2026/02/image.png"

# 网络资源
api_endpoint: "https://api.weather.com/forecast"
api_endpoint: "http://internal-service/data"
```

#### 系统职责 vs Tool 职责

| 职责 | 系统 | Tool |
|------|:----:|:----:|
| URI 格式校验 | ✓ | |
| 协议合法性检查 | ✓ | |
| 相对路径解析 | ✓ | |
| 资源实际访问 | | ✓ |
| 内容读取/解析 | | ✓ |
| 错误处理 | | ✓ |

> **设计原则**：`resource` 是声明式的资源引用，系统只做格式校验，不做运行时访问。Tool 接收到的是解析后的资源路径，由 Tool 决定如何访问和处理资源内容。

### 2.5 字段属性

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
| `traits` | array | 否 | 能力特征列表，描述字段支持的运算能力（详见 2.10） |
| `semantic_role` | string | 否 | 语义角色，描述字段的业务语义类别（详见 2.10） |

### 2.6 数组类型与 items

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

#### 类型完整性规则

**当字段类型为 `array` 时，必须通过 `items` 完整定义其元素结构，直到最末级节点的类型为基本类型（`string`、`number`、`boolean`、`datetime`、`resource`）。**

```yaml
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

> **唯一例外**：当 `array` 配合 `options` 实现多选时，不需要定义 `items`，因为 `options` 已隐含元素为字符串类型。

### 2.7 对象类型

当字段类型为 `object` 时，子字段直接平铺在父字段下，不使用 `properties` 包装：

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

#### 类型完整性规则

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

### 2.8 单选和多选（options）

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

### 2.9 简写语法

当字段仅需声明类型时，可使用简写格式（默认 `required: true`，无描述）：

```yaml
query: string
prompt: string
context: string
```

### 2.10 能力语义系统

能力语义系统用于描述字段的运算能力和业务语义，支持平台级的条件判断、UI 自动生成和技能自动编排。

#### 三层结构

| 层级 | 属性 | 必需 | 作用 |
|------|------|------|------|
| Type | `type` | 是 | 数据结构类型，决定基础校验 |
| Trait | `traits` | 否 | 运算能力描述，决定表达式合法性 |
| Semantic Role | `semantic_role` | 否 | 业务语义类别，用于自动编排 |

#### Trait（能力特征）

Trait 描述字段支持的运算能力，平台根据 Trait 决定哪些操作符合法。

**核心 Trait 列表：**

| Trait | 说明 | 支持的操作 | 适用类型 |
|-------|------|-----------|----------|
| `comparable` | 可比较大小 | `>`, `<`, `>=`, `<=`, `between` | `number`, `datetime` |
| `scorable` | 可作为评价指标 | `top_k`, `max`, `min`, `sorting` | `number` |
| `equatable` | 可判断相等 | `==`, `!=`, `in`, `not_in` | 所有基本类型 |
| `emptiable` | 可判断空值 | `is_empty`, `is_not_empty` | `string`, `array`, `object`, `datetime`, `resource` |
| `countable` | 可统计数量 | `count`, `count > n` | `array` |
| `searchable` | 可模糊匹配 | `contains`, `starts_with`, `ends_with`, `like` | `string` |

**Type 与 Trait 的对应关系：**

| Type | 可声明的 Trait |
|------|---------------|
| `number` | `comparable`, `scorable`, `equatable`, `emptiable`（不推荐） |
| `string` | `equatable`, `emptiable`, `searchable` |
| `boolean` | `equatable` |
| `datetime` | `comparable`, `equatable`, `emptiable` |
| `resource` | `equatable`, `emptiable` |
| `array` | `countable`, `emptiable` |
| `object` | `emptiable` |

**使用示例：**

```yaml
price:
  type: number
  description: 商品价格
  traits:
    - comparable
    - scorable

content:
  type: string
  description: 文档内容
  traits:
    - emptiable
    - searchable

results:
  type: array
  description: 搜索结果
  items:
    title:
      type: string
    score:
      type: number
  traits:
    - countable
    - emptiable
```

#### Semantic Role（语义角色）

Semantic Role 表示字段的业务语义类别，用于技能自动编排和语义匹配。

**设计原则：**
- 必须抽象，不包含具体字段名
- 不表达字段粒度
- 由平台统一定义和注册

**内置 Semantic Role（示例）：**

| 类别 | Role | 说明 |
|------|------|------|
| 数值类 | `score` | 评分值 |
| | `confidence` | 置信度 |
| | `probability` | 概率值 |
| | `amount` | 数量 |
| | `finance` | 金融数值 |
| 文本类 | `content` | 主体内容 |
| | `summary` | 摘要 |
| | `label` | 标签 |
| | `keyword` | 关键词 |
| | `explanation` | 解释说明 |
| 集合类 | `document_list` | 文档列表 |
| | `file_list` | 文件列表 |
| | `record_list` | 记录列表 |

**使用示例：**

```yaml
price:
  type: number
  description: 商品价格
  traits:
    - comparable
  semantic_role: finance

analysis:
  type: string
  description: 分析结论
  traits:
    - emptiable
    - searchable
  semantic_role: summary
```

#### 能力语义的作用

| 能力 | 依赖层级 | 说明 |
|------|----------|------|
| 条件表达式校验 | Type + Trait | 编译期校验操作符合法性 |
| UI 自动生成 | Type + Trait | 根据 Trait 决定 UI 组件和操作选项 |
| 技能自动编排 | Trait + Semantic Role | 根据能力和语义匹配技能 |

> **注意**：能力语义主要用于 `output_schema`，帮助平台理解技能输出的能力特征。`input_schema` 中也可使用，但主要场景是输出定义。

---

## 3. 执行步骤 (steps)

### 3.1 概述

每个 Step 代表一个执行单元，按顺序执行。支持四种类型：

| 类型         | 说明                 | 是否调用 LLM |
| ---------- | ------------------ | -------- |
| `tool`     | 调用外部工具             | 否        |
| `prompt`   | 调用 LLM 生成响应        | 是        |
| `template` | 文本模板渲染（变量替换、表达式求值） | 否        |
| `await`    | 暂停执行，等待用户输入        | 否        |

### 3.2 Step 通用属性

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
| `tool` | **不适用** | 工具通过 `output.put(key, value)` 直接写入上下文，输出 key 由工具实现决定 |

语法：
```markdown
**type**: template  **varName**: report
```

特性：
- **直接访问**：通过 `{{varName}}` 访问输出，无需 `.value` 后缀
- **与 output_schema 配合**：varName 应与 output_schema 中定义的字段名对齐，确保最终输出正确
- **命名规则**：必须符合 `^[a-z][a-z0-9_]*$` 模式，且不能与已有 stepName、其他 varName 或输入字段名冲突

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

### 3.3 变量与模板语法

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

**数组字段投影：**

使用 `[*]` 语法从结构化数组中提取单一字段，生成一个简单数组：

```prompt
# 假设 result = [{content: "x", score: 80}, {content: "y", score: 90}]

所有分数：{{result[*].score}}
# 结果：[80, 90]

所有内容：{{result[*].content}}
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

使用 `{{break}}` 提前终止循环。`break` 可以独立使用，也可以嵌套在 `{{#when}}` 条件中实现条件中断。

**基本语法：**

```template
{{#for items}}
{{#when _index >= 3}}
{{break}}
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
{{break}}
{{/when}}
{{/for}}
```

**带累计逻辑的中断：**

```template
{{#for orders}}
订单 {{order_id}}：¥{{amount}}
{{#when _index >= 4}}
（仅显示前 5 条记录）
{{break}}
{{/when}}
{{/for}}
```

**break 语法规则：**

| 规则 | 说明 |
|------|------|
| 作用范围 | 仅在 `{{#for}}` 循环体内有效 |
| 执行效果 | 立即终止当前循环，继续执行循环后的内容 |
| 嵌套循环 | 仅终止最内层循环 |
| 条件嵌套 | 可嵌套在 `{{#when}}` 中实现条件中断 |

**嵌套循环中的 break：**

```template
{{#for categories}}
分类：{{name}}
{{#for items}}
{{#when price > 1000}}
{{break}}
{{/when}}
  - {{title}}: ¥{{price}}
{{/for}}
{{/for}}
```

> 上例中，`{{break}}` 仅终止内层 `items` 循环，外层 `categories` 循环继续执行。

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

## 4. Step 类型详解

### 4.1 Tool 步骤

调用已注册的外部工具（HTTP API、数据库、内部服务等）。工具执行后会将输出写入执行上下文，后续步骤可通过输出的 key 名直接引用。

#### 配置格式

##### step: <step_name>

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
| `tool` | 是 | 工具名称（需在平台中已注册） |
| `input` | 是 | 工具输入参数（支持模板语法） |
| `output_schema` | 是 | 声明工具输出的 key 及其类型 |

> **注意**：Tool 步骤**不需要** `varName`。工具的输出 key 由工具定义决定，技能开发者需查阅工具文档了解其输出字段。

#### 输入参数

在 YAML 配置块中，`input` 键下定义工具的输入参数：

```yaml
input:
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
input:
  tags:
    - "{{tag1}}"
    - "{{tag2}}"
    - fixed_tag
```

### 4.2 Prompt 步骤

将 Prompt 模板发送给 LLM，获取生成结果。

#### 配置格式

~~~markdown
### step: <step_name>

**type**: prompt
**varName**: <variable_name>

```prompt
<prompt_template>
```
~~~

| 字段 | 必需 | 说明 |
|------|------|------|
| `type` | 是 | 必须为 `prompt` |
| `varName` | 是 | 输出变量名 |
| ` ```prompt` 代码块 | 是 | Prompt 模板内容 |

#### 示例

~~~markdown
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
~~~

### 4.3 Template 步骤

纯文本模板渲染，只做变量替换，**不调用 LLM 也不调用 Tool**。渲染结果以 varName 为键直接存入上下文。

适用场景：
- 拼接变量和文字生成一段文本（如订单确认、通知消息）
- 组合多个前置步骤的输出为统一格式
- 不需要 LLM 智能生成的纯模板渲染

#### 配置格式

~~~markdown
### step: <step_name>

**type**: template
**varName**: <variable_name>

```template
模板内容，支持 {{variable}} 语法
```
~~~

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

### 4.4 Await 步骤

暂停 Skill 执行，等待用户提供额外输入后继续执行。用于人机交互控制流场景。

#### 配置格式

~~~markdown
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
~~~

| 字段 | 必需 | 说明 |
|------|------|------|
| `type` | 是 | 必须为 `await` |
| `message` | 是 | 向用户展示的提示信息 |
| `input_schema` | 是 | 用户输入的结构定义 |

> **注意**：Await 步骤**不需要** `varName`。用户提交输入后，`input_schema` 中定义的每个字段直接以字段名为 key 写入执行上下文。

---

## 5. 内置工具

Aegis 提供一组内置工具用于处理常见的数据操作，这些工具无需额外注册即可在技能中使用。

### 5.1 array.zip

将多个简单数组按索引位置组装为一个结构化数组（对象数组）。

**输入参数**：

| 参数 | 类型 | 必需 | 说明 |
|------|------|------|------|
| `arrays` | object | 是 | 键值对，key 为目标字段名，value 为源数组变量 |

**输出**：

| 字段 | 类型 | 说明 |
|------|------|------|
| `result` | array | 组装后的结构化数组 |

**使用示例**：

~~~markdown
### step: combine_data

**type**: tool
**tool**: array.zip

```yaml
input:
  arrays:
    name: "{{names}}"
    score: "{{scores}}"
    grade: "{{grades}}"
output_schema:
  result:
    type: array
    description: 组装后的学生成绩数组
    items:
      name:
        type: string
      score:
        type: number
      grade:
        type: string
```
~~~

### 5.2 array.flatten

将嵌套数组展平为一维数组。

**输入参数**：

| 参数 | 类型 | 必需 | 说明 |
|------|------|------|------|
| `array` | array | 是 | 要展平的嵌套数组 |
| `depth` | number | 否 | 展平深度，默认为 1 |

### 5.3 array.unique

对数组去重，返回不包含重复元素的新数组。

**输入参数**：

| 参数 | 类型 | 必需 | 说明 |
|------|------|------|------|
| `array` | array | 是 | 要去重的数组 |
| `key` | string | 否 | 对于结构化数组，指定用于判断重复的字段名 |

### 5.4 array.sort

对数组进行排序。

**输入参数**：

| 参数 | 类型 | 必需 | 说明 |
|------|------|------|------|
| `array` | array | 是 | 要排序的数组 |
| `key` | string | 否 | 对于结构化数组，指定排序字段 |
| `order` | string | 否 | 排序方向：`asc`（升序，默认）或 `desc`（降序） |

### 5.5 array.filter

根据条件过滤数组元素。

**输入参数**：

| 参数 | 类型 | 必需 | 说明 |
|------|------|------|------|
| `array` | array | 是 | 要过滤的数组 |
| `condition` | string | 是 | 过滤条件表达式，支持 `==`、`!=`、`>`、`<`、`>=`、`<=` |

### 5.6 内置工具与投影语法的选择

| 需求 | 推荐方式 | 说明 |
|------|----------|------|
| 提取单一字段组成简单数组 | `{{arr[*].field}}` 投影语法 | 简洁高效，适合简单场景 |
| 多个数组组装为对象数组 | `array.zip` 工具 | 支持任意数量的数组组合 |
| 数组排序、过滤、去重 | 对应的 `array.*` 工具 | 提供完整的数据操作能力 |
| 复杂的数组变换 | 工具组合使用 | 通过多个步骤完成复杂操作 |

---

## 6. 最佳实践

### 6.1 命名规范

- **Step 名称**：使用动词或动词短语，如 `calculate_total`、`prepare_summary`
- **相关输入输出变量名称**：使用与业务含义相匹配的名词或动名词以及短语，如 `file`、`summary`、`result`、`report`
- **输入参数**：使用下划线命名法，如 `order_id`、`unit_price`
- **工具名称**：使用点号分隔命名空间，如 `database.query`、`search_api`
- **varName**：与 output_schema 中的字段名对齐，如输出定义了 `report` 字段，则 varName 也用 `report`

### 6.2 设计原则

1. **单一职责**：每个 Skill 应专注于一个任务
2. **步骤复用**：通过步骤组合实现复杂逻辑
3. **明确输入输出**：清晰定义输入输出结构
4. **错误处理**：在条件分支中处理各种情况
5. **链式调用**：上一个技能的 output_schema 可能是下一个技能的 input_schema，保持结构一致

### 6.3 性能优化

1. **减少 LLM 调用**：对于固定格式输出和简单计算，使用 `template` 类型代替 `prompt`
2. **并行执行**：将独立步骤放在不同分支
3. **缓存结果**：在步骤间传递输出而非重复计算

### 6.4 调试技巧

1. **使用 log 工具**：输出中间变量值进行调试（仅限 AtomicSkill）
2. **条件日志**：使用 `when` 条件输出调试信息
3. **分步测试**：单独测试每个步骤

---

## 7. 附录

### 7.1 保留关键词

以下词汇为 DSL 保留关键词，不能用作字段名或变量名：

**系统保留字（以 `_` 开头）：**
- `_` — 循环中的当前元素引用
- `_index` — 循环中的当前索引
- `_input` — 用户原始输入
- 以 `_` 开头的任意其他词（预留给系统扩展）

**DSL 关键词：**
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
- `options`
- `display`
- `mapping`
- `layout`
- `break`
- `traits`
- `semantic_role`

### 7.2 修订历史

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
| 1.1.0 | 2026-02-21 | 新增 UI 展示语义章节：引入 `## ui` 作为必需顶层章节；定义 display 类型；建立组件分类体系；mixed 采用纯布局 DSL 设计 |
| 1.1.0 | 2026-02-23 | 新增审计日志章节：强调审计日志是 Aegis Skill 核心特性；文档化 `log` 工具的使用方法和最佳实践 |
| 1.2.0 | 2026-02-27 | **类型完整性规则**：强制要求 `array` 和 `object` 类型必须完整定义子级结构；**数组字段投影**：新增 `{{arr[*].field}}` 语法；**循环索引**：新增 `{{_index}}` 内置变量；**内置工具**：新增数组操作工具；**统一 when 条件语法** |
| 2.0.0 | 2026-02-27 | **技能分类体系**：引入 `type` 字段区分技能类型；定义 `AtomicSkill` 和 `PresentationSkill`；**文档拆分**：拆分为 README.md（通用规范）、AtomicSkill.md、PresentationSkill.md 三个文件；**审计日志**：明确仅适用于 AtomicSkill |
| 2.1.0 | 2026-02-28 | **循环控制指令**：新增 `{{break}}` 语法支持在循环中提前终止；**datetime 类型**：新增时间基本类型，支持多种输入格式；**resource 类型**：新增资源引用类型，支持 `file://` 和 `http[s]://`；**能力语义系统**：新增 `traits`（能力特征）和 `semantic_role`（语义角色）字段属性，支持条件表达式校验、UI 自动生成和技能自动编排 |

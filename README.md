# Aegis Skill DSL 规范说明文档

> **规范版本：3.1.0**

## 目录

### 通用规范（本文档）
- [1. 概述](#1-概述) — 技能分类、文件骨架、type/version 字段
- [2. 语义类型系统](#2-语义类型系统) — 语义类型定义、Type.constraints 语法
- [3. 输入定义 (input_schema)](#3-输入定义-input_schema) — 输入参数结构定义
- [4. 执行步骤 (steps)](#4-执行步骤-steps) — Step 类型、变量与模板语法
- [5. Step 类型详解](#5-step-类型详解) — tool/prompt/template/await 步骤
- [6. 内置函数](#6-内置函数) — 数组操作函数、函数调用语法
- [7. 最佳实践](#7-最佳实践) — 命名规范、设计原则、性能优化
- [8. 附录](#8-附录) — 保留关键词、修订历史

### 按技能类型分述
- [AtomicSkill.md](./AtomicSkill.md) — 原子技能：output_schema、审计日志、完整示例
- [PresentationSkill.md](./PresentationSkill.md) — 展示技能：UI 展示语义、完整示例
- [CognitiveSkill.md](./CognitiveSkill.md) — 认知技能：internal_flow、动态决策、完整示例

---

## 1. 概述

Aegis Skill DSL 是一种基于 Markdown 的领域特定语言，用于定义可执行的 AI 技能。

### 1.1 技能分类

Aegis 技能分为以下类型：

| 类型 | 说明 | 核心职责 |
|------|------|----------|
| `AtomicSkill` | 原子技能（默认） | 数据处理和业务逻辑，输出结构化数据 |
| `PresentationSkill` | 展示技能 | UI 渲染和数据展示，接收上游技能输出 |
| `CognitiveSkill` | 认知技能 | 复杂决策和动态编排，运行时决策、能力匹配 |

**AtomicSkill** 由以下部分组成：
- 输入定义（input_schema）
- 输出定义（output_schema）
- 执行步骤（steps）

**PresentationSkill** 由以下部分组成：
- 输入定义（input_schema）— 对应上游技能的输出
- 执行步骤（steps）— 可选，用于数据转换或 LLM 生成展示内容
- UI 展示语义（ui）

**CognitiveSkill** 由以下部分组成：
- 输入定义（input_schema）— 复杂决策所需的输入数据
- 内部控制流（internal_flow）— 决策逻辑和能力编排
- 输出定义（output_schema）— 决策结果和处理后的数据

> **设计原则**：AtomicSkill 专注于数据处理，不包含 UI；PresentationSkill 专注于展示，可通过 steps（特别是 prompt/template）进行数据转换和内容生成；CognitiveSkill 专注于复杂决策和动态编排，通过 internal_flow 实现运行时的条件判断和能力选择。技能的串联和编排由 Agent 层负责，DSL 层只需关注单个技能的定义。

Aegis 采用聊天瀑布流作为唯一页面容器，AtomicSkill 负责"数据结构"，PresentationSkill 负责"语义展示"，前端不做智能猜测渲染

### 1.2 AtomicSkill 文件骨架

~~~markdown
# skill: <skill_id>

**version**: <版本号>
**ignore**: <true|false>
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
**ignore**: <true|false>
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

### 1.4 CognitiveSkill 文件骨架

~~~markdown
# skill: <skill_id>

**version**: <版本号>
**ignore**: <true|false>
**timeout**: <超时毫秒数>
**type**: CognitiveSkill

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

### 1.5 技能文件顶层章节

#### AtomicSkill 章节

| 章节 | 必需 | 说明 |
|------|------|------|
| `# skill: <id>` | 是 | 技能唯一标识符，作为一级标题 |
| `**version**: <版本号>` | 否 | 技能版本号，遵循语义化版本（如 `2.0.0`）。未提供时默认为 `1.0.0` |
| `**ignore**: <true\|false>` | 否 | 技能级异常处理，默认为 `false` |
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
| `**ignore**: <true\|false>` | 否 | 技能级异常处理，默认为 `false` |
| `**type**: PresentationSkill` | 是 | 技能类型，必须显式声明 |
| `## description` | 否 | 技能描述文本 |
| `## capabilityTags` | 否 | 能力关键词列表，用于技能路由和匹配 |
| `## input_schema` | 是 | 输入参数结构定义，对应上游 AtomicSkill 的 output_schema |
| `## steps` | 否 | 执行步骤列表，用于数据转换或 LLM 生成展示内容（推荐使用 prompt/template 类型） |
| `## ui` | 是 | UI 展示语义定义 |

> **注意**：PresentationSkill **不包含** `## output_schema` 章节。UI 即其输出。

#### CognitiveSkill 章节

| 章节 | 必需 | 说明 |
|------|------|------|
| `# skill: <id>` | 是 | 技能唯一标识符，作为一级标题 |
| `**version**: <版本号>` | 否 | 技能版本号，遵循语义化版本（如 `3.0.0`） |
| `**ignore**: <true\|false>` | 否 | 技能级异常处理，默认为 `false` |
| `**timeout**: <毫秒数>` | 否 | 执行超时时间（毫秒），默认由平台配置 |
| `**type**: CognitiveSkill` | 是 | 技能类型，必须显式声明 |
| `## description` | 否 | 技能描述文本 |
| `## capabilityTags` | 否 | 能力关键词列表，用于技能路由和匹配 |
| `## input_schema` | 是 | 输入参数结构定义（YAML 格式） |
| `## internal_flow` | 是 | 内部控制流定义，包含决策节点序列 |
| `## output_schema` | 是 | 输出参数结构定义（YAML 格式） |

> **注意**：CognitiveSkill **不包含** `## steps` 和 `## ui` 章节。决策逻辑通过 `internal_flow` 定义。

### 1.6 type 字段

`type` 字段声明在 `version` 之后，格式为：

```markdown
**type**: AtomicSkill
```

或：

```markdown
**type**: PresentationSkill
```

或：

```markdown
**type**: CognitiveSkill
```

**用法说明：**

- **必填性**：`type` 字段在 PresentationSkill 和 CognitiveSkill 中必须显式声明；AtomicSkill 可省略（为默认值）
- **默认值**：未声明时默认为 `AtomicSkill`（向后兼容）
- **取值范围**：`AtomicSkill` | `PresentationSkill` | `CognitiveSkill`
- **结构约束**：不同类型的技能有不同的章节要求（见上表）

### 1.7 version 字段

`version` 字段声明在一级标题 `# skill: <id>` 之后，格式为：

```markdown
**version**: 2.2.0
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

### 1.8 ignore 字段（技能级异常处理）

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

- **可选技能**：某些技能的执行结果不是必需的，失败不应影响整体流程
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

| 层级 | 属性位置 | 作用范围 |
|------|----------|----------|
| 技能级 | `**ignore**:` 顶层字段 | 整个技能执行 |
| 步骤级 | Step 的 `ignore` 属性 | 单个步骤执行 |
| 函数级 | 函数的 `ignore` 参数 | 单次函数调用 |

> **优先级**：步骤级/函数级的 `ignore` 在各自范围内独立生效。技能级 `ignore` 仅在整个技能执行失败（未被步骤级捕获）时生效。

### 1.9 timeout 字段（CognitiveSkill 专用）

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

---

## 2. 语义类型系统

> **3.1.0 新特性**：从 DSL 3.1.0 版本开始，`input_schema` 和 `output_schema` 中**必须**使用语义类型，**禁止**使用原始类型（`string`、`number`、`integer`、`boolean`、`datetime`）。此规则确保技能的输入输出具有明确的业务语义。

### 2.1 设计原则

语义类型系统的核心目标是为技能的输入输出提供**业务语义约束**，而非仅仅是数据结构约束。

| 原则 | 说明 |
|------|------|
| 语义明确 | 每个类型都有明确的业务含义，而非泛化的数据结构 |
| 类型安全 | 编译期校验类型正确性，避免运行时错误 |
| 可组合 | 通过 `array` + `items: SemanticType` 实现集合类型 |
| 可扩展 | 支持通过配置文件注册自定义语义类型 |

### 2.2 内置语义类型

系统预定义以下语义类型，按功能分为七层：

#### 第一层：输入语义类型

| 类型 | 别名 | 说明 | 典型字段 |
|------|------|------|----------|
| `Query` | 查询 | 用户输入的问题或指令 | content, intent, context, constraints |
| `Entity` | 实体 | 业务对象 | entity_type, id, attributes |

#### 第二层：知识语义类型

| 类型 | 别名 | 说明 | 典型字段 |
|------|------|------|----------|
| `Document` | 文档 | 内部/外部文档 | title, content, source, source_type, metadata |
| `Knowledge` | 知识 | 结构化知识 | topic, content, category, confidence, source |
| `SearchResult` | 搜索结果 | 外部搜索返回 | query, title, snippet, url, relevance, timestamp |

#### 第三层：规则语义类型

| 类型 | 别名 | 说明 | 典型字段 |
|------|------|------|----------|
| `Rule` | 规则 | 业务规则/政策条款 | rule_id, name, description, condition, action, priority |
| `Validation` | 校验结果 | 规则校验输出 | rule, passed, message, severity, suggestion |
| `Compliance` | 合规检查 | 合规性评估 | subject, rules_checked, violations, risk_level, summary |

#### 第四层：推理语义类型

| 类型 | 别名 | 说明 | 典型字段 |
|------|------|------|----------|
| `Evidence` | 证据 | 支持推理的论据 | content, evidence_type, confidence, source, source_type |
| `Reasoning` | 推理 | 推理链中的一步 | premise, inference, conclusion, confidence |
| `Conclusion` | 结论 | 最终结论 | content, confidence, reasoning_chain, evidence_summary |

#### 第五层：流程语义类型

| 类型          | 别名  | 说明   | 典型字段                                                     |
| ----------- | --- | ---- | -------------------------------------------------------- |
| `Step`      | 步骤  | 流程步骤 | sequence, action, actor, condition, output               |
| `Procedure` | 流程  | 操作流程 | name, purpose, steps, preconditions, postconditions      |
| `Decision`  | 决策  | 决策点  | question, options, recommendation, rationale, trade_offs |

#### 第六层：度量语义类型

| 类型 | 别名 | 说明 | 典型字段 |
|------|------|------|----------|
| `Score` | 评分 | 评分值（0-1 或百分比） | value, scale, label |
| `Count` | 计数 | 数量统计值 | value, unit |
| `Fact` | 事实 | 单个事实陈述 | content, category, confidence |

#### 第七层：输出语义类型

| 类型 | 别名 | 说明 | 典型字段 |
|------|------|------|----------|
| `Answer` | 回答 | 标准回答 | content, confidence, sources, follow_up |
| `Analysis` | 分析 | 综合分析 | subject, findings, conclusion, recommendations, data_sources |
| `Report` | 报告 | 结构化报告 | title, summary, sections, conclusion, generated_at |

### 2.3 禁止的原始类型

以下原始类型在 `input_schema` 和 `output_schema` 中**禁止使用**：

| 禁止的类型 | 替代方案 |
|------------|----------|
| `string` | 根据语义选择：`Query`（问题）、`Document`（文档）、`Answer`（回答）、`Fact`（事实）等 |
| `number` | 根据语义选择：`Score`（评分）、`Count`（计数）等 |
| `integer` | 使用 `Count` 或包含整数字段的语义类型 |
| `boolean` | 使用 `Validation`（校验结果）或在语义类型内部使用 |
| `datetime` | 在语义类型的字段内部使用，不作为顶层类型 |

**例外情况**：

| 场景 | 允许使用原始类型 | 说明 |
|------|------------------|------|
| Step 的 `output_schema` | ✓ | Tool 返回值映射，允许使用基本类型 |
| 语义类型内部字段 | ✓ | 语义类型的字段定义中可使用基本类型 |

### 2.4 Type.constraints 语法

`Type.constraints` 是一种特殊的类型声明语法，用于表示**对某个语义类型的约束条件**。

**语法格式**：

```yaml
<field_name>:
  type: <SemanticType>.constraints
  required: <true|false>
  description: <字段描述>
```

**语义含义**：

`Type.constraints` 表示该字段是对 `Type` 类型的**约束条件**或**筛选参数**，而非 `Type` 类型本身。系统会校验 `Type` 必须是已注册的语义类型。

**使用示例**：

```yaml
# Query.constraints - 查询的约束条件
time_range:
  type: Query.constraints
  required: false
  description: 时间范围限制（如"最近一周"、"2025年"）

# Document.constraints - 文档的筛选条件
max_length:
  type: Document.constraints
  required: false
  description: 文档最大长度限制

# Analysis.constraints - 分析的输出格式约束
output_format:
  type: Analysis.constraints
  required: false
  description: 输出格式（detailed/summary/bullet）
  options:
    - detailed
    - summary
    - bullet

# Evidence.constraints - 证据收集的约束
evidence_type:
  type: Query.constraints
  required: false
  description: 证据类型限制
  options:
    - supporting
    - opposing
    - neutral
```

**完整技能示例**：

```yaml
# input_schema 示例
question:
  type: Query
  required: true
  description: 用户提出的问题
context:
  type: Document
  required: false
  description: 问题的背景上下文
max_depth:
  type: Query.constraints      # 对 question 字段的约束
  required: false
  description: 最大分解层级（默认2层）
style:
  type: Answer.constraints     # 对 output_schema 中 answer 字段的约束
  required: false
  description: 回答风格
  options:
    - concise
    - detailed
    - bullet

# 对应的 output_schema 示例
answer:
  type: Answer                 # style 字段约束的就是这个输出
  required: true
  description: 生成的回答内容
sub_questions:
  type: array
  items: Query
  description: 分解后的子问题列表
```

> **命名约定**：`Type.constraints` 字段通常约束同名或相关的语义类型字段。例如 `Answer.constraints` 类型的 `style` 字段约束 `output_schema` 中 `Answer` 类型的字段（如 `answer`）。

**Type.constraints 对应关系规则**：

`Type.constraints` 必须在技能的 `input_schema` 或 `output_schema` 中存在对应的 `Type` 语义类型字段，否则"约束"无从约束，语义不完整。

| 约束类型 | 对应字段位置 | 说明 |
|----------|-------------|------|
| `Query.constraints` | `input_schema` 中的 `Query` 字段 | 约束查询条件 |
| `Answer.constraints` | `output_schema` 中的 `Answer` 字段 | 约束回答风格 |
| `Document.constraints` | `input_schema` 或 `output_schema` 中的 `Document` 字段 | 约束文档处理 |
| `Analysis.constraints` | `output_schema` 中的 `Analysis` 字段 | 约束分析格式 |

> **注意**：`input_schema` 和 `output_schema` 是声明式定义，对应关系不依赖定义顺序，只要能在技能范围内找到匹配的语义类型即可。

**Type.constraints vs 直接使用语义类型**：

| 场景 | 使用方式 | 示例 |
|------|----------|------|
| 主体数据 | 直接使用语义类型 | `question: type: Query` |
| 约束条件 | 使用 Type.constraints | `max_length: type: Document.constraints` |
| 筛选参数 | 使用 Type.constraints | `filter_type: type: Entity.constraints` |
| 格式选项 | 使用 Type.constraints | `style: type: Analysis.constraints` |

### 2.5 数组类型

使用 `type: array` + `items: <SemanticType>` 组合表示语义类型的集合：

```yaml
# 证据列表
evidence_list:
  type: array
  required: true
  description: 收集的证据列表
  items: Evidence

# 搜索结果列表
results:
  type: array
  required: true
  description: 搜索结果
  items: SearchResult
  traits:
    - countable
    - emptiable

# 事实要点列表
key_points:
  type: array
  required: false
  description: 关键要点
  items: Fact
  traits:
    - countable
```

**数组元素类型规则**：

| 规则 | 说明 |
|------|------|
| 必须指定 items | `array` 类型必须通过 `items` 指定元素类型 |
| items 使用语义类型 | `items` 的值必须是已注册的语义类型 |
| 支持嵌套 | 元素类型可以是另一个数组或语义类型 |

> **关于泛型操作**：Skill 的 `input_schema` 和 `output_schema` 不支持泛型数组（即不声明 `items` 的数组）。如需实现泛型数组操作（如通用过滤、排序、聚合），应通过 Tool 层实现。Tool 内部可以是泛型的，但 Skill 边界的语义类型必须明确。

### 2.6 类型选择指南

根据业务场景选择合适的语义类型：

| 业务场景 | 推荐类型 | 说明 |
|----------|----------|------|
| 用户输入的问题 | `Query` | 用于问答、搜索、分析等场景 |
| 搜索/检索结果 | `SearchResult` | 来自搜索引擎或知识库的结果 |
| 文档内容 | `Document` | 待分析、待摘要的文档 |
| 摘要/总结 | `Answer` | 对内容的总结回答 |
| 分析结论 | `Analysis` 或 `Conclusion` | 深度分析的结论 |
| 推理过程 | `Reasoning` | 多步推理的中间过程 |
| 证据支撑 | `Evidence` | 支持推理的论据 |
| 评分/打分 | `Score` | 置信度、相关性等评分 |
| 数量统计 | `Count` | 计数、数量等 |
| 事实陈述 | `Fact` | 单个事实或要点 |
| 业务实体 | `Entity` | 通用业务对象 |
| 校验结果 | `Validation` | 规则校验的结果 |
| 完整报告 | `Report` | 结构化的报告输出 |

---

## 3. 输入定义 (input_schema)

### 3.1 基本语法

```yaml
<field_name>:
  type: <SemanticType>
  required: <true|false>
  description: <字段描述>
  default: <默认值>           # 可选
  options: [选项1, 选项2, ...] # 可选，用于枚举约束
```

> **重要**：从 3.1.0 版本开始，`type` 必须使用语义类型或 `Type.constraints` 语法，禁止使用原始类型。

### 3.2 支持的类型

| 类型分类 | 说明 | 示例 |
|----------|------|------|
| 语义类型 | 具有业务语义的类型 | `Query`、`Document`、`Evidence` |
| 约束类型 | `Type.constraints` 语法 | `Query.constraints`、`Document.constraints` |
| 容器类型 | 数组容器 | `array`（必须配合 `items` 使用） |

**已废弃的类型**（3.1.0 起禁止使用）：

| 废弃类型 | 替代方案 |
|----------|----------|
| `string` | 使用语义类型：`Query`、`Answer`、`Fact` 等 |
| `number` | 使用语义类型：`Score`、`Count` 等 |
| `boolean` | 使用 `Validation` 或在语义类型内部定义 |
| `datetime` | 在语义类型字段内部使用 |
| `resource` | 使用 `Document`（带 source 字段） |
| `object` | 使用具体的语义类型：`Entity`、`Evidence` 等 |

### 3.3 datetime 类型（语义类型内部使用）

`datetime` 是语义类型**内部字段**使用的基本类型，用于处理日期和时间。**不能作为顶层 `input_schema` 或 `output_schema` 的类型**。

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

### 3.4 resource 类型（语义类型内部使用）

`resource` 是语义类型**内部字段**使用的基本类型，用于表示需要访问的外部资源。**推荐使用 `Document` 语义类型（带 source 字段）替代顶层 resource 类型**。

#### 支持的协议

| 协议 | 格式 | 说明 |
|------|------|------|
| `file://` | `file://<相对路径>` | 本地文件，路径相对于运行时系统限定的根目录 |
| `http://` | `http://<url>` | HTTP 网络资源 |
| `https://` | `https://<url>` | HTTPS 网络资源 |

### 3.5 字段属性

| 属性 | 类型 | 必需 | 说明 |
|------|------|------|------|
| `type` | string | 是 | 语义类型或 `Type.constraints` |
| `required` | boolean | 否 | 是否必填（默认 true） |
| `description` | string | 否 | 字段描述信息 |
| `default` | any | 否 | 字段默认值 |
| `options` | array | 否 | 枚举选项列表，用于约束取值范围 |
| `traits` | array | 否 | 能力特征列表，描述字段支持的运算能力（详见 3.10） |
| `semantic_role` | string | 否 | 语义角色，描述字段的业务语义类别（详见 3.10） |

### 3.6 数组类型与 items

当字段类型为 `array` 时，使用 `items` 指定语义类型。

#### 使用语义类型作为 items（推荐）

```yaml
# 证据列表 - 使用 Evidence 语义类型
evidence_list:
  type: array
  description: 收集的证据列表
  items: Evidence

# 搜索结果列表 - 使用 SearchResult 语义类型
results:
  type: array
  description: 搜索结果
  items: SearchResult

# 关键要点列表 - 使用 Fact 语义类型
key_points:
  type: array
  description: 关键要点
  items: Fact
```

#### 类型完整性规则

**`array` 类型必须通过 `items` 指定元素的语义类型。**

```yaml
# ⚠️ 不推荐：array 没有定义 items
headers:
  type: array
  description: 表头列表

# ✅ 推荐：使用语义类型
key_points:
  type: array
  description: 关键要点列表
  items: Fact

# ⚠️ 禁止：items 使用原始类型
records:
  type: array
  items:
    type: string

# ✅ 推荐：使用语义类型
records:
  type: array
  items: Entity
```

> **例外**：在 Step 的 `output_schema` 中（Tool 返回值映射），允许使用基本类型作为 items。

### 3.7 对象类型

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

**当字段类型为 `object` 时，应尽可能完整定义其子字段结构，直到最末级节点的类型为基本类型（`string`、`number`、`boolean`、`datetime`、`resource`）。只有该技能对于该object类型内部细节暂时无法获知时，才可以将本级作为定义时的最末级**

```yaml
# ⚠️不推荐：object 没有定义子字段
config:
  type: object
  description: 配置项

# ✅ 推荐：object 定义了子字段直到基本类型
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

### 3.8 单选和多选（options）

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

### 3.9 简写语法（已废弃）

> **3.1.0 废弃说明**：简写语法使用原始类型，在 3.1.0 版本中已废弃。请使用完整的语义类型声明。

```yaml
# ⚠️ 已废弃：原始类型简写
query: string
prompt: string

# ✅ 推荐：使用语义类型
query:
  type: Query
  required: true
  description: 用户查询
prompt:
  type: Document
  required: true
  description: 提示内容
```

### 3.10 能力语义系统

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
| `tool` | **不适用** | 工具通过 `output.put(key, value)` 直接写入上下文，输出 key 由工具实现决定 |

语法：
```markdown
**type**: template  
**varName**: report
```

特性：
- **直接访问**：通过 `{{varName}}` 访问输出，无需 `.value` 后缀
- **与 output_schema 配合**：varName 应与 output_schema 中定义的字段名对齐，确保最终输出正确
- **命名规则**：必须符合 `^[a-z][a-z0-9_]*$` 模式，且不能与已有 stepName、其他 varName 或输入字段名冲突

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
input:
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
input:
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

## 5. Step 类型详解

### 5.1 Tool 步骤

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

### 5.2 Prompt 步骤

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

### 5.3 Template 步骤

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

### 5.4 Await 步骤

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

> **注意**：Await 步骤**不需要** `varName`。用户提交输入后，`input_schema` 中定义的每个字段直接以字段名为 key 写入执行上下文。

#### Await 输入的语义类型

`await` 的 `input_schema` 同样遵循语义类型规范，输入字段分为两类：

| 类型 | 说明 | 语义类型选择 |
|------|------|-------------|
| **业务数据补充** | 补充技能所需的业务数据 | 使用对应的业务语义类型 |
| **过程控制确认** | 影响流程走向的临时决策 | 使用 `Decision` |

**示例**：

```yaml
input_schema:
  confirm:
    type: Decision        # 过程控制 - 用户确认
    required: true
    description: 是否确认订单
  notes:
    type: Answer          # 业务数据 - 用户补充的备注
    required: false
    description: 备注信息（可选）
```

---

## 6. 内置函数

Aegis 提供一组内置函数用于处理常见的数据操作。内置函数可以在模板表达式中直接调用，无需通过 step 配置。

### 6.1 函数调用语法

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
# 在 input 中使用函数
input:
  flat_data: array::flatten({{nested_array}}, 1)
  unique_items: array::unique({{items}})
  sorted_list: array::sort({{scores}}, "value", "desc")
```

```template
# 在模板中使用函数
处理后的数据：{{array::flatten({{data}}, 2)}}
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

### 6.2 array::zip

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

### 6.3 array::flatten

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

### 6.4 array::unique

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

### 6.5 array::sort

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

### 6.6 array::count

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

### 6.7 array::filter

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

### 6.8 exists

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

**与 emptiable trait 的区别**：

| 检查方式 | 判断内容 | 适用场景 |
|----------|----------|----------|
| `exists(value)` | 值是否存在（非 null/undefined） | 检查可选字段是否有值 |
| `is_empty` (emptiable trait) | 值是否为空（空字符串、空数组、空对象） | 检查值的内容是否为空 |

### 6.9 内置函数与投影语法的选择

| 需求 | 推荐方式 | 说明 |
|------|----------|------|
| 提取单一字段组成简单数组 | `{{arr[*].field}}` 投影语法 | 简洁高效，适合简单场景 |
| 多个数组组装为对象数组 | `array::zip()` 函数 | 支持任意数量的数组组合 |
| 数组排序、过滤、去重 | 对应的 `array::*` 函数 | 提供完整的数据操作能力 |
| 复杂的数组变换 | 函数链式调用 | 通过函数组合完成复杂操作 |
| 检查值是否存在 | `exists()` 函数 | 判断可选字段是否有值 |

### 6.10 函数命名空间

当前版本提供以下命名空间：

| 命名空间 | 说明 | 包含函数 | 可省略前缀 |
|----------|------|----------|:----------:|
| `public` | 通用基础函数 | `exists` | ✓ |
| `array` | 数组操作 | `zip`、`flatten`、`unique`、`sort`、`count`、`filter` | — |

> **命名空间省略**：只有 `public` 命名空间的函数可以省略 `namespace::` 前缀直接调用。

> 未来版本可能扩展更多命名空间，如 `string`（字符串操作）、`math`（数学运算）等。

---

## 7. 最佳实践

### 7.1 命名规范

- **Step 名称**：使用动词或动词短语，如 `calculate_total`、`prepare_summary`
- **相关输入输出变量名称**：使用与业务含义相匹配的名词或动名词以及短语，如 `file`、`summary`、`result`、`report`
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

1. **使用 log 工具**：输出中间变量值进行调试（仅限 AtomicSkill）
2. **条件日志**：使用 `when` 条件输出调试信息
3. **分步测试**：单独测试每个步骤

---

## 8. 附录

### 8.1 保留关键词

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
- `traits`
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

### 8.2 修订历史

| 版本 | 日期 | 说明                                                                                                                                                                                                              |
| ----- | ---------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1.0.0 | 2026-02-07 | 初始版本，支持四种 Step 类型                                                                                                                                                                                               |
| 1.0.0 | 2026-02-10 | 移除 Transform 类型；统一步骤输出引用为 `.value` 语法；Step 类型精简为三种（tool / prompt / await）                                                                                                                                       |
| 1.0.0 | 2026-02-10 | 新增 Template 类型 Step：纯文本模板渲染，不调用 LLM 也不调用 Tool；Step 类型扩展为四种                                                                                                                                                      |
| 1.0.0 | 2026-02-10 | 新增 varName 属性：为所有 Step 类型添加变量别名，步骤输出以 varName 为键直接存入上下文                                                                                                                                                         |
| 1.0.0 | 2026-02-10 | 移除 TEXT 格式 output_schema：output_schema 改为必需字段，统一使用 YAML 格式定义                                                                                                                                                    |
| 1.0.0 | 2026-02-10 | varName 改为必填：所有 Step 必须指定 varName，移除无 varName 的 StepOutputWrapper 兼容路径                                                                                                                                          |
| 1.0.0 | 2026-02-11 | Tool 输出模型重构：工具通过 `ToolOutputContext.put(key, value)` 直接写上下文；Tool 步骤不再需要 varName；新增 output_schema 声明（纯可读性）；上下文变量限制为基本类型                                                                                          |
| 1.0.0 | 2026-02-11 | 文档重构：移除 `properties` 关键词，array 用 `items` 平铺子字段，object 子字段直接平铺；重组文档结构为「概述 → 输入 → 输出 → Steps → 示例」                                                                                                                |
| 1.0.0 | 2026-02-11 | Await 去掉 varName，input_schema 字段直接写入上下文；Tool YAML 增加 `input:` 包装，output_schema 改为必需；模板仅保留变量替换，移除条件渲染和循环语法；参数类型系统明确为基本类型 + 数组，不支持复杂对象                                                                            |
| 1.0.0 | 2026-02-11 | 模板语法扩展：`{{}}` 支持简单表达式求值（四则运算、字符串拼接）；示例中 `variable.set` 改为 `template` 表达式                                                                                                                                        |
| 1.0.0 | 2026-02-11 | 模板新增数组索引访问（`{{arr[n].field}}`、`{{arr[#var].field}}`）和循环渲染（`{{#for arr}}...{{/for}}`）                                                                                                                            |
| 1.0.0 | 2026-02-11 | input_schema 和上下文新增 `object` 类型（`Map<String, Object>`）；循环渲染新增 `{{_}}` 引用当前元素；array 上下文存储改为 `List`；更新 database.query 示例                                                                                          |
| 1.0.0 | 2026-02-12 | 新增 version 字段：技能文件支持声明版本号，`skillId + version` 构成全局唯一标识；API 层支持按版本查询和执行                                                                                                                                          |
| 1.0.0 | 2026-02-14 | 将意图关键词intent改为能力关键词capabilityTags                                                                                                                                                                               |
| 1.1.0 | 2026-02-21 | 新增 UI 展示语义章节：引入 `## ui` 作为必需顶层章节；定义 display 类型；建立组件分类体系；mixed 采用纯布局 DSL 设计                                                                                                                                      |
| 1.1.0 | 2026-02-23 | 新增审计日志章节：强调审计日志是 Aegis Skill 核心特性；文档化 `log` 工具的使用方法和最佳实践                                                                                                                                                        |
| 1.2.0 | 2026-02-27 | **类型完整性规则**：强制要求 `array` 和 `object` 类型必须完整定义子级结构；**数组字段投影**：新增 `{{arr[*].field}}` 语法；**循环索引**：新增 `{{_index}}` 内置变量；**内置工具**：新增数组操作工具；**统一 when 条件语法**                                                           |
| 2.0.0 | 2026-02-27 | **技能分类体系**：引入 `type` 字段区分技能类型；定义 `AtomicSkill` 和 `PresentationSkill`；**文档拆分**：拆分为 README.md（通用规范）、AtomicSkill.md、PresentationSkill.md 三个文件；**审计日志**：明确仅适用于 AtomicSkill                                          |
| 2.1.0 | 2026-02-28 | **循环控制指令**：新增 `{{#break}}` 语法支持在循环中提前终止；**datetime 类型**：新增时间基本类型，支持多种输入格式；**resource 类型**：新增资源引用类型，支持 `file://` 和 `http[s]://`；**能力语义系统**：新增 `traits`（能力特征）和 `semantic_role`（语义角色）字段属性，支持条件表达式校验、UI 自动生成和技能自动编排 |
| 2.2.0 | 2026-03-01 | **内置函数系统**：将原"内置工具"重构为"内置函数"，使用 `namespace::function()` 语法直接调用（如 `array::flatten()`）；**函数链式调用**：支持函数嵌套调用；**三层异常处理**：技能级 `**ignore**` 字段、步骤级 `ignore` 属性、函数级 `ignore` 参数，分别控制不同粒度的异常处理行为 |
| 2.3.0 | 2026-03-01 | **CognitiveSkill**：新增认知技能类型，支持 `internal_flow` 定义内部控制流，包含 `decision`（条件分支）、`select`（能力匹配）、`assign`（变量赋值）、`merge`（结果合并）四种节点类型；**动态能力匹配**：select 节点通过 capability 标签动态选择 AtomicSkill；**fallback 机制**：最多两层嵌套的异常处理 |
| 3.0.0 | 2026-03-02 | **foreach 节点**：新增循环遍历节点，支持对数组逐项处理并收集结果，实现多实例推理能力（Map → Collect → Reduce）；**timeout 配置**：CognitiveSkill 新增 `timeout` 字段控制执行超时；**内置函数扩展**：新增 `array::count` 统计数组长度、`public::exists` 判断值存在性，支持省略 `public::` 前缀；**执行模型升级**：CognitiveSkill 从"单步决策"升级为"多实例推理"，支持问题分解求解、多文档分析、多来源抓取等复杂认知场景 |
| 3.1.0 | 2026-03-03 | **语义类型系统**：`input_schema` 和 `output_schema` 必须使用语义类型（Query、Document、Evidence、Answer 等），禁止使用原始类型（string、number、boolean 等）；**Type.constraints 语法**：新增约束类型声明语法用于表示对语义类型的约束条件（如 `Query.constraints`）；**模板语法规范**：明确禁止 Mustache/Handlebars 语法（`#each`、`#if`、`@index`、`this`），统一使用 Aegis 模板语法（`#for`、`#when`、`_index`、`_`）；**语义类型体系**：预定义 19 种语义类型分七层组织（输入、知识、规则、推理、流程、度量、输出）；**数组语义化**：`array` 的 `items` 必须使用语义类型 |

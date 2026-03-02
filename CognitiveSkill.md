# CognitiveSkill 规范

> 本文档是 [Aegis Skill DSL 规范](./README.md) 的 CognitiveSkill 专属部分。
>
> **规范版本：3.0.0**

## 目录

- [1. 概述](#1-概述)
- [2. 内部流程 (internal_flow)](#2-内部流程-internal_flow)
- [3. 节点类型详解](#3-节点类型详解)
- [4. foreach 节点](#4-foreach-节点)
- [5. 表达式语法](#5-表达式语法)
- [6. 变量作用域](#6-变量作用域)
- [7. 异常处理](#7-异常处理)
- [8. 完整示例](#8-完整示例)

---

## 1. 概述

**CognitiveSkill**（认知技能）是一种具备内部控制流程的技能类型，用于实现复杂的认知决策逻辑。它是 Agent 的"认知操作单元"，可以根据条件动态选择和编排 AtomicSkill。

### 1.1 核心特点

- **内部控制流**：包含 `internal_flow`，支持条件分支、能力选择、变量赋值等控制逻辑
- **能力匹配**：通过 `capability` 标签动态选择合适的 AtomicSkill 执行
- **微型执行器**：本质是带条件分支和能力匹配能力的微型执行器
- **不可嵌套**：CognitiveSkill 内部不允许调用其他 CognitiveSkill

### 1.2 与其他技能类型的区别

| 特性 | AtomicSkill | CognitiveSkill | PresentationSkill |
|------|-------------|----------------|-------------------|
| 核心职责 | 单步数据处理 | 认知决策与编排 | UI 渲染展示 |
| 执行单元 | steps | internal_flow | steps + ui |
| 控制流 | 顺序执行 | 条件分支 + 能力选择 | 顺序执行 |
| 调用其他技能 | 否 | 是（仅 AtomicSkill） | 否 |
| 输出 | output_schema | output_schema | ui |

### 1.3 执行模型层次

```
ExecutionPlan（执行计划）
   └── PlanStep → SkillRef
                     ├── AtomicSkill（简单场景）
                     └── CognitiveSkill（复杂场景）
                             ├── AtomicSkill1
                             ├── AtomicSkill2
                             └── AtomicSkill3
```

**层次关系**：
- Plan 是高层骨架，负责整体编排
- CognitiveSkill 负责局部认知控制
- AtomicSkill 是最小执行单元

### 1.4 文件结构

| 章节 | 必需 | 说明 |
|------|------|------|
| `# skill: <id>` | 是 | 技能唯一标识符，建议以 `cognitive_` 前缀 |
| `**version**` | 否 | 技能版本号，默认 `1.0.0` |
| `**ignore**` | 否 | 技能级异常处理，默认 `false` |
| `**timeout**` | 否 | 执行超时时间（毫秒），默认由平台配置 |
| `**type**: CognitiveSkill` | 是 | 技能类型，必须显式声明 |
| `## description` | 否 | 技能描述 |
| `## capabilityTags` | 否 | 能力关键词列表 |
| `## input_schema` | 是 | 输入参数结构定义 |
| `## internal_flow` | 是 | 内部执行流程定义 |
| `## output_schema` | 是 | 输出参数结构定义 |

### 1.5 文件骨架

~~~markdown
# skill: cognitive_<name>

**version**: 3.0.0
**timeout**: 30000
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
<内部流程节点列表>
```

## output_schema

```yaml
<输出参数定义>
```
~~~

### 1.6 与 Strategy 的对应关系

CognitiveSkill 与 Agent Strategy 的关系是"多对一"：每种 StrategyType 有一个默认的 CognitiveSkill 类型。

| StrategyType | 默认 CognitiveSkill | 说明 |
|--------------|---------------------|------|
| `SEARCH_AND_SYNTHESIZE` | `cognitive_summarize` | 搜索并综合 |
| `MULTI_SOURCE_COMPARE` | `cognitive_compare` | 多源对比 |
| `DEEP_ANALYSIS` | `cognitive_analysis` | 深度分析 |
| `STRUCTURED_EXTRACTION` | `cognitive_extraction` | 结构化提取 |
| `STEPWISE_REASONING` | `cognitive_reasoning` | 逐步推理 |
| `TEMPLATE_GENERATION` | `cognitive_generation` | 模板生成 |

---

## 2. 内部流程 (internal_flow)

`internal_flow` 是 CognitiveSkill 特有的必填章节，定义内部执行控制逻辑。

### 2.1 基本结构

`internal_flow` 是一个**有序执行节点数组**：

```yaml
internal_flow:
  - <node1>
  - <node2>
  - <node3>
```

### 2.2 执行规则

| 规则 | 说明 |
|------|------|
| 顺序执行 | 节点按数组顺序依次执行 |
| 变量产生 | 每个节点执行后可产生变量 |
| 作用域 | 变量进入当前上下文作用域 |
| 引用传递 | 后续节点可引用之前产生的变量 |

### 2.3 支持的节点类型

CognitiveSkill 支持以下 5 种节点类型：

| 节点类型 | 说明 | 用途 |
|----------|------|------|
| `decision` | 条件决策 | 根据条件执行不同分支 |
| `select` | 能力选择 | 根据能力标签选择并执行 Skill |
| `assign` | 变量赋值 | 将变量从一个名称复制到另一个 |
| `merge` | 变量合并 | 合并多个变量为一个 |
| `foreach` | 循环遍历 | 对数组逐项处理并收集结果 |

---

## 3. 节点类型详解

### 3.1 decision 节点

条件决策节点，根据表达式结果执行不同的分支。

**结构**：

```yaml
- type: decision
  condition: <expression>
  then:
    - <node>
    - <node>
  else:
    - <node>
```

**字段说明**：

| 字段 | 必需 | 类型 | 说明 |
|------|------|------|------|
| `type` | 是 | string | 固定为 `decision` |
| `condition` | 是 | string | 布尔表达式 |
| `then` | 是 | array | 条件为 `true` 时执行的节点列表 |
| `else` | 否 | array | 条件为 `false` 时执行的节点列表 |

**示例**：

```yaml
- type: decision
  condition: count({{results}}) > 20
  then:
    - type: select
      capability: clustering
      input: results
      as: processed
  else:
    - type: assign
      from: results
      to: processed
```

**嵌套 decision**：

`then` 和 `else` 分支内可以继续嵌套 `decision` 节点：

```yaml
- type: decision
  condition: count({{data}}) > 100
  then:
    - type: decision
      condition: avg({{data}}.score) > 0.8
      then:
        - type: select
          capability: advanced_processing
          as: result
      else:
        - type: select
          capability: standard_processing
          as: result
  else:
    - type: select
      capability: simple_processing
      as: result
```

### 3.2 select 节点

能力选择节点，根据能力标签动态选择并执行一个 AtomicSkill。

**结构**：

```yaml
- type: select
  capability: <string>
  strategy: <string>
  input: <variable>
  as: <variable_name>
  fallback:
    - <node>
```

**字段说明**：

| 字段           | 必需  | 类型     | 说明                                     |
| ------------ | --- | ------ | -------------------------------------- |
| `type`       | 是   | string | 固定为 `select`                           |
| `capability` | 是   | string | 能力标签，用于匹配 Skill 的 capabilityTags       |
| `strategy`   | 否   | string | 技能选择策略（如 `highest_accuracy`、`fastest`） |
| `input`      | 否   | string | 传入 Skill 的输入变量名                        |
| `as`         | 是   | string | 输出变量名，Skill 执行结果保存到此变量                 |
| `fallback`   | 否   | array  | 主能力失败时的备选节点列表                          |

**执行语义**：

1. 查询所有 `capabilityTags` 包含 `capability` 的 Skill
2. 根据 `strategy` 选择一个技能（默认策略由平台决定）
3. 将 `input` 指定的变量作为输入执行 Skill
4. 将结果保存到 `as` 指定的变量

**示例**：

```yaml
- type: select
  capability: summarization
  strategy: highest_accuracy
  input: filtered_data
  as: summary
```

#### fallback 机制

当主能力执行失败时，可通过 `fallback` 指定备选执行逻辑：

```yaml
- type: select
  capability: clustering
  as: clustered
  fallback:
    - type: select
      capability: simple_clustering
      as: clustered
```

**fallback 规则**：

| 规则 | 说明 |
|------|------|
| 触发条件 | 主能力执行失败时触发 |
| 成功继续 | fallback 成功后继续执行后续节点 |
| 失败抛异常 | fallback 也失败则抛出异常 |
| 嵌套限制 | fallback 最多两层嵌套 |

**两层 fallback 示例**：

```yaml
- type: select
  capability: advanced_clustering
  as: result
  fallback:
    - type: select
      capability: standard_clustering
      as: result
      fallback:
        - type: select
          capability: basic_clustering
          as: result
```

### 3.3 assign 节点

变量赋值节点，将一个变量的值复制到另一个变量。

**结构**：

```yaml
- type: assign
  from: <variable>
  to: <variable>
```

**字段说明**：

| 字段 | 必需 | 类型 | 说明 |
|------|------|------|------|
| `type` | 是 | string | 固定为 `assign` |
| `from` | 是 | string | 源变量名 |
| `to` | 是 | string | 目标变量名 |

**执行语义**：

```
context[to] = context[from]
```

**示例**：

```yaml
# 将 input.results 赋值给 processed
- type: assign
  from: input.results
  to: processed
```

**典型用途**：

- 在 decision 的 else 分支中保持变量名一致
- 重命名变量以便后续统一引用

### 3.4 merge 节点

变量合并节点，将多个变量合并为一个。这是控制流的收敛节点（Flow Convergence Node）。

**结构**：

```yaml
- type: merge
  sources:
    - <variable1>
    - <variable2>
  as: <variable_name>
```

**字段说明**：

| 字段 | 必需 | 类型 | 说明 |
|------|------|------|------|
| `type` | 是 | string | 固定为 `merge` |
| `sources` | 是 | array | 要合并的变量名列表 |
| `as` | 是 | string | 合并结果的变量名 |

**执行语义**：

```
context[as] = merge(context[sources[0]], context[sources[1]], ...)
```

**合并规则**：

| 源类型 | 合并方式 |
|--------|----------|
| 数组 + 数组 | 拼接为一个数组 |
| 对象 + 对象 | 字段合并（后者覆盖前者同名字段） |
| 其他组合 | 报错 |

**示例**：

```yaml
# 合并两个搜索结果
- type: merge
  sources:
    - web_results
    - local_results
  as: all_results
```

---

## 4. foreach 节点

`foreach` 是 CognitiveSkill 的**结构控制节点**，用于对数组进行逐项遍历处理，是实现"多实例推理"的核心能力。

### 4.1 引入背景

CognitiveSkill 原有的节点类型（decision、select、assign、merge）仅支持单步决策，缺乏**多实例处理能力**。

**典型场景**：

| 场景 | 说明 |
|------|------|
| 问题分解求解 | 将复杂问题分解为子问题，逐个求解 |
| 多文档分析 | 对多个文档逐一进行摘要或分析 |
| 多来源抓取 | 从多个数据源逐一获取数据 |
| 候选方案评估 | 对多个候选方案逐一评估打分 |

这类场景的本质是：**Map → Collect → Reduce**

### 4.2 基本结构

```yaml
- type: foreach
  items: <expression>
  as: <item_name>
  do:
    - <node>
    - <node>
  collect: <variable_name>
```

**字段说明**：

| 字段 | 必需 | 类型 | 说明 |
|------|------|------|------|
| `type` | 是 | string | 固定为 `foreach` |
| `items` | 是 | string | 要遍历的数组表达式 |
| `as` | 是 | string | 当前循环元素的变量名 |
| `do` | 是 | array | 每次迭代执行的子节点列表 |
| `collect` | 是 | string | 聚合结果的变量名 |

### 4.3 执行语义

**执行步骤**：

1. **计算 items 表达式**
   - 必须为数组，否则抛出异常

2. **逐项遍历**
   - 对每个元素：
     - 创建局部循环变量：`context[as] = 当前元素`
     - 执行 `do` 中的节点序列
     - 记录本轮执行产生的结果

3. **收集结果**
   - 最终：`context[collect] = [iteration_result_1, iteration_result_2, ...]`

**执行流程图**：

```
items = [item1, item2, item3]
           │
           ▼
    ┌──────────────┐
    │  iteration 1 │ → context[as] = item1 → do nodes → result1
    └──────────────┘
           │
           ▼
    ┌──────────────┐
    │  iteration 2 │ → context[as] = item2 → do nodes → result2
    └──────────────┘
           │
           ▼
    ┌──────────────┐
    │  iteration 3 │ → context[as] = item3 → do nodes → result3
    └──────────────┘
           │
           ▼
    context[collect] = [result1, result2, result3]
```

### 4.4 结果收集规则

`collect` 指定的变量名决定收集什么：

| 规则 | 说明 |
|------|------|
| 收集目标 | `do` 中最后一个节点通过 `as` 产生的变量 |
| 变量匹配 | `collect` 的值必须与 `do` 内某个节点的 `as` 一致 |
| 自动聚合 | 系统自动将每轮的结果收集为数组 |

**推荐写法**：

```yaml
- type: foreach
  items: sub_questions
  as: sq
  do:
    - type: select
      capability: deep_search
      input: sq.question
      as: sub_result    # 每轮产生 sub_result
  collect: sub_result   # 收集所有 sub_result
```

**语义**：

- 每轮执行产生 `sub_result`
- 系统自动收集所有 `sub_result`
- 最终 `sub_result = [第1轮结果, 第2轮结果, ...]`

### 4.5 作用域规则

`foreach` 引入**局部循环作用域**：

| 规则 | 说明 |
|------|------|
| `as` 变量 | 循环局部变量，每轮独立 |
| `do` 内变量 | 仅在当前轮可见，不跨轮共享 |
| `collect` 变量 | 全局变量，循环结束后可用 |
| 外部变量 | 只读访问，不允许覆盖 |

**作用域模型**：

```
GlobalContext
    │
    ├── input variables (只读)
    │
    └── foreach
          │
          ├── foreachContext (iteration 1) ─┐
          ├── foreachContext (iteration 2) ─┼── 独立上下文副本
          └── foreachContext (iteration N) ─┘
                      │
                      ▼
              Merge to GlobalContext (only collect result)
```

> **设计原则**：每次 iteration 都是新的上下文实例，不共享内部变量，避免变量污染。

### 4.6 do 内允许的节点类型

`foreach` 的 `do` 块内允许以下节点类型：

| 节点类型 | 是否允许 | 说明 |
|----------|:--------:|------|
| `decision` | ✓ | 条件分支 |
| `select` | ✓ | 能力选择 |
| `assign` | ✓ | 变量赋值 |
| `merge` | ✓ | 变量合并 |
| `foreach` | ✗ | **不允许嵌套** |

> **嵌套限制**：当前版本不允许 `foreach` 嵌套 `foreach`，未来版本可能支持单层嵌套。

### 4.7 异常处理

`foreach` 内部异常遵循 CognitiveSkill 的 `ignore` 规则：

**当 `ignore: false`（默认）**：

| 场景 | 行为 |
|------|------|
| 任意一轮失败 | 整个 foreach 失败 |
| foreach 失败 | 整个 Skill 失败 |

**当 `ignore: true`**：

| 场景 | 行为 |
|------|------|
| 单轮失败 | 该轮跳过，继续下一轮 |
| 全部失败 | 整个 Skill 视为失败 |
| 至少一轮成功 | 收集成功轮次的结果 |

**示例**：

```yaml
# 技能级 ignore: true
- type: foreach
  items: documents
  as: doc
  do:
    - type: select
      capability: summarization
      input: doc
      as: summary
  collect: summaries
# 如果某个文档处理失败，跳过该文档，继续处理其他文档
```

### 4.8 空数组处理

当 `items` 为空数组时：

| 场景 | 行为 |
|------|------|
| items = [] | `collect` 生成空数组 `[]` |
| 空数组是否成功 | 取决于 `output_schema` 约束 |

```yaml
# 如果 output_schema 中 summaries 是 required: true
# 空数组 [] 满足类型约束，技能执行成功
```

### 4.9 使用示例

#### 示例 1：子问题逐个搜索

```yaml
- type: foreach
  items: aspects.sub_questions
  as: sq
  do:
    - type: select
      capability: deep_search
      input: sq.question
      as: sub_evidence
  collect: sub_evidence
```

**结果**：

```yaml
sub_evidence:
  - { ...子问题1的搜索结果... }
  - { ...子问题2的搜索结果... }
  - { ...子问题3的搜索结果... }
```

#### 示例 2：多文档摘要

```yaml
- type: foreach
  items: documents
  as: doc
  do:
    - type: select
      capability: summarization
      input: doc
      as: summary_part
  collect: summaries
```

#### 示例 3：带条件判断的循环

```yaml
- type: foreach
  items: candidates
  as: candidate
  do:
    - type: decision
      condition: {{candidate.score}} > 0.5
      then:
        - type: select
          capability: detailed_evaluation
          input: candidate
          as: evaluation
      else:
        - type: assign
          from: candidate
          to: evaluation
  collect: evaluations
```

### 4.10 与 merge 的区别

| 特性 | foreach | merge |
|------|---------|-------|
| 类型 | 结构控制节点 | 数据合并节点 |
| 作用 | 多次执行 | 单次聚合 |
| 输出 | 产生数组 | 合并变量 |
| 阶段 | Map 阶段 | Reduce 阶段 |

**推荐模式**：

```
foreach → merge → reasoning
  │         │         │
  │         │         └── 最终推理
  │         └── 合并多个 foreach 结果
  └── 逐项处理
```

### 4.11 执行模型影响

引入 `foreach` 后，CognitiveSkill 的执行模型升级为：

```
ExecutionPlan
    └── CognitiveSkill
            ├── decision
            ├── foreach ← 新增
            │     ├── select
            │     ├── decision
            │     └── assign
            ├── merge
            └── select
```

> **架构意义**：`foreach` 使 CognitiveSkill 从"单步决策"升级为"多实例推理"，具备了：
> - 多跳推理能力
> - 分解式认知能力
> - 复杂决策能力
> - 结构化分析能力

---

## 5. 表达式语法

### 5.1 变量引用

在 `condition` 表达式中使用 `{{variable}}` 引用变量：

```yaml
condition: count({{results}}) > 10
```

支持点号访问嵌套字段：

```yaml
condition: {{config.threshold}} > 0.5
```

### 5.2 比较运算符

| 运算符 | 说明 | 示例 |
|--------|------|------|
| `>` | 大于 | `count({{arr}}) > 10` |
| `<` | 小于 | `{{score}} < 0.5` |
| `>=` | 大于等于 | `{{count}} >= 100` |
| `<=` | 小于等于 | `{{rate}} <= 0.8` |
| `==` | 等于 | `{{status}} == "active"` |
| `!=` | 不等于 | `{{type}} != "invalid"` |

### 5.3 逻辑运算符

| 运算符 | 说明 | 示例 |
|--------|------|------|
| `and` | 逻辑与 | `count({{arr}}) > 10 and avg({{arr}}.score) > 0.6` |
| `or` | 逻辑或 | `empty({{data}}) or count({{data}}) < 5` |

### 5.4 内置函数

| 函数 | 参数 | 返回类型 | 说明 |
|------|------|----------|------|
| `count(x)` | array | number | 返回数组长度 |
| `avg(x.field)` | array | number | 返回数组中指定字段的平均值 |
| `sum(x.field)` | array | number | 返回数组中指定字段的总和 |
| `max(x.field)` | array | number | 返回数组中指定字段的最大值 |
| `min(x.field)` | array | number | 返回数组中指定字段的最小值 |
| `exists(x)` | any | boolean | 判断变量是否存在且非 null |
| `empty(x)` | any | boolean | 判断变量是否为空（null、空字符串、空数组） |

**示例**：

```yaml
# 数组长度判断
condition: count({{results}}) > 20

# 平均值判断
condition: avg({{scores}}.value) < 0.6

# 存在性判断
condition: exists({{optional_data}})

# 空值判断
condition: empty({{filter_result}})

# 组合判断
condition: count({{items}}) > 10 and avg({{items}}.score) >= 0.7
```

---

## 6. 变量作用域

### 6.1 作用域规则

CognitiveSkill 使用**全局单作用域**：

| 来源 | 进入方式 |
|------|----------|
| `input_schema` 定义的输入 | 自动进入 context，通过 `input.<field>` 或直接 `<field>` 访问 |
| `select` 节点输出 | 通过 `as` 字段指定的变量名进入 context |
| `assign` 节点输出 | 通过 `to` 字段指定的变量名进入 context |
| `merge` 节点输出 | 通过 `as` 字段指定的变量名进入 context |

### 6.2 变量引用方式

**输入变量引用**：

```yaml
# 方式一：带 input 前缀
input: input.results

# 方式二：直接引用（如果无冲突）
input: results
```

**中间变量引用**：

```yaml
# 引用前面节点产生的变量
- type: select
  capability: clustering
  as: clustered

- type: select
  capability: filtering
  input: clustered    # 引用上一步的输出
  as: filtered
```

### 6.3 输出映射

`output_schema` 中定义的字段从 context 中获取：

```yaml
output_schema:
  summary:
    type: string
    description: 最终摘要
```

系统会自动从 context 中查找名为 `summary` 的变量作为输出。

### 6.4 foreach 作用域

`foreach` 节点引入局部作用域（详见 [4.5 作用域规则](#45-作用域规则)）：

| 变量类型 | 作用域 |
|----------|--------|
| `as` 变量 | 循环局部，每轮独立 |
| `do` 内产生的变量 | 循环局部，不跨轮共享 |
| `collect` 变量 | 全局，循环结束后可用 |

---

## 7. 异常处理

### 7.1 技能级 ignore

CognitiveSkill 支持技能级 `ignore` 属性：

```markdown
**ignore**: true
```

### 7.2 成功判定规则

**当 `ignore: false`（默认）时**：

技能成功需要同时满足：
1. 整个执行流程没有未捕获的异常
2. `output_schema` 中所有 `required: true` 的字段都成功生成

**当 `ignore: true` 时**：

技能成功仅需要：
- `output_schema` 中所有 `required: true` 的字段都成功生成

> 即使某些 select 节点失败，只要最终能生成必需的输出字段，技能即视为成功。

### 7.3 foreach 异常处理

当 CognitiveSkill 包含 `foreach` 时，异常处理规则如下：

**当 `ignore: false`**：

| 场景 | 行为 |
|------|------|
| 任意一轮失败 | 整个 foreach 失败，技能失败 |

**当 `ignore: true`**：

| 场景 | 行为 |
|------|------|
| 单轮失败 | 跳过该轮，继续下一轮 |
| 全部失败 | 技能视为失败 |
| 至少一轮成功 | 收集成功轮次结果 |

### 7.4 fallback 与 ignore 的配合

```yaml
# 技能级 ignore: true
# select 有 fallback

- type: select
  capability: advanced_analysis
  as: result
  fallback:
    - type: select
      capability: basic_analysis
      as: result
```

执行顺序：
1. 尝试 `advanced_analysis`
2. 失败则尝试 `basic_analysis`（fallback）
3. fallback 也失败：
   - 若技能 `ignore: true`：检查是否能生成必需输出
   - 若技能 `ignore: false`：抛出异常

### 7.5 成功状态定义（含 foreach）

当 CognitiveSkill 包含 `foreach` 时：

| 条件 | 是否成功 |
|------|:--------:|
| `collect` 变量成功生成 | ✓ |
| `items` 为空数组，`collect` 为 `[]` | 取决于 `output_schema` 约束 |
| `ignore: true` 且全部轮次失败 | ✗ |
| `output_schema` required 字段缺失 | ✗ |

---

## 8. 完整示例

### 8.1 结果优化器（cognitive_result_optimizer）

~~~markdown
# skill: cognitive_result_optimizer

**version**: 3.0.0
**type**: CognitiveSkill

## description

针对执行结果进行优化，最终得到总结文案。根据数据量和质量动态选择处理策略。

## capabilityTags

- 结果优化
- 决策
- 总结
- 数据处理

## input_schema

```yaml
results:
  type: array
  required: true
  description: 待优化的数组结果
  items:
    content:
      type: string
    score:
      type: number
```

## internal_flow

```yaml
# 步骤1：根据数据量决定是否需要聚类
- type: decision
  condition: count({{results}}) > 20
  then:
    - type: select
      capability: clustering
      strategy: highest_accuracy
      input: results
      as: clustered
  else:
    - type: assign
      from: results
      to: clustered

# 步骤2：根据质量评分决定是否需要过滤
- type: decision
  condition: avg({{clustered}}.score) < 0.6
  then:
    - type: select
      capability: filtering
      input: clustered
      as: filtered
  else:
    - type: assign
      from: clustered
      to: filtered

# 步骤3：生成摘要
- type: select
  capability: summarization
  input: filtered
  as: summary
```

## output_schema

```yaml
summary:
  type: string
  required: true
  description: 优化后的结果文本
```
~~~

### 8.2 多源对比器（cognitive_compare）

~~~markdown
# skill: cognitive_compare

**version**: 3.0.0
**type**: CognitiveSkill

## description

对多个来源的数据进行对比分析，生成对比报告。

## capabilityTags

- 多源对比
- 数据分析
- 报告生成

## input_schema

```yaml
sources:
  type: array
  required: true
  description: 多个数据源
  items:
    source_name:
      type: string
    data:
      type: array
      items:
        type: object
```

## internal_flow

```yaml
# 步骤1：判断数据源数量
- type: decision
  condition: count({{sources}}) > 2
  then:
    # 多源：先聚合再对比
    - type: select
      capability: multi_source_aggregation
      input: sources
      as: aggregated
    - type: select
      capability: comparative_analysis
      input: aggregated
      as: comparison
  else:
    # 双源：直接对比
    - type: select
      capability: pairwise_comparison
      input: sources
      as: comparison

# 步骤2：生成对比报告
- type: select
  capability: report_generation
  input: comparison
  as: report
```

## output_schema

```yaml
report:
  type: object
  required: true
  description: 对比分析报告
  similarities:
    type: array
    description: 相似点列表
    items:
      type: string
  differences:
    type: array
    description: 差异点列表
    items:
      type: string
  conclusion:
    type: string
    description: 总结结论
```
~~~

### 8.3 深度分析器（cognitive_analysis）

~~~markdown
# skill: cognitive_analysis

**version**: 3.0.0
**type**: CognitiveSkill

## description

对复杂问题进行深度分析，支持多轮推理和证据收集。

## capabilityTags

- 深度分析
- 推理
- 证据收集

## input_schema

```yaml
question:
  type: string
  required: true
  description: 待分析的问题
context:
  type: string
  required: false
  description: 背景上下文
```

## internal_flow

```yaml
# 步骤1：问题分解
- type: select
  capability: question_decomposition
  input: question
  as: sub_questions

# 步骤2：针对每个子问题收集证据
- type: select
  capability: evidence_collection
  input: sub_questions
  as: evidence
  fallback:
    - type: select
      capability: basic_search
      input: sub_questions
      as: evidence

# 步骤3：判断证据是否充分
- type: decision
  condition: count({{evidence}}) >= 3 and avg({{evidence}}.confidence) > 0.7
  then:
    # 证据充分：直接推理
    - type: select
      capability: reasoning
      input: evidence
      as: analysis
  else:
    # 证据不足：补充搜索后推理
    - type: select
      capability: supplementary_search
      input: evidence
      as: more_evidence
    - type: merge
      sources:
        - evidence
        - more_evidence
      as: all_evidence
    - type: select
      capability: reasoning
      input: all_evidence
      as: analysis
```

## output_schema

```yaml
analysis:
  type: object
  required: true
  description: 分析结果
  conclusion:
    type: string
    description: 结论
  reasoning_chain:
    type: array
    description: 推理链
    items:
      type: string
  confidence:
    type: number
    description: 置信度
```
~~~

### 8.4 带 fallback 的聚类器

~~~markdown
# skill: cognitive_clustering

**version**: 3.0.0
**ignore**: true
**type**: CognitiveSkill

## description

对数据进行智能聚类，支持多级降级策略。

## capabilityTags

- 聚类
- 数据处理
- 降级处理

## input_schema

```yaml
data:
  type: array
  required: true
  description: 待聚类的数据
  items:
    type: object
```

## internal_flow

```yaml
# 尝试高级聚类，失败则降级
- type: select
  capability: advanced_clustering
  input: data
  as: clusters
  fallback:
    - type: select
      capability: standard_clustering
      input: data
      as: clusters
      fallback:
        - type: select
          capability: simple_grouping
          input: data
          as: clusters
```

## output_schema

```yaml
clusters:
  type: array
  required: true
  description: 聚类结果
  items:
    cluster_id:
      type: number
    members:
      type: array
      items:
        type: object
```
~~~

### 8.5 多文档分析器（cognitive_multi_doc_analyzer）— 使用 foreach

~~~markdown
# skill: cognitive_multi_doc_analyzer

**version**: 3.0.0
**timeout**: 60000
**ignore**: true
**type**: CognitiveSkill

## description

对多个文档进行逐一分析，收集各文档的摘要和关键信息，最终生成综合报告。展示 foreach 节点的典型用法。

## capabilityTags

- 多文档分析
- 批量处理
- 摘要生成
- 综合报告

## input_schema

```yaml
documents:
  type: array
  required: true
  description: 待分析的文档列表
  items:
    title:
      type: string
      description: 文档标题
    content:
      type: string
      description: 文档内容
    source:
      type: string
      description: 文档来源
```

## internal_flow

```yaml
# 步骤1：逐个文档进行摘要
- type: foreach
  items: documents
  as: doc
  do:
    - type: select
      capability: document_summarization
      input: doc
      as: doc_summary
  collect: summaries

# 步骤2：合并所有摘要
- type: merge
  sources:
    - summaries
  as: all_summaries

# 步骤3：根据文档数量决定综合策略
- type: decision
  condition: count({{all_summaries}}) > 5
  then:
    # 文档较多：先聚类再综合
    - type: select
      capability: topic_clustering
      input: all_summaries
      as: clustered_summaries
    - type: select
      capability: comprehensive_synthesis
      input: clustered_summaries
      as: final_report
  else:
    # 文档较少：直接综合
    - type: select
      capability: direct_synthesis
      input: all_summaries
      as: final_report
```

## output_schema

```yaml
summaries:
  type: array
  required: true
  description: 各文档的摘要列表
  items:
    title:
      type: string
      description: 文档标题
    summary:
      type: string
      description: 文档摘要
    key_points:
      type: array
      description: 关键要点
      items:
        type: string
final_report:
  type: object
  required: true
  description: 综合分析报告
  overview:
    type: string
    description: 总体概述
  themes:
    type: array
    description: 主题列表
    items:
      type: string
  conclusion:
    type: string
    description: 结论
```
~~~

### 8.6 问题分解求解器（cognitive_decompose_solver）— foreach 高级用法

~~~markdown
# skill: cognitive_decompose_solver

**version**: 3.0.0
**timeout**: 120000
**type**: CognitiveSkill

## description

将复杂问题分解为子问题，逐个求解后合并答案。展示 foreach 与 decision 的组合使用。

## capabilityTags

- 问题分解
- 逐步推理
- 答案合成

## input_schema

```yaml
question:
  type: string
  required: true
  description: 待解答的复杂问题
context:
  type: string
  required: false
  description: 问题背景上下文
```

## internal_flow

```yaml
# 步骤1：问题分解
- type: select
  capability: question_decomposition
  input: question
  as: aspects

# 步骤2：逐个子问题搜索证据
- type: foreach
  items: aspects.sub_questions
  as: sq
  do:
    - type: decision
      condition: {{sq.complexity}} > 0.7
      then:
        # 复杂子问题：使用深度搜索
        - type: select
          capability: deep_search
          input: sq.question
          as: sub_evidence
      else:
        # 简单子问题：使用快速搜索
        - type: select
          capability: quick_search
          input: sq.question
          as: sub_evidence
  collect: all_evidence

# 步骤3：合并所有证据
- type: merge
  sources:
    - all_evidence
  as: merged_evidence

# 步骤4：综合推理
- type: select
  capability: synthesis_reasoning
  input: merged_evidence
  as: final_answer
```

## output_schema

```yaml
aspects:
  type: object
  required: true
  description: 问题分解结果
  sub_questions:
    type: array
    description: 子问题列表
    items:
      question:
        type: string
      complexity:
        type: number
all_evidence:
  type: array
  required: true
  description: 各子问题的证据
  items:
    type: object
final_answer:
  type: object
  required: true
  description: 最终答案
  answer:
    type: string
    description: 综合答案
  reasoning:
    type: string
    description: 推理过程
  confidence:
    type: number
    description: 置信度
```
~~~

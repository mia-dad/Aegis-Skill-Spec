# CognitiveSkill 规范

> 本文档是 [Aegis Skill DSL 规范](./README.md) 的 CognitiveSkill 专属部分。
>
> **规范版本：3.4.0**

## 目录

- [1. 概述](#1-概述)
- [2. 内部流程 (internal_flow)](#2-内部流程-internal_flow)
- [3. 节点类型详解](#3-节点类型详解)
- [4. 表达式语法](#4-表达式语法)
- [5. 变量作用域](#5-变量作用域)
- [6. 运行模式](#6-cognitiveskill-的运行模式)
- [7. output_schema 校验策略](#7-output_schema-校验策略)
- [8. 异常与成功处理](#8-异常与成功处理)
- [9. 完整示例](#9-完整示例)

---

## 1. 概述

**CognitiveSkill**（认知技能）是一种具备内部控制流程的技能类型，用于实现复杂的认知决策逻辑。
它是 Agent 的"认知操作单元"，可以根据条件动态选择和编排 AtomicSkill。
是系统Agent 模式的实现载体（CoT、ReAct、ToT 等）

### 1.1 核心特点

- **内部控制流**：包含 `internal_flow`，支持条件分支、能力选择、变量赋值等控制逻辑
- **能力匹配**：通过 `capability` 标签动态选择合适的 AtomicSkill 执行
- **微型执行器**：本质是带条件分支和能力匹配能力的微型执行器
- **不可嵌套**：CognitiveSkill 内部不允许调用其他 CognitiveSkill

### 1.2 与其他技能类型的区别

| 特性     | AtomicSkill   | CognitiveSkill   |
| ------ | ------------- | ---------------- |
| 核心职责   | 单步数据处理        | 认知决策与编排          |
| 执行单元   | steps         | internal_flow    |
| 控制流    | 顺序执行          | 条件分支 + 能力选择      |
| 调用其他技能 | 否             | 是（仅 AtomicSkill） |
| 输出     | output_schema | output_schema    |
| UI渲染   | 有ui           | 无ui              |

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
| `**mode**` | 是 | Agent 运行模式（Direct/CoT/ReAct/Decompose/Retrieve/Compare/Generate） |
| `## description` | 否 | 技能描述 |
| `## capabilityTags` | 否 | 能力关键词列表 |
| `## input_schema` | 是 | 输入参数结构定义 |
| `## internal_flow` | 是 | 内部执行流程定义 |
| `## output_schema` | 是 | 输出参数结构定义 |

### 1.5 文件骨架

~~~markdown
# skill: cognitive_reasoning<name>

**version**: 3.1.0
**timeout**: 30000
**type**: CognitiveSkill
**mode**: CoT                         # 声明模式


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

| StrategyType        | 默认 CognitiveSkill      | 说明    |
| ------------------- | ---------------------- | ----- |
| `SEARCH_SYNTHESIZE` | `cognitive_summarize`  | 搜索并综合 |
| `COMPARE`           | `cognitive_compare`    | 多源对比  |
| `ANALYSIS`          | `cognitive_analysis`   | 深度分析  |
| `EXTRACTION`        | `cognitive_extraction` | 结构化提取 |
| `GENERATION`        | `cognitive_generation` | 模板生成  |

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

CognitiveSkill 支持以下 6 种节点类型：

| 节点类型       | 说明   | 用途                |
| ---------- | ---- | ----------------- |
| `decision` | 条件决策 | 根据条件执行不同分支        |
| `select`   | 能力选择 | 根据能力标签选择并执行 Skill |
| `assign`   | 变量赋值 | 将变量从一个名称复制到另一个    |
| `merge`    | 变量合并 | 合并多个变量为一个         |
| `foreach`  | 循环遍历 | 对数组逐项处理并收集结果      |
| `loop`     | 条件循环 | 支持 ReAct 模式的迭代执行  |

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
      input:
        Query: results
      output:
        Answer: processed
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
          input:
            Query: data
          output:
            Answer: result
      else:
        - type: select
          capability: standard_processing
          input:
            Query: data
          output:
            Answer: result
  else:
    - type: select
      capability: simple_processing
      input:
        Query: data
      output:
        Answer: result
```

### 3.2 select 节点

能力选择节点，根据能力标签动态选择并执行一个 AtomicSkill。

**结构**：

```yaml
- type: select
  capability: <string>
  strategy: <string>
  input:
    <角色语义>: <变量名>              # 角色 → 变量
  output:
    <角色语义>: <变量名>              # 角色 → 变量
  fallback:
    - <node>
```

示例
~~~markdown
  ## internal_flow

  ```yaml
  # 节点内，角色名自动带上模式前缀
  - type: select
    capability: reasoning
    input:
      Query: question                   # = CoT.Query → question 变量
    output:
      Conclusion: conclusion            # = CoT.Conclusion → conclusion 变量
      Confidence: confidence            # = CoT.Confidence → confidence 变量
  - type: assign
    from: conclusion
    to: analysis
~~~
**字段说明**：

| 字段           | 必需  | 类型     | 说明                                     |
| ------------ | --- | ------ | -------------------------------------- |
| `type`       | 是   | string | 固定为 `select`                           |
| `capability` | 是   | string | 能力标签，用于匹配 Skill 的 capabilityTags       |
| `input`      | 是   | object | 语义角色:变量名，成对出现                          |
| `output`     | 是   | object | 语义角色:变量名，成对出现，Skill 执行结果保存到对应变量 |
| `fallback`   | 否   | array  | 主能力失败时的备选节点列表                          |

**执行语义**：

1. 查询所有 `capabilityTags` 包含 `capability` 的 Skill
2. 根据 `strategy` 选择一个技能（默认策略由平台决定）
3. 将 `input` 指定的变量作为输入执行 Skill
4. 将结果保存到 `output` 指定的变量

**角色匹配机制**：

`input` 和 `output` 中的角色名会根据 CognitiveSkill 的 `**mode**` 自动补全模式前缀：

| CognitiveSkill mode | select 中写 | 实际匹配 |
|---------------------|-------------|----------|
| CoT | `Query` | `CoT.Query` |
| ReAct | `Task` | `ReAct.Task` |
| Retrieve | `Query` | `Retrieve.Query` |

系统根据角色（而非字段名）匹配 AtomicSkill 的输入输出字段：

```
select.input.Query: question
        ↓
匹配 AtomicSkill.input_schema 中 roles 包含 CoT.Query 的字段
        ↓
找到字段 prompt (roles: [CoT.Query, Direct.Query])
        ↓
将 question 的值传给 prompt
```

> **关键点**：字段名可以不同（如 `question` vs `prompt`），只要语义角色匹配即可。这使得 AtomicSkill 可以被多种模式的 CognitiveSkill 复用。

**fallback 机制**

当主能力执行失败时，可通过 `fallback` 指定备选执行逻辑：

```yaml
- type: select
  capability: clustering
  input:
    Query: question                   # = CoT.Query → question 变量
  output:
    Conclusion: conclusion            # = CoT.Conclusion → conclusion 变量     
  fallback:
    - type: select
      capability: simple_clustering
      input:
        Query: question
      output:
        Conclusion: conclusion           
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
  input:
    Query: question                   # = CoT.Query → question 变量
  output:
    Conclusion: result            # = CoT.Conclusion → result 变量
  fallback:
    - type: select
      capability: standard_clustering
      input:
        Query: question
      output:
        Conclusion: result
      fallback:
        - type: select
          capability: basic_clustering
          input:
            Query: question
          output:
            Conclusion: result
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

| 字段     | 必需  | 类型     | 说明           |
| ------ | --- | ------ | ------------ |
| `type` | 是   | string | 固定为 `assign` |
| `from` | 是   | string | 源变量名         |
| `to`   | 是   | string | 目标变量名        |

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

### 3.5. foreach 节点

`foreach` 是 CognitiveSkill 的**结构控制节点**，用于对数组进行逐项遍历处理，是实现"多实例推理"的核心能力。

#### 3.5.1 引入背景

CognitiveSkill 原有的节点类型（decision、select、assign、merge）仅支持单步决策，缺乏**多实例处理能力**。

**典型场景**：

| 场景 | 说明 |
|------|------|
| 问题分解求解 | 将复杂问题分解为子问题，逐个求解 |
| 多文档分析 | 对多个文档逐一进行摘要或分析 |
| 多来源抓取 | 从多个数据源逐一获取数据 |
| 候选方案评估 | 对多个候选方案逐一评估打分 |

这类场景的本质是：**Map → Collect → Reduce**

#### 3.5.2 基本结构

```yaml
- type: foreach
  items: <expression>
  as: <item_name>
  do:
    - <node>
    - <node>
  collect: <variable_name>
  accumulated_context: <variable_name>    # 可选，累积上下文
```

**字段说明**：

| 字段 | 必需 | 类型 | 说明 |
|------|------|------|------|
| `type` | 是 | string | 固定为 `foreach` |
| `items` | 是 | string | 要遍历的数组表达式 |
| `as` | 是 | string | 当前循环元素的变量名 |
| `do` | 是 | array | 每次迭代执行的子节点列表 |
| `collect` | 是 | string | 聚合结果的变量名 |
| `accumulated_context` | 否 | string | 累积上下文变量名，每轮结果累积传递给下一轮 |

#### 3.5.3 执行语义

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

#### 3.5.4 结果收集规则

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
      input:
	    Query:  sq.question                   # = CoT.Query → question 变量
	  output:
        Conclusion: sub_result  
  collect: sub_result   # 收集所有 sub_result
```

**语义**：

- 每轮执行产生 `sub_result`
- 系统自动收集所有 `sub_result`
- 最终 `sub_result = [第1轮结果, 第2轮结果, ...]`

#### 3.5.5 作用域规则

`foreach` 引入**局部循环作用域**，同时支持**累积上下文**机制：

| 规则 | 说明 |
|------|------|
| `as` 变量 | 循环局部变量，每轮独立 |
| `do` 内变量 | 仅在当前轮可见，不跨轮共享 |
| `collect` 变量 | 全局变量，循环结束后可用 |
| `accumulated_context` 变量 | **累积传递**，每轮结果追加后传给下一轮 |
| 外部变量 | 只读访问，不允许覆盖 |

**作用域模型**：

```
GlobalContext
    │
    ├── input variables (只读)
    │
    └── foreach
          │
          ├── foreachContext (iteration 1) ─── accumulated = []
          │         ↓ 结果追加
          ├── foreachContext (iteration 2) ─── accumulated = [result1]
          │         ↓ 结果追加
          └── foreachContext (iteration N) ─── accumulated = [result1, ..., resultN-1]
                      │
                      ▼
              Merge to GlobalContext (collect = [result1, ..., resultN])
```

**累积上下文 vs 普通收集**：

| 特性 | `collect` | `accumulated_context` |
|------|-----------|----------------------|
| 可见时机 | 循环结束后 | 每轮迭代中可访问之前所有轮次的结果 |
| 用途 | 收集所有结果 | 让后续轮次能参考前面的结果 |
| 典型场景 | 并行独立处理 | 依赖链式推理（如子问题求解需参考已解决的子问题） |

**累积上下文示例**：

```yaml
- type: foreach
  items: sub_questions
  as: sq
  accumulated_context: prior_findings    # 累积上下文变量名
  collect: all_answers
  do:
    - type: select
      capability: sub_question_solving
      input:
        SubQuestion: sq.question
        PriorFindings: prior_findings    # 引用之前轮次的累积结果
      output:
        SubAnswer: answer
```

在此示例中：
- 第 1 轮：`prior_findings = []`
- 第 2 轮：`prior_findings = [第1轮的answer]`
- 第 N 轮：`prior_findings = [第1轮answer, ..., 第N-1轮answer]`

> **设计原则**：每次 iteration 的局部变量相互隔离，但通过 `accumulated_context` 可以实现受控的跨轮次信息传递。

#### 3.5.6 do 内允许的节点类型

`foreach` 的 `do` 块内允许以下节点类型：

| 节点类型 | 是否允许 | 说明 |
|----------|:--------:|------|
| `decision` | ✓ | 条件分支 |
| `select` | ✓ | 能力选择 |
| `assign` | ✓ | 变量赋值 |
| `merge` | ✓ | 变量合并 |
| `foreach` | ✗ | **不允许嵌套** |

> **嵌套限制**：当前版本不允许 `foreach` 嵌套 `foreach`，未来版本可能支持单层嵌套。

#### 3.5.7 异常处理

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
      input:
	    Query:  doc                   # = CoT.Query → question 变量
	  output:
        Conclusion: summary        
  collect: summaries
# 如果某个文档处理失败，跳过该文档，继续处理其他文档
```

#### 3.5.8 空数组处理

当 `items` 为空数组时：

| 场景 | 行为 |
|------|------|
| items = [] | `collect` 生成空数组 `[]` |
| 空数组是否成功 | 取决于 `output_schema` 约束 |

```yaml
# 如果 output_schema 中 summaries 是 required: true
# 空数组 [] 满足类型约束，技能执行成功
```

#### 3.5.9 使用示例

##### 示例 1：子问题逐个搜索

```yaml
- type: foreach
  items: aspects.sub_questions
  as: sq
  do:
    - type: select
      capability: deep_search
      input:
	     ActionInput: sq.question                  
	  output:
         Conclusion: sub_evidence            # = CoT.Conclusion → conclusion 变量
  collect: sub_evidence
```

**结果**：

```yaml
sub_evidence:
  - { ...子问题1的搜索结果... }
  - { ...子问题2的搜索结果... }
  - { ...子问题3的搜索结果... }
```

##### 示例 2：多文档摘要

```yaml
- type: foreach
  items: documents
  as: doc
  do:
    - type: select
      capability: summarization
      input:
        Query: doc
      output:
        Answer: summary_part
  collect: summaries
```

##### 示例 3：带条件判断的循环

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
          input:
            Query: candidate
          output:
            Answer: evaluation
      else:
        - type: assign
          from: candidate
          to: evaluation
  collect: evaluations
```

#### 3.5.10 与 merge 的区别

| 特性  | foreach | merge     |
| --- | ------- | --------- |
| 类型  | 结构控制节点  | 数据合并节点    |
| 作用  | 多次执行    | 单次聚合      |
| 输出  | 产生数组    | 合并变量      |
| 阶段  | Map 阶段  | Reduce 阶段 |

**推荐模式**：

```
foreach → merge → reasoning
  │         │         │
  │         │         └── 最终推理
  │         └── 合并多个 foreach 结果
  └── 逐项处理
```

#### 3.5.11 执行模型影响

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

### 3.6 loop 节点

`loop` 是 CognitiveSkill 的**结构控制节点**，主要用于 ReAct 模式中的条件循环控制。

#### 3.6.1 引入背景

为解决 ReAct 模式中"思考→行动→观察"的迭代循环，引入 `loop` 节点，支持：
- 条件退出（达成目标时停止）
- 最大迭代限制（防止无限循环）
- 跨轮次上下文共享（observation 传递给下一轮 thinking）

#### 3.6.2 基本结构

```yaml
- type: loop
  max_iterations: <number>              # 最大迭代次数，防止无限循环
  exit_condition: "<expression>"        # 退出条件表达式
  do:
    - <node>
    - <node>
  final: <expression>                   # 循环结束后提取的最终结果
  as: <variable_name>                   # 最终结果的变量名
```

**字段说明**：

| 字段 | 必需 | 类型 | 说明 |
|------|------|------|------|
| `type` | 是 | string | 固定为 `loop` |
| `max_iterations` | 是 | number | 最大迭代次数，超过则强制退出 |
| `exit_condition` | 是 | string | 退出条件表达式，为 true 时退出循环 |
| `do` | 是 | array | 每次迭代执行的子节点列表 |
| `final` | 是 | string | 循环结束后提取最终结果的表达式 |
| `as` | 是 | string | 最终结果存入的变量名 |

#### 3.6.3 作用域规则

`loop` 的作用域规则与 `foreach` 不同：**loop 内的变量在各轮次之间共享**。

| 规则 | 说明 |
|------|------|
| `do` 内变量 | **跨轮次共享**，下一轮可访问上一轮产生的变量 |
| 外部变量 | 可读可写（与 foreach 不同） |
| `as` 变量 | 全局变量，循环结束后可用 |

**作用域模型**：

```
GlobalContext
    │
    └── loop (共享上下文)
          │
          ├── iteration 1: thought, action, observation
          │         ↓ 变量保留
          ├── iteration 2: 可访问上轮 observation → 产生新 thought, action, observation
          │         ↓ 变量保留
          └── iteration N: 可访问所有历史变量
                      │
                      ▼
              提取 final 表达式 → 存入 as 变量 → 写入 GlobalContext
```

**与 foreach 的区别**：

| 特性 | foreach | loop |
|------|---------|------|
| 迭代方式 | 遍历数组元素 | 条件循环 |
| 轮次间变量 | **隔离**（需 accumulated_context 显式共享） | **共享**（自动继承上轮变量） |
| 退出条件 | 数组遍历完毕 | exit_condition 为 true 或达到 max_iterations |
| 典型用途 | 批量处理独立项 | ReAct 迭代推理 |

#### 3.6.4 完整示例（ReAct 模式）

```yaml
- type: loop
  max_iterations: 10
  exit_condition: "{{should_stop}}"
  do:
    # 思考：分析当前状态，决定下一步行动
    - type: select
      capability: thinking
      input:
        Task: task
        Observation: observation        # 引用上一轮的观察结果
      output:
        Thought: thought
        Action: action
        ActionInput: action_input

    # 行动：执行选定的工具
    - type: select
      capability: action_execution
      input:
        Action: action
        ActionInput: action_input
      output:
        Observation: observation        # 产生新的观察，供下一轮使用

    # 判断是否完成
    - type: select
      capability: completion_check
      input:
        Task: task
        Observation: observation
      output:
        Answer: should_stop             # 控制退出条件
  final: thought.final_answer
  as: answer
```

**执行流程**：
1. 第 1 轮：`observation` 为 `null`（未定义变量默认为 null），thinking 基于 task 产生初始 thought/action
2. 第 2-N 轮：`observation` 为上一轮的结果，thinking 可参考历史观察
3. 当 `should_stop == true` 或达到 `max_iterations` 时退出
4. 提取 `thought.final_answer` 存入 `answer`

> **未定义变量处理**：loop 第一轮引用的未定义变量（如 `observation`）值为 `null`。被调用的 AtomicSkill 应能处理 null 输入，或在 input_schema 中将该字段设为 `required: false`。

> **final 表达式说明**：`final: thought.final_answer` 假设 thinking 能力返回的 `thought` 对象包含 `final_answer` 字段。实际开发中，需确保被调用的 AtomicSkill 的 output_schema 定义了相应的结构。


### 3.7 完整节点列表

  | 节点         |  节点中文说明   |是否涉及技能串联         | 是否需要角色 |
  |-----------|--------|------|------|
  | decision  |  条件分支   |  ❌ 纯控制流           | ❌ 不需要  |
  | select    |  能力选择  | ✅ 调用 AtomicSkill | ✅ 需要   |
  | assign    | 变量赋值           |❌ 纯变量操作          | ❌ 不需要  |
  | merge     |变量合并           |❌ 纯变量操作          | ❌ 不需要  |
  | foreach   | 数组遍历           |❌ 纯控制流           | ❌ 不需要  |
  | loop      | 条件循环（支持 ReAct） |❌ 纯控制流           | ❌ 不需要  |
---



## 4. 表达式语法

### 4.1 变量引用

在 `condition` 表达式中使用 `{{variable}}` 引用变量：

```yaml
condition: count({{results}}) > 10
```

支持点号访问嵌套字段：

```yaml
condition: {{config.threshold}} > 0.5
```

### 4.2 比较运算符

| 运算符  | 说明   | 示例                       |
| ---- | ---- | ------------------------ |
| `>`  | 大于   | `count({{arr}}) > 10`    |
| `<`  | 小于   | `{{score}} < 0.5`        |
| `>=` | 大于等于 | `{{count}} >= 100`       |
| `<=` | 小于等于 | `{{rate}} <= 0.8`        |
| `==` | 等于   | `{{status}} == "active"` |
| `!=` | 不等于  | `{{type}} != "invalid"`  |

### 4.3 逻辑运算符

| 运算符 | 说明 | 示例 |
|--------|------|------|
| `and` | 逻辑与 | `count({{arr}}) > 10 and avg({{arr}}.score) > 0.6` |
| `or` | 逻辑或 | `empty({{data}}) or count({{data}}) < 5` |

### 4.4 内置函数

技能能够使用dsl平台的内置函数，使用方法与总章说明一致。

| 函数                    | 参数    | 返回类型    | 说明                      |
| --------------------- | ----- | ------- | ----------------------- |
| `array::count(x)`     | array | number  | 返回数组长度                  |
| `array::avg(x.field)` | array | number  | 返回数组中指定字段的平均值           |
| `array::sum(x.field)` | array | number  | 返回数组中指定字段的总和            |
| `array::max(x.field)` | array | number  | 返回数组中指定字段的最大值           |
| `array::min(x.field)` | array | number  | 返回数组中指定字段的最小值           |
| `exists(x)`           | any   | boolean | 判断变量是否存在且非 null         |
| `empty(x)`            | any   | boolean | 判断变量是否为空（null、空字符串、空数组） |

**示例**：

```yaml
# 数组长度判断
condition: array::count({{results}}) > 20

# 平均值判断
condition: array::avg({{scores}}.value) < 0.6

# 存在性判断
condition: exists({{optional_data}})

# 空值判断
condition: empty({{filter_result}})

# 组合判断
condition: array::count({{items}}) > 10 and array::avg({{items}}.score) >= 0.7
```

---

## 5. 变量作用域

### 5.1 作用域规则

CognitiveSkill 使用**全局单作用域**：

| 来源                   | 进入方式                                             |
| -------------------- | ------------------------------------------------ |
| `input_schema` 定义的输入 | 自动进入 context，通过 `input.<field>` 或直接 `<field>` 访问 |
| `select` 节点输出        | 通过 `as` 字段指定的变量名进入 context                       |
| `assign` 节点输出        | 通过 `to` 字段指定的变量名进入 context                       |
| `merge` 节点输出         | 通过 `as` 字段指定的变量名进入 context                       |

### 5.2 变量引用方式

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
  input:
    Query: data
  output:
    Answer: clustered

- type: select
  capability: filtering
  input:
    Query: clustered    # 引用上一步的输出
  output:
    Answer: filtered
```

### 5.3 输出映射

`output_schema` 中定义的字段从 context 中获取：

```yaml
output_schema:
  summary:
    type: Answer
    description: 最终摘要
```

系统会自动从 context 中查找名为 `summary` 的变量作为输出。

### 5.4 foreach 作用域

`foreach` 节点引入局部作用域（详见 [3.5.5 作用域规则](#355-作用域规则)）：

| 变量类型 | 作用域 |
|----------|--------|
| `as` 变量 | 循环局部，每轮独立 |
| `do` 内产生的变量 | 循环局部，不跨轮共享 |
| `collect` 变量 | 全局，循环结束后可用 |

---



## 6. CognitiveSkill 的运行模式

每个具体的 CognitiveSkill 都对应着一种 Agent 运行模式。

> **语义角色定义**：关于 Agent 运行模式和语义角色的完整定义，请参见 [README.md 第 2.3 节](./README.md#23-语义角色-semantic-role)。

### 6.1 模式与 CognitiveSkill 的关系

CognitiveSkill 通过 `**mode**` 字段声明其对应的 Agent 运行模式：

```markdown
**mode**: CoT
```

声明模式后，该技能的 `select` 节点中的角色名会自动带上模式前缀。例如在 CoT 模式下，`Query` 等价于 `CoT.Query`。

---
  


## 7. output_schema 校验策略

### 7.1 与 AtomicSkill 的差异

CognitiveSkill 的 `internal_flow` 支持 `decision` 节点进行条件分支，不同分支可能通过 `select` 调用不同的 AtomicSkill，产生不同语义类型的输出。这导致 `output_schema` 中某些字段的语义类型在**声明期无法确定**。

**典型场景**：

```yaml
- type: decision
  condition: {{data_type}} == "document"
  then:
    - type: select
      capability: document_analysis
      input:
        Query: data
      output:
        Analysis: result           # 产出 Analysis 角色
  else:
    - type: select
      capability: data_extraction
      input:
        Query: data
      output:
        Answer: result             # 产出 Answer 角色
```

上例中，`result` 字段的语义角色取决于运行时条件，无法在声明期确定。

### 7.2 校验规则差异

因此，CognitiveSkill 与 AtomicSkill 在 `output_schema` 校验上采用不同策略：

| 校验阶段 | AtomicSkill | CognitiveSkill |
| ---- | ----------- | -------------- |
| 声明期  | 严格校验语义类型一致性 | **仅校验语法正确性**   |
| 运行时  | -           | 校验实际输出与声明的兼容性  |

### 7.3 声明期校验

CognitiveSkill 在声明期（解析时）仅执行以下校验：

| 校验项 | 说明 |
|--------|------|
| `output_schema` 存在性 | 必须声明 `output_schema` |
| YAML 语法正确性 | 确保 schema 可正确解析 |
| 字段结构完整性 | 遵循类型完整性规则（array 需要 items 等） |

**不执行**的校验：

| 不校验项 | 原因 |
|----------|------|
| internal_flow 各分支输出与 output_schema 的语义类型一致性 | 分支输出类型在声明期无法确定 |
| select 节点输出类型与 output_schema 声明类型的匹配 | 依赖运行时语义匹配 |

### 7.4 运行时校验

CognitiveSkill 执行完成后，进行以下运行时校验：

| 校验项 | 说明 | 失败处理 |
|--------|------|----------|
| required 字段存在 | 检查 `required: true` 的字段是否存在 | 抛出运行时异常 |
| 字段值非 null | 必需字段的值不能为 null | 抛出运行时异常 |
| 类型基本兼容 | 输出值的基础类型与声明一致（array、object、string 等） | 抛出运行时异常 |

### 7.5 语义匹配机制

CognitiveSkill 的输出字段在被下游消费时，通过**运行时语义匹配**而非声明期类型检查来确保正确性：

```
CognitiveSkill 输出 → SemanticInputMapper → 下游 Skill 输入
                            ↓
                    多策略字段匹配：
                                        1. 语义类型匹配
                    1. 相似度匹配
```

**核心原则**：

> `output_schema` 的作用是**契约提示**而非**强类型约束**。
>
> - 告诉调用方"可能"的输出结构
> - 声明 `required` 约束确保必需字段存在
> - 实际类型校验由运行时语义匹配机制完成

### 7.6 output_schema 与 internal_flow 的映射规则

CognitiveSkill 执行完成后，系统从 ExecutionContext 中**按 output_schema 字段名**提取输出值：

```
internal_flow 执行完成
        ↓
遍历 output_schema 中声明的每个字段
        ↓
从 ExecutionContext 中按字段名提取值
        ↓
校验 required / 类型
        ↓
组装为最终输出 Map
```

**关键规则**：

| 规则 | 说明 |
|------|------|
| 字段名匹配 | output_schema 中的字段名必须与 context 中的变量名一致 |
| 显式赋值 | 需要通过 `as` 或 `assign` 将值存入正确的变量名 |
| 无自动映射 | 系统不会自动推断字段映射关系 |

**错误示例**：

```yaml
# internal_flow 最后一步
- type: select
  capability: reasoning
  input:
    Query: question
  output:
    Conclusion: reasoning_result    # 存入 context["reasoning_result"]

# output_schema
analysis:                  # 期望从 context["analysis"] 提取
  type: string
  required: true
```

上例中，`reasoning_result` 与 `analysis` 名称不匹配，导致 `analysis` 字段为 null，校验失败。

**正确做法**：使用 `assign` 节点进行字段映射。

### 7.7 使用 assign 提取输出字段

当被调用的 AtomicSkill 输出结构与 CognitiveSkill 的 output_schema 不同时，必须使用 `assign` 节点进行字段提取和映射。

**典型场景**：

```yaml
# reasoning skill 输出结构
{
  "conclusion": "...",
  "reasoning_chain": [...],
  "confidence": 0.85
}

# cognitive_analysis 的 output_schema
analysis:
  type: string
  required: true
confidence:
  type: Score
  required: false
```

**解决方案**：

```yaml
# 步骤 N：调用 reasoning，输出存入临时变量
- type: select
  capability: reasoning
  input:
    Query: question
    Evidence: evidence
  output:
    Conclusion: reasoning_result    # 临时变量，不是最终输出

# 步骤 N+1：提取 output_schema 所需字段
- type: assign
  from: reasoning_result.conclusion
  to: analysis           # 映射到 output_schema 中的 analysis

- type: assign
  from: reasoning_result.confidence
  to: confidence         # 映射到 output_schema 中的 confidence
```

**最佳实践**：

| 实践 | 说明 |
|------|------|
| 临时变量命名 | 使用 `xxx_result` 后缀标识中间结果 |
| 显式提取 | 为每个 output_schema 字段添加对应的 assign |
| 支持嵌套访问 | assign 支持点号路径如 `result.field.subfield` |
| 统一模式 | 所有 CognitiveSkill 都应遵循此模式 |

**完整示例**：

```yaml
internal_flow:
  # 执行业务逻辑
  - type: select
    capability: analysis
    input:
      Query: question
    output:
      Analysis: analysis_result

  # 提取输出字段（与 output_schema 对应）
  - type: assign
    from: analysis_result.summary
    to: summary

  - type: assign
    from: analysis_result.confidence
    to: confidence

  - type: assign
    from: analysis_result.details
    to: details
```

### 7.8 设计理念

这种设计符合 CognitiveSkill 的**动态决策**本质：

| 设计考量 | 说明 |
|----------|------|
| 灵活性 | 允许不同分支产出不同语义类型的输出 |
| 显式映射 | 通过 assign 明确指定字段对应关系 |
| 实用性 | 运行时能正确匹配到下游输入即可 |
| 简洁性 | 避免复杂的联合类型声明语法 |
| 兼容性 | 与现有 SemanticInputMapper 匹配机制协同工作 |

---

## 8. 异常与成功处理

### 8.1 技能级 ignore

CognitiveSkill 支持技能级 `ignore` 属性：

```markdown
**ignore**: true
```

### 8.2 成功判定规则

**当 `ignore: false`（默认）时**：

技能成功需要同时满足：
1. 整个执行流程没有未捕获的异常
2. `output_schema` 中所有 `required: true` 的字段都成功生成

**当 `ignore: true` 时**：

技能成功仅需要：
- `output_schema` 中所有 `required: true` 的字段都成功生成

> 即使某些 select 节点失败，只要最终能生成必需的输出字段，技能即视为成功。

### 8.3 foreach 异常处理

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

### 8.4 fallback 与 ignore 的配合

```yaml
# 技能级 ignore: true
# select 有 fallback

- type: select
  capability: advanced_analysis
  input:
    Query: data
  output:
    Analysis: result
  fallback:
    - type: select
      capability: basic_analysis
      input:
        Query: data
      output:
        Analysis: result
```

执行顺序：
1. 尝试 `advanced_analysis`
2. 失败则尝试 `basic_analysis`（fallback）
3. fallback 也失败：
   - 若技能 `ignore: true`：检查是否能生成必需输出
   - 若技能 `ignore: false`：抛出异常

### 8.5 成功状态定义（含 foreach）

当 CognitiveSkill 包含 `foreach` 时：

| 条件 | 是否成功 |
|------|:--------:|
| `collect` 变量成功生成 | ✓ |
| `items` 为空数组，`collect` 为 `[]` | 取决于 `output_schema` 约束 |
| `ignore: true` 且全部轮次失败 | ✗ |
| `output_schema` required 字段缺失 | ✗ |

---

## 9. 完整示例

本章按 Agent 运行模式分类，为每种模式提供完整的 CognitiveSkill 示例。

> **注意**：`**mode**` 字段声明技能对应的 Agent 运行模式，声明后 `select` 节点中的角色名自动带上模式前缀。

### 9.1 Direct 模式 — 简单问答（cognitive_direct_answer）

Direct 模式适用于单步处理、无复杂推理的简单问答场景。

~~~markdown
# skill: cognitive_direct_answer

**version**: 3.4.0
**type**: CognitiveSkill
**mode**: Direct

## description

直接响应模式示例。用于简单问答场景，无需复杂推理，直接调用能力返回答案。

## capabilityTags

- 简单问答
- 直接响应
- 快速回答

## input_schema

```yaml
query:
  type: string
  required: true
  description: 用户问题
  roles:
    - Direct.Query
```

## internal_flow

```yaml
# 直接调用问答能力
- type: select
  capability: 简单问答,直接响应,快速回答
  input:
    Query: query
  output:
    Answer: answer
```

## output_schema

```yaml
answer:
  type: string
  required: true
  description: 回答内容
  roles:
    - Direct.Answer
```
~~~

### 9.2 CoT 模式 — 思维链推理（cognitive_reasoning）

CoT（Chain of Thought）模式适用于需要分步推理、逐步得出结论的复杂问题。

~~~markdown
# skill: cognitive_reasoning

**version**: 3.4.0
**type**: CognitiveSkill
**mode**: CoT

## description

思维链推理模式示例。通过分步思考，逐步推导出最终结论，适用于复杂推理、数学问题等场景。

## capabilityTags

- 推理
- 思维链
- 逻辑分析
- 复杂问题

## input_schema

```yaml
question:
  type: string
  required: true
  description: 待推理的问题
  roles:
    - CoT.Query
context:
  type: string
  required: false
  description: 背景信息
```

## internal_flow

```yaml
# 步骤1：生成推理链
- type: select
  capability: chain_of_thought
  input:
    Query: question
  output:
    ReasoningChain: reasoning_chain
    Thought: thoughts

# 步骤2：基于推理链得出结论
- type: select
  capability: conclusion_generation
  input:
    ReasoningChain: reasoning_chain
  output:
    Conclusion: conclusion
    Confidence: confidence
```

## output_schema

```yaml
reasoning_chain:
  type: array
  required: true
  description: 推理步骤链
  roles:
    - CoT.ReasoningChain
  items:
    step:
      type: string
    reasoning:
      type: string
conclusion:
  type: string
  required: true
  description: 最终结论
  roles:
    - CoT.Conclusion
confidence:
  type: number
  required: false
  description: 推理置信度
  roles:
    - CoT.Confidence
```
~~~

### 9.3 ReAct 模式 — 推理行动循环（cognitive_react_agent）

ReAct 模式适用于需要与外部工具交互的任务，通过"思考→行动→观察"循环完成目标。

~~~markdown
# skill: cognitive_react_agent

**version**: 3.4.0
**timeout**: 120000
**type**: CognitiveSkill
**mode**: ReAct

## description

ReAct 模式示例。通过"思考→行动→观察"循环，与外部工具交互完成任务。适用于需要多次工具调用的复杂任务。

## capabilityTags

- 工具调用
- 推理行动
- 多步任务
- 智能助手

## input_schema

```yaml
task:
  type: string
  required: true
  description: 任务目标描述
  roles:
    - ReAct.Task
```

## internal_flow

```yaml
# 使用 loop 节点实现 ReAct 循环
- type: loop
  max_iterations: 10
  exit_condition: "{{should_stop}}"
  do:
    # 思考：分析当前状态，决定下一步行动
    - type: select
      capability: thinking
      input:
        Task: task
        Observation: observation
      output:
        Thought: thought
        Action: action
        ActionInput: action_input

    # 行动：执行选定的工具
    - type: select
      capability: action_execution
      input:
        Action: action
        ActionInput: action_input
      output:
        Observation: observation

    # 判断是否完成
    - type: select
      capability: completion_check
      input:
        Task: task
        Observation: observation
      output:
        Answer: should_stop
  final: thought.final_answer
  as: answer
```

## output_schema

```yaml
answer:
  type: string
  required: true
  description: 最终答案
  roles:
    - ReAct.Answer
```
~~~

### 9.4 Decompose 模式 — 问题分解求解（cognitive_decompose_solver）

Decompose 模式适用于复杂多步骤任务，通过问题分解、子问题求解、综合答案完成。

~~~markdown
# skill: cognitive_decompose_solver

**version**: 3.4.0
**timeout**: 120000
**type**: CognitiveSkill
**mode**: Decompose

## description

分解求解模式示例。将复杂问题分解为子问题，逐个求解后综合答案。适用于复杂多步骤任务。

## capabilityTags

- 问题分解
- 逐步推理
- 答案合成
- 复杂任务

## input_schema

```yaml
question:
  type: string
  required: true
  description: 待解答的复杂问题
  roles:
    - Decompose.Question
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
  input:
    Question: question
  output:
    SubQuestions: sub_questions

# 步骤2：逐个子问题求解
- type: foreach
  items: sub_questions
  as: sq
  do:
    - type: select
      capability: sub_question_solving
      input:
        SubQuestion: sq
      output:
        SubAnswer: sub_answer       # select 的输出是一个包含多字段的对象
  collect: sub_answer               # 收集每轮的 sub_answer 对象

# 步骤3：综合所有子答案
- type: select
  capability: answer_synthesis
  input:
    Question: question
    SubAnswers: sub_answer          # 引用 foreach 收集的结果
  output:
    Synthesis: final_answer
```

## output_schema

```yaml
sub_questions:
  type: array
  required: true
  description: 分解后的子问题列表
  roles:
    - Decompose.SubQuestions
  items:
    question:
      type: string
    priority:
      type: number
sub_answer:
  type: array
  required: true
  description: 各子问题的答案（foreach 收集结果）
  items:
    answer:
      type: string
    evidence:
      type: string
final_answer:
  type: string
  required: true
  description: 综合后的最终答案
  roles:
    - Decompose.Synthesis
```
~~~

### 9.5 Retrieve 模式 — 检索增强问答（cognitive_rag）

Retrieve 模式适用于需要外部知识支持的查询，通过检索→综合→回答完成。

~~~markdown
# skill: cognitive_rag

**version**: 3.4.0
**type**: CognitiveSkill
**mode**: Retrieve

## description

检索增强模式示例。先检索相关信息，再基于检索结果生成回答。适用于需要外部知识的问答场景。

## capabilityTags

- 检索增强
- RAG
- 知识问答
- 信息检索

## input_schema

```yaml
query:
  type: string
  required: true
  description: 检索查询
  roles:
    - Retrieve.Query
```

## internal_flow

```yaml
# 步骤1：检索相关文档
- type: select
  capability: document_retrieval
  input:
    Query: query
  output:
    SearchResults: search_results

# 步骤2：筛选和排序结果
- type: decision
  condition: array::count({{search_results}}) > 10
  then:
    - type: select
      capability: result_reranking
      input:
        Query: query
        SearchResults: search_results
      output:
        Context: context
  else:
    - type: assign
      from: search_results
      to: context

# 步骤3：基于上下文生成回答
- type: select
  capability: answer_generation
  input:
    Query: query
    Context: context
  output:
    Answer: answer
    Sources: sources
```

## output_schema

```yaml
answer:
  type: string
  required: true
  description: 基于检索的回答
  roles:
    - Retrieve.Answer
sources:
  type: array
  required: false
  description: 引用来源
  roles:
    - Retrieve.Sources
  items:
    title:
      type: string
    url:
      type: string
```
~~~

### 9.6 Compare 模式 — 多源对比分析（cognitive_compare）

Compare 模式适用于方案比较、竞品分析等需要多源数据对比的场景。

~~~markdown
# skill: cognitive_compare

**version**: 3.4.0
**type**: CognitiveSkill
**mode**: Compare

## description

多源对比模式示例。收集多个来源的数据，进行多维度对比分析，给出结论和建议。

## capabilityTags

- 对比分析
- 竞品分析
- 方案比较
- 多维评估

## input_schema

```yaml
subject:
  type: string
  required: true
  description: 对比主题
  roles:
    - Compare.Subject
sources:
  type: array
  required: true
  description: 对比对象列表
  roles:
    - Compare.Sources
  items:
    name:
      type: string
    description:
      type: string
dimensions:
  type: array
  required: false
  description: 对比维度
  items:
    type: string
```

## internal_flow

```yaml
# 步骤1：逐个收集各对象的详细信息
- type: foreach
  items: sources
  as: source
  do:
    - type: select
      capability: source_analysis
      input:
        Source: source
        Dimension: dimensions
      output:
        Analysis: source_detail
  collect: source_details

# 步骤2：进行多维度对比
- type: select
  capability: multi_dimension_compare
  input:
    Subject: subject
    Sources: source_details
    Dimension: dimensions
  output:
    Similarities: similarities
    Differences: differences

# 步骤3：生成分析结论和建议
- type: select
  capability: recommendation_generation
  input:
    Subject: subject
    Similarities: similarities
    Differences: differences
  output:
    Analysis: analysis
    Recommendation: recommendation
```

## output_schema

```yaml
similarities:
  type: array
  required: true
  description: 相似点列表
  roles:
    - Compare.Similarities
  items:
    type: string
differences:
  type: array
  required: true
  description: 差异点列表
  roles:
    - Compare.Differences
  items:
    dimension:
      type: string
    comparison:
      type: string
analysis:
  type: string
  required: true
  description: 对比分析结论
  roles:
    - Compare.Analysis
recommendation:
  type: string
  required: false
  description: 推荐建议
  roles:
    - Compare.Recommendation
```
~~~

### 9.7 Generate 模式 — 内容生成（cognitive_content_generator）

Generate 模式适用于模板驱动或约束驱动的内容创作，如报告生成、文档创作等。

~~~markdown
# skill: cognitive_content_generator

**version**: 3.4.0
**type**: CognitiveSkill
**mode**: Generate

## description

内容生成模式示例。基于需求、模板和约束条件，生成符合要求的内容。适用于报告生成、文档创作等场景。

## capabilityTags

- 内容生成
- 报告生成
- 文档创作
- 模板填充

## input_schema

```yaml
requirement:
  type: string
  required: true
  description: 生成需求描述
  roles:
    - Generate.Requirement
template:
  type: string
  required: false
  description: 内容模板（可选）
  roles:
    - Generate.Template
constraints:
  type: array
  required: false
  description: 生成约束条件
  roles:
    - Generate.Constraints
  items:
    type: string
```

## internal_flow

```yaml
# 步骤1：生成初稿
- type: select
  capability: draft_generation
  input:
    Requirement: requirement
    Template: template
    Constraints: constraints
  output:
    Draft: draft

# 步骤2：质量检查
- type: select
  capability: quality_check
  input:
    Draft: draft
    Requirement: requirement
    Constraints: constraints
  output:
    Score: quality_score
    Issues: issues

# 步骤3：根据质量决定是否需要优化
- type: decision
  condition: {{quality_score}} < 0.8
  then:
    # 质量不足：进行优化
    - type: select
      capability: content_refinement
      input:
        Draft: draft
        Issues: issues
      output:
        Refined: refined
    - type: assign
      from: refined
      to: output
  else:
    # 质量达标：直接输出
    - type: assign
      from: draft
      to: output
```

## output_schema

```yaml
output:
  type: string
  required: true
  description: 最终生成的内容
  roles:
    - Generate.Output
```
~~~

### 9.8 综合示例 — 带 fallback 的深度分析（cognitive_analysis）

展示多种节点组合使用，包含 fallback 降级策略。

~~~markdown
# skill: cognitive_analysis

**version**: 3.4.0
**type**: CognitiveSkill
**mode**: CoT

## description

深度分析认知技能。对复杂问题进行深度分析，支持证据收集、多轮推理和降级策略。

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
  roles:
    - CoT.Query
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
  input:
    Query: question
  output:
    SubQuestions: sub_questions

# 步骤2：收集证据（带 fallback）
- type: select
  capability: evidence_collection
  input:
    Query: sub_questions
  output:
    Evidence: evidence
  fallback:
    - type: select
      capability: basic_search
      input:
        Query: sub_questions
      output:
        SearchResults: evidence

# 步骤3：判断证据充分性
- type: decision
  condition: array::count({{evidence}}) >= 3
  then:
    # 证据充分：直接推理
    - type: select
      capability: reasoning
      input:
        Query: question
        Evidence: evidence
      output:
        Conclusion: reasoning_result
        Confidence: confidence
  else:
    # 证据不足：补充搜索后推理
    - type: select
      capability: deep_search
      input:
        Query: question
      output:
        SearchResults: more_evidence
    - type: merge
      sources:
        - evidence
        - more_evidence
      as: all_evidence
    - type: select
      capability: reasoning
      input:
        Query: question
        Evidence: all_evidence
      output:
        Conclusion: reasoning_result
        Confidence: confidence

# 步骤4：提取输出字段
- type: assign
  from: reasoning_result
  to: analysis
```

## output_schema

```yaml
analysis:
  type: string
  required: true
  description: 分析结果
  roles:
    - CoT.Conclusion
confidence:
  type: number
  required: false
  description: 分析置信度
  roles:
    - CoT.Confidence
```
~~~

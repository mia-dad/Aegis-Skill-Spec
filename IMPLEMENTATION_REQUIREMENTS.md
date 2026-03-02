# Aegis Skill DSL 平台实现需求说明

> **文档版本**：1.3.0
> **规范版本**：1.1.0 → 1.2.0 → 2.0.0 → 2.1.0 → 2.2.0 → 2.3.0
> **日期**：2026-03-01
> **目标受众**：DSL 平台开发者

---

## 1. 变更概述

本文档涵盖 Aegis Skill DSL 从 **1.1.0** 到 **2.3.0** 的完整升级内容，分为多个阶段：

| 升级阶段 | 主要变更 |
|----------|----------|
| 1.1.0 → 1.2.0 | 类型完整性规则、数组字段投影、循环索引、内置工具、统一 when 条件语法 |
| 1.2.0 → 2.0.0 | 技能分类体系（AtomicSkill / PresentationSkill）、文档拆分 |
| 2.0.0 → 2.1.0 | 循环控制指令 `{{break}}`；新增 `datetime` 和 `resource` 类型；能力语义系统（traits + semantic_role） |
| 2.1.0 → 2.2.0 | 内置函数系统重构（`namespace::function()` 语法）；函数链式调用；`ignore` 异常处理参数；Step `ignore` 属性 |
| 2.2.0 → 2.3.0 | CognitiveSkill 类型：internal_flow 控制流、decision/select/assign/merge 四种节点类型、动态能力匹配、fallback 异常处理 |

---

# 第一部分：1.1.0 → 1.2.0 变更

## 1.1 类型完整性规则

**优先级**：P0（必须）

**需求描述**：强制要求 `array` 和 `object` 类型必须完整定义子级结构，直到最末级节点的类型为基本类型（`string`、`number`、`boolean`）。

**适用范围**：`input_schema` 和 `output_schema`

**校验规则**：

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

**唯一例外**：当 `array` 配合 `options` 实现多选时，不需要定义 `items`（`options` 已隐含元素为字符串类型）

**实现要点**：
1. 解析器遍历 schema 树时，递归检查每个 `array` 和 `object` 节点
2. `array` 必须有 `items`，`object` 必须有子字段定义
3. 递归直到叶子节点为 `string`、`number` 或 `boolean`
4. 违反规则时报错，提示具体字段路径

**错误消息示例**：

```
ERROR: input_schema.headers: array 类型必须定义 items
ERROR: output_schema.records.items: object 类型必须定义子字段结构
```

---

## 1.2 数组字段投影语法

**优先级**：P0（必须）

**需求描述**：新增 `{{arr[*].field}}` 语法，从结构化数组中提取单一字段，生成一个简单数组。

**语法定义**：

| 语法 | 说明 | 示例 |
|------|------|------|
| `{{arr[*].field}}` | 提取数组中每个元素的 `field` 字段，组成新的简单数组 | `{{result[*].score}}` |

**使用示例**：

```yaml
# 假设 result = [{content: "x", score: 80}, {content: "y", score: 90}]

所有分数：{{result[*].score}}
# 结果：[80, 90]

所有内容：{{result[*].content}}
# 结果：["x", "y"]
```

**实现要点**：
1. 模板引擎识别 `[*]` 语法
2. 遍历数组，提取每个元素的指定字段
3. 返回新的简单数组
4. 支持链式访问：`{{arr[*].nested.field}}`

---

## 1.3 循环索引变量

**优先级**：P0（必须）

**需求描述**：在循环渲染 `{{#for}}` 内部新增 `{{_index}}` 内置变量，获取当前迭代的索引值（从 0 开始）。

**语法示例**：

```template
{{#for result}}
第 {{_index + 1}} 条：{{region}}，{{product}}，销售量：{{amount}}
{{/for}}
```

**循环内置变量汇总**：

| 变量 | 说明 |
|------|------|
| `{{_}}` | 当前迭代元素本身（已有功能） |
| `{{_index}}` | 当前迭代索引（从 0 开始）**[新增]** |

**实现要点**：
1. 进入循环时初始化 `_index = 0`
2. 每次迭代后 `_index++`
3. `_index` 支持表达式运算，如 `{{_index + 1}}`
4. 循环结束后清理 `_index` 变量

**模板引擎实现示例（伪代码）**：

```java
public class TemplateEngine {

    /**
     * 执行 {{#for array}}...{{/for}} 循环渲染
     */
    public String executeForLoop(String arrayName, String bodyTemplate, Context ctx) {
        List<?> array = (List<?>) ctx.get(arrayName);
        if (array == null || array.isEmpty()) {
            return "";
        }

        StringBuilder result = new StringBuilder();

        for (int i = 0; i < array.size(); i++) {
            Object element = array.get(i);

            // 创建循环作用域上下文
            Context loopCtx = ctx.createChildScope();

            // [1.2.0 新增] 设置循环索引变量
            loopCtx.put("_index", i);

            // 设置当前元素引用（用于简单数组）
            loopCtx.put("_", element);

            // 如果是结构化对象，展开字段到上下文
            if (element instanceof Map) {
                Map<String, Object> map = (Map<String, Object>) element;
                for (Map.Entry<String, Object> entry : map.entrySet()) {
                    loopCtx.put(entry.getKey(), entry.getValue());
                }
            }

            // 渲染循环体
            String rendered = renderTemplate(bodyTemplate, loopCtx);
            result.append(rendered);
        }

        return result.toString();
    }

    /**
     * 表达式求值，支持 _index 参与运算
     * 示例：{{_index + 1}} → 1, 2, 3, ...
     */
    public Object evaluateExpression(String expr, Context ctx) {
        // 解析表达式中的变量引用
        // _index 作为普通数字变量参与四则运算
        // 示例：_index + 1, _index * 2, _index == 0
        // ...
    }
}
```

**循环作用域规则**：

| 规则 | 说明 |
|------|------|
| 作用域隔离 | `_index` 和 `_` 仅在当前循环体内有效 |
| 嵌套循环 | 内层循环的 `_index` 覆盖外层，外层 `_index` 不可见 |
| 循环结束 | 自动清理 `_index` 和 `_`，不污染外层上下文 |
| 变量优先级 | 循环变量 > 步骤输出 > 输入参数 |

**嵌套循环示例**：

```template
{{#for departments}}
部门 {{_index + 1}}：{{name}}
  {{#for members}}
  - 成员 {{_index + 1}}：{{memberName}}
  {{/for}}
{{/for}}
```

> 内层 `{{_index}}` 指的是 `members` 数组的索引，外层 `departments` 的索引在内层不可直接访问。

---

## 1.4 内置数组操作工具

**优先级**：P1（重要）

**需求描述**：新增一组内置工具用于处理常见的数组操作，无需额外注册即可使用。

### 1.4.1 array.zip

将多个简单数组按索引位置组装为一个结构化数组。

**输入参数**：

| 参数 | 类型 | 必需 | 说明 |
|------|------|------|------|
| `arrays` | object | 是 | 键值对，key 为目标字段名，value 为源数组变量 |

**输出**：

| 字段 | 类型 | 说明 |
|------|------|------|
| `result` | array | 组装后的结构化数组 |

**错误处理**：当输入数组长度不一致时，以最短数组长度为准，忽略多余元素。

**使用示例**：

```yaml
input:
  arrays:
    name: "{{names}}"
    score: "{{scores}}"
    grade: "{{grades}}"
output_schema:
  result:
    type: array
    items:
      name:
        type: string
      score:
        type: number
      grade:
        type: string
```

### 1.4.2 array.flatten

将嵌套数组展平为一维数组。

| 参数 | 类型 | 必需 | 说明 |
|------|------|------|------|
| `array` | array | 是 | 要展平的嵌套数组 |
| `depth` | number | 否 | 展平深度，默认为 1 |

### 1.4.3 array.unique

对数组去重，返回不包含重复元素的新数组。

| 参数 | 类型 | 必需 | 说明 |
|------|------|------|------|
| `array` | array | 是 | 要去重的数组 |
| `key` | string | 否 | 对于结构化数组，指定用于判断重复的字段名 |

### 1.4.4 array.sort

对数组进行排序。

| 参数 | 类型 | 必需 | 说明 |
|------|------|------|------|
| `array` | array | 是 | 要排序的数组 |
| `key` | string | 否 | 对于结构化数组，指定排序字段 |
| `order` | string | 否 | 排序方向：`asc`（升序，默认）或 `desc`（降序） |

### 1.4.5 array.filter

根据条件过滤数组元素。

| 参数 | 类型 | 必需 | 说明 |
|------|------|------|------|
| `array` | array | 是 | 要过滤的数组 |
| `condition` | string | 是 | 过滤条件表达式，支持 `==`、`!=`、`>`、`<`、`>=`、`<=` |

---

## 1.5 统一 when 条件语法

**优先级**：P0（必须）

**需求描述**：统一步骤级 `when` 条件执行和模板级 `{{#when}}` 条件渲染的语法规范。

### 1.5.1 步骤级 when 条件

在 Step 的 YAML 配置块中定义，控制整个步骤是否执行：

```yaml
when:
  expr: "{{region}} == null"
```

**语法要点**：
- 变量引用使用 `{{variable}}` 格式
- 整个表达式必须用**双引号**包裹（YAML 语法要求）
- 字符串字面量使用转义双引号 `\"`

**示例**：

```yaml
when:
  expr: "{{status}} == \"active\""
```

```yaml
when:
  expr: "{{count}} > 10 && {{enabled}} == true"
```

### 1.5.2 模板级条件渲染

在模板内部使用 `{{#when condition}}...{{/when}}` 进行条件渲染：

**基本语法**：

```template
{{#when condition}}
条件为真时显示的内容
{{/when}}
```

**带 else 分支**：

```template
{{#when condition}}
条件为真时显示
{{:else}}
条件为假时显示
{{/when}}
```

**语法要点**：
- 条件表达式直接写在 `{{#when ...}}` 内部，**不需要**额外的 `{{}}`
- 字符串字面量使用普通双引号 `"`

**示例**：

```template
{{#when status == "active"}}
状态：激活
{{:else}}
状态：未激活
{{/when}}
```

### 1.5.3 支持的操作符

| 操作符 | 说明 | 示例 |
|--------|------|------|
| `==` | 等于 | `{{status}} == "active"` |
| `!=` | 不等于 | `{{status}} != "pending"` |
| `>` | 大于 | `{{count}} > 10` |
| `<` | 小于 | `{{count}} < 10` |
| `>=` | 大于等于 | `{{count}} >= 10` |
| `<=` | 小于等于 | `{{count}} <= 10` |
| `&&` | 逻辑与 | `{{a}} == true && {{b}} == false` |
| `\|\|` | 逻辑或 | `{{a}} == null \|\| {{b}} != ""` |

### 1.5.4 步骤级 vs 模板级对比

| 特性 | 步骤级 `when` | 模板级 `{{#when}}` |
|------|--------------|-------------------|
| 作用范围 | 控制整个步骤是否执行 | 控制模板内部分内容是否渲染 |
| 语法位置 | YAML 配置块内 | 模板代码块内 |
| 变量引用 | 需要 `{{variable}}` | 直接使用变量名 |
| 字符串引号 | 转义双引号 `\"` | 普通双引号 `"` |

---

## 1.6 1.1.0 → 1.2.0 测试要点

### 类型完整性规则

| 测试场景 | 预期结果 |
|----------|----------|
| array 类型无 items | 解析报错 |
| object 类型无子字段 | 解析报错 |
| array + options（多选） | 正常解析，不要求 items |
| 嵌套 array 未完整定义 | 解析报错，提示具体路径 |

### 数组字段投影

| 测试场景 | 预期结果 |
|----------|----------|
| `{{arr[*].field}}` 投影 | 返回简单数组 |
| `{{arr[*].nested.field}}` 链式投影 | 正确提取嵌套字段 |
| 空数组投影 | 返回空数组 `[]` |

### for 循环与索引

| 测试场景 | 预期结果 |
|----------|----------|
| `{{_index}}` 在循环内 | 返回当前索引（从 0 开始） |
| `{{_index + 1}}` 表达式 | 正确计算（从 1 开始） |
| `{{_index}}` 在循环外 | 变量未定义或为 null |
| `{{_}}` 引用简单数组元素 | 返回当前元素值 |
| 嵌套循环内层 `{{_index}}` | 返回内层循环索引 |
| 嵌套循环外层 `{{_index}}` 不可见 | 内层无法访问外层索引 |
| 循环结束后 `{{_index}}` | 变量已清理，不污染上下文 |

### 内置数组工具

| 测试场景 | 预期结果 |
|----------|----------|
| array.zip 等长数组 | 正确组装 |
| array.zip 不等长数组 | 按最短数组处理 |
| array.zip 空数组 | 返回空数组 |
| array.sort 升序 | 正确排序 |
| array.filter 条件过滤 | 返回符合条件的元素 |

### when 条件语法

| 测试场景 | 预期结果 |
|----------|----------|
| 步骤级 when 条件为 true | 执行步骤 |
| 步骤级 when 条件为 false | 跳过步骤 |
| 模板级 {{#when}} 条件为 true | 渲染内容 |
| 模板级 {{#when}} 条件为 false | 不渲染或渲染 else |

---

# 第二部分：1.2.0 → 2.0.0 变更

## 2. 技能分类体系

本次升级将 Aegis Skill DSL 从 **1.2.0** 升级至 **2.0.0**，引入**技能分类体系**，将原有的单一 Skill 结构拆分为两种类型：

| 技能类型 | 核心职责 | 必需章节 | 禁止章节 |
|----------|----------|----------|----------|
| `AtomicSkill` | 数据处理、业务逻辑 | input_schema, output_schema, steps | ui |
| `PresentationSkill` | UI 渲染、数据展示 | input_schema, ui | output_schema |

**设计理念**：分离数据处理与 UI 展示，AtomicSkill 负责"数据结构"，PresentationSkill 负责"语义展示"。技能的串联编排由 Agent 层负责，DSL 层只需关注单个技能的定义。

---

## 2.1 需要实现的功能点（2.0.0）

### 2.1.1 新增 `type` 字段解析

**优先级**：P0（必须）

**需求描述**：
- 在技能文件顶层新增 `**type**` 字段解析
- 位置：在 `**version**` 之后

**解析规则**：

```markdown
# skill: example_skill

**version**: 2.0.0
**type**: AtomicSkill   ← 新增字段
```

**字段定义**：

| 属性 | 说明 |
|------|------|
| 格式 | `**type**: <SkillType>` |
| 取值 | `AtomicSkill` \| `PresentationSkill` |
| 默认值 | `AtomicSkill`（向后兼容） |
| 必填性 | 建议显式声明，AtomicSkill 可省略 |

**实现要点**：
1. 解析器需支持从 Markdown 提取 `**type**:` 后的值
2. 若未声明 type，默认为 `AtomicSkill`
3. type 值需进行枚举校验，非法值应报错

**代码示例（伪代码）**：

```java
public enum SkillType {
    AtomicSkill,
    PresentationSkill,
    CognitiveSkill  // 预留，暂未实现
}

// 解析逻辑
String typeValue = extractField(markdown, "type");
SkillType type = (typeValue == null) ? SkillType.AtomicSkill : SkillType.valueOf(typeValue);
```

---

### 2.1.2 技能结构校验规则

**优先级**：P0（必须）

**需求描述**：根据 `type` 字段值，校验技能文件的章节结构。

**AtomicSkill 校验规则**：

| 章节 | 校验规则 |
|------|----------|
| `## output_schema` | **必需**，缺失应报错 |
| `## steps` | **必需**，且至少包含一个 step |
| `## ui` | **禁止**，存在应报错 |

**PresentationSkill 校验规则**：

| 章节 | 校验规则 |
|------|----------|
| `## input_schema` | **必需**，缺失应报错 |
| `## ui` | **必需**，缺失应报错 |
| `## steps` | **可选**，允许存在（用于数据转换） |
| `## output_schema` | **禁止**，存在应报错 |

**错误消息示例**：

```
ERROR: AtomicSkill 'order_process' 缺少必需的 output_schema 章节
ERROR: AtomicSkill 'order_process' 不应包含 ui 章节，UI 展示应由 PresentationSkill 负责
ERROR: PresentationSkill 'order_display' 不应包含 output_schema 章节
```

---

### 2.1.3 技能数据模型更新

**优先级**：P0（必须）

**需求描述**：更新技能数据模型，增加 type 字段。

**数据库变更**：

```sql
-- 为 skills 表增加 type 字段
ALTER TABLE skills ADD COLUMN type VARCHAR(32) NOT NULL DEFAULT 'AtomicSkill';

-- 添加枚举约束（可选，取决于数据库类型）
ALTER TABLE skills ADD CONSTRAINT chk_skill_type
    CHECK (type IN ('AtomicSkill', 'PresentationSkill', 'CognitiveSkill'));

-- 创建索引（用于按类型查询）
CREATE INDEX idx_skills_type ON skills(type);
```

**API 响应变更**：

```json
{
  "skillId": "order_process",
  "version": "2.0.0",
  "type": "AtomicSkill",  // 新增字段
  "description": "订单处理技能",
  "inputSchema": { ... },
  "outputSchema": { ... },
  "steps": [ ... ]
}
```

---

### 2.1.4 PresentationSkill 的 UI 渲染支持

**优先级**：P1（重要）

**需求描述**：实现 PresentationSkill 的 `## ui` 章节解析和渲染。

**UI 定义结构**：

```yaml
display: <display_type>
mapping:
  <ui_field>: "{{data_field}}"
config:
  <config_key>: <value>
```

**支持的 Display 类型**：

| 类型 | 说明 | 实现优先级 |
|------|------|------------|
| `text` | 纯文本 | P0 |
| `markdown` | Markdown 渲染 | P0 |
| `table` | 表格 | P0 |
| `list` | 列表 | P1 |
| `card` | 卡片 | P1 |
| `chart` | 图表 | P2 |
| `file` | 文件下载 | P1 |
| `form` | 表单 | P2 |
| `mixed` | 混合布局 | P1 |

**Mapping 模板解析**：

UI 的 mapping 字段支持模板语法 `{{variable}}`，需复用现有模板引擎：

```yaml
mapping:
  content: "{{report}}"        # 引用 input_schema 或 steps 输出
  title: "{{title}}"
  rows: "{{data}}"             # 支持数组引用
```

**数据来源优先级**：
1. `## steps` 的输出变量（如果有 steps）
2. `## input_schema` 定义的输入字段

---

### 2.1.5 PresentationSkill Steps 支持

**优先级**：P1（重要）

**需求描述**：PresentationSkill 的 `## steps` 为可选章节，主要用于：

1. **数据格式转换**：使用 template step
2. **LLM 生成展示内容**：使用 prompt step
3. **数据聚合计算**：使用 template step 进行表达式计算

**执行流程**：

```
PresentationSkill 执行流程：

1. 接收 input_schema 数据（来自上游 AtomicSkill）
         ↓
2. [可选] 执行 steps（数据转换/LLM 生成）
         ↓
3. 将上下文数据映射到 UI 定义
         ↓
4. 渲染 UI 组件
```

**实现要点**：
- Steps 执行逻辑复用现有 AtomicSkill 的 step 执行器
- Steps 输出写入上下文后，UI mapping 可引用

---

### 2.1.6 Mixed 布局支持

**优先级**：P1（重要）

**需求描述**：实现 `display: mixed` 类型，支持在一个页面组合多个展示组件。

**Mixed 布局结构**：

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
```

**实现要点**：
1. `layout` 是数组，按顺序渲染
2. 每个元素包含 `type`、`mapping`、`config`（可选）
3. 递归复用各 display 类型的渲染逻辑

---

### 2.1.7 审计日志限制

**优先级**：P2（建议）

**需求描述**：将审计日志 (`log` 工具) 限制为 AtomicSkill 专属功能。

**实现要点**：
1. 在 PresentationSkill 中调用 `log` 工具应产生警告或忽略
2. 文档层面已明确：审计日志仅适用于 AtomicSkill

**可选实现方式**：
- 方式 A：运行时忽略 PresentationSkill 的 log 调用
- 方式 B：解析时校验，PresentationSkill 禁止使用 log 工具

---

## 2.2 向后兼容性（2.0.0）

### 2.2.1 兼容性策略

| 场景 | 处理方式 |
|------|----------|
| 未声明 type 的技能 | 默认视为 `AtomicSkill` |
| 1.x 版本技能文件 | 按 AtomicSkill 处理，保持原有行为 |
| 同时包含 ui 和 output_schema | 报错，提示需明确技能类型 |

### 2.2.2 迁移指南

**现有技能迁移**：

1. **纯数据处理技能**：添加 `**type**: AtomicSkill`，移除 `## ui` 章节
2. **含 UI 的技能**：拆分为两个技能文件
   - `xxx.md` → AtomicSkill（数据处理）
   - `xxx_display.md` → PresentationSkill（UI 展示）

---

---

# 第三部分：2.0.0 → 2.1.0 变更

## 3. 循环控制指令

### 3.1 break 语法

**优先级**：P0（必须）

**需求描述**：在 `{{#for}}` 循环中新增 `{{break}}` 控制指令，支持提前终止循环。

**语法定义**：

```template
{{#for array}}
{{break}}
{{/for}}
```

**使用场景**：

| 场景 | 示例 |
|------|------|
| 限制输出数量 | 只显示前 N 条记录 |
| 查找终止 | 找到目标后立即退出 |
| 条件中断 | 满足特定条件时停止遍历 |

### 3.2 与 when 条件嵌套

`{{break}}` 可嵌套在 `{{#when}}` 条件中，实现条件中断：

**基本语法**：

```template
{{#for items}}
{{#when _index >= 3}}
{{break}}
{{/when}}
处理第 {{_index + 1}} 项：{{name}}
{{/for}}
```

**查找终止示例**：

```template
{{#for users}}
{{#when status == "active"}}
找到活跃用户：{{name}}
{{break}}
{{/when}}
{{/for}}
```

**带累计逻辑的中断**：

```template
{{#for orders}}
订单 {{order_id}}：¥{{amount}}
{{#when _index >= 4}}
（仅显示前 5 条记录）
{{break}}
{{/when}}
{{/for}}
```

### 3.3 嵌套循环行为

`{{break}}` 仅终止最内层循环，外层循环继续执行：

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

> 上例中，内层 `items` 循环遇到 `price > 1000` 时终止，但外层 `categories` 循环继续。

### 3.4 实现要点

**语法规则**：

| 规则 | 说明 |
|------|------|
| 作用范围 | 仅在 `{{#for}}` 循环体内有效 |
| 执行效果 | 立即终止当前循环，继续执行循环后的内容 |
| 嵌套循环 | 仅终止最内层循环 |
| 条件嵌套 | 可嵌套在 `{{#when}}` 中实现条件中断 |
| 循环外使用 | 应报错或忽略（建议报错） |

**模板引擎实现（伪代码）**：

```java
public class TemplateEngine {

    // 自定义异常用于 break 控制流
    private static class BreakException extends RuntimeException {}

    /**
     * 执行 {{#for array}}...{{/for}} 循环渲染
     * [2.1.0] 支持 {{break}} 提前终止
     */
    public String executeForLoop(String arrayName, String bodyTemplate, Context ctx) {
        List<?> array = (List<?>) ctx.get(arrayName);
        if (array == null || array.isEmpty()) {
            return "";
        }

        StringBuilder result = new StringBuilder();

        for (int i = 0; i < array.size(); i++) {
            Object element = array.get(i);

            // 创建循环作用域上下文
            Context loopCtx = ctx.createChildScope();
            loopCtx.put("_index", i);
            loopCtx.put("_", element);

            // 展开结构化对象字段
            if (element instanceof Map) {
                Map<String, Object> map = (Map<String, Object>) element;
                for (Map.Entry<String, Object> entry : map.entrySet()) {
                    loopCtx.put(entry.getKey(), entry.getValue());
                }
            }

            try {
                // 渲染循环体
                String rendered = renderTemplate(bodyTemplate, loopCtx);
                result.append(rendered);
            } catch (BreakException e) {
                // [2.1.0] 捕获 break，终止循环
                break;
            }
        }

        return result.toString();
    }

    /**
     * 处理 {{break}} 指令
     * [2.1.0] 新增
     */
    private void handleBreak() {
        throw new BreakException();
    }

    /**
     * 渲染模板，识别 {{break}} 指令
     */
    private String renderTemplate(String template, Context ctx) {
        // 解析模板内容
        // 遇到 {{break}} 时调用 handleBreak()
        if (template.contains("{{break}}")) {
            // 处理 break 前的内容
            int breakIndex = template.indexOf("{{break}}");
            String beforeBreak = template.substring(0, breakIndex);
            String rendered = processTemplate(beforeBreak, ctx);

            // 检查 break 是否在 when 条件中
            // 如果 when 条件为 true，则触发 break
            if (isBreakActive(template, ctx)) {
                throw new BreakException();
            }
            return rendered + processTemplate(template.substring(breakIndex + 9), ctx);
        }
        return processTemplate(template, ctx);
    }
}
```

**替代实现方案（标志位）**：

```java
public String executeForLoop(String arrayName, String bodyTemplate, Context ctx) {
    List<?> array = (List<?>) ctx.get(arrayName);
    StringBuilder result = new StringBuilder();

    for (int i = 0; i < array.size(); i++) {
        Context loopCtx = ctx.createChildScope();
        loopCtx.put("_index", i);
        loopCtx.put("_", array.get(i));
        loopCtx.put("_break", false);  // 初始化 break 标志

        String rendered = renderTemplate(bodyTemplate, loopCtx);
        result.append(rendered);

        // [2.1.0] 检查 break 标志
        if ((Boolean) loopCtx.get("_break")) {
            break;
        }
    }

    return result.toString();
}

// 处理 {{break}} 时设置标志
private void handleBreak(Context ctx) {
    ctx.put("_break", true);
}
```

### 3.5 错误处理

| 场景 | 处理方式 |
|------|----------|
| `{{break}}` 在循环外 | 报错：`break 只能在 for 循环内使用` |
| `{{break}}` 语法错误（如 `{{break }}`） | 报错：`无效的 break 语法` |
| 多余参数（如 `{{break 2}}`） | 报错或忽略（建议报错：`break 不接受参数`） |

### 3.6 测试要点

| 测试场景 | 预期结果 |
|----------|----------|
| 简单 break | 立即终止循环，输出 break 前的内容 |
| break 在 when 条件中（条件为 true） | 终止循环 |
| break 在 when 条件中（条件为 false） | 继续循环 |
| 嵌套循环内层 break | 仅终止内层，外层继续 |
| 嵌套循环外层 break | 终止外层循环 |
| break 在空循环中 | 无操作 |
| break 在循环外 | 报错 |
| break 后的循环体内容 | 不执行 |

**测试用例示例**：

```yaml
# 测试1：限制输出数量
input:
  items: [{name: "A"}, {name: "B"}, {name: "C"}, {name: "D"}, {name: "E"}]
template: |
  {{#for items}}
  - {{name}}
  {{#when _index >= 2}}
  {{break}}
  {{/when}}
  {{/for}}
expected: |
  - A
  - B
  - C

# 测试2：查找终止
input:
  users: [{name: "张三", status: "inactive"}, {name: "李四", status: "active"}, {name: "王五", status: "active"}]
template: |
  {{#for users}}
  {{#when status == "active"}}
  找到：{{name}}
  {{break}}
  {{/when}}
  {{/for}}
expected: |
  找到：李四
```

---

## 3.7 datetime 基本类型

**优先级**：P0（必须）

**需求描述**：新增 `datetime` 基本类型，用于处理日期和时间。系统自动识别多种输入格式，统一转换为标准输出格式。

### 3.7.1 支持的输入格式

| 格式类型 | 示例 | 说明 |
|----------|------|------|
| 仅日期（ISO） | `2026-02-28` | 标准日期格式 |
| 仅日期（斜杠） | `2026/02/28` | 斜杠分隔 |
| 仅日期（紧凑） | `20260228` | 8 位数字 |
| 仅时间（完整） | `14:30:00` | 时:分:秒 |
| 仅时间（简短） | `14:30` | 时:分，秒默认为 00 |
| 仅时间（紧凑） | `143000` | 6 位数字 |
| 日期+时间（空格） | `2026-02-28 14:30:00` | 空格分隔 |
| 日期+时间（T） | `2026-02-28T14:30:00` | ISO 8601 格式 |
| 日期+时间（简短） | `2026/02/28 14:30` | 省略秒 |

### 3.7.2 输出格式

**统一输出格式**：

```
yyyy年MM月dd日 HH:mm:ss
```

示例：`2026年02月28日 14:30:00`

### 3.7.3 格式转换规则

| 输入 | 输出 | 说明 |
|------|------|------|
| `2026-02-28` | `2026年02月28日 00:00:00` | 仅日期，时间补 00:00:00 |
| `14:30` | `2026年02月28日 14:30:00` | 仅时间，日期补当天 |
| `14:30:45` | `2026年02月28日 14:30:45` | 仅时间，日期补当天 |
| `2026-02-28 14:30:00` | `2026年02月28日 14:30:00` | 完整日期时间 |
| `2026/02/28 14:30` | `2026年02月28日 14:30:00` | 秒补 00 |

### 3.7.4 实现要点

**解析器实现（伪代码）**：

```java
public class DateTimeParser {

    // 支持的日期格式
    private static final String[] DATE_PATTERNS = {
        "yyyy-MM-dd",
        "yyyy/MM/dd",
        "yyyyMMdd"
    };

    // 支持的时间格式
    private static final String[] TIME_PATTERNS = {
        "HH:mm:ss",
        "HH:mm",
        "HHmmss"
    };

    // 支持的日期时间格式
    private static final String[] DATETIME_PATTERNS = {
        "yyyy-MM-dd HH:mm:ss",
        "yyyy-MM-dd'T'HH:mm:ss",
        "yyyy-MM-dd HH:mm",
        "yyyy/MM/dd HH:mm:ss",
        "yyyy/MM/dd HH:mm"
    };

    // 统一输出格式
    private static final String OUTPUT_FORMAT = "yyyy年MM月dd日 HH:mm:ss";

    /**
     * 解析并格式化时间字符串
     */
    public String parseAndFormat(String input) {
        LocalDateTime dateTime = parse(input);
        return dateTime.format(DateTimeFormatter.ofPattern(OUTPUT_FORMAT));
    }

    /**
     * 解析时间字符串，自动识别格式
     */
    public LocalDateTime parse(String input) {
        if (input == null || input.trim().isEmpty()) {
            return null;
        }

        input = input.trim();

        // 尝试完整日期时间格式
        for (String pattern : DATETIME_PATTERNS) {
            try {
                return LocalDateTime.parse(input,
                    DateTimeFormatter.ofPattern(pattern));
            } catch (DateTimeParseException e) {
                // 继续尝试下一个格式
            }
        }

        // 尝试仅日期格式（补充时间为 00:00:00）
        for (String pattern : DATE_PATTERNS) {
            try {
                LocalDate date = LocalDate.parse(input,
                    DateTimeFormatter.ofPattern(pattern));
                return date.atStartOfDay();
            } catch (DateTimeParseException e) {
                // 继续尝试下一个格式
            }
        }

        // 尝试仅时间格式（补充日期为当天）
        for (String pattern : TIME_PATTERNS) {
            try {
                LocalTime time = LocalTime.parse(input,
                    DateTimeFormatter.ofPattern(pattern));
                return LocalDate.now().atTime(time);
            } catch (DateTimeParseException e) {
                // 继续尝试下一个格式
            }
        }

        throw new IllegalArgumentException(
            "无法解析时间格式: " + input);
    }
}
```

**Schema 校验**：

```java
// 类型校验扩展
public void validateType(String type, Object value) {
    switch (type) {
        case "string":
            // 字符串校验
            break;
        case "number":
            // 数字校验
            break;
        case "boolean":
            // 布尔校验
            break;
        case "datetime":
            // [2.1.0] 时间校验
            if (value instanceof String) {
                dateTimeParser.parse((String) value);  // 会抛出异常如果格式不正确
            } else {
                throw new ValidationException("datetime 类型的值必须是字符串");
            }
            break;
        // ...
    }
}
```

### 3.7.5 前端支持

**输入组件**：

| 场景 | 推荐组件 |
|------|----------|
| 仅日期 | 日期选择器（DatePicker） |
| 仅时间 | 时间选择器（TimePicker） |
| 日期+时间 | 日期时间选择器（DateTimePicker） |
| 自由输入 | 文本框 + 格式提示 |

**组件选择逻辑**：

前端可根据 `placeholder` 或额外的 `format` 属性判断显示哪种选择器：

```yaml
# 显示日期选择器
start_date:
  type: datetime
  placeholder: 请选择日期

# 显示时间选择器
alarm_time:
  type: datetime
  placeholder: 请选择时间

# 显示日期时间选择器
meeting_time:
  type: datetime
  placeholder: 请选择日期和时间
```

### 3.7.6 测试要点

| 测试场景 | 输入 | 预期输出 |
|----------|------|----------|
| ISO 日期 | `2026-02-28` | `2026年02月28日 00:00:00` |
| 斜杠日期 | `2026/02/28` | `2026年02月28日 00:00:00` |
| 紧凑日期 | `20260228` | `2026年02月28日 00:00:00` |
| 完整时间 | `14:30:00` | `当天年月日 14:30:00` |
| 简短时间 | `14:30` | `当天年月日 14:30:00` |
| 紧凑时间 | `143000` | `当天年月日 14:30:00` |
| ISO 日期时间 | `2026-02-28T14:30:00` | `2026年02月28日 14:30:00` |
| 空格日期时间 | `2026-02-28 14:30:00` | `2026年02月28日 14:30:00` |
| 无效格式 | `abc` | 报错 |
| 空值 | `null` | `null` 或报错（根据 required） |

---

## 3.8 resource 资源引用类型

**优先级**：P0（必须）

**需求描述**：新增 `resource` 基本类型，用于声明式地引用外部资源。系统负责 URI 格式校验，Tool 负责实际访问和处理。

### 3.8.1 支持的协议

| 协议 | 格式 | 说明 |
|------|------|------|
| `file://` | `file://<相对路径>` | 本地文件，路径相对于运行时限定的根目录 |
| `http://` | `http://<url>` | HTTP 网络资源 |
| `https://` | `https://<url>` | HTTPS 网络资源 |

### 3.8.2 file:// 路径解析

`file://` 协议使用**相对路径**，基于系统为当前技能限定的工作目录：

```
file://data/report.xlsx
       ↓
系统根目录 + /data/report.xlsx
       ↓
/var/aegis/workspace/skill_123/data/report.xlsx  （实际路径，对 DSL 不可见）
```

**设计原则**：
- DSL 层无法获知系统的真实文件路径
- 系统运行时负责将相对路径解析为实际位置
- 安全隔离：每个技能只能访问其限定目录内的文件

### 3.8.3 实现要点

**URI 解析器（伪代码）**：

```java
public class ResourceParser {

    // 支持的协议
    private static final Set<String> SUPPORTED_SCHEMES =
        Set.of("file", "http", "https");

    /**
     * 校验 resource 值的格式
     */
    public void validate(String uri) {
        if (uri == null || uri.trim().isEmpty()) {
            throw new ValidationException("resource 值不能为空");
        }

        URI parsed;
        try {
            parsed = new URI(uri);
        } catch (URISyntaxException e) {
            throw new ValidationException("无效的 URI 格式: " + uri);
        }

        String scheme = parsed.getScheme();
        if (scheme == null) {
            throw new ValidationException("缺少协议前缀: " + uri);
        }

        if (!SUPPORTED_SCHEMES.contains(scheme.toLowerCase())) {
            throw new ValidationException(
                "不支持的协议: " + scheme + "，支持: file, http, https");
        }

        // file:// 协议特殊校验
        if ("file".equalsIgnoreCase(scheme)) {
            validateFilePath(parsed.getPath());
        }
    }

    /**
     * 校验文件路径（防止路径穿越）
     */
    private void validateFilePath(String path) {
        if (path == null || path.isEmpty()) {
            throw new ValidationException("file:// 路径不能为空");
        }

        // 禁止绝对路径
        if (path.startsWith("/")) {
            throw new ValidationException(
                "file:// 必须使用相对路径，不能以 / 开头");
        }

        // 禁止路径穿越
        if (path.contains("..")) {
            throw new ValidationException(
                "file:// 路径不能包含 '..' 路径穿越");
        }
    }

    /**
     * 解析 file:// 为实际路径（运行时调用）
     */
    public Path resolveFilePath(String uri, Path workspaceRoot) {
        URI parsed = URI.create(uri);
        String relativePath = parsed.getPath();

        // 去除开头的 / （如果有）
        if (relativePath.startsWith("/")) {
            relativePath = relativePath.substring(1);
        }

        Path resolved = workspaceRoot.resolve(relativePath).normalize();

        // 安全检查：确保解析后的路径仍在 workspace 内
        if (!resolved.startsWith(workspaceRoot)) {
            throw new SecurityException("路径越界: " + uri);
        }

        return resolved;
    }
}
```

**Schema 校验扩展**：

```java
// 类型校验扩展
public void validateType(String type, Object value) {
    switch (type) {
        // ... 其他类型
        case "resource":
            // [2.1.0] 资源引用校验
            if (value instanceof String) {
                resourceParser.validate((String) value);
            } else {
                throw new ValidationException("resource 类型的值必须是字符串");
            }
            break;
    }
}
```

### 3.8.4 运行时处理

**系统职责**：

| 阶段 | 职责 |
|------|------|
| 解析时 | 校验 URI 格式、协议合法性 |
| 执行时 | 将 `file://` 相对路径解析为实际路径 |
| 传递给 Tool | 传递解析后的资源信息 |

**Tool 接收的数据结构**：

```java
public class ResolvedResource {
    private String originalUri;    // 原始 URI: "file://data/report.xlsx"
    private String scheme;         // 协议: "file"
    private String resolvedPath;   // 解析后路径: "/var/aegis/workspace/xxx/data/report.xlsx"
                                   // 对于 http[s]:// 则为原始 URL
}
```

### 3.8.5 前端支持

| 协议 | 推荐组件 |
|------|----------|
| `file://` | 文件选择器（限定目录内） |
| `http[s]://` | URL 输入框 + 格式校验 |

### 3.8.6 安全考虑

| 风险 | 缓解措施 |
|------|----------|
| 路径穿越攻击 | 禁止 `..`，校验解析后路径在 workspace 内 |
| 任意文件读取 | 限制 `file://` 只能访问限定目录 |
| SSRF（服务端请求伪造） | `http[s]://` 访问由 Tool 负责，可配置白名单 |

### 3.8.7 测试要点

| 测试场景 | 输入 | 预期结果 |
|----------|------|----------|
| 有效 file:// | `file://data/report.xlsx` | 校验通过 |
| 有效 https:// | `https://api.example.com/data` | 校验通过 |
| 缺少协议 | `data/report.xlsx` | 报错：缺少协议前缀 |
| 不支持的协议 | `ftp://server/file` | 报错：不支持的协议 |
| 路径穿越 | `file://../../../etc/passwd` | 报错：路径不能包含 .. |
| 绝对路径 | `file:///var/data/file` | 报错：必须使用相对路径 |
| 空值 | `null` 或 `""` | 报错（如 required）或通过（如非 required） |

---

## 3.9 能力语义系统

**优先级**：P0（必须）

**需求描述**：引入三层能力语义系统，用于统一描述字段的运算能力和业务语义，支持条件表达式校验、UI 自动生成和技能自动编排。

### 3.9.1 三层结构

| 层级 | 属性 | 必需 | 作用 |
|------|------|------|------|
| Type | `type` | 是 | 数据结构类型，决定基础校验 |
| Trait | `traits` | 否 | 运算能力描述，决定表达式合法性 |
| Semantic Role | `semantic_role` | 否 | 业务语义类别，用于自动编排 |

### 3.9.2 Trait 定义

**核心 Trait 列表：**

| Trait | 说明 | 支持的操作符 |
|-------|------|-------------|
| `comparable` | 可比较大小 | `>`, `<`, `>=`, `<=`, `between` |
| `scorable` | 可作为评价指标 | `top_k`, `max`, `min`, `sorting`, `normalization` |
| `equatable` | 可判断相等 | `==`, `!=`, `in`, `not_in` |
| `emptiable` | 可判断空值 | `is_empty`, `is_not_empty` |
| `countable` | 可统计数量 | `count`, `count > n`, `count == n` |
| `searchable` | 可模糊匹配 | `contains`, `starts_with`, `ends_with`, `like` |

**Type 与 Trait 合法性矩阵：**

| Type | comparable | scorable | equatable | emptiable | countable | searchable |
|------|:----------:|:--------:|:---------:|:---------:|:---------:|:----------:|
| `number` | ✓ | ✓ | ✓ | △ | | |
| `string` | | | ✓ | ✓ | | ✓ |
| `boolean` | | | ✓ | | | |
| `datetime` | ✓ | | ✓ | ✓ | | |
| `resource` | | | ✓ | ✓ | | |
| `array` | | | | ✓ | ✓ | |
| `object` | | | | ✓ | | |

> ✓ 合法，△ 不推荐但允许，空白 非法

### 3.9.3 Semantic Role 定义

**内置 Semantic Role（平台维护）：**

```java
public enum SemanticRole {
    // 数值类
    SCORE,          // 评分值
    CONFIDENCE,     // 置信度
    PROBABILITY,    // 概率值
    AMOUNT,         // 数量
    FINANCE,        // 金融数值

    // 文本类
    CONTENT,        // 主体内容
    SUMMARY,        // 摘要
    LABEL,          // 标签
    KEYWORD,        // 关键词
    EXPLANATION,    // 解释说明

    // 集合类
    DOCUMENT_LIST,  // 文档列表
    FILE_LIST,      // 文件列表
    RECORD_LIST     // 记录列表
}
```

**扩展策略**：
- 核心 Role 内置于平台
- 扩展 Role 通过注册制添加
- 禁止自由定义无限 Role

### 3.9.4 实现要点

**Schema 解析扩展**：

```java
public class FieldSchema {
    private String name;
    private String type;
    private String description;
    private boolean required;

    // [2.1.0] 能力语义
    private List<String> traits;      // 能力特征列表
    private String semanticRole;       // 语义角色

    // 嵌套结构
    private FieldSchema items;         // array 元素
    private Map<String, FieldSchema> properties;  // object 子字段
}
```

**Trait 合法性校验**：

```java
public class TraitValidator {

    // Type -> 合法 Trait 集合
    private static final Map<String, Set<String>> VALID_TRAITS = Map.of(
        "number", Set.of("comparable", "scorable", "equatable", "emptiable"),
        "string", Set.of("equatable", "emptiable", "searchable"),
        "boolean", Set.of("equatable"),
        "datetime", Set.of("comparable", "equatable", "emptiable"),
        "resource", Set.of("equatable", "emptiable"),
        "array", Set.of("countable", "emptiable"),
        "object", Set.of("emptiable")
    );

    /**
     * 校验字段的 traits 是否合法
     */
    public void validate(FieldSchema field) {
        if (field.getTraits() == null || field.getTraits().isEmpty()) {
            return;  // traits 是可选的
        }

        Set<String> validTraits = VALID_TRAITS.get(field.getType());

        for (String trait : field.getTraits()) {
            if (!validTraits.contains(trait)) {
                throw new ValidationException(
                    String.format("类型 '%s' 不支持 trait '%s'，允许的 traits: %s",
                        field.getType(), trait, validTraits));
            }
        }
    }
}
```

**表达式操作符校验**：

```java
public class ExpressionValidator {

    // Trait -> 支持的操作符
    private static final Map<String, Set<String>> TRAIT_OPERATORS = Map.of(
        "comparable", Set.of(">", "<", ">=", "<=", "between"),
        "scorable", Set.of("top_k", "max", "min", "sorting"),
        "equatable", Set.of("==", "!=", "in", "not_in"),
        "emptiable", Set.of("is_empty", "is_not_empty"),
        "countable", Set.of("count"),
        "searchable", Set.of("contains", "starts_with", "ends_with", "like")
    );

    /**
     * 校验表达式中的操作符是否被字段的 traits 支持
     */
    public void validateOperator(String fieldName, String operator,
                                  List<String> traits) {
        for (String trait : traits) {
            Set<String> supportedOps = TRAIT_OPERATORS.get(trait);
            if (supportedOps != null && supportedOps.contains(operator)) {
                return;  // 找到支持该操作符的 trait
            }
        }

        throw new ValidationException(
            String.format("字段 '%s' 不支持操作符 '%s'，请检查 traits 定义",
                fieldName, operator));
    }
}
```

### 3.9.5 平台能力获得

| 能力 | 依赖 | 说明 |
|------|------|------|
| 条件表达式校验 | Type + Trait | 编译期校验操作符合法性 |
| UI 自动生成 | Type + Trait | 根据 Trait 决定 UI 组件和操作选项 |
| 技能自动编排 | Trait + Semantic Role | 根据能力和语义匹配技能 |

### 3.9.6 测试要点

| 测试场景 | 预期结果 |
|----------|----------|
| number + comparable | 允许 `>`, `<` 等操作 |
| string + comparable | 解析报错：类型不支持该 trait |
| number 字段使用 `contains` | 表达式报错：不支持该操作符 |
| array + countable | 允许 `count` 操作 |
| 未声明 traits 的字段 | 正常解析，不做 trait 相关校验 |
| 无效的 semantic_role | 报错或警告（取决于实现策略） |

---

# 第四部分：技术实现建议

## 4. 技术实现建议

### 4.1 解析器改动

```
SkillParser
├── parseHeader()          // 解析 skill id, version, type
│   └── parseType()        // [2.0.0] 解析 type 字段
├── validateStructure()    // [2.0.0] 根据 type 校验章节
├── parseInputSchema()
│   └── validateTypeCompleteness()  // [1.2.0] 类型完整性校验
├── parseOutputSchema()    // 仅 AtomicSkill
│   └── validateTypeCompleteness()  // [1.2.0] 类型完整性校验
├── parseSteps()
└── parseUI()              // 仅 PresentationSkill
```

### 4.2 模板引擎改动（1.2.0 + 2.1.0）

```
TemplateEngine
├── parseExpression()
│   └── parseProjection()       // [1.2.0] 解析 {{arr[*].field}} 投影语法
├── executeLoop()               // [1.2.0] 增强循环渲染
│   ├── createLoopScope()       // 创建循环作用域
│   ├── setIndexVariable()      // 设置 _index = 当前索引
│   ├── setElementVariable()    // 设置 _ = 当前元素
│   ├── expandObjectFields()    // 展开结构化对象字段
│   └── handleBreak()           // [2.1.0] 处理 {{break}} 控制流
├── evaluateCondition()         // [1.2.0] 统一 when 条件求值
│   ├── parseStepLevelWhen()    // 步骤级 when（YAML 格式）
│   └── parseTemplateLevelWhen() // 模板级 {{#when}}
├── parseBreak()                // [2.1.0] 识别 {{break}} 指令
└── renderTemplate()
```

**executeLoop 实现要点**：

```java
// 循环执行核心逻辑
for (int i = 0; i < array.size(); i++) {
    Context loopCtx = ctx.createChildScope();  // 隔离作用域

    loopCtx.put("_index", i);                  // [1.2.0] 循环索引
    loopCtx.put("_", array.get(i));            // 当前元素引用

    if (array.get(i) instanceof Map) {         // 结构化数组
        ((Map<?, ?>) array.get(i)).forEach((k, v) ->
            loopCtx.put(k.toString(), v));     // 展开字段
    }

    result.append(renderTemplate(body, loopCtx));
}
```

**投影语法实现要点**：

```java
// 伪代码示例
Object evaluateProjection(String expr, Context ctx) {
    // 解析 arr[*].field 模式
    Matcher m = Pattern.compile("(\\w+)\\[\\*\\]\\.(.+)").matcher(expr);
    if (m.matches()) {
        String arrayName = m.group(1);
        String fieldPath = m.group(2);
        List<?> array = (List<?>) ctx.get(arrayName);
        return array.stream()
            .map(item -> getField(item, fieldPath))
            .collect(Collectors.toList());
    }
    // 其他表达式处理...
}
```

### 4.3 执行器改动

```
SkillExecutor
├── executeAtomicSkill()
│   ├── executeSteps()
│   ├── validateOutput()
│   └── return structuredOutput
│
└── executePresentationSkill()
    ├── executeSteps()      // 可选
    ├── resolveMapping()    // 解析 UI mapping
    └── renderUI()          // 返回 UI 渲染结果
```

### 4.4 内置工具实现（1.2.0）

```
BuiltinTools
├── ArrayZip        // 多数组组装
├── ArrayFlatten    // 数组展平
├── ArrayUnique     // 数组去重
├── ArraySort       // 数组排序
└── ArrayFilter     // 数组过滤
```

**array.zip 实现要点**：

```java
// 伪代码示例
public class ArrayZip implements BuiltinTool {
    public void execute(Map<String, Object> input, ToolOutputContext output) {
        Map<String, List<?>> arrays = (Map<String, List<?>>) input.get("arrays");

        // 找最短数组长度
        int minLength = arrays.values().stream()
            .mapToInt(List::size)
            .min()
            .orElse(0);

        // 按索引组装
        List<Map<String, Object>> result = new ArrayList<>();
        for (int i = 0; i < minLength; i++) {
            Map<String, Object> item = new HashMap<>();
            for (Map.Entry<String, List<?>> entry : arrays.entrySet()) {
                item.put(entry.getKey(), entry.getValue().get(i));
            }
            result.add(item);
        }

        output.put("result", result);
    }
}
```

### 4.5 前端渲染器

```
UIRenderer
├── renderText()
├── renderMarkdown()
├── renderTable()
├── renderList()
├── renderCard()
├── renderChart()
├── renderFile()
├── renderForm()
└── renderMixed()
    └── recursively render layout items
```

---

## 5. 测试要点（2.0.0）

### 5.1 单元测试

| 测试场景 | 预期结果 |
|----------|----------|
| 解析 `**type**: AtomicSkill` | type = AtomicSkill |
| 解析 `**type**: PresentationSkill` | type = PresentationSkill |
| 未声明 type | type = AtomicSkill（默认） |
| 非法 type 值 | 报错 |
| AtomicSkill 缺少 output_schema | 报错 |
| AtomicSkill 包含 ui | 报错 |
| PresentationSkill 缺少 ui | 报错 |
| PresentationSkill 包含 output_schema | 报错 |
| PresentationSkill 无 steps | 正常执行 |
| PresentationSkill 有 steps | steps 输出可在 ui mapping 引用 |

### 5.2 集成测试

| 测试场景 | 预期结果 |
|----------|----------|
| AtomicSkill → PresentationSkill 串联 | 数据正确传递，UI 正常渲染 |
| Mixed 布局渲染 | 所有子组件按顺序正确渲染 |
| PresentationSkill 使用 prompt step | LLM 生成内容可在 UI 展示 |

---

# 第五部分：实施计划

## 6. 文件变更清单

本次规范升级涉及以下文件变更：

| 文件 | 变更类型 | 说明 |
|------|----------|------|
| `README.md` | 修改 | 重构为通用规范，移除类型专属内容 |
| `AtomicSkill.md` | 新增 | AtomicSkill 专属规范：output_schema、审计日志、示例 |
| `PresentationSkill.md` | 新增 | PresentationSkill 专属规范：UI 展示语义、示例 |

**README.md 主要变更**：
- 版本号从 1.2.0 升至 2.0.0
- 新增技能分类章节（1.1）
- 新增 AtomicSkill/PresentationSkill 文件骨架（1.2, 1.3）
- 新增 type 字段说明（1.5）
- 移除 output_schema 章节（移至 AtomicSkill.md）
- 移除 UI 展示语义章节（移至 PresentationSkill.md）
- 移除审计日志章节（移至 AtomicSkill.md）
- 移除所有完整示例（分别移至对应文件）

---

## 7. 实施建议

### 7.1 分阶段实施

**第一阶段：1.2.0 功能（P0）**：
1. 类型完整性规则校验
2. 数组字段投影 `{{arr[*].field}}`
3. 循环索引 `{{_index}}`
4. 统一 when 条件语法

**第二阶段：1.2.0 功能（P1）**：
1. 内置数组工具（array.zip、array.flatten 等）

**第三阶段：2.0.0 功能（P0）**：
1. type 字段解析
2. 技能结构校验规则
3. 数据模型更新

**第四阶段：2.0.0 功能（P1）**：
1. PresentationSkill UI 渲染
2. PresentationSkill steps 支持
3. mixed 布局

**第五阶段：2.0.0 功能（P2）**：
1. 审计日志限制
2. chart/form 等复杂组件
3. 迁移工具

**第六阶段：2.1.0 功能（P0）**：
1. 循环控制指令 `{{break}}`
2. `{{break}}` 与 `{{#when}}` 条件嵌套
3. 嵌套循环 break 行为
4. `datetime` 基本类型解析和格式化
5. 多种日期时间输入格式支持
6. 统一输出格式 `yyyy年MM月dd日 HH:mm:ss`
7. `resource` 资源引用类型
8. `file://` 相对路径解析和安全校验
9. `http[s]://` URL 格式校验
10. 能力语义系统：`traits` 属性解析
11. 能力语义系统：`semantic_role` 属性解析
12. Type-Trait 合法性校验
13. 表达式操作符与 Trait 校验

### 7.2 风险点

| 风险 | 影响 | 缓解措施 |
|------|------|----------|
| 类型完整性规则破坏现有技能 | 现有未完整定义的 schema 解析失败 | 提供迁移期警告而非报错，给予过渡时间 |
| 现有技能不兼容 2.0.0 | 升级后解析失败 | 默认 type=AtomicSkill，保持兼容 |
| UI 渲染复杂度 | 前端工作量大 | 分优先级实现 display 类型 |
| 技能拆分工作量 | 需重构现有含 UI 的技能 | 提供迁移指南和工具 |

---

## 附录 A：Git 变更记录

```
# 2026-02-27 提交记录

3a360af feat: 升级规范至 1.2.0 版本
    - 类型完整性规则
    - 数组字段投影 {{arr[*].field}}
    - 循环索引 {{_index}}
    - 内置数组工具
    - 统一 when 条件语法

# 待提交变更（2.0.0 → 2.1.0）
- README.md (modified): 版本升至 2.1.0，新增 {{break}}、datetime、resource、能力语义系统
- AtomicSkill.md (modified): 版本升至 2.1.0，output_schema 新增 traits 和 semantic_role
- PresentationSkill.md (modified): 版本升至 2.1.0
- IMPLEMENTATION_REQUIREMENTS.md (modified): 新增 2.1.0 完整实现要求
```

---

## 附录 B：相关文档

- [README.md](./README.md) - 通用规范
- [AtomicSkill.md](./AtomicSkill.md) - AtomicSkill 规范
- [PresentationSkill.md](./PresentationSkill.md) - PresentationSkill 规范

---

# 第六部分：2.1.0 → 2.2.0 变更

## 4. 内置函数系统

### 4.1 概述

**优先级**：P0（必须）

**需求描述**：将原"内置工具"重构为"内置函数"系统。函数可以在模板表达式中直接调用，无需通过 step 配置，提供更简洁的数据处理能力。

**核心变更**：

| 原设计（内置工具） | 新设计（内置函数） |
|------------------|------------------|
| 需要定义完整 step | 在表达式中直接调用 |
| `array.flatten` 点号分隔 | `array::flatten()` 双冒号语法 |
| 输出写入上下文 key | 返回值可直接赋值 |
| 不支持链式调用 | 支持函数嵌套链式调用 |

### 4.2 函数调用语法

**语法格式**：

```
namespace::function(arg1, arg2, ..., [ignore: true])
```

**语法组成**：

| 组成部分 | 说明 | 示例 |
|----------|------|------|
| `namespace` | 函数所属命名空间 | `array` |
| `::` | 命名空间分隔符（双冒号） | — |
| `function` | 函数名（小写 + 下划线） | `flatten`, `unique` |
| `(...)` | 参数列表 | `({{data}}, 1)` |
| `ignore` | 可选最后参数，异常处理 | `ignore: true` |

**解析要点**：

```java
public class FunctionParser {

    // 函数调用正则：namespace::function(args)
    private static final Pattern FUNCTION_PATTERN =
        Pattern.compile("(\\w+)::(\\w+)\\((.*)\\)");

    /**
     * 解析函数调用表达式
     */
    public FunctionCall parse(String expr) {
        Matcher m = FUNCTION_PATTERN.matcher(expr);
        if (!m.matches()) {
            return null;  // 不是函数调用
        }

        String namespace = m.group(1);
        String functionName = m.group(2);
        String argsStr = m.group(3);

        // 解析参数（支持嵌套函数调用）
        List<Object> args = parseArguments(argsStr);

        // 提取 ignore 参数
        boolean ignore = extractIgnoreParam(args);

        return new FunctionCall(namespace, functionName, args, ignore);
    }

    /**
     * 解析参数列表，支持嵌套函数
     */
    private List<Object> parseArguments(String argsStr) {
        List<Object> args = new ArrayList<>();
        // 处理嵌套括号、字符串、变量引用等
        // 递归解析嵌套函数调用
        // ...
        return args;
    }
}
```

### 4.3 可选参数与默认值

**需求描述**：函数支持可选参数，未传入时使用默认值，简化调用方式。

**参数省略规则**：

| 规则 | 说明 |
|------|------|
| 从右向左省略 | 只能省略末尾的可选参数 |
| 不能跳过参数 | 如需指定第 3 个参数，必须传入第 1、2 个 |
| ignore 始终最后 | `ignore: true` 必须是最后一个参数 |

**调用示例**：

```yaml
# 完整调用
result: array::flatten({{data}}, 2)

# 省略 depth，使用默认值 1
result: array::flatten({{data}})

# 排序：省略 key 和 order
sorted: array::sort({{numbers}})

# 排序：指定 key，省略 order（默认 asc）
sorted: array::sort({{items}}, "score")
```

**实现要点**：

```java
public class ParameterDef {
    private String name;
    private String type;
    private boolean required;      // 是否必需
    private Object defaultValue;   // 可选参数的默认值
}

public class FunctionExecutor {

    /**
     * 填充可选参数的默认值
     */
    private List<Object> fillDefaults(FunctionSignature sig, List<Object> providedArgs) {
        List<Object> result = new ArrayList<>(providedArgs);
        List<ParameterDef> params = sig.getParameters();

        // 从已提供参数数量开始，填充剩余可选参数的默认值
        for (int i = providedArgs.size(); i < params.size(); i++) {
            ParameterDef param = params.get(i);
            if (param.isRequired()) {
                throw new MissingRequiredParameterException(
                    sig.getName(), param.getName());
            }
            result.add(param.getDefaultValue());
        }
        return result;
    }
}
```

**各函数默认值定义**：

| 函数 | 参数 | 默认值 |
|------|------|--------|
| `array::flatten` | `depth` | `1` |
| `array::unique` | `key` | `null`（按元素本身去重） |
| `array::sort` | `key` | `null`（按元素本身排序） |
| `array::sort` | `order` | `"asc"` |
| `array::filter` | — | 无可选参数（condition 必需） |
| 所有函数 | `ignore` | `false` |

### 4.4 链式调用实现

**语法示例**：

```yaml
# 嵌套调用：先展平，再去重
result: array::unique(array::flatten({{nested_data}}, 1))

# 多层嵌套：展平 → 过滤 → 排序
top_items: array::sort(
    array::filter(
        array::flatten({{data}}, 1),
        "score > 60"
    ),
    "score",
    "desc"
)
```

**实现要点**：

```java
public class FunctionExecutor {

    /**
     * 执行函数调用（支持嵌套）
     */
    public Object execute(FunctionCall call, Context ctx) {
        // 1. 先递归执行参数中的嵌套函数
        List<Object> resolvedArgs = new ArrayList<>();
        for (Object arg : call.getArgs()) {
            if (arg instanceof FunctionCall) {
                // 递归执行嵌套函数
                Object result = execute((FunctionCall) arg, ctx);
                resolvedArgs.add(result);
            } else if (arg instanceof String && isVariableRef((String) arg)) {
                // 解析变量引用 {{var}}
                resolvedArgs.add(resolveVariable((String) arg, ctx));
            } else {
                resolvedArgs.add(arg);
            }
        }

        // 2. 获取并执行目标函数
        BuiltinFunction fn = getFunctionByName(
            call.getNamespace(), call.getFunctionName());

        try {
            return fn.invoke(resolvedArgs);
        } catch (Exception e) {
            if (call.isIgnore()) {
                return null;  // ignore: true 时返回 null
            }
            throw new FunctionExecutionException(
                call.getNamespace() + "::" + call.getFunctionName(), e);
        }
    }
}
```

### 4.5 ignore 参数实现

**语法**：

```yaml
# 默认：失败抛异常
result: array::flatten({{data}}, 1)

# 显式忽略：失败返回 null
safe_result: array::flatten({{data}}, 1, ignore: true)
```

**解析规则**：

```java
/**
 * 从参数列表中提取 ignore 参数，该参数在函数调用时可以忽略配置，忽略时采用ignore: false
 * ignore 必须是最后一个参数，格式为 "ignore: true" 或 "ignore: false"
 */
private boolean extractIgnoreParam(List<Object> args) {
    if (args.isEmpty()) {
        return false;
    }

    Object lastArg = args.get(args.size() - 1);
    if (lastArg instanceof String) {
        String s = ((String) lastArg).trim();
        if (s.startsWith("ignore:")) {
            args.remove(args.size() - 1);  // 从参数列表移除
            return s.contains("true");
        }
    }
    return false;
}
```

**执行行为**：

| ignore 值 | 执行成功 | 执行失败 |
|-----------|----------|----------|
| `false`（默认） | 返回结果 | 抛出 `FunctionExecutionException`，终止 Skill |
| `true` | 返回结果 | 返回 `null`，继续执行 |

### 4.6 函数注册与命名空间

**内置函数注册**：

```java
public class BuiltinFunctionRegistry {

    private final Map<String, Map<String, BuiltinFunction>> functions = new HashMap<>();

    public BuiltinFunctionRegistry() {
        // 注册 array 命名空间
        registerNamespace("array", Map.of(
            "zip", new ArrayZipFunction(),
            "flatten", new ArrayFlattenFunction(),
            "unique", new ArrayUniqueFunction(),
            "sort", new ArraySortFunction(),
            "filter", new ArrayFilterFunction()
        ));

        // 未来可扩展其他命名空间
        // registerNamespace("string", ...);
        // registerNamespace("math", ...);
    }

    public BuiltinFunction get(String namespace, String name) {
        Map<String, BuiltinFunction> ns = functions.get(namespace);
        if (ns == null) {
            throw new UnknownNamespaceException(namespace);
        }
        BuiltinFunction fn = ns.get(name);
        if (fn == null) {
            throw new UnknownFunctionException(namespace, name);
        }
        return fn;
    }
}
```

**函数接口定义**：

```java
public interface BuiltinFunction {

    /**
     * 函数签名（用于文档和类型检查）
     */
    FunctionSignature getSignature();

    /**
     * 执行函数
     */
    Object invoke(List<Object> args) throws FunctionExecutionException;
}

public class FunctionSignature {
    private String namespace;
    private String name;
    private List<ParameterDef> parameters;
    private String returnType;

    // 示例：array::flatten(array, depth?, ignore?) → array
}
```

### 4.7 各函数实现规范

#### array::zip

**函数签名**：

```
array::zip(arrays: object, [ignore: boolean]) → array
```

**实现**：

```java
public class ArrayZipFunction implements BuiltinFunction {

    @Override
    public Object invoke(List<Object> args) {
        Map<String, List<?>> arrays = (Map<String, List<?>>) args.get(0);

        // 找最短数组长度
        int minLen = arrays.values().stream()
            .mapToInt(List::size)
            .min()
            .orElse(0);

        // 按索引组装
        List<Map<String, Object>> result = new ArrayList<>();
        for (int i = 0; i < minLen; i++) {
            Map<String, Object> item = new HashMap<>();
            for (Map.Entry<String, List<?>> entry : arrays.entrySet()) {
                item.put(entry.getKey(), entry.getValue().get(i));
            }
            result.add(item);
        }
        return result;
    }
}
```

#### array::flatten

**函数签名**：

```
array::flatten(array: array, depth?: number, [ignore: boolean]) → array
```

**实现**：

```java
public class ArrayFlattenFunction implements BuiltinFunction {

    @Override
    public Object invoke(List<Object> args) {
        List<?> array = (List<?>) args.get(0);
        int depth = args.size() > 1 ? ((Number) args.get(1)).intValue() : 1;

        return flatten(array, depth);
    }

    private List<Object> flatten(List<?> array, int depth) {
        List<Object> result = new ArrayList<>();
        for (Object item : array) {
            if (item instanceof List && depth > 0) {
                result.addAll(flatten((List<?>) item, depth - 1));
            } else {
                result.add(item);
            }
        }
        return result;
    }
}
```

#### array::unique

**函数签名**：

```
array::unique(array: array, key?: string, [ignore: boolean]) → array
```

#### array::sort

**函数签名**：

```
array::sort(array: array, key?: string, order?: string, [ignore: boolean]) → array
```

#### array::filter

**函数签名**：

```
array::filter(array: array, condition: string, [ignore: boolean]) → array
```

---

## 5. 三层异常处理机制

### 5.1 概述

**优先级**：P0（必须）

**需求描述**：引入三层异常处理机制，分别控制技能级、步骤级、函数级的异常处理行为。

| 层级 | 属性/参数 | 作用范围 | 默认值 |
|------|-----------|----------|--------|
| 技能级 | `**ignore**: true` | 整个技能执行 | `false` |
| 步骤级 | Step 的 `ignore` 属性 | 单个步骤执行 | `false` |
| 函数级 | 函数的 `ignore` 参数 | 单次函数调用 | `false` |

---

## 5.2 技能级 ignore 属性

### 5.2.1 语法定义

```markdown
# skill: optional_enrichment

**version**: 2.2.0
**ignore**: true
**type**: AtomicSkill
```

### 5.2.2 行为定义

| ignore 值 | 技能执行成功 | 技能执行失败 |
|-----------|-------------|-------------|
| `false`（默认） | 正常返回输出 | 抛出异常，终止整个执行流程 |
| `true` | 正常返回输出 | 返回 `null`，允许上层编排继续执行 |

### 5.2.3 实现要点

**Skill 数据模型扩展**：

```java
public class Skill {
    private String id;
    private String version;
    private boolean ignore;       // [2.2.0] 技能级异常处理
    private SkillType type;
    private String description;
    private InputSchema inputSchema;
    private OutputSchema outputSchema;
    private List<Step> steps;
    // ...
}
```

**解析器扩展**：

```java
public class SkillParser {

    /**
     * 解析 **ignore**: true/false
     */
    private boolean parseIgnore(String content) {
        Pattern p = Pattern.compile("\\*\\*ignore\\*\\*:\\s*(true|false)",
            Pattern.CASE_INSENSITIVE);
        Matcher m = p.matcher(content);
        if (m.find()) {
            return "true".equalsIgnoreCase(m.group(1));
        }
        return false;  // 默认 false
    }
}
```

**执行器扩展**：

```java
public class SkillExecutor {

    public Object executeSkill(Skill skill, Map<String, Object> input) {
        try {
            // 正常执行技能
            return doExecute(skill, input);
        } catch (SkillExecutionException e) {
            if (skill.isIgnore()) {
                // [2.2.0] ignore: true 时返回 null
                log.warn("Skill '{}' failed but ignored: {}",
                    skill.getId(), e.getMessage());
                return null;
            }
            throw e;  // 默认：重新抛出异常
        }
    }
}
```

### 5.2.4 适用场景

| 场景 | 说明 |
|------|------|
| 可选数据增强 | 附加信息获取失败不影响主流程 |
| 降级处理 | 上层编排根据 `null` 做备用逻辑 |
| 批量编排 | 允许部分技能失败，不中断整体 |

---

## 5.3 步骤级 ignore 属性

### 5.3.1 语法定义

**Markdown 行内格式**：

```markdown
**type**: tool  **ignore**: true
**tool**: external_api
```

**YAML 配置块格式**：

```yaml
ignore: true
input:
  query: "{{search_term}}"
```

### 5.3.2 行为定义

| ignore 值 | 步骤执行成功 | 步骤执行失败 |
|-----------|-------------|-------------|
| `false`（默认） | 正常输出 | 抛出异常，终止 Skill 执行 |
| `true` | 正常输出 | 输出为 `null`，继续执行后续步骤 |

### 5.3.3 适用的 Step 类型

| Step 类型 | 支持 ignore | 典型失败场景 |
|-----------|:-----------:|-------------|
| `tool` | ✓ | 工具超时、网络错误、API 异常 |
| `prompt` | ✓ | LLM 超时、API 限流、内容审核拦截 |
| `template` | — | 纯模板渲染，几乎不会失败 |
| `await` | — | 等待用户输入，不涉及失败 |

### 5.3.4 实现要点

**Step 解析扩展**：

```java
public class Step {
    private String name;
    private StepType type;
    private String tool;          // for tool step
    private String varName;       // for prompt/template
    private WhenCondition when;
    private boolean ignore;       // [2.2.0] 新增

    // ...
}

public class StepParser {

    /**
     * 解析 **ignore**: true/false
     */
    private boolean parseIgnore(String line) {
        Pattern p = Pattern.compile("\\*\\*ignore\\*\\*:\\s*(true|false)");
        Matcher m = p.matcher(line);
        if (m.find()) {
            return "true".equalsIgnoreCase(m.group(1));
        }
        return false;  // 默认 false
    }
}
```

**Step 执行扩展**：

```java
public class StepExecutor {

    public void executeStep(Step step, Context ctx) {
        try {
            switch (step.getType()) {
                case TOOL:
                    executeTool(step, ctx);
                    break;
                case PROMPT:
                    executePrompt(step, ctx);
                    break;
                // ...
            }
        } catch (StepExecutionException e) {
            if (step.isIgnore()) {
                // [2.2.0] ignore: true 时，输出 null 并继续
                log.warn("Step '{}' failed but ignored: {}",
                    step.getName(), e.getMessage());

                // 将输出设为 null
                if (step.getType() == StepType.TOOL) {
                    // tool 的 output_schema 中所有字段设为 null
                    for (String key : step.getOutputSchema().keySet()) {
                        ctx.put(key, null);
                    }
                } else if (step.getVarName() != null) {
                    ctx.put(step.getVarName(), null);
                }
            } else {
                // 默认行为：重新抛出异常
                throw e;
            }
        }
    }
}
```

---

## 5.4 三层 ignore 的关系

| 层级 | 属性位置 | 作用范围 | 失败时行为 |
|------|----------|----------|------------|
| 技能级 | `**ignore**:` 顶层字段 | 整个技能 | 返回 `null` |
| 步骤级 | Step 的 `ignore` 属性 | 单个步骤 | 该步骤输出为 `null` |
| 函数级 | 函数最后参数 `ignore: true` | 单次函数调用 | 函数返回 `null` |

**优先级说明**：

1. 函数级 `ignore` 在函数调用时独立生效
2. 步骤级 `ignore` 在步骤执行时独立生效（包括步骤内未被函数 ignore 捕获的异常）
3. 技能级 `ignore` 在整个技能执行失败时生效（包括未被步骤 ignore 捕获的异常）

**示例场景**：

```yaml
# 技能级 ignore: true
# 若某步骤失败且该步骤 ignore: false → 技能失败 → 技能级 ignore 生效 → 返回 null

# 步骤级 ignore: true
# 若步骤内函数失败且函数 ignore: false → 函数抛异常 → 步骤级 ignore 生效 → 步骤输出 null

# 函数级 ignore: true
# 函数执行失败 → 函数返回 null → 步骤继续执行
```

---

## 6. 2.1.0 → 2.2.0 测试要点

### 6.1 函数调用语法

| 测试场景 | 预期结果 |
|----------|----------|
| `array::flatten({{data}}, 1)` | 正确展平数组 |
| `array::unknown({{data}})` | 报错：未知函数 |
| `unknown::flatten({{data}})` | 报错：未知命名空间 |
| `array.flatten({{data}})` | 报错：语法错误（点号不是函数调用） |

### 6.2 链式调用

| 测试场景 | 预期结果 |
|----------|----------|
| `array::unique(array::flatten({{data}}, 1))` | 先展平后去重 |
| 三层嵌套调用 | 从内到外依次执行 |
| 嵌套函数某层失败（无 ignore） | 整体失败 |
| 嵌套函数某层失败（有 ignore） | 该层返回 null，外层继续 |

### 6.3 ignore 参数

| 测试场景 | 预期结果 |
|----------|----------|
| 无 ignore 参数，执行成功 | 返回结果 |
| 无 ignore 参数，执行失败 | 抛出异常 |
| `ignore: true`，执行成功 | 返回结果 |
| `ignore: true`，执行失败 | 返回 null |
| `ignore: false`，执行失败 | 抛出异常 |

### 6.4 步骤级 ignore 属性

| 测试场景 | 预期结果 |
|----------|----------|
| tool step，无 ignore，成功 | 正常输出 |
| tool step，无 ignore，失败 | 终止 Skill |
| tool step，ignore: true，失败 | 输出 null，继续执行 |
| prompt step，ignore: true，失败 | varName 为 null，继续执行 |
| 后续步骤引用 null 输出 | 可通过 {{#when}} 判断 |

### 6.5 技能级 ignore 属性

| 测试场景 | 预期结果 |
|----------|----------|
| 技能无 ignore，执行成功 | 正常返回输出 |
| 技能无 ignore，执行失败 | 抛出异常，终止执行流程 |
| 技能 ignore: true，执行成功 | 正常返回输出 |
| 技能 ignore: true，执行失败 | 返回 null，上层继续 |
| 技能 ignore: true + 步骤 ignore: true | 步骤失败返回 null，技能继续；技能最终失败返回 null |

### 6.6 向后兼容性

| 测试场景 | 预期结果 |
|----------|----------|
| 旧版 `array.zip` 工具调用 | 【需决策】报错或兼容 |
| 不使用函数的技能 | 正常执行 |
| 不使用 ignore 的技能 | 正常执行，默认 ignore=false |

---

## 7. 实施建议（2.2.0）

### 7.1 分阶段实施

**阶段一（P0）**：
1. 函数调用语法解析器
2. 函数注册与执行框架
3. 5 个 array 函数实现
4. 函数级 ignore 参数支持

**阶段二（P0）**：
1. 链式调用支持
2. 步骤级 ignore 属性解析
3. 步骤失败处理扩展
4. 技能级 ignore 属性解析
5. 技能失败处理扩展

**阶段三（P1）**：
1. 函数类型检查
2. 错误消息优化
3. 性能优化

### 7.2 向后兼容策略

**选项 A：完全迁移**
- 移除旧的 `array.zip` 等内置工具支持
- 强制使用新函数语法
- 清晰但有迁移成本

**选项 B：兼容期**
- 保留旧语法支持，产生 deprecation 警告
- 给予 1-2 个版本的过渡期
- 平滑迁移但维护成本高

**建议**：选项 A，因为 DSL 刚投入使用，尽早统一语法。

### 7.3 风险点

| 风险 | 影响 | 缓解措施 |
|------|------|----------|
| 函数嵌套解析复杂 | 解析器实现困难 | 使用递归下降解析器，充分测试 |
| ignore 滥用 | 掩盖真实错误 | 文档强调使用场景，建议配合日志 |
| 性能影响 | 复杂链式调用可能较慢 | 缓存函数查找，优化参数解析 |

---

# 第六部分：2.2.0 → 2.3.0 变更（CognitiveSkill）

## 8. CognitiveSkill 概述

**优先级**：P0（必须）

**需求描述**：新增 CognitiveSkill 技能类型，用于复杂决策和动态编排场景。与 AtomicSkill/PresentationSkill 不同，CognitiveSkill 使用 `internal_flow` 替代 `steps`，通过决策节点实现运行时的条件判断和能力匹配。

**核心特性**：
- 内部控制流（internal_flow）：有序执行的决策节点序列
- 四种节点类型：decision（条件分支）、select（能力匹配）、assign（变量赋值）、merge（结果合并）
- 动态能力匹配：select 节点通过 capability 标签动态选择 AtomicSkill
- fallback 机制：最多两层嵌套的异常处理

---

## 9. internal_flow 解析器

**优先级**：P0（必须）

### 9.1 数据结构

```java
public class CognitiveSkill extends BaseSkill {
    private List<FlowNode> internalFlow;  // 内部控制流节点序列
}

public abstract class FlowNode {
    private String name;       // 节点名称
    private String type;       // 节点类型：decision/select/assign/merge
    private FlowNode fallback; // 可选，fallback 节点（最多两层）
}

public class DecisionNode extends FlowNode {
    private List<Branch> branches;     // 条件分支列表
    private String defaultTarget;      // 默认跳转目标（可选）
}

public class Branch {
    private String condition;  // 条件表达式
    private String target;     // 目标节点名
}

public class SelectNode extends FlowNode {
    private String capability;         // 能力标签
    private Map<String, Object> input; // 输入参数映射
    private String outputVar;          // 输出变量名
}

public class AssignNode extends FlowNode {
    private Map<String, String> assignments; // 变量赋值映射 key -> expression
}

public class MergeNode extends FlowNode {
    private List<SourceMapping> sources; // 合并来源列表
    private String outputVar;            // 输出变量名
}

public class SourceMapping {
    private String var;   // 变量名
    private String field; // 可选，字段名
    private String as;    // 可选，别名
}
```

### 9.2 解析流程

```
internal_flow YAML
       ↓
   YAML 解析
       ↓
   节点类型识别
       ↓
   节点对象构建
       ↓
   fallback 递归解析
       ↓
   节点引用验证
       ↓
   CognitiveSkill 对象
```

### 9.3 校验规则

| 校验项 | 说明 | 错误消息 |
|--------|------|----------|
| 节点名唯一性 | 每个节点名在 flow 内唯一 | `节点名 '{name}' 重复` |
| 节点类型合法性 | 必须是 decision/select/assign/merge | `未知节点类型: {type}` |
| 分支目标存在性 | decision 的 target 必须指向已定义节点 | `分支目标 '{target}' 不存在` |
| fallback 深度 | fallback 最多两层嵌套 | `fallback 嵌套超过最大深度(2)` |
| capability 非空 | select 节点必须定义 capability | `select 节点缺少 capability 定义` |
| sources 非空 | merge 节点必须定义 sources | `merge 节点缺少 sources 定义` |

---

## 10. 节点执行器

**优先级**：P0（必须）

### 10.1 decision 节点执行

```java
public class DecisionNodeExecutor implements FlowNodeExecutor<DecisionNode> {

    @Override
    public String execute(DecisionNode node, ExecutionContext ctx) {
        for (Branch branch : node.getBranches()) {
            boolean result = expressionEvaluator.evaluate(branch.getCondition(), ctx);
            if (result) {
                return branch.getTarget();  // 返回跳转目标
            }
        }
        return node.getDefaultTarget();  // 无匹配则返回默认目标
    }
}
```

### 10.2 select 节点执行

```java
public class SelectNodeExecutor implements FlowNodeExecutor<SelectNode> {

    @Override
    public ExecutionResult execute(SelectNode node, ExecutionContext ctx) {
        // 1. 根据 capability 查找匹配的 AtomicSkill
        AtomicSkill skill = skillRegistry.findByCapability(node.getCapability());
        if (skill == null) {
            throw new SkillNotFoundException(node.getCapability());
        }

        // 2. 构建输入参数
        Map<String, Object> input = templateEngine.resolve(node.getInput(), ctx);

        // 3. 执行技能
        Object result = skillExecutor.execute(skill, input);

        // 4. 将结果写入上下文
        ctx.put(node.getOutputVar(), result);

        return ExecutionResult.success();
    }
}
```

### 10.3 assign 节点执行

```java
public class AssignNodeExecutor implements FlowNodeExecutor<AssignNode> {

    @Override
    public ExecutionResult execute(AssignNode node, ExecutionContext ctx) {
        for (Map.Entry<String, String> entry : node.getAssignments().entrySet()) {
            Object value = expressionEvaluator.evaluate(entry.getValue(), ctx);
            ctx.put(entry.getKey(), value);
        }
        return ExecutionResult.success();
    }
}
```

### 10.4 merge 节点执行

```java
public class MergeNodeExecutor implements FlowNodeExecutor<MergeNode> {

    @Override
    public ExecutionResult execute(MergeNode node, ExecutionContext ctx) {
        Map<String, Object> merged = new LinkedHashMap<>();

        for (SourceMapping source : node.getSources()) {
            Object value = ctx.get(source.getVar());

            // 提取特定字段
            if (source.getField() != null && value instanceof Map) {
                value = ((Map<?, ?>) value).get(source.getField());
            }

            // 使用别名或原变量名作为 key
            String key = source.getAs() != null ? source.getAs() : source.getVar();
            merged.put(key, value);
        }

        ctx.put(node.getOutputVar(), merged);
        return ExecutionResult.success();
    }
}
```

---

## 11. 表达式求值器

**优先级**：P0（必须）

### 11.1 支持的表达式函数

| 函数 | 语法 | 说明 |
|------|------|------|
| `count()` | `count({{array}})` | 数组元素数量 |
| `avg()` | `avg({{array}})` 或 `avg({{array}}, field)` | 数值平均值 |
| `sum()` | `sum({{array}})` 或 `sum({{array}}, field)` | 数值求和 |
| `max()` | `max({{array}})` 或 `max({{array}}, field)` | 最大值 |
| `min()` | `min({{array}})` 或 `min({{array}}, field)` | 最小值 |
| `exists()` | `exists({{var}})` | 变量是否存在且非 null |
| `empty()` | `empty({{array}})` | 数组是否为空 |

### 11.2 条件表达式语法

复用现有的 when 条件表达式语法，支持：

- 比较运算符：`==`, `!=`, `>`, `<`, `>=`, `<=`
- 逻辑运算符：`&&`, `||`
- 函数调用：`count({{results}}) > 5`
- 字段访问：`{{item.score}} > 80`

### 11.3 实现示例

```java
public class ExpressionEvaluator {

    public Object evaluate(String expression, ExecutionContext ctx) {
        // 1. 解析表达式 AST
        Expression ast = parser.parse(expression);

        // 2. 替换变量引用
        ast = resolveVariables(ast, ctx);

        // 3. 执行表达式求值
        return ast.evaluate();
    }

    private Object evaluateFunction(String name, List<Object> args) {
        switch (name) {
            case "count":
                return ((List<?>) args.get(0)).size();
            case "avg":
                return calculateAverage((List<?>) args.get(0),
                    args.size() > 1 ? (String) args.get(1) : null);
            case "sum":
                return calculateSum((List<?>) args.get(0),
                    args.size() > 1 ? (String) args.get(1) : null);
            case "exists":
                return args.get(0) != null;
            case "empty":
                return args.get(0) == null || ((List<?>) args.get(0)).isEmpty();
            default:
                throw new UnknownFunctionException(name);
        }
    }
}
```

---

## 12. 能力匹配器

**优先级**：P0（必须）

### 12.1 能力匹配规则

`select` 节点通过 `capability` 标签匹配 AtomicSkill。匹配规则：

1. 技能必须在 `capabilityTags` 中声明该能力
2. 多个技能匹配时，选择第一个匹配的技能
3. 无匹配时抛出异常或执行 fallback

### 12.2 技能注册表

```java
public class SkillRegistry {

    private Map<String, List<AtomicSkill>> capabilityIndex;

    public void register(AtomicSkill skill) {
        for (String capability : skill.getCapabilityTags()) {
            capabilityIndex
                .computeIfAbsent(capability, k -> new ArrayList<>())
                .add(skill);
        }
    }

    public AtomicSkill findByCapability(String capability) {
        List<AtomicSkill> skills = capabilityIndex.get(capability);
        if (skills == null || skills.isEmpty()) {
            return null;
        }
        return skills.get(0);  // 返回第一个匹配的技能
    }
}
```

---

## 13. fallback 机制

**优先级**：P0（必须）

### 13.1 fallback 处理流程

```
主节点执行
    ↓
 执行成功？ → 是 → 继续下一节点
    ↓ 否
 有 fallback？ → 否 → 抛出异常
    ↓ 是
执行 fallback 节点
    ↓
fallback 成功？ → 是 → 继续下一节点
    ↓ 否
fallback 有 fallback？ → 否 → 抛出异常
    ↓ 是
执行二级 fallback
    ↓
成功 → 继续 / 失败 → 抛出异常
```

### 13.2 实现示例

```java
public class CognitiveSkillExecutor {

    public Object execute(CognitiveSkill skill, Map<String, Object> input) {
        ExecutionContext ctx = new ExecutionContext(input);

        for (FlowNode node : skill.getInternalFlow()) {
            executeWithFallback(node, ctx, 0);
        }

        return ctx.buildOutput(skill.getOutputSchema());
    }

    private void executeWithFallback(FlowNode node, ExecutionContext ctx, int depth) {
        if (depth > 2) {
            throw new FallbackDepthExceededException();
        }

        try {
            FlowNodeExecutor<?> executor = executorRegistry.get(node.getType());
            executor.execute(node, ctx);
        } catch (Exception e) {
            if (node.getFallback() != null) {
                executeWithFallback(node.getFallback(), ctx, depth + 1);
            } else {
                throw e;
            }
        }
    }
}
```

---

## 14. CognitiveSkill 测试要点

**优先级**：P0（必须）

### 14.1 解析器测试

| 测试场景 | 预期结果 |
|----------|----------|
| 完整的 CognitiveSkill 定义 | 正确解析所有节点 |
| 缺少 internal_flow | 解析错误 |
| 未知节点类型 | 解析错误 |
| 节点名重复 | 解析错误 |
| fallback 超过两层 | 解析错误 |
| decision 分支目标不存在 | 解析错误 |

### 14.2 执行器测试

| 测试场景 | 预期结果 |
|----------|----------|
| decision 条件匹配第一个分支 | 跳转到对应目标 |
| decision 无匹配使用默认 | 跳转到 defaultTarget |
| select 找到匹配技能 | 执行技能并存储结果 |
| select 无匹配技能 | 抛出异常或执行 fallback |
| assign 变量赋值 | 变量正确写入上下文 |
| merge 合并多个变量 | 生成正确的合并对象 |

### 14.3 表达式测试

| 测试场景 | 预期结果 |
|----------|----------|
| `count({{arr}})` | 返回数组长度 |
| `avg({{arr}}, "score")` | 返回指定字段的平均值 |
| `exists({{undefined}})` | 返回 false |
| `empty({{empty_arr}})` | 返回 true |
| `{{a}} > 10 && {{b}} < 5` | 正确计算逻辑表达式 |

### 14.4 集成测试

| 测试场景 | 预期结果 |
|----------|----------|
| 完整的决策流程 | 正确执行所有节点 |
| fallback 成功恢复 | 执行 fallback 后继续 |
| 二级 fallback | 一级失败后执行二级 |
| 能力匹配执行 | 动态选择并执行技能 |
| 循环决策流程 | 正确处理决策循环 |

---

## 15. 实施建议（CognitiveSkill）

### 15.1 分阶段实施

**阶段一（P0）**：
1. CognitiveSkill 数据模型定义
2. internal_flow 解析器
3. 四种节点类型执行器
4. 基础表达式函数

**阶段二（P0）**：
1. fallback 机制实现
2. 能力匹配器
3. 与 AtomicSkill 的集成

**阶段三（P1）**：
1. 表达式函数扩展
2. 执行追踪和调试
3. 性能优化

### 15.2 与现有系统的集成

| 集成点 | 说明 |
|--------|------|
| 解析器入口 | SkillParser 根据 type 分发到 CognitiveSkillParser |
| 执行器入口 | SkillExecutor 根据 type 分发到 CognitiveSkillExecutor |
| 能力注册 | AtomicSkill 注册时建立 capability 索引 |
| 上下文共享 | CognitiveSkill 与子技能共享执行上下文 |

### 15.3 风险点

| 风险 | 影响 | 缓解措施 |
|------|------|----------|
| 能力匹配不明确 | 选择错误的技能 | 建立严格的 capability 命名规范 |
| fallback 滥用 | 掩盖真实错误 | 记录 fallback 执行日志，监控频率 |
| 循环跳转 | 无限循环 | 设置最大执行步数限制 |
| 性能影响 | 多层嵌套执行较慢 | 优化节点执行，考虑并行执行独立分支 |

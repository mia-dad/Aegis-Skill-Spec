# Aegis Skill DSL 规范 - 待改进项分析

> 从技能开发者的角度，对规范完整性的评估和改进建议

**分析日期**: 2026-03-08
**规范版本**: 3.4.2

---

## ✅ 规范的优势（能满足的需求）

### 1. 数据处理能力完善
- 变量引用：`{{stepName.field}}`、`{{stepName.value}}`
- 数组操作：索引访问 `[n]`、字段投影 `[*].field`、`.length` 属性
- 简单表达式：算术运算 `+ - * /`、字符串拼接
- 内置函数：`array::flatten`、`array::filter`、`array::sort` 等

### 2. 流程控制基本够用
- 步骤级条件执行：`when.expr`
- 模板级条件渲染：`{{#when condition}}`
- 循环遍历：`{{#for array}}` + `_index`、`#break`
- 异常处理：`ignore` 参数

### 3. 外部交互丰富
- 预置工具：`db_select`、`db_insert`、`db_update`、`http_request`、`text_file`、`json_select`、`log`
- 自定义工具扩展机制
- 支持参数化查询（防注入）

### 4. 人机交互支持
- `await` 步骤：暂停等待用户输入
- `ui` 展示语义：多种展示类型

### 5. 调试和审计
- `debug()` 函数
- `log` 工具（审计日志）
- 层级归属信息自动关联

---

## ⚠️ 规范的不足（可能无法满足的需求）

### 1. 缺少基础比较和逻辑函数

**当前状态**：只有操作符，没有函数

```yaml
# ❌ 不支持
{{#when gt(count, 10)}}
{{#when exists(data)}}

# ✅ 只能用操作符
{{#when count > 10}}
```

**影响**：
- 无法将比较逻辑封装为函数
- 条件表达式只能写在操作符位置

**建议补充**：
```
public::gt(a, b) → boolean
public::lt(a, b) → boolean
public::gte(a, b) → boolean
public::lte(a, b) → boolean
public::eq(a, b) → boolean
public::ne(a, b) → boolean
public::exists(value) → boolean
public::empty(value) → boolean
```

**优先级**：高

---

### 2. 字符串处理能力薄弱

**当前状态**：只支持 `+` 拼接

**缺少功能**：
- 字符串截取
- 大小写转换
- 正则匹配
- 字符串分割/替换
- 去除空格

**影响场景**：
- 格式化电话号码、身份证号
- 处理文件扩展名
- 生成编号
- 清洗用户输入

**建议补充**：
```
public::substring(str, start, end?) → string
public::uppercase(str) → string
public::lowercase(str) → string
public::trim(str) → string
public::replace(str, pattern, replacement) → string
public::split(str, delimiter) → array
public::join(array, separator) → string
public::contains(str, substring) → boolean
public::matches(str, pattern) → boolean
```

**优先级**：高

---

### 3. 数学函数缺失

**当前状态**：只有四则运算 `+ - * /`

**缺少功能**：
- 取整、四舍五入
- 绝对值
- 最大值/最小值
- 随机数
- 幂运算

**影响场景**：
- 价格计算（需要四舍五入）
- 排名计算
- 分配算法

**建议补充**：
```
public::round(number, decimals?) → number
public::floor(number) → number
public::ceil(number) → number
public::abs(number) → number
public::min(a, b) → number
public::max(a, b) → number
public::random() → number
public::pow(base, exponent) → number
public::sqrt(number) → number
```

**优先级**：高

---

### 4. 日期时间处理缺失

**当前状态**：有 `datetime` 类型，但没有操作函数

**缺少功能**：
- 获取当前时间
- 日期加减
- 日期格式化
- 时间差计算

**影响场景**：
- 过期时间判断
- 报表时间范围
- 定时任务
- 日期显示格式化

**建议补充**：
```
public::now() → datetime
public::format(date, pattern) → string
public::parse(str, pattern) → datetime
public::addDays(date, days) → datetime
public::addHours(date, hours) → datetime
public::diff(date1, date2, unit?) → number
public::year(date) → number
public::month(date) → number
public::day(date) → number
```

**优先级**：高

---

### 5. 对象操作能力不足

**当前状态**：对象只能平铺访问

**缺少功能**：
- 对象合并
- 对象键获取
- 深度访问
- 动态字段访问

**影响场景**：
- 合并配置
- 动态字段访问
- 处理嵌套 JSON

**建议补充**：
```
public::merge(obj1, obj2, ...?) → object
public::keys(obj) → array
public::values(obj) → array
public::get(obj, path) → any
public::set(obj, path, value) → object
```

**优先级**：中

---

### 6. 类型转换缺失

**当前状态**：无法在 string/number/boolean 之间转换

**影响场景**：
- 从数据库读取的数字是字符串，需要计算
- API 返回的 "true"/"false" 字符串需要转为布尔值
- 用户输入需要类型校验和转换

**建议补充**：
```
public::toInt(value) → number
public::toFloat(value) → number
public::toString(value) → string
public::toBoolean(value) → boolean
public::typeOf(value) → string
```

**优先级**：高

---

### 7. 复杂条件逻辑受限

**当前状态**：模板中的 `{{#when}}` 不支持多分支

**当前写法**：
```yaml
{{#when score >= 90}}
优秀
{{/when}}
{{#when score >= 80}}
良好
{{/when}}
{{#when score >= 60}}
及格
{{/when}}
```

**影响**：需要写多个 `{{#when}}`，代码冗长

**建议**：支持 else-if 链式语法
```yaml
{{#when score >= 90}}
优秀
{{:elif score >= 80}}
良好
{{:elif score >= 60}}
及格
{{:else}}
不及格
{{/when}}
```

**优先级**：中

---

### 8. 数组操作仍有缺口

**当前状态**：有 `array::filter`，但缺少常用操作

**缺少功能**：
- `map`：对每个元素转换
- `find`：查找第一个符合条件的元素
- `some`/`every`：判断是否存在/全部满足
- `reduce`：归约聚合
- `groupBy`：分组

**影响场景**：
- 从订单数组中提取所有 ID
- 检查数组中是否包含某个值
- 分组统计

**建议补充**：
```
array::map(array, projection) → array
array::find(array, condition) → object
array::some(array, condition) → boolean
array::every(array, condition) → boolean
array::reduce(array, accumulator, initial) → any
array::groupBy(array, key) → object
array::reverse(array) → array
array::slice(array, start, end?) → array
array::concat(array1, array2) → array
```

**优先级**：中

---

## 📊 总体评价

| 方面 | 评分 | 说明 |
|------|------|------|
| **基础数据处理** | 7/10 | 基本够用，但字符串、数学、日期处理薄弱 |
| **流程控制** | 8/10 | when/for/ignore 基本满足，但复杂逻辑需要多个步骤 |
| **外部交互** | 9/10 | 工具系统设计良好，预置工具丰富 |
| **类型系统** | 8/10 | 语义类型设计优秀，但缺少类型转换 |
| **调试支持** | 9/10 | log、debug、审计日志完善 |
| **学习曲线** | 7/10 | 自定义模板语法需要学习，但整体清晰 |

**总体结论**：规范能满足 **70-80%** 的常见技能开发需求。

---

## 🎯 改进建议优先级

### 高优先级（必须补充）
1. **类型转换函数**：`toInt`、`toString`、`toBoolean`
2. **字符串函数**：`substring`、`trim`、`split`、`join`、`contains`
3. **数学函数**：`round`、`floor`、`ceil`、`abs`
4. **比较函数**：`gt`、`lt`、`eq`、`exists`、`empty`
5. **日期函数**：`now`、`format`、`addDays`、`diff`

### 中优先级（建议补充）
1. **对象操作**：`merge`、`keys`、`get`、`set`
2. **高级数组操作**：`map`、`find`、`reduce`、`groupBy`
3. **条件语法增强**：支持 `{{:elif}}` 链式语法
4. **更多字符串函数**：`replace`、`matches`、`uppercase`、`lowercase`

### 低优先级（可选）
1. **随机数**：`random`
2. **高级数学**：`pow`、`sqrt`
3. **数组操作**：`reverse`、`slice`、`concat`

---

## 📝 实现建议

### 函数命名规范

延续现有规范：
- `public::` 命名空间用于通用函数（可省略前缀）
- `array::` 命名空间用于数组操作
- 新增 `string::` 命名空间用于字符串操作
- 新增 `date::` 命名空间用于日期操作

### 函数签名规范

```
<namespace>::<function>(param1: type, param2?: type, [ignore: boolean]) → return_type
```

### 向后兼容性

所有新增函数均为**可选使用**，不影响现有技能的正常运行。

---

## 🔗 相关文档

- [README.md - 内置函数](./README.md#3-内置函数)
- [AtomicSkill.md - 模板语法](./AtomicSkill.md#43-变量与模板语法)

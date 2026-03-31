---
name: dslquery
description: Use when building DSL query filters, sorting, or pagination in JavaScript/TypeScript projects that use the dslquery library
tools: Read, Edit, Write, Grep, Glob
---

# dslquery - DSL 查询构建

当代码中需要构建 DSL 查询（过滤、排序、分页）时，使用 `dslquery` 库的 API 直接生成。

## 导入

```javascript
import {
  Query,
  and, or,
  equals, notEquals, greaterThan, greaterThanOrEquals,
  lessThan, lessThanOrEquals,
  startsWith, endsWith, contains,
  isIn, notIn, between,
  isnull, notnull,
  asc, desc,
  buildFilter, buildFilterExpression
} from 'dslquery';
```

## 快速参考

### 操作符

| 函数 | DSL 输出 | 用法 |
|------|----------|------|
| `equals(field, val)` | `(field eq val)` | 精确匹配 |
| `notEquals(field, val)` | `(field ne val)` | 不等于 |
| `greaterThan(field, val)` | `(field gt val)` | 大于 |
| `greaterThanOrEquals(field, val)` | `(field ge val)` | 大于等于 |
| `lessThan(field, val)` | `(field lt val)` | 小于 |
| `lessThanOrEquals(field, val)` | `(field le val)` | 小于等于 |
| `startsWith(field, val)` | `(field sw val)` | 前缀匹配 |
| `endsWith(field, val)` | `(field ew val)` | 后缀匹配 |
| `contains(field, val)` | `(field ct val)` | 包含 |
| `isIn(field, arr)` | `(field in [...])` | 在列表中 |
| `notIn(field, arr)` | `(field ni [...])` | 不在列表中 |
| `between(field, start, end)` | `(field bt start,end)` | 范围 |
| `isnull(field)` | `(field isn)` | 为空 |
| `notnull(field)` | `(field inn)` | 不为空 |

### 逻辑组合

```javascript
// AND 组合
and(equals("status", "active"), greaterThan("age", 18))
// → (and(status eq active)(age gt 18))

// OR 组合
or(equals("role", "admin"), equals("role", "moderator"))
// → (or(role eq admin)(role eq moderator))

// 嵌套组合：status=active 且 (role=admin 或 role=moderator)
and(
  equals("status", "active"),
  or(
    equals("role", "admin"),
    equals("role", "moderator")
  )
)
// → (and(status eq active)(or(role eq admin)(role eq moderator)))

// 深层嵌套
and(
  equals("status", "active"),
  or(
    and(greaterThan("age", 18), lessThan("age", 65)),
    equals("vip", true)
  )
)
// → (and(status eq active)(or(and(age gt 18)(age lt 65))(vip eq true)))
```

### 排序

```javascript
// 单字段
asc("name")                     // → name asc
desc("createdAt")               // → createdAt desc

// 多字段链式
desc("createdAt").asc("name")   // → createdAt desc,name asc
asc("name").desc("age")         // → name asc,age desc
```

## 核心模式

### 1. 构建查询（过滤 + 分页 + 排序）

```javascript
const query = new Query()
  .withLimit(20)
  .withFilter(and(
    equals("name", "张三"),
    greaterThan("age", 18)
  ))
  .withSort(desc("createdAt").asc("name"));

query.goto(3); // 跳到第 3 页，skip = (3-1) * limit

// 使用
query.filter // "(and(name eq %E5%BC%A0%E4%B8%89)(age gt 18))"
query.sort   // "createdAt desc,name asc"
query.limit  // 20
query.skip   // 40
```

### 2. 表单搜索过滤器（buildFilter）

将表单数据映射为 DSL 过滤条件，自动忽略空值：

```javascript
const rules = {
  keyword: (value) => contains("name", value),
  status: (value) => equals("status", value),
  dateRange: (value) => between("createdAt", value[0], value[1]),
  default: (form) => equals("deleted", false) // 始终附加的默认条件
};

const form = { keyword: "test", status: "active", dateRange: ["2025-01-01", "2025-12-31"] };
const tenantCondition = equals("tenantId", "t001"); // 额外条件（如租户隔离）

// 返回 DSL 字符串
buildFilter(rules, form, tenantCondition);
// "(and(name ct test)(status eq active)(createdAt bt 2025-01-01,2025-12-31)(deleted eq false)(tenantId eq t001))"

// 返回 Expression 对象（可传给 Query.withFilter）
const expr = buildFilterExpression(rules, form, tenantCondition);
new Query().withFilter(expr);
```

**rules 规则：**
- 每个 key 对应 form 中的字段名，value 是 `(value, form) => Expression | Expression[]`
- `default` 是特殊 key，`(form) => Expression | undefined`，始终执行
- form 中值为 `""` / `null` / `undefined` / `[]` 的字段自动跳过
- boolean 值（`true`/`false`）不会被跳过
- 一个 rule 可返回数组（多个条件会自动展开）：
  ```javascript
  const rules = {
    search: (value) => [
      contains("name", value),
      contains("email", value)
    ]
  };
  const form = { search: "test" };
  buildFilter(rules, form);
  // → "(and(name ct test)(email ct test))"
  ```
- rule 函数可访问完整 form 对象（第二个参数）：
  ```javascript
  const rules = {
    name: (value, form) => form.exactMatch ? equals("name", value) : contains("name", value)
  };
  ```

### 3. JSON 序列化

Expression 对象自带 `toJSON()`，可直接放入请求参数：

```javascript
const params = {
  filter: and(equals("name", "test")),
  sort: "name asc"
};
JSON.stringify(params); // {"filter":"(and(name eq test))","sort":"name asc"}

// 单个表达式也可以，会自动包裹 and()
const params2 = { filter: equals("name", "test") };
JSON.stringify(params2); // {"filter":"(and(name eq test))"}
```

### 4. Query 分页 API

```javascript
const query = new Query()         // limit=10, skip=0
  .withLimit(20)                  // ✅ 可链式，返回 this
  .withFilter(expr)               // ✅ 可链式，返回 this
  .withSort(sort)                 // ✅ 可链式，返回 this
  .withSkip(10);                  // ✅ 可链式，返回 this

// ⚠️ 以下方法返回 void，不可链式调用
query.goto(3);                    // 跳到第 3 页 → skip = 40
query.gotoOffset(-1);             // 相对偏移 → skip = 20
query.onTotal(219);               // 设置总数
query.maxPage;                    // 11 (ceil(219/20))
```

## 注意事项

- 特殊字符（括号等）会自动 URL 编码，无需手动处理
- `buildFilter` 返回字符串，`buildFilterExpression` 返回 Expression 对象

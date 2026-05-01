---
name: dct-schema
description: 产物格式查询。返回当前阶段对应的产物 Schema 文件路径，用于拼接到 sailor prompt 中。纯只读。
argument-hint: "<需求主题>"
---

# dct-schema（产物格式查询）

返回当前阶段的产物 Schema 文件路径列表。与流程无关，独立 Skill。

---

## 调用方式

```
Skill("dct-schema", args="<需求主题>")
```

---

## 执行步骤

1. 读取 `prd/<需求主题>/status.md` 确定当前阶段
2. 根据当前阶段返回对应的 schema 文件路径

---

## 返回格式

```json
{
  "schemas": ["schema/normalization.md"],
  "stage": "阶段一：需求标准化"
}
```

---

## 阶段→Schema 映射

| 阶段 | Schema 文件 |
|------|-------------|
| 阶段一 | `schema/normalization.md` |
| 阶段二 | `schema/design.md`、`schema/api.json.md` |
| 阶段三 | `schema/planning.md` |
| 阶段四 | `schema/execution.md` |
| 阶段五 | `schema/review.md` |

---

## captain 使用方式

```javascript
const schemaResult = Skill("dct-schema", args="<需求主题>");
// 拼接到 sailor prompt:
// 产物格式参考：参考 schema/normalization.md 获取格式规范
```

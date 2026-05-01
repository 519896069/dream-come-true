---
name: dct-loop
description: 阶段循环体。读取 status.md 判断当前阶段状态，返回 action 给 captain 执行。纯决策，不调用 Agent()。
argument-hint: "<需求主题> [fast]"
---

# dct-loop（阶段循环体）

阶段循环的决策核心。读取 status.md → 判断当前该做什么 → 返回 action + 数据。**不调 Agent()，Agent() 由 captain 负责。**

---

## 执行模式

| 模式 | args 包含 | 用户确认行为 |
|------|----------|-------------|
| **深度模式**（默认） | 仅 `<需求主题>` | AskUserQuestion 等待用户决策 |
| **快速模式** | `<需求主题> fast` | confirm 自动返回 `pass`，跳过用户交互 |

---

## 调用方式

```
# 深度模式（默认）
Skill("dct-loop", args="<需求主题>")

# 快速模式
Skill("dct-loop", args="<需求主题> fast")
```

---

## 阶段配置

阶段配置从 [reference/stages.md](reference/stages.md) 读取。

---

## 执行流程

### 步骤1：读取状态

读取 `prd/<需求主题>/status.md`。

### 步骤2：定位当前阶段

遍历阶段进度表，找第一个"用户确认"为 `[ ]` 的阶段。

如果所有阶段"用户确认"=`[✅]` → 返回 `action: "done"`。

### 步骤3：决策 → 返回 action

检查当前阶段的"产物"和"AI审查"列，从 [stages.md](reference/stages.md) 读取阶段配置：

```
产物=[ ]
  ├─ stages.md 并行=true → action: "parallel"（读 plan.md 拆 tasks）
  └─ 并行=false → action: "sailor"
产物=[x]
  ├─ AI审查=跳过（查 stages.md）→ action: "confirm"
  └─ AI审查=需要
      ├─ AI审查=[ ]   → action: "inspector"
      └─ AI审查=[x]   → action: "confirm"
```

**并行拆分逻辑**：查 stages.md 当前阶段 `并行=true` → 读 `prd/<需求主题>/{project}-plan.md` 中的"执行单元"表格 → 每个执行单元生成一个 task，key/project/artifacts/plan_file 均隔离。详见 [reference/parallel-convention.md](reference/parallel-convention.md)。

---

## 返回格式

### action: "sailor" — 派遣 sailor 执行阶段

```json
{
  "action": "sailor",
  "stepname": "<阶段名>（从 stages.md 读取）",
  "stage": "<阶段编号>",
  "dispatch": {
    "stage_skill": "<skill名称>（从 stages.md 读取）",
    "stage_name": "<阶段名>",
    "effort_level": "<effort级别>（从 stages.md 读取）",
    "prev_artifacts": ["<上阶段产物路径>"],
    "current_artifacts": ["<当前阶段产物路径>"]
  }
}
```

### action: "parallel" — 并行派遣多 sailor

当前阶段 `并行=true` 且产物未生成时返回。dct-loop 读取上阶段产物 `{project}-plan.md` 的"执行单元"表格，拆分为独立 task。

```json
{
  "action": "parallel",
  "stage": "<阶段编号>",
  "tasks": [
    {
      "key": "<unit-id>（从 plan.md 执行单元表读取）",
      "stage_skill": "<skill名称>（从 stages.md 读取）",
      "stage_name": "<阶段名>",
      "effort_level": "<effort级别>",
      "scope": {
        "key": "<unit-id>",
        "project": "<项目名>",
        "artifacts": ["<产物目录>"],
        "plan_file": "prd/<需求主题>/<project>-plan.md"
      }
    }
  ]
}
```

tasks 数量 = plan.md 执行单元表的行数。captain 并发派遣全部 task，全部成功后统一更新 status.md。详见 [reference/parallel-convention.md](reference/parallel-convention.md)。

---

### action: "inspector" — 派遣 dct-inspector 审查

```json
{
  "action": "inspector",
  "stepname": "<阶段名>",
  "stage": "<阶段编号>",
  "artifacts": ["<当前阶段产物路径>"],
  "checkpoint_path": "prd/<需求主题>/checkpoint.md"
}
```

### action: "confirm" — 用户确认

#### 快速模式

跳过 AskUserQuestion，直接返回：

```json
{
  "action": "pass",
  "stepname": "<阶段名>",
  "stage": "<阶段编号>"
}
```

#### 深度模式（默认）

dct-loop 内部调用 `AskUserQuestion`，将用户选择转为以下三种之一返回：

**用户选"通过"：**
```json
{
  "action": "pass",
  "stepname": "<阶段名>",
  "stage": "<阶段编号>"
}
```

**用户选"修改意见"：**
```json
{
  "action": "revise",
  "stepname": "<阶段名>",
  "stage": "<阶段编号>",
  "feedback": "<用户修改意见原文>",
  "artifacts": ["<当前阶段产物路径>"]
}
```

**用户选"打回"：**
```json
{
  "action": "reject",
  "stepname": "<阶段名>",
  "stage": "<阶段编号>"
}
```

AskUserQuestion 格式：

```javascript
AskUserQuestion({
  questions: [{
    question: "请确认{阶段名}产物：",
    header: "产物确认",
    options: [
      { label: "通过", description: "确认产物无误，继续下一阶段" },
      { label: "修改意见", description: "有问题需要修改" },
      { label: "打回", description: "重新执行阶段，描述有什么问题" }
    ],
    multiSelect: false
  }]
})
```

### action: "done" — 全部完成

```json
{
  "action": "done",
  "stepname": "全部完成",
  "artifacts": ["<所有阶段产物汇总>"]
}
```

### action: "error" — 校验不通过

```json
{
  "action": "error",
  "stepname": "<阶段名>",
  "message": "<错误原因>"
}
```

---

## 决策速查

| 产物 | AI审查(配置) | AI审查(状态) | 并行 | → action |
|------|-------------|-------------|------|----------|
| `[ ]` | - | - | false | `sailor` |
| `[ ]` | - | - | true | `parallel` |
| `[x]` | 跳过 | - | - | `confirm` |
| `[x]` | 需要 | `[ ]` | - | `inspector` |
| `[x]` | 需要 | `[x]` | - | `confirm` |
| `[✅]`* | - | `[✅]`* | - | `done` |

> *所有阶段均为 `[✅]`；AI审查(配置) 从 stages.md 读取

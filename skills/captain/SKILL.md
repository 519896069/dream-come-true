---
name: captain
description: "帮助用户完成 idea 到代码实现的完整流程，覆盖需求分析、方案设计、计划执行到测试验收。当用户提到需求开发、功能实现、完整项目开发时调用。"
---

# 工作流编排（主线程执行）

主线程在执行此 Skill 期间扮演**调度者**角色：

```
入口（locate / init）→ 循环（调 dct-loop → 根据 action 执行 → 再调 dct-loop → ...）→ 结束
```

> **本 Skill 不派遣 sub-agent 来执行编排逻辑**（避免 sub-agent 嵌套派遣限制），而是由主线程直接读取本指令并按流程执行。

---

## 执行模式

| 模式 | 触发条件 | 用户确认行为 |
|------|---------|-------------|
| **深度模式**（默认） | 未指定模式时 | 每个阶段产物需用户确认（通过/修改/打回） |
| **快速模式** | 上下文含"快速模式"或 "fast" | AI 自动决策，跳过所有用户确认 |

模式在 captain 启动时从上下文判断。快速模式下 dct-loop 的 confirm 自动返回 `pass`。

---

## 工具约定

| 工具 | 用途 |
|------|------|
| `Skill("dct-loop", args="<需求主题> [fast]")` | 循环体：读状态 → 返回 action。快速模式时追加 `fast` 参数 |
| `Skill("dct-schema", args="<需求主题>")` | 产物 Schema 查询（拼接到 sailor prompt） |
| `Agent({ subagent_type: "dream-come-true:sailor:sailor", ... })` | 阶段执行 |
| `Agent({ subagent_type: "dream-come-true:dct-inspector:dct-inspector", ... })` | AI 审查 |
| `Edit(...)` | 仅写 status.md "用户确认"列 |

| 工具 | 用途 |
|------|------|
| `Skill("dct-loop", args="<需求主题>")` | 循环体：读状态 → 返回 action |
| `Skill("dct-schema", args="<需求主题>")` | 产物 Schema 查询（拼接到 sailor prompt） |
| `Agent({ subagent_type: "dream-come-true:sailor:sailor", ... })` | 阶段执行 |
| `Agent({ subagent_type: "dream-come-true:dct-inspector:dct-inspector", ... })` | AI 审查 |
| `Edit(...)` | 仅写 status.md "用户确认"列 |

---

## ⛔ 红线规则

**最高优先级 — 违反即严重流程错误。详见 [reference/red-lines.md](reference/red-lines.md)。**

| 规则 | 一句话摘要 |
|------|-----------|
| R1 | 禁止主线程 Write/Edit PRD 业务文件（除 status.md） |
| R2 | 禁止主线程 Read PRD 业务内容（除 status.md） |
| R3 | 必须先调 dct-loop，不以用户 prompt 为准 |
| R4 | 禁止主线程替代 sailor；AI 审查由 captain 直接派遣 dct-inspector |
| R5 | 禁止主线程直接调用业务 Skill（dct-normalization/design/planning/execution/testing/standards） |
| R6 | 所有用户决策必须由用户本人回复，禁止代答 |

---

## 一、入口

主线程启动后，**首先告知当前模式**，然后判断"断点续传"还是"新需求"。

```
输出: "dream-come-true 插件启动 — <深度模式/快速模式>。需求主题：<需求主题>"
```

### 1.1 检查 status.md

```
检查 prd/<需求主题>/status.md 是否存在
  → 不存在 → 新需求 → 执行 1.2
  → 存在   → 断点续传 → 跳过 1.2，直接进入 二、阶段循环
```

### 1.2 新需求初始化

1. 创建 `prd/<需求主题>/` 目录
2. 读取 [reference/status-template.md](reference/status-template.md)
3. 替换 `<需求主题>` 和 `<YYYY-MM-DD>` 为实际值
4. 写入 `prd/<需求主题>/status.md`

初始化完成后进入阶段循环（dct-loop 会返回"阶段一 sailorable"）。

---

## 二、阶段循环

```
┌──────────────────────────────────────────────────┐
│                    phase loop                     │
│                                                   │
│   captain ─→ Skill("dct-loop") ─→ action          │
│     │         (快速模式: 追加 fast 参数)            │
│     │                                           │
│     ├─ sailor    → Agent(sailor)    → 循环       │
│     ├─ parallel  → Agent(sailor)×N  → 循环       │
│     │             (并发，全成功后汇聚；            │
│     │              快速模式：失败自动重试1次)       │
│     ├─ inspector → Agent(inspector) → 循环       │
│     ├─ pass      → Edit status.md   → 循环       │
│     ├─ reject    → Edit status.md   → 循环       │
│     │             (仅深度模式，快速模式无此路径)    │
│     ├─ revise    → Agent(sailor,revise)          │
│     │             → Agent(inspector)  → 循环     │
│     │             (仅深度模式，快速模式无此路径)    │
│     ├─ done      → 结束                          │
│     └─ error     → AskUserQuestion / 中断        │
│               (深度模式问用户，快速模式直接中断)     │
│                                                   │
└──────────────────────────────────────────────────┘
```

**快速模式核心规则**：处理完任何 action 后，**不输出中间文字**，立即调 `Skill("dct-loop", args="<需求主题> fast")` 进入下一轮。仅在 `done` 时输出最终结果、`error` 时输出错误信息。每次可见文字输出都会被 runtime 视为本轮结束，导致流程停顿。

---

### 2.1 action: `sailor` — 派遣阶段执行

**🚨 `subagent_type` 必须是 `dream-come-true:sailor:sailor`，绝不能用 Skill 名作为 Agent 类型。**

```javascript
const { dispatch } = result;
const { schemas } = Skill("dct-schema", args="<需求主题>");

Agent({
  name: `${dispatch.stage_name}执行者`,
  description: `${dispatch.stage_name} - ${需求主题}`,
  subagent_type: "dream-come-true:sailor:sailor",
  prompt: `/effort ${dispatch.effort_level}

你是阶段执行 Agent，负责完成单个阶段的完整工作流程。

## 输入参数
- 运行模式：normal
- 执行模式：${mode}（深度/快速，影响是否跳过用户确认）
- 需求主题：${需求主题}
- 阶段名：${dispatch.stage_name}
- skill名称：${dispatch.stage_skill}
- effort级别：${dispatch.effort_level}
- 上阶段产物：${dispatch.prev_artifacts.join(', ') || '无'}
- 当前阶段产物：${dispatch.current_artifacts.join(', ')}

## 产物格式参考
${schemas.map(s => `参考 ${s} 获取格式规范`).join('\n')}

请按照标准工作流执行，完成后返回 JSON 格式结果。`
})
```

**sailor 返回后**：

| sailor 状态 | 深度模式处理 | 快速模式处理 |
|-------------|-------------|-------------|
| `"完成"` | 循环（调 dct-loop） | 直接调 `Skill("dct-loop", args="... fast")`，不输出文字 |
| `"失败"` 且含"缺少产物格式" | `AskUserQuestion` 报告用户，等待确认后重试 | 输出错误，流程中断 |
| 其他失败 | `AskUserQuestion` 报告错误 | 输出错误，流程中断 |

### 2.2 action: `parallel` — 并发派遣多 sailor

**并行阶段触发（stages.md 中 `并行=true`）。dct-loop 从 plan.md 读取执行单元，返回 N 个 task。**

```javascript
const { tasks } = result;

// 1. 并发派遣（每个 task 的 scope 保证写隔离）
const agents = tasks.map(task =>
  Agent({
    name: `${task.scope.key}执行者`,
    description: `阶段四 - ${task.scope.project}`,
    subagent_type: "dream-come-true:sailor:sailor",
    prompt: `/effort ${task.effort_level}

你是阶段执行 Agent，负责单个执行单元的代码实现。

## 输入参数
- 运行模式：normal
- 需求主题：${需求主题}
- 阶段名：${task.stage_name}
- skill名称：${task.stage_skill}
- effort级别：${task.effort_level}
- scope.project：${task.scope.project}
- scope.key：${task.scope.key}
- scope.artifacts：${task.scope.artifacts.join(', ')}
- scope.plan_file：${task.scope.plan_file}

## 并行约定
遵守 dct-loop/reference/parallel-convention.md 的 P1~P5 规则。
只实现 scope.project 一个项目，不触碰其他执行单元的文件。

请按照标准工作流执行，完成后返回 JSON 格式结果。`
  })
);

// 2. 等待全部返回
wait_all(agents);

// 3. 按结果分组
succeeded = tasks.filter(r => r.status === "完成");
failed    = tasks.filter(r => r.status !== "完成");

// 4. 原子决策
if (failed 为空) {
  所有 sailor 已各自更新产物文件；
  captain 统一 Edit status.md "产物"=[x]；
  直接调 `Skill("dct-loop", args="... fast")`，不输出文字；
} else if (mode === "快速") {
  // 快速模式：自动重试失败的执行单元（最多1轮），仍失败则中断
  if (retry已完成) {
    输出: `快速模式：${failed.length} 个执行单元重试仍失败，流程中断。失败：${failed.map(f => f.key + ": " + f.error).join("; ")}`;
    结束流程；
  } else {
    retry已完成 = true;
    只重派 failed 的 task → 回到 步骤2；
  }
} else {
  // 深度模式：等待用户决策
  AskUserQuestion({
    question: `${succeeded.length} 个完成。${failed.length} 个失败：${failed.map(f => f.key + ": " + f.error).join("; ")}。`,
    header: "并行结果",
    options: [
      { label: "仅重试失败", description: "只重新执行失败的执行单元" },
      { label: "全部重试", description: "所有执行单元重新执行" },
      { label: "中断", description: "暂停流程，人工介入" }
    ]
  });
  if "仅重试失败" → 只派 failed 的 task → 回到 步骤2；
  if "全部重试" → 全量重派 → 回到 步骤1；
  if "中断" → 退出流程；
}
```

**并行隔离保证**：每个 task 的 `scope.artifacts` 指向独立目录，天然无竞态。status.md 由 captain 在全部成功后统一写入。

---

### 2.3 action: `inspector` — 派遣 AI 审查

**AI审查=跳过的阶段（见 stages.md），sailor 返回后直接进用户确认，不执行 inspector。**

```javascript
Agent({
  name: `${result.stepname}审查者`,
  description: `${result.stepname}审查 - ${需求主题}`,
  subagent_type: "dream-come-true:dct-inspector:dct-inspector",
  prompt: `/effort high

你是 AI 审查 Agent，负责审查阶段产物的一致性。

## 输入参数
- 需求主题：${需求主题}
- 阶段名：${result.stepname}
- 产物文件列表：${result.artifacts.join(', ')}
- checkpoint路径：${result.checkpoint_path}

## 审查要求

按照 dct-inspector agent 的审查规则执行：
1. 读取 checkpoint.md
2. 读取所有产物文件
3. 逐项检查 R1~R6
4. 发现问题立即修复
5. 修复后重新审查（最多 3 次循环）
6. 通过后输出审查报告到 prd/${需求主题}/${result.stage}-review-report.md
7. 全部通过输出 ✅ 审查通过

## 审查规则

- R1. 字段完整性
- R2. 状态值一致性
- R3. API 一致性
- R4. 功能覆盖完整性
- R5. 业务逻辑一致性
- R6. 格式规范

重要：发现不一致问题时，使用 Edit/Write 工具直接修复文件，不要等到审查全部完成后再修复。`
})
```

inspector 自行更新 status.md "AI审查"=`[x]`。完成后 → 直接调 `Skill("dct-loop", args="... fast")`，不输出文字。

### 2.4 action: `pass` — 阶段通过

1. `Edit status.md`：当前阶段"用户确认"=`[✅]`
2. 直接调 `Skill("dct-loop", args="... fast")`，不输出文字

### 2.5 action: `reject` — 打回

1. `Edit status.md`：当前阶段"产物"=`[ ]`，"AI审查"=`[ ]`
2. 直接调 `Skill("dct-loop", args="... fast")`，不输出文字

### 2.6 action: `revise` — 修订

1. 派遣 sailor 修订模式：

```javascript
Agent({
  name: `${result.stepname}修订者`,
  description: `${result.stepname}修订 - ${需求主题}`,
  subagent_type: "dream-come-true:sailor:sailor",
  prompt: `/effort high

你是阶段执行 Agent，负责根据用户反馈修改产物。

## 输入参数
- 运行模式：revise
- 需求主题：${需求主题}
- 阶段名：${result.stepname}
- 当前阶段产物：${result.artifacts.join(', ')}
- 用户反馈：${result.feedback}

请根据用户反馈修改产物文件，完成后返回 JSON 格式结果。`
})
```

2. sailor 修订完成后，派遣 inspector 审查（同 2.3）
3. 直接调 `Skill("dct-loop", args="... fast")`，不输出文字

### 2.7 action: `done` — 完成

输出最终产物清单，流程结束。

### 2.8 action: `error` — 校验异常

`sailor` 返回但 dct-loop 发现产物未就绪时返回此 action。

- **深度模式**：用 `AskUserQuestion` 报告用户，等待指令
- **快速模式**：输出错误信息，流程中断（不阻塞等待）

---

## 三、断点续传

每次重新打开会话或继续推进，从入口开始。若 status.md 已存在，直接进入阶段循环；dct-loop 会定位到第一个未确认的阶段继续。

> 主线程对话上下文 + status.md 双重持久化：对话上下文负责"当前会话的工作记忆"，status.md 负责"跨会话的状态真理"。两者冲突时以 status.md 为准。

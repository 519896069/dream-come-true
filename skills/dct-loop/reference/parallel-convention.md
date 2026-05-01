# 并行阶段 Skill 约定

并行阶段 sailor 调用同一个 Skill 的多个实例，每个实例只负责一个执行单元。

---

## 一、输入约定

sailor prompt 中传入 `scope` 参数，Skill 据此确定本实例的职责范围：

| 字段 | 含义 | 示例 |
|------|------|------|
| `scope.key` | 执行单元 ID | `<unit-id>` |
| `scope.project` | 本实例负责的项目 | `<project>` |
| `scope.artifacts` | 本实例独享的产物文件/目录 | `<产物目录>` |
| `scope.plan_file` | 本项目的 plan 文件 | `prd/<需求>/<project>-plan.md` |

---

## 二、隔离约束

违反以下任一条 = 流程错误。

| 规则 | 说明 |
|------|------|
| P1. 写隔离 | 只能 Write/Edit 在 `scope.artifacts` 范围内的文件 |
| P2. 读隔离 | 可以 Read 任意文件，但优先用 `scope.plan_file` 获取本单元计划 |
| P3. 不写 status.md | 并行模式下 status.md 由 captain 在全部成功后统一写入 |
| P4. 不跨项目 | 不读写其他项目的产物目录 |
| P5. 独立分支 | 每个执行单元在自己的 git 分支上工作，分支名包含 `scope.key` |

---

## 三、返回约定

每个并行实例必须返回确定性的 pass/fail：

```json
{
  "key": "<unit-id>",
  "project": "<project>",
  "status": "完成" | "失败",
  "files": ["<本单元产物列表>"],
  "error": "<失败原因（仅失败时）>"
}
```

---

## 四、失败处理

- 本单元失败不波及并行中的其他单元
- 返回失败后，captain 单独重试本单元
- 重试时从 `scope.plan_file` 读取计划，重新执行

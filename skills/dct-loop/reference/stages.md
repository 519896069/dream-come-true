# 阶段配置表

| 阶段 | 阶段名称 | Skill | 固定产物 | Effort | AI审查 | 并行 |
|------|----------|-------|----------|--------|--------|------|
| 阶段一 | 需求标准化 | `dct-normalization` | `checkpoint.md`, `user-stories.md` | high | 跳过 | false |
| 阶段二 | 方案设计 | `dct-design` | `design.md`, `api.json`, `test-case.md` | **max** | 需要 | false |
| 阶段三 | 计划编写 | `dct-planning` | `{project}-plan.md` | **max** | 需要 | false |
| 阶段四 | 计划执行 | `dct-execution` | 各项目代码文件 | high | 需要 | **true** |
| 阶段五 | 代码审查 | `dct-review` | `review-log.md` | high | 需要 | false |

---

## 并行阶段拆分依据

当 `并行 = true` 时，dct-loop 读取上阶段产物 `{project}-plan.md` 中的"执行单元"表格，拆分为独立任务。

执行单元约定格式：

```markdown
## 执行单元

| 单元ID | 项目 | 产物目录 |
|--------|------|---------|
| U-001 | gxadmin-console | frontend/gxadmin-console/ |
| U-002 | api | backend/api/ |
```

每个执行单元对应一个并行 sailor 实例，产物目录天然隔离。

---

## 断点续传

| 阶段 | 续传方式 |
|------|----------|
| 阶段一~三 | 读取 `status.md`，重新执行该阶段 Skill |
| 阶段四 | 不支持断点续传（代码生成需重新执行） |
| 阶段五 | 读取 `status.md`，从"展示审查结果→等待用户确认"继续 |

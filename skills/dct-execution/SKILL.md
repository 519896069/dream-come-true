---
name: dct-execution
description: 阶段四：计划执行。基于实现计划，执行各项目代码生成。支持串行和并行两种模式。
argument-hint: "<需求主题> [scope.project]"
effort-recommendation: high
---

# 阶段四：计划执行

## 用户确认模式

| 模式 | 条件 | 第二步（分支确认） |
|------|------|-------------------|
| 深度模式（默认） | 执行模式 ≠ fast | AskUserQuestion 确认分支名 |
| 快速模式 | 执行模式 = fast | 自动生成分支名，跳过确认 |

## 并行/串行判断

根据 sailor prompt 中是否传入 `scope` 参数选择执行分支：

| 模式 | 条件 | 行为 |
|------|------|------|
| 并行模式 | `scope.project` 存在 | 只实现 `scope.project` 的项目，遵守 P1~P5 隔离约束 |
| 串行模式 | `scope` 不存在 | 逐个实现所有项目（保持现有流程） |

**并行模式约定**：详见 `dct-loop/reference/parallel-convention.md`。核心规则：
- P1. 只写 `scope.artifacts` 范围内的文件
- P2. 用 `scope.plan_file` 获取本单元计划
- P3. 不写 status.md（并行时由 captain 统一写）
- P4. 不读/写其他项目目录
- P5. git 分支名包含 `scope.key`

## 前置条件

- 阶段三已完成：各项目的 `prd/{date}-{需求}/{project_name}-plan.md` 已生成

## 代码生成完成标准

**编译通过 + 单元测试通过**

## 执行步骤

### 第一步：准备环境

1. 确保每个项目的 `dev` 分支最新：`git fetch` + `git pull`
2. 确认开发分支命名规则：`dev_v{迭代版本号}/feature_{topic}_{git_username}`

### 第二步：确认分支名称

#### 快速模式

自动生成分支名称，跳过用户确认：

```
dev_v{迭代版本号}/feature_{topic}_{git_username}
```

生成后直接进入第三步。

#### 深度模式（默认）

使用 AskUserQuestion 工具让用户确认开发分支名称后才能继续。不得使用标准输出文本提问。

### 第三步：执行代码实现

#### 并行模式（`scope.project` 存在）

只实现 `scope.project` 一个项目。不触碰其他项目。

**执行步骤：**

1. 切换到项目根目录
   - 如果是前端项目需要加载 Skill(impeccable:frontend-design)
2. 从 dev 分支拉功能分支 `dev_v{迭代版本号}/feature_{topic}_{git_username}-{scope.key}`
3. 读取 `scope.plan_file` 获取本单元计划
4. 调用 Skill(superpowers:writing-plans) 执行计划
5. 进行代码review，并修复review的问题，**保证符合review标准**
6. 进行单元测试，循环修复，直到**单元测试通过，编译通过**

**返回格式：**
```json
{
  "key": "U-001",
  "project": "gxadmin-console",
  "status": "完成",
  "files": ["frontend/gxadmin-console/..."]
}
```

#### 串行模式（`scope` 不存在）

按 plan 列表顺序，逐个执行所有项目的代码实现。

不派遣子 Agent。sailor 自己使用 Read/Write/Edit/Bash 工具，按项目顺序逐个实现代码。

**每个项目的执行步骤：**
同上（跳过分支名中的 `-{scope.key}` 后缀）。

---

**执行要求**
1. 必须确保逻辑完整性
2. plan计划会提供实现细节但是不一定对，需要实现时保证逻辑完整性和代码完整性
3. 单元测试必须覆盖所有代码分支
4. 每个函数必须是可测试的

**review要求**
1. 确保代码的正确性
 - 必须确保字段和真实数据库字段对应，且类型正确，调用Mcp(mysql)查看对照
 - bigint unsigned 字段，都使用了正确的onion.Id类型 而不是uint64类型
 - 代码分层正确
   - **不可以** service层直接调用数据库
   - **不可以** service层进行参数处理而不是controller层
   - **不可以** repository层过于复杂，里面存在大量业务逻辑

### 第四步：检查所有产物

按项目列表逐个检查：
1. 代码是否已实现
2. 编译是否通过
3. 单元测试是否通过

如有项目未通过，修复该项目代码实现，直到单元测试通过，编译通过。

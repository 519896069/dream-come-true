---
name: dct-review
description: 阶段五：四层审查流程（交叉检查 + API一致性 + 单元测试 + 编译测试），修复BUG。
argument-hint: "<需求主题>"
effort-recommendation: high
---

# 阶段五：审查

四层审查流程，逐层执行，任何一层失败则修复后重跑，全部通过后才进入用户确认。

---

## 第一层：数据库与 Entity 交叉检查

确保 Entity 定义与真实数据库表结构一致。

### 前置步骤：识别修改模块

1. **读取 plan 文件列表**
   ```bash
   ls prd/<需求主题>/*-plan.md
   ```

2. **从每个 plan.md 提取项目信息**
   - 项目路径（如 `backend/{project_name}/`、`frontend/{project_name}/`）
   - 模块名（如 `scheduled_backup_config`、`vault`）
   - 项目类型（Go/前端）

3. **构建测试矩阵**：收集所有项目路径和模块，按项目类型分组。

### 1.1 Entity 与数据库交叉验证

> 此检查必须通过 MCP MySQL 工具查询真实数据库，禁止仅做静态代码分析。

对每个修改过的 Entity 执行以下验证：

a. **提取 Entity 字段**：读取 Entity 文件，收集所有带 `gorm:"column:xxx"` tag 的字段名
   ```bash
   grep -oP 'gorm:"column:\K[^"]+' <entity_file>
   ```

b. **查询数据库表结构**：使用 MCP `mysql__list_tables` 工具获取表的列定义
   ```sql
   SELECT COLUMN_NAME FROM INFORMATION_SCHEMA.COLUMNS
   WHERE TABLE_SCHEMA = DATABASE() AND TABLE_NAME = '<table_name>'
   ```

c. **交叉比对**：
   - Entity 中 `gorm:"column:xxx"` 的 xxx 必须存在于数据库列中
   - 数据库中的列不强制要求在 Entity 中全部定义（允许 Entity 只映射部分列）
   - **不一致则报错并修复**：删除 Entity 中多余的字段，或提醒用户添加缺失的数据库列

d. **检查 json tag 与前端字段名一致性**：对比 Entity 的 `json` tag 与前端类型定义中的字段名

### 1.2 失败处理

发现不一致问题时，直接修复代码，重新检查直到全部通过（最多 3 次）。

---

## 第二层：API 一致性检查

确保 api.json（OpenAPI 3.0）、后端实现、前端代码、design.md 中的契约层代码四方一致。

### 2.1 api.json vs 后端接口

> 基准：`prd/<需求主题>/api.json`（OpenAPI 3.0 格式）

对每个修改过的接口执行以下验证：

a. **接口地址一致性**
   - 提取 api.json 中的接口路径和 HTTP 方法
   - 提取后端路由注册代码中的接口路径和 HTTP 方法
   - 对比两者是否完全一致（包括路径参数、HTTP 方法）

b. **请求字段一致性**
   - 提取 api.json 中 requestBody 的字段名、类型、required、校验规则
   - 提取后端 DTO/Request 结构体的字段定义（包括 `json` tag、`binding` tag）
   - 对比：字段名必须一致、类型必须匹配、必填约束必须对齐
   - 检查边界约束：api.json 中的 `minLength`/`maxLength`/`pattern`/`minimum`/`maximum` 与 binding tag 是否一致

c. **响应字段一致性**
   - 提取 api.json 中 responses 的字段名、类型
   - 提取后端 Entity/Response 结构体的字段定义
   - 对比：字段名必须一致、类型必须匹配、嵌套结构必须对齐

d. **不一致处理**
   - **以 api.json 为准**，修改后端代码使其与 api.json 一致
   - 同步修改受影响的单元测试
   - 重新检查直到全部通过（最多 3 次）

### 2.2 api.json vs 前端调用

> 基准：`prd/<需求主题>/api.json`（OpenAPI 3.0 格式）

对每个修改过的接口执行以下验证：

a. **TS 类型一致性**
   - 提取前端 `types/` 或 `interfaces/` 目录中的类型定义
   - 对比 api.json 中 schemas 定义的字段
   - 检查：字段名必须一致、类型必须匹配（`string`↔`string`、`number`↔`integer`、`boolean`↔`boolean`）
   - 检查：前端使用的字段必须在 api.json schema 中存在（不允许使用未定义的字段）

b. **API 调用一致性**
   - 提取前端 `api/` 或 `services/` 目录中的请求路径和方法
   - 与 api.json 中的路径和 HTTP 方法对比
   - 检查：路径参数是否正确传递、查询参数是否正确拼接

c. **表单一致性**
   - 若涉及表单提交，检查表单字段是否满足 api.json requestBody 的要求
   - 检查：表单字段名与 api.json 请求字段名一致
   - 检查：表单校验规则与 api.json 的约束一致（required、maxLength、pattern 等）
   - 检查：表单提交的数据结构与 api.json requestBody 结构一致

d. **不一致处理**
   - **以 api.json 为准**，修改前端代码使其与 api.json 一致
   - 同步修改受影响的单元测试
   - 重新检查直到全部通过（最多 3 次）

### 2.3 api.json vs design.md 契约层代码

> 确保 design.md 中的 Entity/DTO/TypeScript 代码与 api.json 一致。

a. **Entity vs api.json response schema**
   - Entity 的 json tag 字段名必须与 api.json response schema 的字段名一致

b. **DTO vs api.json requestBody**
   - DTO 的 json tag 字段名必须与 api.json requestBody 的字段名一致
   - DTO 的 binding tag 约束必须与 api.json 的 required/校验规则一致

c. **TypeScript 类型 vs api.json schema**
   - 前端 TypeScript 类型的字段必须与 api.json schema 的字段一致

d. **不一致处理**
   - **以 api.json 为准**，修改 design.md 中的代码使其与 api.json 一致

---

## 第三层：单元测试

验证 GORM 映射、JSON 往返、DB 约束等单元测试。

### 执行步骤（按项目遍历）

对测试矩阵中的**每个项目**执行以下步骤：

#### Go 后端项目

1. **单元测试**（针对修改的模块）
   ```bash
   go test -gcflags=all=-l ./module/<module_name>/... -v -cover
   ```
   记录：测试总数、通过数、覆盖率。

2. **失败处理**：分析失败原因，修复代码或测试，重新运行直到全部通过（最多 3 次）。

#### 前端项目

1. **单元测试 + 覆盖率**（针对修改的模块）
   ```bash
   npx jest --coverage --testPathIgnorePatterns="e2e" <module_name>
   ```
   记录：测试总数、通过数、语句覆盖率、分支覆盖率。
   要求：分支覆盖率 >= 80%。不足则补充测试用例。

2. **失败处理**：分析失败原因，修复代码或测试，重新运行直到全部通过（最多 3 次）。

### 测试覆盖验证

对比 plan.md 中的任务列表，检查每个模块是否有对应测试文件。缺失则补充。

---

## 第四层：编译测试

确保代码能正确编译，无语法错误和类型错误。

### 执行步骤（按项目遍历）

对测试矩阵中的**每个项目**执行以下步骤：

#### Go 后端项目

1. **编译检查**
   ```bash
   cd <project_path> && go build ./... && go vet ./...
   ```

2. **失败处理**：分析编译错误，修复代码，重新编译直到通过（最多 3 次）。

#### 前端项目

1. **编译检查**
   ```bash
   cd <project_path> && npx tsc --noEmit
   ```

2. **失败处理**：分析编译错误，修复代码，重新编译直到通过（最多 3 次）。

---

## 第五步：生成审查报告

输出 `prd/<需求主题>/review-log.md`，格式参考：

```markdown
# 审查验收报告 - <需求主题>

## 审查概览
| 项目 | 值 |
|------|-----|
| 需求主题 | <需求主题> |
| 审查日期 | <YYYY-MM-DD> |
| 审查范围 | <动态：列出所有审查的项目和模块> |

---

## 一、数据库与 Entity 交叉检查

<按项目遍历，每个项目一个子章节>

### 1.X <project_name>
- Entity 文件
- 数据库表
- 检查结果

---

## 二、API 一致性检查

### 2.1 API 定义 vs 后端接口

<按项目遍历，每个项目一个子章节>

#### 2.1.X <project_name>
- 接口地址一致性
- 请求字段一致性
- 响应字段一致性
- 修复记录

### 2.2 API 定义 vs 前端调用

<按项目遍历，每个项目一个子章节>

#### 2.2.X <project_name>
- TS 类型一致性
- API 调用一致性
- 表单一致性
- 修复记录

---

## 三、单元测试

<按项目遍历，每个项目一个子章节>

### 3.X <project_name>
- 测试总数
- 通过数
- 覆盖率

---

## 四、编译测试

<按项目遍历，每个项目一个子章节>

### 4.X <project_name>
- 编译结果

---

## 五、问题汇总
### 5.1 发现的问题
### 5.2 已修复的问题

---

## 六、审查结论
| 项目 | 交叉检查 | API一致性 | 单元测试 | 编译测试 |
|------|:-------:|:--------:|:-------:|:-------:|
| <project_1> | | | | |
| <project_2> | | | | |
```

---

## 重要规则

1. **每层失败都必须修复**：不能带着失败进入下一层
2. **最多重试 3 次**：同一层修复后重跑 3 次仍失败则报告用户
3. **API 定义为准**：API 定义（api.json）是唯一事实来源，后端和前端必须与之对齐
4. **修复即改代码**：发现的 bug 直接修复，不是记录后交给别人
5. **同步更新测试**：修复代码时必须同步更新受影响的单元测试

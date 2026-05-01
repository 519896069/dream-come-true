---
name: dct-design
description: 阶段二：方案设计。基于标准化需求，进行方案设计，产出审批通过的设计方案和 api.json。
argument-hint: "<需求主题>"
effort-recommendation: max
---

# 阶段二：方案设计

## 阶段产物

| 产物 | 说明 |
|------|------|
| `prd/{date}-{需求}/design.md` | 设计方案文档 |
| `prd/{date}-{需求}/api.json` | 接口设计文档（OpenAPI 3.0 格式） |
| `prd/{date}-{需求}/test-case.md` | 测试用例文档 |

---
**方案设计要求：**
1. 设计方案必须基于上一步骤的输出，不得与上一步骤的输出冲突
2. 设计方案必须符合用户需求，不得超出用户需求范围
3. 必须要有证据，不能凭空假设
4. 需要跟用户事无巨细确认需求点，所有问题使用 AskUserQuestion 工具呈现选项，不得使用标准输出文本提问
5. 需要包含每个项目之间数据流的流程图
6. 需要明确说明每个项目的功能点和接口定义
7. 最小范围读取文件，只读取涉及到的文件，不可以全项目扫描

**契约层代码要求（必须包含）：**
8. 必须包含 Entity 结构体定义（含 gorm tag、json tag），可直接复制使用
9. 必须包含 DTO 请求/响应结构体（含 binding tag），定义 API 契约
10. 必须包含 TypeScript 类型定义（interface/type），与 api.json schema 对应
11. 必须包含 SQL DDL（CREATE TABLE 语句）
12. **不包含实现细节**：不写 Controller/Service/Repository 实现、不写前端组件 JSX/TSX

**api.json 产出要求：**
13. 必须产出 `prd/{date}-{需求}/api.json`，遵循 OpenAPI 3.0.0 格式
14. 所有接口必须包含完整的 parameters、requestBody、responses 定义
15. 所有字段必须有中文 `description`
16. 需要鉴权的接口必须带 `security: [{ "BearerAuth": [] }]`
17. requestBody 必须包含校验规则（required、minLength、maxLength、pattern、minimum、maximum）
18. 每个接口必须定义错误响应（至少 401 和 500）
19. 重复结构必须抽取为 `components.schemas`，通过 `$ref` 引用

**test-case.md 产出要求：**
20. 必须覆盖 checkpoint.md 中所有验收点
21. 集成测试覆盖：功能测试、边界测试、流程测试
22. E2E 测试覆盖：用户操作路径、异常场景
23. 每个测试用例必须关联验收点（checkpoint ID）

## 执行步骤

### 第一步：使用 AskUserQuestion 工具询问用户，确定涉及的项目范围，调用 `Skill("standards", args="<项目范围>")`，获取项目规范。

使用 AskUserQuestion 工具跟用户确认可能涉及的项目，减少读取范围。所有选项必须使用 AskUserQuestion 呈现，不得使用标准输出文本提问。

### 第二步：方案设计

调用 `Skill("superpowers:brainstorming", args="<方案设计要求> <上阶段产物（从 status.md 读取）> <产物格式>")`，等待其返回标准化输出。

**输出文件：**
- `prd/{date}-{需求}/design.md`
- `prd/{date}-{需求}/api.json`
- `prd/{date}-{需求}/test-case.md`

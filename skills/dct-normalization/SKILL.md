---
name: dct-normalization
description: 需求标准化。规范化需求内容，生成用户故事和检查清单。
argument-hint: "<需求主题>"
effort-recommendation: high
---

# 阶段一：需求标准化

## 描述

step1、step2 ： 只做分析问答，不产出任何文件。
step3 ： 汇总上面的分析结果和用户回答，产出最终需求标准化结果。

## 推荐 workflow

```
write-spec  →  产出功能规格文档（问题陈述、用户故事粗纲）
     ↓
synthesize-research  →  补充用户洞察、验证需求假设
     ↓
手动/协作  →  产出细化的用户故事 + 验收标准
```

### 具体做法

**Step 1 — 调用 `Skill("product-management:write-spec", args="<需求验收点>")`**
把你有的需求描述、背景信息给它，让它帮你：
- 梳理需求背景和目标
- 识别缺失信息
- 给出初步用户故事框架

**Step 2 — 询问如果有用户调研数据，调用 `Skill("product-management:synthesize-research", args="<需求验收点>")`**

把访谈记录、问卷、反馈数据给它，帮你：
- 提炼用户核心诉求
- 验证需求真伪
- 补充验收维度

**Step 3 — 汇总上面的分析结果和用户回答，按照 `<产物格式>` 产出最终需求标准化结果**

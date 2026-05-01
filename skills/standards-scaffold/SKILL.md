---
name: standards-scaffold
description: Use to initialize a new empty standards project structure
---

# Standards Scaffold

初始化新项目的 standards 规范目录结构。

## 功能

需要在.claude/skills/目录中，创建完整的 standards 目录结构，并基于模板生成 SKILL.md 文件。

## 生成的目录结构

.claude/skills/目录下，创建完整的standards/目录结构。

```
standards/
├── SKILL.md
└── reference/
    ├── backend/
    │   ├── patterns/
    │   └── testing/
    ├── business/
    └── frontend/
```

## 映射模板

参考 `standards-scaffold/reference/SKILL.md` 模板，创建标准化的项目映射表和业务规则映射表。

## 使用方式

当用户说"创建 standards"、"初始化规范"、"新建 standards 项目"时使用此 skill。

## 执行步骤

1. 创建 `standards/` 目录
2. 创建 `standards/reference/` 目录
3. 创建 `standards/reference/backend/patterns/` 目录
4. 创建 `standards/reference/backend/testing/` 目录
5. 创建 `standards/reference/business/` 目录
6. 创建 `standards/reference/frontend/` 目录
7. 基于 `standards-scaffold/reference/SKILL.md` 模板生成 `standards/SKILL.md`

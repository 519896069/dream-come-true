# Dream Come True

一个 Claude Code 插件，将需求从 Idea 到代码实现的完整流程自动化。覆盖需求分析、方案设计、计划编写、代码生成、审查验收五个阶段。

## 功能特点

- **全流程编排**：从需求标准化到代码交付，一条命令走完
- **双模式支持**：深度模式（逐步确认）和快速模式（AI 自动决策）
- **断点续传**：会话中断后可从上次进度继续
- **并行执行**：支持多项目并行代码生成，自动隔离冲突
- **AI 审查**：自动检查产物一致性，发现问题自动修复

## 工作流阶段

| 阶段 | 说明 | 产物 |
|------|------|------|
| 阶段一：需求标准化 | 将模糊需求转化为结构化验收标准 | `checkpoint.md`、`user-stories.md` |
| 阶段二：方案设计 | 技术方案设计、API 定义、测试用例 | `design.md`、`api.json`、`test-case.md` |
| 阶段三：计划编写 | 拆分实现计划，支持并行任务规划 | `{project}-plan.md` |
| 阶段四：计划执行 | 按计划生成代码，支持 Git 分支管理 | 各项目代码文件 |
| 阶段五：审查 | 四层审查：交叉检查 + API 一致性 + 单元测试 + 编译测试 | `review-log.md` |

## 核心组件

### Skills

- **captain** — 主编排器，管理整个需求生命周期，调度各阶段执行
- **dct-loop** — 阶段循环决策核心，读取状态返回下一步 action
- **dct-design** — 方案设计阶段
- **dct-execution** — 代码执行阶段，支持串行/并行
- **dct-review** — 四层审查流程
- **dct-schema** — 产物格式定义查询
- **standards-scaffold** / **standards-business-analysis** — 标准化工具

### Agents

- **sailor** — 阶段执行 Agent，负责单个阶段的完整工作流
- **dct-inspector** — AI 审查 Agent，检查阶段产物一致性并自动修复

## 使用方式

在 Claude Code 中调用 captain skill 即可启动完整流程：

```
/captain <需求主题>
```

快速模式（跳过所有确认，AI 自动决策）：

```
/captain <需求主题> 快速模式
```

## 安装

```bash
# 克隆到 Claude Code 插件目录
cd ~/.claude/plugins.local
git clone https://github.com/519896069/dream-come-true.git
```

或通过 marketplace 安装：

```json
{
  "plugins": [
    {
      "source": "https://github.com/519896069/dream-come-true"
    }
  ]
}
```

## 项目结构

```
dream-come-true/
├── .claude-plugin/
│   ├── plugin.json          # 插件配置
│   └── marketplace.json     # 市场发布配置
├── agents/
│   ├── sailor/              # 阶段执行 Agent
│   │   ├── sailor.md
│   │   └── reference/       # 参考文档（业务规则、架构模式等）
│   └── dct-inspector/       # AI 审查 Agent
│       └── dct-inspector.md
├── skills/
│   ├── captain/             # 主编排器
│   ├── dct-loop/            # 阶段循环体
│   ├── dct-design/          # 方案设计
│   ├── dct-execution/       # 计划执行
│   ├── dct-review/          # 审查流程
│   ├── dct-schema/          # 产物 Schema
│   ├── dct-normalization/   # 需求标准化
│   ├── dct-planning/        # 计划编写
│   ├── standards-scaffold/  # 标准化脚手架
│   └── standards-business-analysis/  # 业务分析
└── README.md
```

## License

MIT

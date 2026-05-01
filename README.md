# Dream Come True

Claude Code 插件 —— 从 Idea 到代码，一条命令搞定。

## 安装

在 Claude Code 中执行：

```bash
claude plugin marketplace add dream-come-true
```

或手动添加到项目 `.claude/settings.json`：

```json
{
  "plugins": [
    {
      "source": "https://github.com/519896069/dream-come-true.git"
    }
  ]
}
```

## 使用

```
/captain <需求主题>
```

快速模式（AI 自动决策，跳过确认）：

```
/captain <需求主题> 快速模式
```

## 流转图

```
  用户需求
     │
     ▼
┌─────────────────┐
│ 阶段一：需求标准化  │  checkpoint.md / user-stories.md
└────────┬────────┘
         │ 用户确认
         ▼
┌─────────────────┐
│ 阶段二：方案设计    │  design.md / api.json / test-case.md
└────────┬────────┘
         │ AI审查 → 用户确认
         ▼
┌─────────────────┐
│ 阶段三：计划编写    │  {project}-plan.md
└────────┬────────┘
         │ AI审查 → 用户确认
         ▼
┌─────────────────┐
│ 阶段四：计划执行    │  代码文件（支持并行）
└────────┬────────┘
         │ AI审查 → 用户确认
         ▼
┌─────────────────┐
│ 阶段五：审查验收    │  review-log.md（四层审查）
└────────┬────────┘
         │ AI审查 → 用户确认
         ▼
       交付 ✅
```

## License

MIT

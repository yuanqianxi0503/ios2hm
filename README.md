# iOS → 鸿蒙功能迁移工具

## 工具简介
本工具基于 Claude Code 多 Agent 架构，自动完成 iOS 功能到鸿蒙端的代码迁移。
用户只需提供迁移任务描述和项目路径，工具自动分析 iOS 实现、制定鸿蒙开发方案、
逐功能点完成开发和编译验证，全程无需反复介入。

---

## 快速开始

**Step 1：将本工具目录加入 Claude Code 会话**

**Step 2：将 iOS 和鸿蒙项目目录加入会话**

/add-dir /path/to/ios-project
/add-dir /path/to/hm-project

**Step 3：填写迁移请求（可选，也可直接对话描述）**

编辑 .migration/request.md：

任务名称：{任务名称}
任务描述：{本次要迁移的页面或功能}
iOS 项目路径：{/path/to/ios}
鸿蒙项目路径：{/path/to/hm}

**Step 4：启动 Claude Code，开始自动迁移**

---

## 目录结构

自动化迁移工具/
├── .claude/
│   ├── settings.local.json                              ← Claude Code 的权限配置文件
│   ├── agents/
│   │   ├── ios-analyst.md                               ← iOS 分析师
│   │   ├── hm-developer.md                              ← 鸿蒙开发者
│   │   └── hm-build-fixer.md                            ← 鸿蒙编译修复专家
│   └── skills/
│       ├── ios-init-task-overview-note/SKILL.md         ← iOS 初始任务概述笔记 skill
│       ├── hm-init-dev-plan-note/SKILL.md               ← 鸿蒙初始开发方案笔记 skill
│       ├── ios-feature-note/SKILL.md                    ← iOS 某个功能点如何实现笔记 skill
│       ├── hm-feature-dev/SKILL.md                      ← 鸿蒙某个功能点开发 + 写笔记 skill
│       └── hm-build-and-fix/SKILL.md                    ← 鸿蒙编译验证与自动修复 skill
├── CLAUDE.md                                            ← 身份定义、整体流程、主循环逻辑
├── README.md                                            ← 项目说明
├── .gitignore                                           ← Git 忽略文件配置
├── .ios-notes/                                          ← iOS 端笔记（ios-analyst 可读写，其他可读）
│   ├── task-overview.md                                 ← 任务相关代码的目录结构、功能点列表
│   └── note-{feature-id}.md                             ← 各功能点实现细节（运行时生成）
├── .hm-notes/                                           ← 鸿蒙端笔记（hm-developer 可读写，其他可读）
│   ├── dev-plan.md                                      ← 鸿蒙开发方案、文件规划
│   ├── note-{feature-id}.md                             ← 各功能点开发笔记（运行时生成）
│   └── build-log-{feature-id}.md                        ← 各功能点编译修复日志（运行时生成）
├── .build-logs/                                         ← 编译日志（运行时生成）
│   └── F-{feature-id}/
│       ├── attempt-1.log                                ← 各次编译的原始日志
│       ├── attempt-2.log
│       └── attempt-3.log
└── .migration/                                          ← 迁移状态管理（主 agent 可读写）
    ├── task-status.md                                   ← 任务步骤规划、进度日志
    ├── request.md                                       ← 迁移任务描述、项目路径
    └── hm-dev-result.md                                 ← 临时结果文件

---

## 工作流程

每个功能点的完整流程：

1. **iOS 分析**：ios-analyst 分析 iOS 代码，生成实现笔记
2. **鸿蒙开发**：hm-developer 基于笔记实现鸿蒙代码
3. **编译验证**：hm-build-fixer 编译验证，自动修复编译错误（最多 3 次）
4. **下一功能点**：循环执行上述步骤，直到所有功能点完成

---

## 自动编译修复

工具内置编译验证与自动修复能力：

- ✅ 每个功能点开发完成后自动编译
- ✅ 自动修复常见编译错误（导入缺失、类型错误、语法错误等）
- ✅ 最多尝试 3 次修复，完整记录修复过程
- ✅ 无法自动修复时转人工介入

编译日志存放在：
- 原始日志：.build-logs/F-{feature-id}/attempt-*.log
- 修复记录：.hm-notes/build-log-{feature-id}.md

---

## 中断恢复

任务执行中途关闭会话后，重新启动 Claude Code 即可自动从上次中断处继续，
无需任何手动操作。恢复逻辑基于 .migration/task-status.md 中的进度日志。

---

## 何时需要人工介入

工具会在以下情况暂停并等待输入：
- 需要确认产品层面的决策（如 iOS 和鸿蒙在交互上是否保持一致）
- 需要确认鸿蒙特有的技术方案选择
- 编译错误修复超过 3 次仍失败
- 发现架构性问题（如循环依赖、模块缺失）

其他情况（iOS 笔记信息不足、规范有歧义、简单编译错误）工具会自主决策并在笔记中注明差异。

---

## 查看迁移结果

任务完成后，工具在对话中输出完成报告，包含：
- 已完成功能点列表
- 每个功能点创建/修改的文件
- 编译修复情况统计
- 与 iOS 实现的差异说明
- 遗留问题

详细内容可查看：
- 开发笔记：.hm-notes/note-{feature-id}.md
- 编译修复记录：.hm-notes/build-log-{feature-id}.md
- 任务进度：.migration/task-status.md

---

## 常见问题

**Q：编译失败时工具如何处理？**

A：工具会自动分析编译错误并尝试修复，最多尝试 3 次。若仍失败，会在对话中展示错误摘要并等待人工介入。

**Q：如何查看某个功能点的编译修复过程？**

A：查看 .hm-notes/build-log-{feature-id}.md，包含每次编译的错误摘要和修复措施。

**Q：编译日志会占用很多空间吗？**

A：工具采用策略 A（全部保留），便于复盘调试。如需清理，可手动删除 .build-logs/ 目录，不影响迁移结果。

**Q：工具是否支持自定义编译命令？**

A：是的，可在鸿蒙项目构建配置中自定义。工具默认使用 hvigorw assembleHap 增量编译。
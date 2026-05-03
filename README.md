# iOS → 鸿蒙功能迁移工具

## 工具简介
本工具基于 Claude Code 多 Agent 架构，自动完成 iOS 功能到鸿蒙端的代码迁移。
用户只需提供迁移任务描述和项目路径，工具自动分析 iOS 实现、制定鸿蒙开发方案、
逐功能点完成开发，全程无需反复介入。

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
│   ├── settings.local.json 							← Claude Code 的权限配置文件
│   ├── agents/
│   │   ├── ios-analyst.md  			 			← iOS分析师
│   │   └── hm-developer.md			 			← 鸿蒙开发者
│   └── skills/
│       ├── ios-init-task-overview-note/SKILL.md		←iOS初始 任务概述笔记skill
│       ├── hm-init-dev-plan-note/SKILL.md     		 	←鸿蒙初始 开发方案笔记skill
│       ├── ios-feature-note/SKILL.md       	 			←ios某个功能点如何实现笔记skill
│       ├── hm-feature-dev/SKILL.md       				←鸿蒙os某个功能点开发+写笔记skill
├── CLAUDE.md          				 			←身份定义、整体流程、主循环逻辑、
├── README.md								←项目说明
├── .gitignore                            					← Git忽略文件配置
├── .ios-notes/                   			 			←此文件夹内容ios端的subagent 可读写，鸿蒙subagent 可读，主agent可读。
    ├── task-overview.md						 		←该任务相关代码的目录结构；任务包含的功能点有哪些，各个功能点的实现概述。
    ├── note-{feature-id}.md				 			←（实际初始不存在，运行过程中为每个功能点动态生成，包含实现进度 和 实现方式 ）
├── .hm-notes/              			 				←此文件夹内容鸿蒙端的subagent 可读写，主agent可读。
    ├── dev-plan.md						 			←需要在项目的什么位置新增文件夹，文件夹的目录结构是什么，初始的开发方案
    ├── note-{feature-id}.md					 		←（实际初始不存在，运行过程中为每个功能点动态生成，包含实现进度 和 实现方式 ）
└── .migration/                  			 			←此文件夹内容主agent可读写，ios subagent 和 鸿蒙subagent 可读。
    ├── task-status.md						 		←主agent在此处动态规划 任务的步骤，记录各个步骤的完成情况（进度日志）。
    ├── request.md        			 					←初始人工提供的信息： ios & 鸿蒙项目位置，本次需要迁移的任务是什么，功能点有哪些。
    └── hm-dev-result.md            						← 鸿蒙开发结果（临时文件）

---

## 中断恢复

任务执行中途关闭会话后，重新启动 Claude Code 即可自动从上次中断处继续，
无需任何手动操作。恢复逻辑基于 .migration/task-status.md 中的进度日志。

---

## 何时需要人工介入

工具会在以下情况暂停并等待输入：
- 需要确认产品层面的决策（如 iOS 和鸿蒙在交互上是否保持一致）
- 需要确认鸿蒙特有的技术方案选择

其他情况（iOS 笔记信息不足、规范有歧义）工具会自主决策并在笔记中注明差异。

---

## 查看迁移结果

任务完成后，工具在对话中输出完成报告，包含：
- 已完成功能点列表
- 每个功能点创建/修改的文件
- 与 iOS 实现的差异说明
- 遗留问题

详细内容可查看 .hm-notes/ 目录下各功能点的笔记文件。
# Skill：鸿蒙编译验证与自动修复

## 我的职责

对指定功能点涉及的鸿蒙代码进行编译验证，
若存在编译错误则自动修复，
最多尝试 5 次修复，
超过则转人工介入。

## ⚠️ 执行承诺

在开始任何操作之前，先在内部确认以下承诺：
- 我不会在写入 .migration/hm-dev-result.md 之前停止
- "我将要执行编译"不等于"我已执行编译"，说完必须立即执行
---

## 输入

从调用方 prompt 中读取：
- feature_id：当前功能点 ID（如 F-002）

读取以下文件：
- .hm-notes/note-{feature-id}.md：了解该功能点创建/修改了哪些文件
- .migration/request.md：获取鸿蒙项目路径
- .hm-notes/build-log-{feature-id}.md：若存在，读取历史修复记录

---

## 执行流程

本 Skill 采用三段式执行,必须执行到最终收尾环节，在此之前不得退出：
1. 阶段一：准备 - 检查环境，初始化变量
2. 阶段二：主循环 - 编译 → 判断 → 修复（最多循环 5 次）
3. 阶段三：收尾 - 根据执行结果选择对应的收尾流程

---

### 阶段一：准备

**1. 读取项目路径**

从 .migration/request.md 中读取：
- HM_PROJECT_PATH = 鸿蒙项目路径
- TOOL_ROOT = 工具项目路径

后续所有操作中：
- 笔记和日志文件路径基于 TOOL_ROOT（或使用相对路径，因为当前工作目录就是 TOOL_ROOT）
- 编译命令在 HM_PROJECT_PATH 下执行
- 修复代码在 HM_PROJECT_PATH 下修改

**2. 检查构建工具**

执行：(cd {HM_PROJECT_PATH} && hvigorw --version)

若失败，进入【阶段三：收尾 - 异常退出】：
  reason: 鸿蒙项目构建工具不可用，项目路径：{HM_PROJECT_PATH}

**3. 初始化变量**

读取 .hm-notes/build-log-{feature-id}.md（若存在）获取已尝试次数
若不存在，attempt_count = 0
否则，从日志中提取 attempt_count

若 attempt_count >= 5，进入【阶段三：收尾 - 超限退出】

**4. 创建日志目录**

执行：mkdir -p .build-logs/F-{feature-id}/

完成准备后，进入【阶段二：主循环】

### 阶段二：主循环

本阶段是一个循环，每次循环包含：编译 → 解析 → 修复

循环条件：attempt_count < 5

**每次循环的执行步骤：**

---

#### 循环步骤 1：执行编译

attempt_count 递增 1

执行编译命令（注意：重定向在子 shell 外部，确保日志写入自动化工具项目）：
(cd {HM_PROJECT_PATH} && hvigorw assembleHap --mode module -p product=default --analyze=normal --parallel --incremental --daemon 2>&1) > .build-logs/F-{feature-id}/attempt-{attempt_count}.log

关键点：
- 这里是hvigorw 而不是./hvigorw,因为系统的.zshrc里配置了：alias hvigorw='node /Applications/DevEco-studio.app/Contents/Tools/hvigor/bin/hvigorw.js'
- cd 在子 shell 内部，只影响编译命令
- 2>&1 在子 shell 内部，合并输出流
- > 重定向在子 shell 外部，基于自动化工具项目的当前目录

读取日志文件：
.build-logs/F-{feature-id}/attempt-{attempt_count}.log

若文件不存在或为空，进入【阶段三：收尾 - 异常退出】：
  reason: 编译命令执行失败，未生成日志文件

---

#### 循环步骤 2：解析编译结果

在日志中搜索关键字：
- 成功标志：BUILD SUCCESS
- 失败标志：ERROR、BUILD FAILED

判断结果：
1. 若包含 BUILD SUCCESS → 进入【阶段三：收尾 - 成功收尾】
2. 若包含 ERROR 或 BUILD FAILED → 继续【循环步骤 3】
3. 若都不包含 → 进入【阶段三：收尾 - 异常退出】，reason: 构建状态不明确

---

#### 循环步骤 3：提取错误信息

使用 grep 提取错误上下文：
grep -n -B 2 -A 2 "ERROR" .build-logs/F-{feature-id}/attempt-{attempt_count}.log

存储为 ERROR_CONTEXT

若未提取到任何内容，进入【阶段三：收尾 - 异常退出】：
  reason: 编译失败但未找到错误信息

---

#### 循环步骤 4：分析错误类型

从 ERROR_CONTEXT 中提取：
- 错误文件路径和行号
- 错误类型（导入缺失、类型错误、参数错误、语法错误、模块错误）
- 错误描述

统计各类错误数量，存储为 ERROR_SUMMARY

检查重复错误（对比上一次编译的错误）：
若检测到重复错误，进入【阶段三：收尾 - 超限退出】：
  reason: 发现重复错误，自动修复无效

---

#### 循环步骤 5：执行修复

修复策略：
1. 只修复编译错误，不改变业务逻辑
2. 一次只修复一类错误（ERROR_SUMMARY 中数量最多的）
3. 参照 .hm-notes/dev-plan.md 中的规范
4. 参照鸿蒙项目中已有的同类实现（使用 Grep 查找）

常见错误的修复方向：
- 导入缺失：搜索项目中该符号的已有导入，添加到错误文件
- 类型错误：添加类型注解（优先使用明确类型，复杂情况用 any）
- 参数错误：查找方法定义，补充参数或修正调用
- 语法错误：直接修正（分号、括号、拼写）
- 模块错误：执行依赖安装（仅尝试一次）
  依赖安装命令：(cd {HM_PROJECT_PATH} && hvigorw clean && ohpm install && cd entry && ohpm install)

修复后记录到 FIXED_FILES 列表：
- 文件路径
- 错误类型
- 修复内容（简要描述，不超过 50 字）

---

#### 循环步骤 6：更新修复日志

初始化日志文件（若不存在）：
创建 .hm-notes/build-log-{feature-id}.md

初始内容：
# 编译修复日志：F-{feature-id}

## 总修复次数：0 / 5

---

追加本次修复记录：
### 第 {attempt_count} 次编译
时间：{时间戳，格式：YYYY-MM-DD HH:mm}

**编译结果：** 失败

**错误摘要：**
{ERROR_SUMMARY 的格式化输出，每类错误一行，示例：}
- 导入缺失：3 处
- 类型错误：1 处

**本次修复重点：**
{修复数量最多的错误类型}（共 {数量} 处）

**已修复：**
{FIXED_FILES 的格式化输出，每个文件一行，示例：}
- entry/src/main/ets/pages/StockList.ets：添加 promptAction 导入

**未修复（需人工介入）：**
{列出跳过的错误，若无则写"无"}

**完整日志路径：**
.build-logs/F-{feature-id}/attempt-{attempt_count}.log

---

更新总修复次数为：{attempt_count} / 5

---

#### 循环步骤 7：判断是否继续循环

若 attempt_count < 5：
  回到【循环步骤 1】（重新编译验证）

若 attempt_count >= 5：
  进入【阶段三：收尾 - 超限退出】

---

### 阶段三：收尾

根据执行情况，选择以下三种收尾流程之一：

---

#### 收尾情况1：成功收尾

**触发条件：** 编译通过（日志中包含 BUILD SUCCESS）

**执行步骤：**

1. 更新修复日志

若 .hm-notes/build-log-{feature-id}.md 存在，追加：
### ✅ 编译成功
时间：{时间戳}
总修复次数：{attempt_count}
最终日志：.build-logs/F-{feature-id}/attempt-{attempt_count}.log

若不存在（首次编译即通过），创建日志：
# 编译修复日志：F-{feature-id}

## 总修复次数：0 / 5

### ✅ 编译成功
时间：{时间戳}
初次编译即通过，无需修复
最终日志：.build-logs/F-{feature-id}/attempt-1.log

2. 写入执行结果

写入 .migration/hm-dev-result.md（完整覆盖）：
status: completed
feature_id: F-{feature-id}
reason: 编译通过{若 attempt_count > 0，追加：（修复 {attempt_count} 次）}

3. 结束执行

---

#### 收尾情况2：异常退出

**触发条件：** 遇到无法处理的异常情况（如工具不可用、日志文件未生成、状态不明确等）

**执行步骤：**

1. 更新修复日志（若已有编译尝试）

若 .hm-notes/build-log-{feature-id}.md 存在，在末尾追加：
### ⚠️ 异常终止
时间：{时间戳}
已尝试次数：{attempt_count}
异常原因：{具体原因}

若不存在（准备阶段就失败），创建日志：
# 编译修复日志：F-{feature-id}

## 总修复次数：0 / 5

### ⚠️ 异常终止
时间：{时间戳}
已尝试次数：0
异常原因：{具体原因}

2. 写入执行结果

写入 .migration/hm-dev-result.md（完整覆盖）：
status: need_human_info
feature_id: F-{feature-id}
reason: {异常原因}。{若有日志，追加：详细日志：.build-logs/F-{feature-id}/attempt-{attempt_count}.log}

3. 结束执行

---

#### 收尾情况3：超限退出

**触发条件：** 修复次数达到 5 次仍未通过，或发现重复错误

**执行步骤：**

1. 更新修复日志

在 .hm-notes/build-log-{feature-id}.md 末尾追加：
### ❌ 修复失败
时间：{时间戳}
已尝试 {attempt_count} 次修复仍未编译通过

**失败原因：**
{若是重复错误：连续出现相同错误，自动修复无效}
{若是达到上限：已达最大修复次数 5 次}

**最后一次错误摘要：**
{ERROR_SUMMARY 的格式化输出}

**完整日志路径：**
.build-logs/F-{feature-id}/attempt-{attempt_count}.log

**建议：**
- 检查是否存在架构性问题（如循环依赖）
- 人工检查日志文件中的核心错误
- 确认是否需要调整 iOS 笔记或鸿蒙开发方案

2. 写入执行结果

写入 .migration/hm-dev-result.md（完整覆盖）：
status: need_human_info
feature_id: F-{feature-id}
reason: 编译错误修复失败。核心错误类型：{ERROR_SUMMARY 中最多的前 2 种}。详细日志：.build-logs/F-{feature-id}/attempt-{attempt_count}.log。修复记录：.hm-notes/build-log-{feature-id}.md

3. 结束执行

---

## 约束

- 编译命令格式：(cd {HM_PROJECT_PATH} && ... 2>&1) > .build-logs/F-{feature-id}/attempt-{n}.log
  cd 在子 shell 内，重定向在子 shell 外，确保日志写入 TOOL_ROOT
- 修复代码时使用 HM_PROJECT_PATH 的绝对路径定位文件
- 笔记和日志写入时使用相对路径（基于 TOOL_ROOT）或绝对路径
- 禁止在 HM_PROJECT_PATH 下创建 .build-logs/、.hm-notes/ 等目录
- .build-logs/ 目录已在项目初始化时创建
- 每个功能点使用独立子目录：.build-logs/F-{feature-id}/
- 全部保留所有编译日志（策略 A），不删除任何 attempt-*.log
- 最多尝试 5 次修复
- 每次修复只聚焦一类错误
- 不修改 IOS_ROOT 中的任何文件
- 不修改 TOOL_ROOT/.migration/（除 hm-dev-result.md）
- 不修改 TOOL_ROOT/.ios-notes/
- 不修改 TOOL_ROOT/.hm-notes/note-{feature-id}.md 和 dev-plan.md
- 只修复编译错误，不改变业务逻辑
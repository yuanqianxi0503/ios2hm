# Skill：鸿蒙单次编译日志判断与修复

## 我的职责

读取主 Agent 已生成的一次鸿蒙编译日志，判断本次编译是否通过。

- 若编译通过：写入编译成功结果。
- 若编译失败且仍可自动修复：根据本次日志修复一次编译错误，写入 fixed 结果，交由主 Agent 重新编译。
- 若无法修复或已超过最大自动修复次数：写入 need_human_info 结果。

本 Skill 不执行编译命令，不负责循环验证。编译循环由 CLAUDE.md 的 hm_build 步骤控制。

## 执行承诺

在开始任何操作之前，先在内部确认以下承诺：
- 我不会执行 hvigorw、ohpm install 或任何编译 / 构建命令。
- 我不会在写入 .migration/hm-dev-result.md 之前停止。
- 我每次只处理一个 build_log_path，并且最多修复一轮。

---

## 输入

从调用方 prompt 中读取：
- feature_id：当前功能点 ID（如 F-002）
- attempt_no：当前是第几次编译（如 1）
- build_log_path：本次编译日志路径（如 .build-logs/F-002/attempt-1.log）
- max_attempts：最大自动修复次数，默认 5

读取以下文件：
- {build_log_path}：本次编译日志
- .hm-notes/note-{feature-id}.md：了解该功能点创建 / 修改了哪些文件
- .hm-notes/dev-plan.md：了解鸿蒙项目开发规范
- .migration/request.md：获取鸿蒙项目路径
- .hm-notes/build-log-{feature-id}.md：若存在，读取历史修复记录

---

## 执行流程

### Step 1：准备与校验

1. 从 .migration/request.md 中读取：
   - HM_PROJECT_PATH = 鸿蒙项目路径
   - TOOL_ROOT = 工具项目路径

2. 读取 build_log_path。

3. 若 build_log_path 不存在或为空：
   - 写入 .hm-notes/build-log-{feature-id}.md，记录"日志缺失或为空"
   - 写入 .migration/hm-dev-result.md：

```text
status: need_human_info
feature_id: {feature_id}
reason: 编译日志不存在或为空：{build_log_path}
```

4. 若日志存在且非空，继续 Step 2。

---

### Step 2：判断编译结果

在日志中搜索关键字：
- 成功标志：BUILD SUCCESS
- 失败标志：ERROR、BUILD FAILED、Hvigor ERROR、hvigor ERROR

判断规则：

1. 若日志包含 BUILD SUCCESS：
   - 不修复代码
   - 写入 .hm-notes/build-log-{feature-id}.md 成功记录
   - 写入 .migration/hm-dev-result.md：

```text
status: completed
feature_id: {feature_id}
reason: 编译通过，日志：{build_log_path}
```

2. 若日志包含失败标志：
   - 若 attempt_no > max_attempts，进入 Step 5：转人工
   - 否则进入 Step 3：分析并修复一次

3. 若日志既不包含成功标志，也不包含失败标志：
   - 写入 .hm-notes/build-log-{feature-id}.md，记录"构建状态不明确"
   - 写入 .migration/hm-dev-result.md：

```text
status: need_human_info
feature_id: {feature_id}
reason: 构建状态不明确，日志：{build_log_path}
```

---

### Step 3：分析错误

从日志中提取错误上下文，优先关注：
- ERROR 前后 2 行
- BUILD FAILED 附近内容
- Hvigor ERROR / hvigor ERROR 附近内容
- 明确包含文件路径、行号、模块名、符号名的错误行

将错误归类为以下类型之一：
- 导入缺失
- 类型错误
- 参数错误
- 语法错误
- 模块 / 依赖错误
- 路由 / 页面注册错误
- 资源引用错误
- 其他无法判断的错误

若无法提取有效错误信息：
- 写入 .hm-notes/build-log-{feature-id}.md，记录"编译失败但未找到有效错误信息"
- 写入 .migration/hm-dev-result.md：

```text
status: need_human_info
feature_id: {feature_id}
reason: 编译失败但未找到有效错误信息，日志：{build_log_path}
```

若在历史 build-log 中发现同一个文件、同一类错误已经修复过但仍重复出现：
- 不继续盲目修复
- 写入 status: need_human_info，reason 说明重复错误和日志路径

---

### Step 4：修复一次

修复策略：
1. 只修复编译错误，不改变业务逻辑。
2. 一次只聚焦最明确或数量最多的一类错误。
3. 优先修复本功能点已创建 / 修改的文件。
4. 参照 .hm-notes/dev-plan.md 中的规范。
5. 参照鸿蒙项目中已有同类实现，使用 Grep / Glob 查找，不自行发明项目规范。

常见错误的修复方向：
- 导入缺失：搜索项目中该符号的已有导入方式，添加到错误文件。
- 类型错误：补充明确类型或修正类型转换；复杂且局部无业务风险时才使用 any。
- 参数错误：查找方法定义，补充参数或修正调用。
- 语法错误：修正括号、分号、拼写、装饰器位置等语法问题。
- 模块 / 依赖错误：只修正明显的 import 路径、模块名或配置引用；不得执行安装命令。
- 路由 / 页面注册错误：参照已有页面注册方式补齐路由、页面路径或入口配置。
- 资源引用错误：参照已有资源命名和引用方式修正路径或资源名。

修复完成后，写入 .hm-notes/build-log-{feature-id}.md：

```markdown
### 第 {attempt_no} 次编译
时间：{YYYY-MM-DD HH:mm}

**编译结果：** 失败

**日志路径：**
{build_log_path}

**错误摘要：**
- {错误类型}：{核心错误描述}

**本次修复：**
- {文件路径}：{修复内容}

**后续动作：**
交由主 Agent 重新编译验证。

---
```

然后写入 .migration/hm-dev-result.md：

```text
status: fixed
feature_id: {feature_id}
reason: 已根据 {build_log_path} 修复一次：{错误类型}，请重新编译
```

---

### Step 5：转人工

以下情况进入本步骤：
- attempt_no > max_attempts 且日志仍失败
- 错误需要产品 / 架构决策
- 错误来自缺失模块、缺失依赖、构建环境异常，无法通过代码局部修复
- 重复错误说明自动修复无效
- 修复会改变业务逻辑

写入 .hm-notes/build-log-{feature-id}.md：

```markdown
### 第 {attempt_no} 次编译
时间：{YYYY-MM-DD HH:mm}

**编译结果：** 失败，需人工介入

**日志路径：**
{build_log_path}

**失败原因：**
{具体原因}

**建议：**
- 人工检查本次日志中的核心错误
- 确认是否需要调整 iOS 笔记、鸿蒙开发方案或项目依赖

---
```

写入 .migration/hm-dev-result.md：

```text
status: need_human_info
feature_id: {feature_id}
reason: {具体原因}。详细日志：{build_log_path}。修复记录：.hm-notes/build-log-{feature-id}.md
```

---

## 输出文件

### 文件 ①：.hm-notes/build-log-{feature-id}.md

每次处理日志都必须追加记录。
若文件不存在，先创建：

```markdown
# 编译修复日志：{feature_id}

---
```

### 文件 ②：.migration/hm-dev-result.md

每次退出前完整覆盖写入，不追加。

可能的 status：
- completed：本次日志显示编译通过
- fixed：本次日志显示编译失败，已修复一次，主 Agent 应重新编译
- need_human_info：无法自动修复或需要人工介入

格式：

```text
status: completed | fixed | need_human_info
feature_id: F-xxx
reason: {说明}
```

---

## 约束

- 不执行编译命令。
- 不执行依赖安装命令。
- 不删除任何 attempt-*.log。
- 不修改 IOS_ROOT 中的任何文件。
- 不修改 TOOL_ROOT/.migration/（除 hm-dev-result.md）。
- 不修改 TOOL_ROOT/.ios-notes/。
- 不修改 TOOL_ROOT/.hm-notes/note-{feature-id}.md 和 dev-plan.md。
- 修复代码时使用 HM_PROJECT_PATH 的绝对路径定位文件。
- 笔记和日志写入时使用相对路径（基于 TOOL_ROOT）或绝对路径。
- 只修复编译错误，不改变业务逻辑。

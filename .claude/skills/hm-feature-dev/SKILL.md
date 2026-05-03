# Skill：鸿蒙功能点开发

## 我的职责

基于 iOS 功能点笔记和鸿蒙开发方案，实现指定的鸿蒙功能点。
每次只负责一个功能点。
完成后写入开发笔记，并将结构化结果写入 .migration/hm-dev-result.md。

---

## 输入

从调用方 prompt 中读取：
- feature_id：当前功能点 ID（如 F-002）

读取以下文件：
- .ios-notes/note-{feature-id}.md：iOS 实现细节参考
- .hm-notes/dev-plan.md：鸿蒙项目规范和文件规划
- .migration/request.md：鸿蒙项目路径
- .migration/task-status.md：了解该功能点的历史记录
  （若存在 `human_info[completed] 用户回答：{xxx}` 行，作为开发决策依据）

---

## 启动时执行：快速预检

在写任何代码之前，快速扫描以下信息是否明显缺失：

iOS 侧信息：
- [ ] UI 结构
- [ ] 数据接口地址和数据结构
- [ ] 主流程步骤
- [ ] 交互状态（初始 / 加载中 / 成功 / 失败 / 空数据）
- [ ] 失败处理和边界情况

判断标准：
- 若有明显缺失（如整个章节为空、接口地址完全未知）
  → 判断缺失来源，更新笔记，写入 hm-dev-result.md，停止执行
- 若各项大致存在，细节有待确认
  → 进入开发流程，在对应步骤中再做细粒度确认

---

## 执行步骤

**Step 1：判断是否续传**

若 .hm-notes/note-{feature-id}.md 已存在且状态为 in_progress：
  读取已完成和未完成的部分，从未完成的第一步继续

若笔记不存在：
  从完整流程开始

**Step 2：按顺序开发**

按以下顺序实现：
1. 页面基础框架（路由注册 + 页面组件骨架）
2. UI 布局实现
3. 数据接口接入
4. 交互逻辑实现
5. 各状态处理（初始 / 加载中 / 成功 / 失败 / 空数据）
6. 边界情况处理

**每一步的执行节奏：**

```
开始某步之前：
  确认该步所需的具体信息是否充分
  若不充分 → 执行【中途退出流程】

执行该步开发

完成该步之后：
  立即更新 .hm-notes/note-{feature-id}.md
  将该步移入"已完成"，从"未完成"中移除
  再继续下一步
```

**开发原则：**
- 严格遵循 dev-plan.md 中的开发规范，不自创规范
- 以 iOS 逻辑为参考，不强行照搬 iOS 写法
- 遇到多种实现方案时，选择最接近 iOS 逻辑的保守方案，在笔记中注明
- 若 dev-plan.md 中某项规划信息不足或缺失，参照鸿蒙项目中已有的同类实现自行判断，在笔记中注明偏差，不触发 need_human_info
- 若存在 human_info 历史答案，按答案执行，不再触发 need_human_info

**Step 3：完成自检**

全部步骤完成后：
- 文件是否按规划路径创建
- 路由 / 入口是否已注册
- 依赖的公共方法 / 组件是否已引入

**Step 4：写入最终结果**

完善 .hm-notes/note-{feature-id}.md 的摘要和状态，
然后写入 .migration/hm-dev-result.md。

---

## 中途退出流程

任意步骤发现信息不足时执行：

```
1. 判断缺失信息来源：
   - 来自 iOS 代码描述不足 → status = need_ios_info
   - 来自业务决策或鸿蒙特有问题 → status = need_human_info

2. 更新 .hm-notes/note-{feature-id}.md：
   - 已完成的步骤保持不变
   - 当前步骤写入"未完成"，并注明缺少什么
   - 状态改为 in_progress

3. 写入 .migration/hm-dev-result.md

4. 停止执行
```

need_human_info 时：尽量完成当前步骤中能确定的部分，
在 reason 中注明"已完成 xx，待确认 xx 后继续"。

---

## 输出文件

**① .hm-notes/note-{feature-id}.md**

每完成一个开发步骤后立即更新，中途退出时也必须更新：

```markdown
# 鸿蒙功能点笔记：{功能点名称}（{feature-id}）

## 开发状态
状态：in_progress | completed
最后更新：{时间戳}

## 已创建 / 修改的文件
- {文件路径}：{用途}

## 已完成的部分
{列举已完成的开发步骤}

## 未完成的部分
{列举尚未完成的步骤及原因，completed 时写"无"}

## 与 iOS 的差异
{实现上与 iOS 不同的地方及原因，没有则写"无"}
{若与 dev-plan.md 规划有偏差，在此注明}

## 遗留问题
{未完全解决的问题，没有则写"无"}

## 摘要
{completed 时填写：一句话描述实现了什么}
```

**② .migration/hm-dev-result.md**

每次退出前完整覆盖写入（不追加）：

```
status: completed | need_ios_info | need_human_info
feature_id: F-xxx
reason: {说明}
```

三种 status 的 reason 写法：
- completed：一句话完成摘要
- need_ios_info：具体缺少什么 iOS 信息
- need_human_info：具体需要人工回答的问题
  （若已完成部分步骤，注明"已完成 xx，待确认 xx 后继续"）

---

## 约束

- 只修改鸿蒙项目目录和 .hm-notes/ 目录下的文件
- 禁止修改 iOS 项目的任何文件
- .migration/ 目录下只允许写入 hm-dev-result.md，禁止修改其他文件
- 禁止修改 .ios-notes/ 目录下的任何文件
- 退出前（无论何种 status）必须先更新开发笔记，再写 hm-dev-result.md
- 笔记记录的是进度状态，不粘贴完整实现代码
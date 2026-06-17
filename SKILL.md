---
name: chaoxing-assistant
description: Automate 超星学习通 (Chaoxing) operations — login, browse courses, complete and submit assignments via playwright-cli.
allowed-tools: Bash(playwright-cli:*) Bash(npx:*) Bash(npm:*)
---

# 超星学习通自动化操作

使用 playwright-cli 浏览器自动化完成超星学习通（i.chaoxing.com / mooc1.chaoxing.com）的登录、课程浏览、作业作答与提交。

## 前置条件

确保 `playwright-cli` 已全局安装：

```bash
npm install -g @playwright/cli@latest
```

## 整体工作流

```
登录 → 进入课程列表 → 找到目标课程 → 进入作业页 → 逐题作答 → 保存并提交
```

---

## 一、登录

### 1.1 打开登录页

超星学习通入口 URL 格式通常为 `https://i.chaoxing.com/base?t=<timestamp>`。直接打开后会自动跳转到登录页：

```bash
playwright-cli open --browser=chrome "https://i.chaoxing.com/base?t=<timestamp>"
```

### 1.2 登录页识别

登录页面 URL 为 `passport2.chaoxing.com/login`，包含：
- 手机号/超星号文本框
- 学习通密码文本框
- 「登录」按钮
- 右侧可选的 APP 扫码登录

### 1.3 填写账号密码并登录

```bash
# 先 snapshot 获取 ref
playwright-cli snapshot

# 填写手机号（通常是 ref=e10 的 textbox "手机号/超星号"）
playwright-cli fill e10 "手机号"

# 填写密码（通常是 ref=e12 的 textbox "学习通密码"）
playwright-cli fill e12 "密码"

# 点击登录按钮（通常是 ref=e17 的 button "登录"）
playwright-cli click e17
```

> **注意**：每次打开页面 ref 可能不同，务必先 snapshot 确认 ref 编号。

---

## 二、进入课程列表

登录成功后进入个人空间（i.chaoxing.com/base），左侧菜单栏有「课程」菜单项。

```bash
# snapshot 确认页面已加载
playwright-cli snapshot

# 点击左侧菜单的「课程」
playwright-cli click "getByRole('menuitem', { name: '课程' })"
```

> **备选方案**：如果 menuitem 名称带图标字符（如 ` 课程`），需要包含图标字：
> ```bash
> playwright-cli click "getByRole('menuitem', { name: ' 课程' })"
> ```

### 课程列表结构

课程列表在右侧的 iframe 中展示。每个课程卡片包含：
- 课程封面图
- 课程名称（link）
- 学校/教师名
- 开课时间（可能）

直接通过课程 URL 导航可绕过 iframe 操作，更快进入目标课程。

---

## 三、进入目标课程

### 方式 A：从 snapshot 中找到课程链接的 URL，直接 goto（推荐）

在课程列表 snapshot 中找到目标课程（如「C语言程序设计进阶与实践」），提取其链接 URL，直接导航：

```bash
playwright-cli goto "https://mooc1.chaoxing.com/visit/stucoursemiddle?courseid=XXX&clazzid=XXX&cpi=XXX&ismooc2=1&v=2"
```

### 方式 B：通过 locator 点击

课程在 iframe 内，需要使用 `iframe` content frame 定位：

```bash
playwright-cli click "getByRole('link', { name: '课程名称' })"
```

---

## 四、进入作业页面

课程页面左侧有导航栏，包含：课程门户、AI助教、任务、章节、讨论、**作业**、考试、资料等。

```bash
# snapshot 确认课程页面加载
playwright-cli snapshot

# 点击「作业」
playwright-cli click "getByRole('link', { name: '作业' })"
```

作业列表显示在 iframe 中，每个作业项包含：
- 作业名称（如「第六周 课后作业」）
- 状态（未交 / 已完成 / 待批阅）
- 可能有「智能分析」「作答记录」等链接

---

## 五、打开目标作业

```bash
# snapshot 确认作业列表
playwright-cli snapshot

# 点击目标作业（在 iframe 中，如 ref=f26e30）
playwright-cli click f26e30
```

作业通常会在**新标签页**中打开（URL 为 `mooc1.chaoxing.com/mooc-ans/mooc2/work/dowork`）。

```bash
# 切换到新标签页
playwright-cli tab-select 1
playwright-cli snapshot
```

---

## 六、阅读并作答

### 作业页面结构

页面顶部有「暂时保存」和「提交」按钮。主体包括：
- 作业标题和题量/满分信息
- 题目列表（18 道题可滚动）
- 底部页码导航

### 两种题目类型

#### 类型 A：简答题（富文本编辑器）

每道简答题包含一个 UEditor 富文本编辑器，嵌入在 `iframe#ueditor_N` 中（N 从 0 开始）。

**核心作答方法**——通过 eval 直接向 iframe 的 body 写入 HTML：

```bash
playwright-cli eval \
  "el => { const doc = el.contentDocument || el.contentWindow.document; if (doc?.body) { doc.body.innerHTML = '<p>答案内容</p>'; return 'done'; } return 'failed'; }" \
  e<iframe_ref>
```

> **关键要点**：
> - 答案内容用 HTML 格式：`<p>段落</p>`、`<strong>加粗</strong>`、`<pre>代码块</pre>`
> - iframe 的 ref 从 snapshot 中找到（如 e174, e322, e470...）
> - 每题对应的 iframe ref 不同，必须逐个 snapshot 确认
> - 返回 `'done'` 表示写入成功

**格式化文本示例**：

```html
<p><strong>标题</strong></p>
<p>1. 步骤一：...</p>
<p>2. 步骤二：...</p>
<pre>int x = 0;
for (int i = 0; i < n; i++) {
    x += a[i];
}</pre>
```

#### 类型 B：程序题（代码文本框）

程序题使用普通 `<textarea>` 或 Monaco/CodeMirror 编辑器。

```bash
# 先 snapshot 找到代码编辑器的 ref
playwright-cli snapshot

# 使用 fill 填充代码
playwright-cli fill e<textarea_ref> "#include <stdio.h>
int main() {
    // 代码内容
    return 0;
}"
```

> **注意**：代码中的 `\n` 会被自动转义，无需手动处理。`"` 和 `%` 在 bash 中需要格外小心。建议使用 Playwright locator 方式定位。

---

## 七、保存与提交

### 暂时保存

```bash
playwright-cli click "getByRole('link', { name: '暂时保存' })"
```

### 提交

点击「提交」按钮**可能不生效**（页面 JS 事件绑定问题）。优先尝试调用 JS 函数。详见「十一、提交函数完整链」。

**快速提交**：

```bash
# 验证 → 确认提交（最可靠）
playwright-cli eval "submitValidate()"
playwright-cli eval "confirmSubmitWork()"
```

**备选**：

```bash
playwright-cli eval "submitWork()"
```

### 确认提交成功

提交成功后会跳转到确认页面（`mooc1.chaoxing.com/mooc-ans/mooc2/work/prompt`），显示：

```
提交成功
等待教师批阅
提交时间：MM-DD HH:mm
```

```bash
playwright-cli snapshot
# 确认页面显示"提交成功"
```

---

## 八、单选题处理

### 8.1 识别单选题

单选题特征：
- 每个选项旁有 radio button
- 无富文本编辑器
- 通过 `getByRole('radio', { name: '...' })` 定位

```bash
# snapshot 中找到 radio，直接 click 即可选答
playwright-cli click e57   # 选择 ref=e57 的 radio
# 或使用 locator
playwright-cli click "getByRole('radio', { name: 'C 答案描述选择' })"
```

### 8.2 图片题目处理 —— 核心规则 ⚠️

**一旦检测到题目包含图片，立即跳过，不做任何图片解析尝试。**

判断题目含图片的方法：

1. Snapshot 中题干或选项内存在 `<img>` 标签
2. 选项文字不完整，只有 A/B/C/D 标签但无文字描述（公式图）
3. 题干依赖图表（如"图(A)~(E)哪一条曲线"、"如图所示"）

**禁止的行为：**
- ❌ 禁止 curl 下载图片
- ❌ 禁止 screenshot 图片元素
- ❌ 禁止尝试识别图片内容
- ❌ 禁止猜测图片对应的选项
- ❌ 禁止因此停止操作询问用户

**正确的处理方式：**

```
检测到题目 X 含图片 → 跳过不做 → 继续下一题 → 
所有题目遍历完后 → 统一汇报跳过明细
```

**汇报模板：**

```
### 答题明细

| 题号 | 状态 | 答案 | 说明 |
|------|------|------|------|
| Q1 | ⚠️ 跳过 | — | 选项含公式图片 |
| Q2 | ✅ | C | 文字可确定 |
| Q3 | ⚠️ 跳过 | — | 题干含 B-x 曲线图 |
...
```

---

## 九、完成后的操作流程 —— 关键规则 ⚠️

### 9.1 做完所有可答题后，必须执行以下流程：

1. **「暂时保存」**——保存已选答案
2. **不要提交**——绝对不要调用 `submitWork()` 或 `confirmSubmitWork()`
3. **汇报明细**——列出所有题目的作答状态
4. **询问用户**——明确问"是否提交？"
5. **等待批准**——只有用户明确批准后才提交

### 9.2 汇报模板

```
### 作业作答完成

**已答题（X道）：**
| 题号 | 答案 | 分值 |
|------|------|------|

**跳过题（Y道，含图片）：**
| 题号 | 原因 |
|------|------|

**已保存但未提交。** 是否需要我帮你提交？
```

### 9.3 用户批准后提交

```bash
# 只有用户明确说"提交"/"可以提交"/"帮我提交"后才执行
playwright-cli eval "confirmSubmitWork()"

# 确认结果
playwright-cli --raw eval "document.body.innerText"
# 应显示"提交成功"
```

### 9.4 用户不批准

如果用户说"先不提交"/"等等"/"我自己检查"：
- 保持当前状态不动
- 告知用户已保存，可随时回来提交
- 如果用户要求，关闭浏览器

---

## 十、提交函数完整链

超星作业提交涉及多个 JS 函数调用链。按可靠性从高到低排列：

### 核心函数

```bash
# 1. 检查所有可用的提交相关函数
playwright-cli --raw eval \
  "Object.keys(window).filter(k => k.toLowerCase().includes('submit') || k.toLowerCase().includes('save')).join(', ')"

# 典型输出：submitValidate, ready2Submit, submitLock, submitWork, confirmSubmitWork, saveWork, ...
```

### 最可靠的提交流程

```bash
# 步骤 1：验证（返回 true/"validated" 表示通过）
playwright-cli --raw eval "typeof submitValidate==='function' ? (submitValidate() || 'validated') : 'not found'"

# 步骤 2：直接调用 confirmSubmitWork（跳过按钮事件绑定问题）
playwright-cli eval "confirmSubmitWork()"
```

### 备选提交流程

```bash
# 方式 A：submitWork() —— 对某些作业类型有效
playwright-cli eval "submitWork()"

# 方式 B：点击提交按钮
playwright-cli click "getByRole('button', { name: '提交' })"

# 方式 C：JS 触发按钮 click 事件
playwright-cli --raw eval \
  "(function(){var b=document.querySelector('button');if(b&&b.textContent.includes('提交')){b.click();return'clicked';}return'not found';})()"
```

### 常见提交失败原因

| 现象 | 原因 | 解决 |
|------|------|------|
| 点击提交无反应 | 页面按钮绑定的是自定义事件，不是原生 click | 用 `confirmSubmitWork()` 替代 |
| submitWork() 跳转到"未完成"页 | 未通过验证或有必填题未答 | 检查是否所有题目都已作答 |
| 提交后返回"未完成"而非"提交成功" | 验证失败（如简答题内容为空） | 确保所有题目有内容，重新保存后再提交 |

### 确认提交结果

提交成功后页面 URL 变为 `prompt?`，显示：
- 「提交成功」+「等待教师批阅」+ 提交时间 → ✅ 成功
- 「未完成」 → ❌ 失败，需返回重试

```bash
playwright-cli --raw eval "document.body.innerText"
# 检查输出是否包含"提交成功"
```

---

## 十一、常见问题与技巧

### 11.1 账号密码安全

如果用户在对话中提供了账号密码，用完即忘。**不要将密码写入 skill 文件或任何持久化文件。**

可以向用户询问是否需要使用 `--persistent` 保存登录态，下次操作无需重新登录：

```bash
playwright-cli open --browser=chrome --persistent "https://i.chaoxing.com/base?t=<ts>"
```

### 11.2 ref 编号变化

每次页面重新加载，ref 编号都会重新分配。**每次 snapshot 后必须重新读取 ref**，不能复用之前的编号。

### 11.3 iframe 内的元素

超星大量使用 iframe。Playwright 操作 iframe 内元素的优先级：

1. **直接用 ref**：`playwright-cli click f26e30`（带 `f` 前缀的 ref 表示在 iframe 内）
2. **eval 注入**：`playwright-cli eval "..." e<ref>`
3. **run-code 脚本**：复杂交互使用 `run-code`

### 11.4 输出过大

作业页面 snapshot 通常非常大（30KB+），使用 `--raw` 截取关键信息：

```bash
playwright-cli --raw eval "document.body.innerText.substring(0, 1000)"
```

### 11.5 作业题型判断

超星作业题主要分三类：

| 类型 | 特征 | 分值 | 作答方式 |
|------|------|------|----------|
| 简答题 | 有 UEditor iframe 编辑器 | 5-10分/题 | eval 注入 HTML |
| 程序题 | 有 textbox + 「运行」按钮 | 10-20分/题 | fill 填充代码 |
| 单选题 | 有 radio button | 约5-15分/题 | click radio |

简答题通常考察：问题理解、子问题分解、输入/输出描述、测试设计、伪代码、流程图描述、具体实例计算。

程序题直接要求编写可运行 C 代码。

单选题直接选择即可，但常含图片公式。

### 11.6 多标签页管理

```bash
playwright-cli tab-list        # 列出所有标签页
playwright-cli tab-select 0    # 切回第 0 个标签页
playwright-cli tab-close 1     # 关闭第 1 个标签页
```

### 11.7 浏览器生命周期

```bash
playwright-cli close           # 关闭当前浏览器实例
playwright-cli close-all       # 关闭所有浏览器实例
playwright-cli kill-all        # 强制杀掉所有浏览器进程
```

---

## 十二、完整操作示例

### 示例 1：完整简答+程序题（C 语言课程）

```bash
# 1. 打开并登录
playwright-cli open --browser=chrome "https://i.chaoxing.com/base?t=1781683144248"
playwright-cli snapshot
playwright-cli fill e10 "13900000000"
playwright-cli fill e12 "password"
playwright-cli click e17

# 2. 进入课程
playwright-cli goto "https://mooc1.chaoxing.com/visit/stucoursemiddle?courseid=XXX&clazzid=XXX..."

# 3. 进入作业
playwright-cli snapshot
playwright-cli click "getByRole('link', { name: '作业' })"

# 4. 打开目标作业（ifrane内链接）
playwright-cli snapshot
playwright-cli click f26e30

# 5. 切换标签页
playwright-cli tab-select 1
playwright-cli snapshot

# 6. 逐题作答 - 简答题
playwright-cli eval "el => { const doc = el.contentDocument || el.contentWindow.document; if (doc?.body) { doc.body.innerHTML = '<p>答案</p>'; return 'done'; } return 'failed'; }" e174

# 7. 逐题作答 - 程序题
playwright-cli fill e1239 "C 代码..."

# 8. 保存
playwright-cli click "getByRole('link', { name: '暂时保存' })"

# 9. 提交
playwright-cli eval "submitWork()"

# 10. 确认
playwright-cli snapshot
```

### 示例 2：单选题部分作答（含图片，不提交）

```bash
# 1-5 同上，打开作业后 snapshot 获取题目

# 6. 只答能确定的题
playwright-cli click "getByRole('radio', { name: 'C 正确答案选择' })"
playwright-cli click "getByRole('radio', { name: 'B 正确答案选择' })"

# 7. 不确定的题目（含图片）跳过不选

# 8. 只保存不提交
playwright-cli click "getByRole('link', { name: '暂时保存' })"

# 9. 关闭浏览器
playwright-cli close-all

# 10. 告知用户哪些题空着、需要手动补答
```

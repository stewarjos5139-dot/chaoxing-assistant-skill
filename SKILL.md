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

### 8.2 题目含图片场景——关键判断流程

超星题目中经常包含嵌入图片（公式、图表、曲线等）。图片有两种来源：

| 来源 | URL 模式 | 说明 |
|------|----------|------|
| CDN 图片 | `p.ananas.chaoxing.com/star3/origin/xxx.png` | 需要登录态 |
| 本地图片 | `mooc1.chaoxing.com/sqp/imgV2?f=xxx` | 同样需要 cookie |

**尝试获取图片的方法：**

1. **snapshot --boxes** 获取图片位置信息：
   ```bash
   playwright-cli snapshot --boxes
   ```
   输出中包含 `[box=x,y,w,h]` 定位信息，可帮助判断图片在哪个题目和选项中。

2. **screenshot 单个图片元素**：
   ```bash
   # 截取选项 A 的图片（ref 从 snapshot 获取）
   playwright-cli screenshot e31 --filename=q1_a.png
   ```

3. **curl 下载图片（大概率 403）**：
   ```bash
   curl -o img.png "https://p.ananas.chaoxing.com/star3/origin/xxx.png"
   ```
   > ⚠️ CDN 通常返回 403，因为缺少浏览器 cookie。优先使用 screenshot 方式。

4. **提取所有图片 URL**：
   ```bash
   playwright-cli --raw eval "Array.from(document.querySelectorAll('img')).map(i => ({src: i.src, w: i.width, h: i.height}))"
   ```

> ⚠️ **关键限制**：当前环境无法真正"看"图片内容（`[Unsupported Image]`）。如果图片包含关键信息（公式、坐标轴、角度值等）且无法从文字上下文推断，则无法确定答案。

---

## 九、不确定性处理——何时停止并询问用户

### 9.1 必须停止的情况

遇到以下任一情况，**立即停止作答**，向用户说明原因并请求帮助：

1. **图片包含关键信息且无法识别**：选项或题干中的图片（公式、图表、曲线）无法从文字推断内容
2. **题目文字不完整**：题干依赖图片中的变量定义或数值
3. **多张图片之间存在逻辑关系**：如图表对应的曲线需要看图匹配
4. **答案有多种合理解释**：且无法从上下文确定出题意图

### 9.2 停止时的操作模板

```
⚠️ 按照你的指示，遇到不能确定答案的情况，先停下来询问。

### 题目分析结果：

**能确定答案的题目（X道）：**
| 题号 | 答案 | 解析 |
|------|------|------|

**无法确定的题目（Y道，含图片）：**
| 题号 | 问题 | 
|------|------|
| Q1 | XXX 图片内容无法识别 |
| Q3 | B-x 曲线图无法看到 |
```

### 9.3 不要猜测图片内容

**禁止**直接猜测公式图片对应的选项。即使知道物理公式的正确答案（如半圆形导线的 B=μ₀I/(4R)），也无法知道 ABCD 哪个选项对应此公式，猜错概率很高。

---

## 十、部分作答与不提交策略

### 10.1 跳过无法确定的题目

对于单选题，无法确定的题目直接跳过不选：

```bash
# 只点击确定题目的 radio，不确定的直接跳过
playwright-cli click "getByRole('radio', { name: 'C 正确选项描述' })"
```

### 10.2 保存但不提交

```bash
# 先「暂时保存」
playwright-cli click "getByRole('link', { name: '暂时保存' })"

# ⚠️ 绝对不要调用 submitWork() 或点击「提交」按钮
# 绝对不要调用 confirmSubmitWork()
```

### 10.3 关闭浏览器

```bash
playwright-cli close-all
```

### 10.4 用户后续操作指南

告知用户：
- 已保存但未提交的作业会保留当前作答状态
- 下次打开可从上次中断处继续
- 用户需自行识别图片题目并补答后提交

---

## 十一、提交函数完整链

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

## 十二、常见问题与技巧

### 12.1 账号密码安全

如果用户在对话中提供了账号密码，用完即忘。**不要将密码写入 skill 文件或任何持久化文件。**

可以向用户询问是否需要使用 `--persistent` 保存登录态，下次操作无需重新登录：

```bash
playwright-cli open --browser=chrome --persistent "https://i.chaoxing.com/base?t=<ts>"
```

### 12.2 ref 编号变化

每次页面重新加载，ref 编号都会重新分配。**每次 snapshot 后必须重新读取 ref**，不能复用之前的编号。

### 12.3 iframe 内的元素

超星大量使用 iframe。Playwright 操作 iframe 内元素的优先级：

1. **直接用 ref**：`playwright-cli click f26e30`（带 `f` 前缀的 ref 表示在 iframe 内）
2. **eval 注入**：`playwright-cli eval "..." e<ref>`
3. **run-code 脚本**：复杂交互使用 `run-code`

### 12.4 输出过大

作业页面 snapshot 通常非常大（30KB+），使用 `--raw` 截取关键信息：

```bash
playwright-cli --raw eval "document.body.innerText.substring(0, 1000)"
```

### 12.5 作业题型判断

超星作业题主要分三类：

| 类型 | 特征 | 分值 | 作答方式 |
|------|------|------|----------|
| 简答题 | 有 UEditor iframe 编辑器 | 5-10分/题 | eval 注入 HTML |
| 程序题 | 有 textbox + 「运行」按钮 | 10-20分/题 | fill 填充代码 |
| 单选题 | 有 radio button | 约5-15分/题 | click radio |

简答题通常考察：问题理解、子问题分解、输入/输出描述、测试设计、伪代码、流程图描述、具体实例计算。

程序题直接要求编写可运行 C 代码。

单选题直接选择即可，但常含图片公式。

### 12.6 多标签页管理

```bash
playwright-cli tab-list        # 列出所有标签页
playwright-cli tab-select 0    # 切回第 0 个标签页
playwright-cli tab-close 1     # 关闭第 1 个标签页
```

### 12.7 浏览器生命周期

```bash
playwright-cli close           # 关闭当前浏览器实例
playwright-cli close-all       # 关闭所有浏览器实例
playwright-cli kill-all        # 强制杀掉所有浏览器进程
```

---

## 十三、完整操作示例

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

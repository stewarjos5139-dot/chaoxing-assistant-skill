# chaoxing-assistant — 超星学习通 自动化 Skill

基于 `playwright-cli` 的浏览器自动化 Skill，用于操作超星学习通（i.chaoxing.com / mooc1.chaoxing.com）：登录、浏览课程、查找作业、逐题作答、保存与提交。

## 先决条件

- Node.js 和 npm 已安装
- 全局安装 `playwright-cli`：

```bash
npm install -g @playwright/cli@latest
```

建议使用 Chrome 浏览器：`--browser=chrome`

## 快速开始

```bash
# 1. 打开并登录
playwright-cli open --browser=chrome "https://i.chaoxing.com/base?t=$(date +%s)"
playwright-cli snapshot
playwright-cli fill e10 "手机号"
playwright-cli fill e12 "密码"
playwright-cli click e17

# 2. 进入课程（直接 goto 课程 URL 最推荐）
playwright-cli goto "https://mooc1.chaoxing.com/visit/stucoursemiddle?courseid=XXX&clazzid=XXX..."

# 3. 进入作业页
playwright-cli click "getByRole('link', { name: '作业' })"
playwright-cli snapshot

# 4. 打开目标作业
playwright-cli click f26e30        # 带 f 前缀 = iframe 内元素
playwright-cli tab-select 1         # 作业在新标签页打开
```

## 支持的三类题目

| 类型 | 识别特征 | 作答方式 |
|------|----------|----------|
| 简答题 | `iframe#ueditor_N` 富文本编辑器 | eval 向 iframe 写入 HTML |
| 程序题 | textbox + "运行" 按钮 | `fill` 填充 C 代码 |
| 单选题 | radio button | `click` 选中 |

### 简答题示例

```bash
playwright-cli eval "el => { const doc = el.contentDocument || el.contentWindow.document; if (doc?.body) { doc.body.innerHTML = '<p>答案内容</p>'; return 'done'; } return 'failed'; }" e174
```

### 程序题示例

```bash
playwright-cli fill e1239 "#include <stdio.h>
int main() {
    // 完整 C 代码
    return 0;
}"
```

### 单选题示例

```bash
playwright-cli click "getByRole('radio', { name: 'C 正确答案描述选择' })"
```

## 图片题目处理 — 核心规则

**一旦检测到题目包含图片（`<img>` 标签），立即跳过，不做任何图片解析尝试。**

禁止的行为：
- ❌ 不 curl 下载图片（CDN 通常返回 403）
- ❌ 不 screenshot 图片元素
- ❌ 不尝试"看"图片内容（环境不支持）
- ❌ 不猜测图片对应的选项
- ❌ 不因此停止并询问用户

正确流程：检测到图片 → 跳过该题 → 继续下一题 → 全部完成后统一汇报。

## 完成后流程 — 核心规则

**完成所有可答题后，绝不自动提交。** 必须执行：

1. **保存** — `playwright-cli click "getByRole('link', { name: '暂时保存' })"`
2. **汇报明细** — 列出每道题的状态（已答/跳过）
3. **询问用户** — 明确问"是否提交？"
4. **等待批准** — 只有用户明确说"提交"/"可以"后才执行提交
5. 用户不同意 → 保持当前状态，告知已保存可随时回来

### 汇报模板

```
### 答题明细

| 题号 | 状态 | 答案 | 说明 |
|------|------|------|------|
| Q1 | ⚠️ 跳过 | — | 选项含公式图片 |
| Q2 | ✅ | C | 文字可确定 |
| Q3 | ⚠️ 跳过 | — | 题干含 B-x 曲线图 |
| Q4 | ✅ | B | 文字可确定 |
...

已保存但未提交。是否需要我帮你提交？
```

## 保存与提交

```bash
# 保存（不提交）
playwright-cli click "getByRole('link', { name: '暂时保存' })"

# 提交（仅用户批准后执行）
playwright-cli eval "confirmSubmitWork()"

# 确认结果
playwright-cli --raw eval "document.body.innerText"
# 应输出："提交成功\n等待教师批阅\n提交时间：..."
```

提交备选：`submitWork()` 或直接点击提交按钮，但 `confirmSubmitWork()` 最可靠。

## 常见问题

- **ref 编号每次页面刷新后变化**：必须先 `snapshot` 再使用 ref
- **iframe 内元素**：带 `f` 前缀的 ref（如 `f26e30`）表示在 iframe 内，可直接使用
- **点击提交无反应**：优先用 `confirmSubmitWork()`，而非点击页面按钮
- **切勿将账号密码写入仓库**：在会话中提供过的密码用完即弃

## 完整示例流程

```bash
# 登录
playwright-cli open --browser=chrome "https://i.chaoxing.com/base?t=1781683144248"
playwright-cli snapshot
playwright-cli fill e10 "13900000000"
playwright-cli fill e12 "password"
playwright-cli click e17

# 进入课程 → 作业
playwright-cli goto "https://mooc1.chaoxing.com/visit/stucoursemiddle?courseid=XXX&clazzid=XXX..."
playwright-cli click "getByRole('link', { name: '作业' })"
playwright-cli snapshot
playwright-cli click f26e30        # 点击目标作业（iframe 内）
playwright-cli tab-select 1         # 切到新标签页

# 逐题作答（简答）
playwright-cli eval "el => { const doc = el.contentDocument || el.contentWindow.document; if (doc?.body) { doc.body.innerHTML = '<p>答案</p>'; return 'done'; } return 'failed'; }" e175

# 逐题作答（程序）
playwright-cli fill e1239 "C 代码..."

# 逐题作答（单选 — 跳过含图片的）
playwright-cli click "getByRole('radio', { name: 'C 正确答案选择' })"

# 保存（不提交！）
playwright-cli click "getByRole('link', { name: '暂时保存' })"

# 汇报明细 → 等用户批准 → 用户说"提交"后：
playwright-cli eval "confirmSubmitWork()"
```

## 文件说明

- `SKILL.md` — 完整操作手册（13 章），包含所有细节和边界情况
- `README.md` — 本文件，快速概览和使用说明

## 许可与免责

此 Skill 仅提供自动化操作流程与示例命令。使用者需自行遵守学校与平台的使用条款和学术诚信政策。作者对因使用自动化导致的任何后果不承担责任。

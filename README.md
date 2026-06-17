# chaoxing-assistant — 超星学习通 自动化 Skill

本仓库提供基于 `playwright-cli` 的操作指令与示例，用于自动化操作超星学习通（i.chaoxing.com / mooc1.chaoxing.com），包括登录、进入课程、打开作业、逐题作答、保存与提交等常见流程。

## 先决条件
- 已安装 Node.js 和 npm。
- 全局安装 `playwright-cli`：

```bash
npm install -g @playwright/cli@latest
```

> 建议使用 Chrome（或 Chromium）进行自动化：`--browser=chrome`。

## 快速开始

1. 打开并登录：

```bash
playwright-cli open --browser=chrome "https://i.chaoxing.com/base?t=$(date +%s)"
playwright-cli snapshot
# 填写账号/密码（先执行 snapshot 获取 ref）
playwright-cli fill <账号_ref> "你的手机或超星号"
playwright-cli fill <密码_ref> "你的密码"
playwright-cli click <登录按钮_ref>
```

2. 进入课程与作业（两种方式）：

- 方式 A（推荐）：在课程列表 snapshot 中复制目标课程的 URL，直接 `goto`：

```bash
playwright-cli goto "https://mooc1.chaoxing.com/visit/stucoursemiddle?courseid=..."
```

- 方式 B：在页面内通过 locator 点击课程（注意 iframe 引用）。

3. 进入作业页面并打开目标作业：

```bash
playwright-cli snapshot
playwright-cli click "getByRole('link', { name: '作业' })"
# 在作业列表中点击目标作业（可能在 iframe 内，会在新标签页打开）
playwright-cli click <作业_ref>
playwright-cli tab-select 1
playwright-cli snapshot
```

## 作答要点

- 简答题（富文本 UEditor）：通常嵌入在 `iframe#ueditor_N`。可通过 `eval` 向 iframe 写入 HTML：

```bash
playwright-cli eval "el => { const doc = el.contentDocument || el.contentWindow.document; if (doc?.body) { doc.body.innerHTML = '<p>答案内容</p>'; return 'done'; } return 'failed'; }" <iframe_ref>
```

- 程序题：使用 `fill` 填充代码编辑器或 textarea：

```bash
playwright-cli fill <textarea_ref> "#include <stdio.h>\nint main() { return 0; }"
```

- 单选题：通过定位 radio 或使用 `getByRole('radio', ...)` 并 `click` 对应选项。

## 保存与提交

- 暂时保存：

```bash
playwright-cli click "getByRole('link', { name: '暂时保存' })"
```

- 提交（优先使用 JS 调用以避开原生 click 绑定问题）：

```bash
# 先验证
playwright-cli --raw eval "typeof submitValidate==='function' ? (submitValidate() || 'validated') : 'not found'"
# 直接调用确认提交
playwright-cli eval "confirmSubmitWork()"
```

备选：`submitWork()`、或尝试触发页面上的提交按钮。

提交成功页面通常跳转到 `prompt` 页面并显示“提交成功”。可用 `snapshot` 或 `--raw eval 'document.body.innerText'` 检查。

## 图片题与不确定性处理

- 题干或选项中若包含图片（公式/图表/曲线）且图片为关键信息，则自动化脚本应停止并提示人工处理。不要盲目猜测图片含义。
- 尝试使用 `snapshot --boxes`、对单个图片元素 `screenshot <ref>` 获取可视化证据；CDN 直接下载图片通常会 403。

## 常见注意事项

- 每次页面刷新后，playwright-cli 的 `ref` 编号会变化，务必先 `snapshot` 再使用 ref。
- 超星大量使用 iframe，优先使用带 `f` 前缀的 iframe ref 或通过 eval 注入操作。
- 切勿在仓库中保存账号密码或把密码写入 skill 文件。若在会话中提供过密码，请在使用后立即删除或重置。

## 故障排查

- 点击提交无反应：优先尝试 `confirmSubmitWork()`；检查是否有未答的必答题。
- 无法下载题中图片：用 `screenshot <ref>` 截图代替直接下载。

## 示例完整流程（简要）

```bash
# 打开并登录
playwright-cli open --browser=chrome "https://i.chaoxing.com/base?t=$(date +%s)"
playwright-cli snapshot
# 填充账号密码并登录（以 snapshot 获取的 ref 为准）
playwright-cli fill e10 "13900000000"
playwright-cli fill e12 "password"
playwright-cli click e17

# 进入课程并打开作业
playwright-cli goto "https://mooc1.chaoxing.com/visit/stucoursemiddle?courseid=XXX&clazzid=XXX"
playwright-cli snapshot
playwright-cli click "getByRole('link', { name: '作业' })"
playwright-cli snapshot
playwright-cli click <目标作业_ref>
playwright-cli tab-select 1

# 作答（示例）
playwright-cli eval "el => { const doc = el.contentDocument || el.contentWindow.document; if (doc?.body) { doc.body.innerHTML = '<p>示例答案</p>'; return 'done'; } return 'failed'; }" e174

# 保存并提交
playwright-cli click "getByRole('link', { name: '暂时保存' })"
playwright-cli eval "confirmSubmitWork()"
```

## 许可与免责

此 skill 仅提供自动化操作流程与示例命令。使用者需自行遵守学校与平台的使用条款与学术诚信政策。作者对用户因使用自动化导致的任何后果不承担责任。

---
文件位置：`SKILL.md` 中包含更详尽的操作说明与示例。若需把 README 扩展为多语言或加入脚本示例（如 PowerShell / Windows 特殊用法），我可以继续补充。

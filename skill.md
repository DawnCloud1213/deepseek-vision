---
name: deepseek-vision
description: "当 API 模型不支持图像识别时，通过 Playwright 自动化 DeepSeek 网页端识图模式进行图片分析。自动启用深度思考。触发词：识图、看图、图片分析、这张图、截图描述、vision、image recognition、deepseek vision。"
version: "1.0.0"
author: Sisyphus

category: vision
tags:
  - deepseek
  - vision
  - image-recognition
  - screenshot
  - playwright
  - web-automation

capabilities:
  - image_recognition
  - screenshot_analysis
  - ui_element_detection
  - diagram_parsing
  - ocr_fallback

languages:
  - zh

related_skills:
  - agent-browser
---

# DeepSeek 网页端识图

当 DeepSeek V4 Pro API 无法直接识别图片时，通过 Playwright 自动化操作 chat.deepseek.com 网页端识图模式完成图片分析。自动启用**深度思考**。

## 调度规则

**此技能必须通过 `Agent` 工具以 `haiku` 模型派发执行**，禁止在主会话中逐步操作浏览器。主会话只接收子代理返回的最终文本结果。

```
Agent(
  subagent_type: "general-purpose",
  model: "haiku",
  description: "DeepSeek识图-{简短描述}",
  prompt: "使用内置 Playwright (mcp__plugin_playwright_playwright__*) 按 deepseek-vision 技能流程操作：
    图片路径：{绝对路径}
    提示词：{评估要求}
    
    完成全部浏览器操作后，返回 DeepSeek 的完整回复文本。"
)
```

多张图片需要评估时，每张图片派发一个独立 Agent，**并行执行**。

## 触发条件

以下任一情况自动触发：
- 用户发送图片但模型返回 "Cannot read image / this model does not support image input"
- 用户要求"识图"、"看图"、"分析这张图"、"描述截图"
- 用户提及 DeepSeek vision / image recognition
- 需要从截图中识别 UI 元素、流程图、架构图等

## 前置条件

- **优先使用内置 Playwright**（`playwright_*` 前缀工具），而非 ECC Playwright（`everything-claude-code_playwright_*`）。
  - ECC Playwright 的文件上传（`browser_file_upload`）在 DeepSeek 上会被 CDP 安全策略拦截（`Not allowed`），不可用。
  - 内置 Playwright 的 `upload_file` 走 `chrome-devtools-mcp` 底层实现，可绕过拦截。
- 内置 Playwright 需连接 9223 端口的独立 Chrome（⚠️ 9222 已被 ECC Playwright 占用，不可混用）：
  ```
  chrome.exe --remote-debugging-port=9223 --user-data-dir=...\mcp-chrome-06b0369 --no-first-run --disable-extensions
  ```
- chat.deepseek.com 已登录（cookie 持久化在 Playwright profile 中）
- 如果未登录，先扫码登录一次（之后永久有效）

## 工作流程

### 步骤 1：导航并决定是否开新对话

```
playwright_navigate_page → https://chat.deepseek.com/
playwright_take_snapshot → 确认在聊天界面（非 /sign_in）
```

如果在 `/sign_in` 页面，先扫码登录。

**自行决定是否开启新对话**：如果当前对话上下文干净（无残留图片/附件），可直接复用；否则点击"开启新对话"按钮（查找含"开启新对话"文本的元素）。

### 步骤 2：切换识图模式 + 深度思考

```
playwright_take_snapshot → 找到 uid
playwright_click → "识图模式" radio
playwright_click → "深度思考" button（确保 pressed 状态）
```

**重要**：必须确保深度思考已选中。如果按钮已经是 active 状态则跳过。

### 步骤 3：上传图片

使用**内置 Playwright** 的 `playwright_upload_file`（注意：不要用 ECC 的 `browser_file_upload`，会被 DeepSeek 拦截）：

```
playwright_click → 上传按钮（触发文件选择器）
playwright_upload_file → uid=上传按钮的uid, filePath="图片绝对路径"
playwright_take_snapshot → 确认出现 button "文件名.png"
```

**注意**：
- 内置 Playwright 工具前缀为 `playwright_*`（不是 `everything-claude-code_playwright_*`）
- `playwright_upload_file` 需要文件选择器处于活跃状态（先点击上传按钮触发）
- ECC 的 `browser_file_upload` / `browser_run_code` + `setInputFiles` 均不可用——DeepSeek 在 CDP 层封堵了 `DOM.setFileInputFiles`
- 支持格式：png, jpg, jpeg, webp, gif, bmp, svg, pdf

### 步骤 4：输入提示词并发送

```
playwright_fill → textbox("给 DeepSeek 发送消息") → 提示词
playwright_take_snapshot → 确认发送按钮非 disabled
playwright_click → 发送按钮
```

默认提示词（如用户未指定）：
- 技术图表："请详细描述这张图片的内容，说明图中展示了什么系统或流程"
- UI 截图："识别这张截图中的UI元素，列出所有可见的按钮、输入框和导航区域"
- 通用："请描述这张图片的内容"

### 步骤 5：等待回复并提取

```
playwright_wait_for → text=["深度思考"], timeout=60000
playwright_evaluate_script → 提取回复文本
```

提取代码：
```javascript
() => {
  const all = document.body.innerText;
  // 从 "已思考" 开始截取到回复结束
  const start = all.indexOf('已思考');
  if (start === -1) return all;
  // 截取思考过程 + 回复内容，去掉尾部的"深度思考"和"内容由 AI 生成"
  let text = all.substring(start);
  text = text.replace(/深度思考\s*内容由 AI 生成，请仔细甄别\s*$/g, '').trim();
  return text;
}
```

### 步骤 6：返回结果

将提取的文本直接返回给用户。格式为：

```
[DeepSeek 识图结果 - 耗时约 XX 秒]

{回复内容}
```

## 异常处理

| 问题 | 处理 |
|------|------|
| 重定向到 /sign_in | 截图登录页，让用户扫码，等待后重试 |
| 上传后发送按钮仍 disabled | 检查深度思考是否开启，点击上传区域后重试 |
| 文件上传失败（`Not allowed`） | **检查是否误用了 ECC Playwright**。必须用内置 `playwright_upload_file`，而非 ECC 的 `browser_file_upload` |
| 回复超时（>60秒无响应） | 刷新页面重试，或告知用户手动检查 |
| uid 编号变化 | **每次操作前必须 snapshot**，不要复用旧 uid |
| 9223 端口 Chrome 未运行 | `powershell Start-Process` 启动独立 Chrome（见前置条件） |

## 备选方案：agent-browser

如果内置 Playwright 不可用，使用 agent-browser 替代：

```bash
agent-browser --profile Default open "https://chat.deepseek.com/"
agent-browser click @{新对话ref}
agent-browser click @{识图模式ref}
agent-browser click @{深度思考ref}
agent-browser eval "document.querySelector('input[type=file]').id='up'"
agent-browser upload "#up" "图片路径"
agent-browser fill @{输入框ref} "提示词"
agent-browser click @{发送按钮ref}
agent-browser wait 25000
agent-browser get text @{回复区域ref}
```

## 注意事项

- 每次识图约耗时 15-30 秒
- 深度思考会增加思考时间但提升分析质量
- 网页端可能有频率限制，短时间内大量请求可能被限
- cookie 登录态通常在数周内有效，过期需重新扫码

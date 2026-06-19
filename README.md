# deepseek-vision

Claude Code skill — DeepSeek 网页端识图。

当 API 模型（如 DeepSeek V4 Pro）不支持图像输入时，通过 Playwright 自动化操作 [chat.deepseek.com](https://chat.deepseek.com) 网页端的识图模式完成图片分析。自动启用**深度思考**。

## 用法

在 Claude Code 中发送图片或使用触发词：
```
识图 / 看图 / 分析这张图 / 描述截图
```

## 工作流程

1. 导航至 chat.deepseek.com
2. 切换识图模式 + 开启深度思考
3. 上传图片
4. 输入提示词并发送
5. 等待回复并提取文本

全程通过子代理（haiku 模型）执行，主会话只接收最终文本结果。

## 前置条件

- 内置 Playwright (`@playwright/mcp`) 连接独立 Chrome（9223 端口）
- chat.deepseek.com 已登录（cookie 持久化）

## 调度规则

必须通过 `Agent` 工具派发，禁止主会话操作浏览器。多图并行派发。

## 安全审查

✅ 通过 — 无硬编码凭据、无 PII 泄露、无内网地址暴露。

## 安装

```bash
git clone https://github.com/DawnCloud1213/deepseek-vision.git \
  ~/.claude/skills/deepseek-vision
```

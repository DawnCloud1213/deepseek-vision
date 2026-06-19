# deepseek-vision

Claude Code skill — DeepSeek 网页端识图。

当 API 模型（如 DeepSeek V4 Pro）不支持图像输入时，通过 Playwright 自动化操作 [chat.deepseek.com](https://chat.deepseek.com) 网页端的识图模式完成图片分析。自动启用**深度思考**。

## 安装

```bash
git clone https://github.com/DawnCloud1213/deepseek-vision.git \
  ~/.claude/skills/deepseek-vision
```

## 依赖

| 组件 | 用途 | 必需 |
|------|------|------|
| `@playwright/mcp` 插件 | 浏览器自动化上传图片并提取回复 | 是 |
| DeepSeek 账号 | 用于 chat.deepseek.com 登录（cookie 持久化） | 是 |

## 配置

### 1. 安装 Playwright 插件

```bash
# 在 Claude Code 中
/install playwright@claude-plugins-official
```

### 2. 登录 DeepSeek

在 Claude Code 中首次使用识图时，agent 会导航至 chat.deepseek.com。如未登录，扫码登录一次即可（cookie 持久化在 Playwright profile 中，数周有效）。

## 用法

在 Claude Code 中发送图片或使用触发词：
```
识图 / 看图 / 分析这张图 / 描述截图
```

多张图片时自动并行派发子代理，同时分析。

## 工作流程

1. 导航至 chat.deepseek.com，决定是否开新对话
2. 切换识图模式 + 开启深度思考
3. 上传图片
4. 输入提示词并发送
5. 等待回复（含深度思考过程）并提取文本

全程通过 haiku 子代理执行，主会话只接收最终文本结果。

## 安全审查

✅ 通过 — 无硬编码凭据、无 PII 泄露、无内网地址暴露。

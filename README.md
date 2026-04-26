---
name: weipaiban-skill
description: 通过 AI 自动创建公众号图文，支持模板搜索、配色方案、文本生成和图片替换
tags: [design, wechat, 公众号, 排版, 图文生成, 模板]
metadata:
  openclaw:
    requires: ['env:WEIPAIBAN_API_KEY']
    optional: ['env:VOLCENGINE_AK', 'env:VOLCENGINE_SK', 'env:VOLCENGINE_TOKEN', 'skill:jimeng-ai', 'bin:python3', 'bin:rembg']
---

# 微排版

## 什么时候使用这个技能

- 用户说「帮我做一个微信图文」「用微排版创建 xxx」时自动触发
- 用户需要基于模板快速生成微信公众号图文作品
- 用户想要 AI 自动完成配色、文本撰写、图片生成等创作流程

## 使用方法

1. 描述你想要的图文主题（如「清明节微信图文」）
2. 从搜索到的模板中选择一个喜欢的
3. AI 自动生成配色方案 → 确认或调整
4. AI 自动生成替换文本 → 确认或调整
5. AI 分析并分类模板中的图片 → 确认分类结果
6. AI 自动生成新图片、去除背景、上传 CDN
7. 所有变更批量提交，返回编辑链接

## 示例

**用户输入：**

> 请帮我用微排版创建一个清明节的微信图文

**技能输出：**

1. 提取标签 `节日`、关键词 `清明`，搜索匹配模板
2. 展示候选模板供用户选择
3. 克隆模板创建作品，分析模板画像（排版类型、色系、密度）
4. 生成清明节主题的清新绿色系配色方案
5. 为每段文本生成清明节主题内容
6. 智能分类图片（场景背景 / 人物 / 装饰 / 文字等），跳过 logo 和二维码
7. 逐张生成新图片并上传
8. 批量更新作品，返回编辑链接：`https://weipaiban.cn/editor/{作品ID}`

## 安全与权限

### 必需

- **凭据**：`WEIPAIBAN_API_KEY`（`sk-` 开头），在 [微排版 API Keys 页面](https://weipaiban.cn/settings/api-keys) 生成
- **外部服务**：微排版 API（`weipaiban.cn`）——模板搜索、作品管理、CDN 上传
- **磁盘写入**：`/tmp/weipaiban-task-{workId}/`（任务中间结果，任务结束后可清理）

### 可选（仅用于 Step 8 图片生成阶段）

- **凭据**：`VOLCENGINE_AK`（必需），`VOLCENGINE_SK` 与 `VOLCENGINE_TOKEN` 至少配置其一（永久凭证用 SK；临时凭证 AKTP/STS 用 TOKEN）
- **依赖技能**：`jimeng-ai`
- **本地二进制**：`python3`（≥3.9）、`rembg`
  - 默认使用 `u2netp` 轻量模型（固定 `-m u2netp`），如果需要提高精度，可以尝试 `u2net` 模型（固定 `-m u2net`）
  - 首次运行将下载 `~/.u2net/u2netp.onnx`（约 4.7MB）
- **外部服务**：即梦 API（图片生成）

### 降级行为与运行时安装

- **缺失可选依赖时**：Step 8 由用户选择跳过，作品保留模板原图，其他步骤正常执行
- **用户同意原则**：技能不会在未经用户同意的情况下执行 `pip install` 或安装其他技能。Step 8a 会先检测依赖，缺失时进入交互点由用户选择「安装」「跳过图片生成」或「取消」
- **关键交互点**：模板选择、配色确认、文本确认、图片分类确认、依赖安装确认、失败重试确认均需用户明确回复

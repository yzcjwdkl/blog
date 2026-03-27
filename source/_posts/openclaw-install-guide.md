---
title: 【AI助手】OpenClaw 安装指南 — 本地 AI 助手跑起来了！
date: 2026-03-25 17:00:00
updated: 2026-03-25 17:50:00
tags:
  - OpenClaw
  - AI助手
categories: 工具分享
cover: https://images.unsplash.com/photo-1629654297299-c8506221ca97?w=800&q=80
description: 完整记录 OpenClaw 本地 AI 助手的安装配置过程，从充值 MiniMax API 到启动 TUI 对话，手把手教程。
---

# 【AI助手】OpenClaw 安装指南 — 本地 AI 助手跑起来了！

一直想给自己整一个本地跑的 AI 助手，不用每次都开浏览器。后来折腾了半天终于把 **OpenClaw** 跑起来了，顺手记录一下安装流程，给有需要的朋友参考，也方便自己以后重装时少走弯路。

**第一步不是装软件，是先充钱** 😄

---

## 第一步：先充钱 — 找一个靠谱的 API 平台

OpenClaw 本质上是一个 AI 网关，得有个大模型的 API 才能跑。国内的话，**MiniMax** 是个不错的选择：

- 接入了 ChatGPT/Gemini/MiniMax 等多种模型
- 价格相对便宜
- 国内直接访问，不用科学上网

我买的是 **29元包月套餐**，每天送一定额度，日常轻度使用完全够了。

具体步骤：

1. 打开 [MiniMax 开放平台](https://www.minimaxi.com/)
2. 找到「API开放平台」注册账号,点击右上角「订阅套餐」，去选一个适合你的包月套餐

---

## 第二步：安装 Node.js

OpenClaw 基于 Node.js 运行，版本需要 **Node 22.16 或更高**。

```bash
node -v
```

确认是 `v22.x.x` 或更高就行。Node 官网 [nodejs.org](https://nodejs.org/) 下载安装包，或者用 Homebrew (`brew install node`)、Winget 等包管理工具都行，看你习惯。

---

## 第三步：安装 OpenClaw

环境准备好了，现在来装主角！

OpenClaw 官方提供了一个**一键安装脚本**，会自动检测系统、安装 OpenClaw、配置好基本环境。

### Mac / Linux / WSL2

```bash
curl -fsSL https://openclaw.ai/install.sh | bash
```

### Windows（PowerShell）

```powershell
iwr -useb https://openclaw.ai/install.ps1 | iex
```

脚本会自动处理一切，按提示一路确认就行。

### 已经有 Node.js 了（npm 安装）

```bash
npm install -g openclaw@latest
openclaw onboard --install-daemon
```

### 安装完成后验证

```bash
openclaw --version
openclaw doctor
openclaw gateway status
```

看到 `Gateway is running` 就说明装好了 🎉

---

## 第四步：配置并启动 — 以 MiniMax 为例

这是最关键的一步，配置好了 AI 模型才能用。

### 引导配置（推荐新手）

运行以下命令，进入交互式配置向导：

```bash
openclaw configure
```

跟着提示一步一步来：

1. Where will the Gateway run? → 选择 Local (this machine)
2. Select sections to configure → 选择 Model
3. Model/auth provider → 选择 MiniMax
4. MiniMax auth method → 选择 MiniMax CN — OAuth (minimaxi.com)

登录授权后会自动弹出登录页，登录并授权即可。

### 配置完成后验证模型

```bash
openclaw models list
```

确认能看到 `minimax/MiniMax-M2.5` 就说明配置成功了

---

## 第五步：启动 TUI，开始对话！

配置完成之后，终于可以开始用了！

### 启动 TUI 界面

```bash
openclaw-tui
```

直接终端里对话，界面支持高亮、代码块什么的，比想象中好用。

> 用 TUI 之前记得先设置好 API Key 环境变量，或者已经在配置里写好了就不用管了。

### 命令行模式

不想用图形界面？直接一行命令：

```bash
openclaw chat "你好！"
```

### 查看帮助

```bash
openclaw --help
```

## 最后说两句

整个安装过程不算复杂，一键脚本省了很多事。最麻烦的反倒是第一步想清楚用哪个 API 平台 😂

配好之后用 TUI 跟 AI 对话，终端直接搞定，不用开浏览器，速度也快。

有什么问题欢迎留言交流！

---

> **参考资料**：[OpenClaw 官方文档](https://docs.openclaw.ai) · [MiniMax 开放平台](https://www.minimax.io/)
>
> 本文更新于 2026-03-25

---
title: Ubuntu 安装 OpenClaw
pubDate: 2026-03-10
categories: ['笔记']
description: 'Ubuntu 系统下 OpenClaw 的安装流程，包括环境准备、安装步骤、模型配置、飞书通道配置及 Skills 安装。'
slug: openclaw-ubuntu-install
---

# Ubuntu 安装 OpenClaw

本文档详细记录在 Ubuntu 系统上从零开始安装和配置 OpenClaw 的完整流程，涵盖环境准备、OpenClaw 安装、onboard 配置向导详解、飞书通道配置以及 Skills 安装。

> **OpenClaw 名称历史**：ClawdBot → MoltBot → OpenClaw（因商标问题多次更名，都是同一个项目）

## 目录

1. [环境准备](#环境准备)
2. [OpenClaw 安装](#openclaw-安装)
3. [Onboard 配置向导详解](#onboard-配置向导详解)
4. [飞书通道配置](#飞书通道配置)
5. [Skills 安装](#skills-安装)
6. [验证与测试](#验证与测试)
7. [常见问题](#常见问题)

---

## 环境准备

### 系统要求

| 项目 | 要求 |
|------|------|
| **操作系统** | Ubuntu 20.04 LTS 或更高版本 |
| **Node.js** | 22.x 或更高版本 |
| **内存** | 建议 2GB 以上 |
| **磁盘空间** | 建议 5GB 以上可用空间 |

### 1. 更新系统包

```bash
sudo apt update && sudo apt upgrade -y
```

### 2. 安装基础依赖

```bash
sudo apt install -y curl git build-essential
```

### 3. 安装 Node.js 22

OpenClaw 要求 Node.js 22 或更高版本。推荐使用 NodeSource 官方源安装：

```bash
# 下载并执行 NodeSource 安装脚本
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -

# 安装 Node.js
sudo apt-get install -y nodejs

# 验证安装
node -v
npm -v
```

预期输出：
```
v22.x.x
10.x.x
```

### 4. 配置 npm 全局包路径（可选）

如果遇到权限问题，可以配置 npm 使用用户目录：

```bash
# 创建全局包目录
mkdir -p ~/.npm-global

# 配置 npm 使用新目录
npm config set prefix ~/.npm-global

# 添加到环境变量
echo 'export PATH=~/.npm-global/bin:$PATH' >> ~/.bashrc
source ~/.bashrc
```

---

## OpenClaw 安装

### 方法一：安装脚本（推荐）

这是最简单和最推荐的安装方式，会自动处理所有依赖：

```bash
# 执行官方安装脚本
curl -fsSL https://openclaw.ai/install.sh | bash
```

安装脚本会自动完成以下操作：
- 检测并安装 Node.js（如果缺失）
- 安装 Git
- 通过 npm 安装 OpenClaw 核心
- 运行初始化向导（onboard）

### 方法二：npm 手动安装

如果你已经配置好 Node.js 环境，可以直接使用 npm 安装：

```bash
# 全局安装 OpenClaw
npm install -g openclaw@latest

# 如果遇到 sharp 构建错误，使用以下命令
SHARP_IGNORE_GLOBAL_LIBVIPS=1 npm install -g openclaw@latest

# 安装网关服务
openclaw onboard --install-daemon
```

### 验证安装

```bash
# 检查 OpenClaw 版本
openclaw --version

# 运行健康检查
openclaw doctor

# 查看状态
openclaw status
```

---

## Onboard 配置向导详解

安装完成后，会自动进入配置向导（`openclaw onboard`）。以下是每一步的详细含义和操作说明。

### 步骤一：风险告知

**界面显示**：
```
⚠️  Risk Warning
OpenClaw has access to your files, messages, and can execute commands.
Do you understand and accept these risks?
```

**含义说明**：
- OpenClaw 是一个具有执行能力的 AI 助手，可以访问你的文件、消息，并能执行系统命令
- 这一步是法律和安全告知，确保你了解使用 OpenClaw 的潜在风险
- 如果 AI 被恶意利用或配置不当，可能会造成数据泄露或系统损坏

**操作**：
- 使用 **向左方向键 ←** 选择 **Yes**
- 按 **Enter 回车键** 确认

> ⚠️ **建议**：生产环境建议配置命令白名单，限制 AI 可执行的命令范围

---

### 步骤二：选择配置模式

**界面显示**：
```
Choose Setup Mode:
○ QuickStart (快速入门)
○ Advanced (高级模式)
○ Manual (手动配置)
```

**各模式说明**：

| 模式 | 说明 | 适用场景 |
|------|------|----------|
| **QuickStart** | 自动选择默认配置，最少交互 | 新手用户，快速体验 |
| **Advanced** | 提供更多自定义选项 | 需要精细控制配置的用户 |
| **Manual** | 完全手动配置，跳过向导 | 已有配置文件，或高级用户 |

**操作**：
- 使用 **上下方向键 ↑↓** 选择
- 按 **Enter 回车键** 确认

> 💡 **推荐**：首次使用选择 QuickStart，后续可通过 `openclaw config` 或编辑配置文件调整

---

### 步骤三：配置 AI 模型 API Key

**界面显示**：
```
Select AI Model Provider:
○ Anthropic (Claude)
○ OpenAI (GPT)
○ Qwen (通义千问)
○ Moonshot (Kimi)
○ Custom (自定义)

Enter API Key for [selected provider]:
```

**含义说明**：
- OpenClaw 本身是 AI 框架，需要连接大语言模型才能工作
- 这一步配置 AI 模型的认证信息
- 不同模型的 token 成本和性能差异较大

**国内推荐选择**：

| 提供商 | 模型 | 免费额度 | 特点 |
|--------|------|----------|------|
| **Qwen（通义千问）** | Qwen-2.5 | OAuth 免费 2000 次/天 | 性价比高，中文支持好 |
| **Moonshot（Kimi）** | Moonshot-v1 | 新用户赠送 token | 长文本处理优秀 |
| **智谱 GLM** | GLM-4 | 新用户赠送 token | 代码能力强 |

**操作**：
1. 选择模型提供商
2. 输入 API Key（或按提示完成 OAuth 登录）

> 🔑 **获取 API Key**：
> - Qwen: https://portal.qwen.ai
> - Moonshot: https://platform.moonshot.cn
> - 智谱：https://open.bigmodel.cn

---

### 步骤四：选择 AI 模型

**界面显示**：
```
Select Model:
○ qwen-portal/coder-model (代码模型)
○ qwen-portal/vision-model (视觉模型)
○ qwen-portal/general-model (通用模型)
```

**含义说明**：
- 同一提供商可能有多个模型，针对不同场景优化
- 代码模型适合编程任务，视觉模型支持图片理解，通用模型适合日常对话

**操作**：
- 根据需求选择模型
- 后续可通过 `openclaw models set <model-id>` 切换

---

### 步骤五：连接即时通讯平台

**界面显示**：
```
Connect Messaging Platform:
○ WhatsApp
○ Telegram
○ Discord
○ Slack
○ Feishu (飞书)
○ DingTalk (钉钉)
○ Skip (跳过)
```

**含义说明**：
- 配置 OpenClaw 的交互入口
- 可以选择多个平台，也可以跳过后续配置
- 国内用户推荐选择飞书或钉钉

**平台对比**：

| 平台 | 适用场景 | 配置难度 |
|------|----------|----------|
| **飞书** | 国内企业/个人 | 中等 |
| **钉钉** | 国内企业 | 中等 |
| **Telegram** | 海外/技术用户 | 简单 |
| **Discord** | 社区/游戏 | 简单 |
| **WhatsApp** | 海外个人 | 简单 |

**操作**：
- 选择要连接的平台
- 或选择 **Skip** 跳过，后续通过 `openclaw channels add` 配置

> 💡 **建议**：首次安装可先跳过，完成基础配置后再添加通道

---

### 步骤六：选择 Skills（技能）

**界面显示**：
```
Install Skills?
○ Yes - Install recommended skills
○ No - Skip for now

Available skills:
- weather (天气查询)
- git-essentials (Git 操作)
- ssh-essentials (SSH 连接)
- network (网络工具)
- image-generate (图片生成)
```

**含义说明**：
- Skills 是 OpenClaw 的功能扩展模块
- 每个 Skill 提供特定领域的能力
- 可以后续通过 ClawHub 安装更多技能

**操作**：
- 选择 **Yes** 安装推荐技能
- 或选择 **No** 跳过，后续通过 `npx clawhub install <skill-name>` 安装

> 💡 **建议**：选择 No 跳过，根据实际需求安装技能，避免不必要的依赖

---

### 步骤七：配置 Hooks（钩子）

**界面显示**：
```
Enable Hooks?
[ ] Message logging (消息日志)
[ ] Command auditing (命令审计)
[ ] Auto-backup (自动备份)

Press Space to toggle, Enter to confirm
```

**含义说明**：
- Hooks 是事件触发器，在特定事件发生时执行操作
- 消息日志：记录所有消息往来
- 命令审计：记录 AI 执行的所有命令
- 自动备份：定期备份配置文件

**操作**：
- 按 **空格键** 选中/取消选中
- 按 **Enter 回车键** 确认

> 🔒 **安全建议**：生产环境建议启用命令审计，便于追踪 AI 操作

---

### 步骤八：启动服务并打开 UI 界面

**界面显示**：
```
Setup Complete!

Starting OpenClaw Gateway...
✓ Gateway started on port 18789

Choose an action:
○ Open the Web UI (打开控制面板)
○ View logs (查看日志)
○ Exit (退出)
```

**含义说明**：
- 配置完成后自动启动网关服务
- 网关是 OpenClaw 的核心服务，负责处理消息和 AI 请求
- Web UI 提供可视化管理界面

**操作**：
1. 等待约 30 秒，网关启动完成
2. 选择 **Open the Web UI**
3. 浏览器自动打开控制面板（地址：http://127.0.0.1:18789）

**界面说明**（Web UI）：
- **Dashboard**：系统状态概览
- **Messages**：消息历史记录
- **Skills**：技能管理
- **Channels**：通道配置
- **Settings**：系统设置

---

### 步骤九：测试连接

**操作**：
1. 打开 Web UI 控制面板
2. 查看网关状态是否为 **Running**
3. 在配置的消息平台发送测试消息
4. 观察日志输出和 AI 响应

**验证命令**：
```bash
# 查看网关状态
openclaw gateway status

# 查看实时日志
openclaw logs --follow

# 发送测试消息
openclaw message send --target <user_id> --message "Hello from OpenClaw"
```

---

## 飞书通道配置

### 概述

飞书是国内企业常用的协作平台，接入 OpenClaw 后可以通过飞书机器人与 AI 交互。

**配置流程概览**：
```
创建飞书应用 → 获取凭证 → 配置权限 → 发布应用 → 配置 OpenClaw → 事件订阅 → 测试
```

---

### 1. 安装飞书插件

```bash
openclaw plugins install @openclaw/feishu
```

安装成功后，重启网关使插件生效：
```bash
openclaw gateway restart
```

---

### 2. 创建飞书应用

#### 2.1 访问飞书开放平台

- **国内版**：https://open.feishu.cn/app
- **国际版（Lark）**：https://open.larksuite.com/app

> ⚠️ **注意**：需要使用企业账号或完成实名认证的个人账号

#### 2.2 创建企业应用

**操作步骤**：
1. 登录后，点击右上角「开发者后台」
2. 点击「创建企业应用」按钮
3. 填写应用信息：
   - **应用名称**：如 "OpenClaw 助手"
   - **应用描述**：如 "AI 智能助手"
   - **应用图标**：上传 144x144 像素的图标
4. 点击「创建」

**界面说明**（创建应用页面）：
- 页面顶部显示应用基本信息
- 左侧是功能菜单（凭证、权限、机器人等）
- 右侧是配置区域

#### 2.3 获取应用凭证

**操作步骤**：
1. 在左侧菜单选择「凭证与基础信息」
2. 复制以下两个凭证：
   - **App ID**：格式为 `cli_xxxxxxxxxxxxxxxx`
   - **App Secret**：一串随机字符

> ⚠️ **重要**：
> - App Secret 只显示一次，请妥善保存
> - 不要将凭证提交到代码仓库
> - 建议使用环境变量或加密存储

---

### 3. 配置应用权限

#### 3.1 批量导入权限

**操作步骤**：
1. 在左侧菜单选择「权限管理」
2. 点击「批量导入」按钮
3. 粘贴以下权限配置：

```json
{
  "scopes": {
    "tenant": [
      "aily:file:read",
      "aily:file:write",
      "application:application.app_message_stats.overview:readonly",
      "application:application:self_manage",
      "application:bot.menu:write",
      "cardkit:card:read",
      "cardkit:card:write",
      "contact:user.employee_id:readonly",
      "corehr:file:download",
      "event:ip_list",
      "im:chat.access_event.bot_p2p_chat:read",
      "im:chat.members:bot_access",
      "im:message",
      "im:message.group_at_msg:readonly",
      "im:message.p2p_msg:readonly",
      "im:message:readonly",
      "im:message:send_as_bot",
      "im:resource"
    ],
    "user": ["aily:file:read", "aily:file:write", "im:chat.access_event.bot_p2p_chat:read"]
  }
}
```

4. 点击「确认导入」

**权限说明**：

| 权限 | 用途 |
|------|------|
| `im:message` | 发送和接收消息 |
| `im:message:send_as_bot` | 以机器人身份发送消息 |
| `im:chat.access_event.bot_p2p_chat:read` | 读取私聊消息 |
| `im:chat.members:bot_access` | 获取群组成员信息 |
| `cardkit:card:write` | 发送卡片消息 |
| `aily:file:read/write` | 文件读写能力 |

#### 3.2 启用机器人能力

**操作步骤**：
1. 在左侧菜单选择「应用功能」>「机器人」
2. 点击「启用机器人能力」
3. 设置机器人名称和头像
4. 点击「保存」

**界面说明**（机器人配置页面）：
- 顶部显示机器人预览效果
- 中间是机器人名称和头像设置
- 底部是功能开关（菜单、快捷指令等）

---

### 4. 配置事件订阅

> ⚠️ **重要前提**：
> 1. 已执行 `openclaw channels add` 添加飞书通道
> 2. 网关正在运行（`openclaw gateway status`）

#### 4.1 选择长连接模式

**操作步骤**：
1. 在左侧菜单选择「事件订阅」
2. 选择「使用长连接接收事件」（WebSocket 模式）
3. 点击「保存」

**两种模式对比**：

| 模式 | 说明 | 适用场景 |
|------|------|----------|
| **长连接（WebSocket）** | 实时推送，无需公网 | 推荐，本地部署 |
| **短连接（HTTP）** | 需要公网回调地址 | 云服务器部署 |

#### 4.2 添加事件

**操作步骤**：
1. 点击「添加事件」按钮（此时按钮应变为可点击状态）
2. 选择事件类型：`im.message.receive_v1`（接收消息事件）
3. 点击「确认」

**界面说明**（事件订阅页面）：
- 顶部显示连接模式
- 中间是已订阅事件列表
- 底部是事件详情和测试工具

---

### 5. 配置 OpenClaw 飞书通道

#### 方法一：使用向导（推荐）

```bash
openclaw channels add
```

**交互流程**：
1. 选择通道类型：**Feishu**
2. 输入 **App ID**（步骤 2.3 获取）
3. 输入 **App Secret**（步骤 2.3 获取）
4. 选择域名：**中国（feishu.cn）**
5. 是否接受群组聊天：**Yes**
6. 确认配置：**Yes**
7. 启动通道：**Open**

#### 方法二：配置文件

编辑 `~/.openclaw/openclaw.json`：

```json5
{
  channels: {
    feishu: {
      enabled: true,
      dmPolicy: "pairing",  // 私聊配对策略
      accounts: {
        main: {
          appId: "cli_xxx",
          appSecret: "xxx",
          botName: "OpenClaw 助手",
          region: "cn"  // cn=中国，global=国际
        }
      }
    }
  }
}
```

配置后重启网关：
```bash
openclaw gateway restart
```

---

### 6. 发布应用

**操作步骤**：
1. 在左侧菜单选择「版本管理与发布」
2. 点击「创建版本」
3. 填写版本号和更新说明
4. 点击「提交审核」
5. 等待审批（企业应用通常自动审批）
6. 审批通过后，点击「发布」

**界面说明**（版本管理页面）：
- 顶部显示当前版本状态
- 中间是版本历史记录
- 底部是发布操作按钮

> ⚠️ **注意**：
> - 每次修改配置后需要重新发布版本
> - 未发布的应用无法在飞书中使用

---

### 7. 配对审批

默认情况下，机器人会要求配对审批才能私聊。

**配对流程**：
1. 在飞书中向机器人发送任意消息
2. 机器人回复配对码（如：`ABCD-1234`）
3. 在终端执行审批命令：

```bash
# 查看配对请求
openclaw pairing list feishu

# 审批配对
openclaw pairing approve feishu ABCD-1234
```

**关闭配对审批（可选）**：

编辑 `~/.openclaw/openclaw.json`：
```json5
{
  channels: {
    feishu: {
      dmPolicy: "open"  // open=开放，pairing=需要配对
    }
  }
}
```

---

### 8. 获取用户和群组 ID

#### 用户 ID（open_id）

格式：`ou_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxx`

**方法一**（推荐）：
1. 启动网关并与机器人私聊
2. 运行 `openclaw logs --follow` 查看日志
3. 日志中会显示 `open_id`

**方法二**：
```bash
openclaw pairing list feishu
```

#### 群组 ID（chat_id）

格式：`oc_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxx`

**方法一**（推荐）：
1. 将机器人添加到群组
2. 在群组中 @提及机器人
3. 运行 `openclaw logs --follow` 查看日志
4. 日志中会显示 `chat_id`

**方法二**：
```bash
# 查看机器人加入的群组
openclaw chat info feishu:chat_id
```

---

### 9. 群组配置示例

#### 允许所有群组，需要 @提及（默认）

```json5
{
  channels: {
    feishu: {
      groupPolicy: "open",
      requireMention: true
    }
  }
}
```

#### 仅允许特定群组

```json5
{
  channels: {
    feishu: {
      groupPolicy: "allowlist",
      groupAllowFrom: ["oc_xxx", "oc_yyy"]
    }
  }
}
```

#### 配置群组内发送者白名单

```json5
{
  channels: {
    feishu: {
      groupPolicy: "allowlist",
      groupAllowFrom: ["oc_xxx"],
      groups: {
        oc_xxx: {
          allowFrom: ["ou_user1", "ou_user2"]  // 只允许特定用户
        }
      }
    }
  }
}
```

---

### 10. 测试对话

**在飞书中测试**：
1. 打开飞书客户端或手机 App
2. 找到 OpenClaw 机器人
3. 发送消息：`你好，请介绍一下自己`
4. 观察机器人回复

**测试命令**：
```
/status      # 查看系统状态
/reset       # 重置对话上下文
/help        # 查看帮助信息
```

---

## Skills 安装

Skills 是 OpenClaw 的扩展功能模块，提供各种专用能力。

### 1. 查看可用 Skills

```bash
# 列出所有技能
openclaw skills list

# 查看技能详情
openclaw skills info <skill-name>

# 检查技能就绪状态
openclaw skills check
```

### 2. 使用 ClawHub 安装 Skills

ClawHub 是 OpenClaw 的技能市场：

```bash
# 搜索技能
npx clawhub search <keyword>

# 安装技能
npx clawhub install <skill-name>

# 更新已安装的技能
npx clawhub update

# 同步所有技能到最新版本
npx clawhub sync
```

### 3. 常用 Skills 推荐

#### 天气技能

```bash
npx clawhub install weather
```

提供当前天气和预报功能，支持全球城市。

#### Git 技能

```bash
npx clawhub install git-essentials
```

提供 Git 版本控制命令和工作流，包括：
- 代码提交、分支管理
- 远程仓库操作
- 冲突解决

#### SSH 技能

```bash
npx clawhub install ssh-essentials
```

提供 SSH 远程访问、密钥管理、隧道等功能。

#### 网络技能

```bash
npx clawhub install network
```

提供 TCP/IP、DNS、路由和网络诊断工具。

#### 图片生成技能

```bash
npx clawhub install image-generate
```

使用内置脚本生成图片，需要准备 prompt。

#### 视频生成技能

```bash
npx clawhub install video-generate
```

使用脚本生成视频，可提供首帧图片。

#### 自我改进技能

```bash
npx clawhub install self-improvement
```

捕获学习、错误和修正，实现持续改进。

### 4. 手动安装 Skills

从 ClawHub 下载技能包后：

```bash
# 解压到技能目录
mkdir -p ~/.openclaw/workspace/skills/<skill-name>
cd ~/.openclaw/workspace/skills/<skill-name>

# 按照技能的 README 配置
```

### 5. 验证 Skills 安装

```bash
# 查看已安装的技能
openclaw skills list --eligible

# 详细输出（包含缺失的依赖）
openclaw skills check --verbose
```

---

## 验证与测试

### 1. 系统健康检查

```bash
# 运行完整诊断
openclaw doctor

# 深度检查
openclaw doctor --deep
```

### 2. 网关状态

```bash
# 查看网关状态
openclaw gateway status

# 查看实时日志
openclaw logs --follow

# 重启网关
openclaw gateway restart
```

### 3. 测试消息发送

```bash
# 发送测试消息（需要配置通道）
openclaw message send --target <user_id> --message "Hello from OpenClaw"
```

### 4. 打开控制面板

```bash
# 在浏览器中打开控制面板
openclaw dashboard
```

访问地址：`http://127.0.0.1:18789/`

### 5. 测试对话

在飞书中向机器人发送消息，测试：
- 文本回复
- 命令响应（如 `/status`、`/reset`）
- 文件处理

---

## 常见问题

### 1. `openclaw: command not found`

**原因**: npm 全局 bin 目录不在 PATH 中

**解决方案**:

```bash
# 查找 npm 全局前缀
npm prefix -g

# 添加到 PATH（~/.bashrc 或 ~/.zshrc）
export PATH="$(npm prefix -g)/bin:$PATH"

# 重新加载配置
source ~/.bashrc
```

### 2. sharp 构建错误

**原因**: 系统安装了全局 libvips

**解决方案**:

```bash
SHARP_IGNORE_GLOBAL_LIBVIPS=1 npm install -g openclaw@latest
```

### 3. 飞书机器人不响应

**检查清单**:
- [ ] 应用已发布并审批
- [ ] 事件订阅包含 `im.message.receive_v1`
- [ ] 已启用长连接
- [ ] 应用权限配置完整
- [ ] 网关正在运行：`openclaw gateway status`
- [ ] 查看日志：`openclaw logs --follow`

### 4. 群组中机器人无响应

**检查清单**:
- [ ] 机器人已添加到群组
- [ ] 使用 @提及机器人（默认行为）
- [ ] `groupPolicy` 未设置为 `"disabled"`
- [ ] 检查 `requireMention` 配置

### 5. 模型连接失败

**解决方案**:

```bash
# 检查模型配置
openclaw models status

# 测试连接
openclaw models status --probe

# 重新认证
openclaw models auth login --provider <provider-name>
```

### 6. 权限错误（EACCES）

**原因**: npm 全局目录权限问题

**解决方案**:

```bash
# 使用用户目录
mkdir -p ~/.npm-global
npm config set prefix ~/.npm-global
export PATH=~/.npm-global/bin:$PATH

# 添加到 ~/.bashrc 永久生效
echo 'export PATH=~/.npm-global/bin:$PATH' >> ~/.bashrc
source ~/.bashrc
```

### 7. 飞书插件安装失败：spawn npm ENOENT

**原因**: Windows 系统 npm 路径问题（Linux 较少见）

**解决方案**:

```bash
# 检查 npm 是否可执行
which npm

# 重新安装 npm
sudo apt install -y npm

# 重新安装插件
openclaw plugins install @openclaw/feishu
```

### 8. 端口 18789 被占用

**解决方案**:

```bash
# 使用其他端口启动服务
openclaw gateway --port 18790

# 或修改配置文件
# 编辑 ~/.openclaw/openclaw.json
{
  gateway: {
    port: 18790
  }
}
```

---

## 常用命令速查

| 命令 | 说明 |
|------|------|
| `openclaw onboard` | 重新进入配置向导 |
| `openclaw status` | 查看运行状态 |
| `openclaw health` | 健康检查 |
| `openclaw gateway start` | 启动服务 |
| `openclaw gateway stop` | 停止服务 |
| `openclaw gateway restart` | 重启服务 |
| `openclaw update` | 更新到最新版本 |
| `openclaw doctor` | 诊断问题 |
| `openclaw uninstall` | 卸载 OpenClaw |
| `openclaw logs --follow` | 查看实时日志 |
| `openclaw dashboard` | 打开控制面板 |
| `openclaw plugins install <plugin>` | 安装插件 |
| `openclaw channels add` | 添加通讯通道 |
| `npx clawhub install <skill>` | 安装技能 |

---

## 参考资源

- [OpenClaw 官方文档](https://docs.openclaw.ai)
- [GitHub 仓库](https://github.com/openclaw/openclaw)
- [ClawHub 技能市场](https://clawhub.com)
- [Discord 社区](https://discord.com/invite/clawd)
- [飞书开放平台](https://open.feishu.cn)
- [腾讯云教程原文](https://cloud.tencent.com/developer/article/2626160)

---

## 更新日志

- **2026-03-10**: 初始版本，完整记录 Ubuntu 安装流程
- **2026-03-25**: 优化 onboard 配置向导详解，补充飞书对接文字说明

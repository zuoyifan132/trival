# OpenClaw使用指南

OpenClaw 安装与配置指南
本文档基于 macOS 环境下的真实安装与配置过程编写，涵盖从零开始搭建 OpenClaw、配置自定义 AI 模型（OpenAI 兼容接口）、接入飞书频道的完整流程。

目录
1. 环境准备
2. 安装 OpenClaw
3. 配置 AI 模型
4. 接入飞书频道
5. 启动 Gateway 与验证
6. 完成配对与对话
7. 最终配置文件参考
8. 常见问题排查

1. 环境准备
1.1 系统要求
项目
要求
操作系统
macOS / Linux / Windows (WSL2)
Node.js
≥ 22（必须）
包管理器
pnpm（推荐）或 npm / bun
1.2 安装 Node.js 22
OpenClaw 是一个 Node.js/TypeScript 项目，需要 Node.js 22 或以上版本。
⚠️ 不建议使用 conda 管理 Node.js。conda 中的 Node.js 版本落后，且与 pnpm/npm 生态兼容性差。推荐使用 brew 或 nvm。
方式 A：使用 Homebrew（macOS 推荐）
# 安装 Node.js 22
brew install node@22

# node@22 是 keg-only，需要手动加入 PATH
echo 'export PATH="/opt/homebrew/opt/node@22/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc

# 验证版本
node --version   # 应显示 v22.x.x
npm --version    # 应显示 10.x.x

方式 B：使用 nvm（可管理多版本）
# 安装 nvm
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash

# 重启终端后
nvm install 22
nvm use 22

成功示例：
$ node --version
v22.22.0
$ npm --version
10.9.4

1.3 安装 pnpm
npm install -g pnpm

# 验证
pnpm --version   # 应显示 10.x.x


2. 安装 OpenClaw
2.1 方式一：全局安装（推荐日常使用）
npm install -g openclaw@latest
# 或
pnpm add -g openclaw@latest

2.2 方式二：从源码安装（开发/调试）
如果你需要调试或修改 OpenClaw，可以直接克隆源码：
git clone https://github.com/openclaw/openclaw.git
cd openclaw

# 安装依赖（约 5-7 分钟）
pnpm install

成功示例（依赖安装完成）：
+ typescript 5.9.3
+ vitest 4.0.18
...
Done in 7m 9s using pnpm v10.23.0

2.3 验证安装
# 全局安装
openclaw --help

# 源码模式
pnpm openclaw --help

首次运行会触发 TypeScript 构建（dist is stale），耐心等待即可。构建完成后会显示完整的命令帮助信息。
📝 后续文档中的命令以源码模式（pnpm openclaw）为例。如果使用全局安装，将 pnpm openclaw 替换为 openclaw 即可。

3. 配置 AI 模型
OpenClaw 支持多种 AI 模型提供商。本节以自定义 OpenAI 兼容接口（如阿里 Ling 模型）为例。
3.1 支持的模型 API 类型
API 类型
适用场景
openai-completions
OpenAI Chat Completions 兼容接口
openai-responses
OpenAI Responses API
anthropic-messages
Anthropic Claude
google-generative-ai
Google Gemini
ollama
本地 Ollama
3.2 配置自定义 OpenAI 兼容模型
编辑配置文件 ~/.openclaw/openclaw.json，在 models.providers 中添加自定义 provider：
{
  "models": {
    "providers": {
      "antchat": {
        "baseUrl": "https://antchat.alipay.com/v1",
        "apiKey": "your-api-key-here",
        "api": "openai-completions",
        "models": [
          {
            "id": "ling_v25_1t_6_2_c_20",
            "name": "Ling v25",
            "reasoning": false,
            "input": ["text"],
            "cost": { "input": 0, "output": 0, "cacheRead": 0, "cacheWrite": 0 },
            "contextWindow": 131072,
            "maxTokens": 8192
          }
        ]
      }
    }
  }
}

字段说明：
字段
说明
baseUrl
API 基础 URL（必须包含 /v1）
apiKey
API 密钥
api
API 协议类型，OpenAI 兼容接口使用 "openai-completions"
| models[ ].id | 模型 ID，与 API 中的 model 参数一致 |
| models[ ].contextWindow | 上下文窗口大小（tokens） |
| models[ ].maxTokens | 最大输出 tokens |
| models[ ].reasoning | 是否为推理模型（如 o1/o3） |
| models[ ].input | 输入类型：["text"] 或 ["text", "image"] |
3.3 设置默认模型
pnpm openclaw models set antchat/ling_v25_1t_6_2_c_20

成功示例：
Default model: antchat/ling_v25_1t_6_2_c_20

这会自动在配置中添加以下内容：
{
  "agents": {
    "defaults": {
      "model": {
        "primary": "antchat/ling_v25_1t_6_2_c_20"
      }
    }
  }
}

3.4 其他常见模型配置示例
Anthropic Claude（直接 API Key）：
pnpm openclaw config set models.providers.anthropic.apiKey "sk-ant-api03-..."
pnpm openclaw models set anthropic/claude-sonnet-4-20250514

OpenAI GPT（直接 API Key）：
pnpm openclaw config set models.providers.openai.apiKey "sk-proj-..."
pnpm openclaw models set openai/gpt-4o


4. 接入飞书频道
飞书（Feishu/Lark）通过 WebSocket 长连接接入 OpenClaw，不需要公网 IP 或域名。
4.1 安装飞书插件
# 如果是全局安装的 OpenClaw
openclaw plugins install @openclaw/feishu

# 如果是从源码运行
pnpm openclaw plugins install ./extensions/feishu

成功示例：
Installing to /Users/yourname/.openclaw/extensions/feishu…
Installing plugin dependencies…
Installed plugin: feishu
Restart the gateway to load plugins.

4.2 在飞书开放平台创建应用
第一步：创建应用
1. 用电脑浏览器打开 飞书开放平台
2. 点击 "创建企业自建应用"
3. 填写应用名称（如 "evan_openclaw"）和描述
4. 点击创建
第二步：获取凭证
进入应用 → 左侧 "凭证与基础信息" → 复制：
● App ID（格式：cli_xxx）
● App Secret
第三步：配置权限
左侧 "权限管理" → "批量导入" → 粘贴以下 JSON → 确认：
{
  "scopes": {
    "tenant": [
      "im:message",
      "im:message.p2p_msg:readonly",
      "im:message.group_at_msg:readonly",
      "im:message:send_as_bot",
      "im:message:readonly",
      "im:resource",
      "im:chat.members:bot_access",
      "im:chat.access_event.bot_p2p_chat:read",
      "contact:user.employee_id:readonly",
      "application:bot.menu:write",
      "contact:contact.base:readonly"
    ],
    "user": [
      "im:chat.access_event.bot_p2p_chat:read"
    ]
  }
}

⚠️ contact:contact.base:readonly 很重要——没有这个权限，Bot 无法读取用户名，会显示为 user604938 而不是真实名字。
第四步：开启机器人能力
左侧 "应用能力" → "机器人" → 开启机器人能力 → 设置机器人名称
第五步：配置事件订阅
⚠️ 重要：此步骤需要 Gateway 已经在运行（详见第 5 节），否则保存时会提示"应用未建立长连接"。如果遇到此提示，先完成第 5 节的 Gateway 启动，再回来操作。
左侧 "事件与回调" → "事件配置" 标签页：
1. 点 "订阅方式" 旁的 ✏️ → 选择 "使用长连接接收事件" → 保存
2. 点右上角 "添加事件" → 搜索 im.message.receive_v1 → 选中并添加
第六步：发布应用
左侧 "版本管理与发布" → "创建版本" → 填写版本号（如 1.0.0）→ 发布
自建应用通常自动审核通过。
4.3 配置 OpenClaw
编辑 ~/.openclaw/openclaw.json，添加飞书频道配置：
{
  "channels": {
    "feishu": {
      "enabled": true,
      "dmPolicy": "pairing",
      "accounts": {
        "main": {
          "appId": "cli_a92bae28f638dcb5",
          "appSecret": "your-app-secret-here",
          "botName": "AI助手"
        }
      }
    }
  }
}

字段说明：
字段
说明
enabled
启用飞书频道
dmPolicy
私聊策略："pairing"（首次对话需配对审批）、"open"（开放）、"allowlist"（白名单）
accounts.main.appId
飞书开放平台的 App ID
accounts.main.appSecret
飞书开放平台的 App Secret
accounts.main.botName
机器人显示名称
4.4 配置操作顺序（关键！）
由于飞书事件订阅需要先建立长连接，推荐的操作顺序是：
1. 安装飞书插件
2. 在飞书开放平台创建应用、获取凭证、配置权限、开启机器人
3. 在 OpenClaw 配置文件中写入飞书凭证
4. 启动 Gateway（此时会自动建立 WebSocket 长连接）
5. 回到飞书开放平台，配置事件订阅（选择长连接 + 添加事件）
6. 发布应用
7. 在飞书中找到 Bot 发送消息 → 获取配对码 → 批准配对


5. 启动 Gateway 与验证
5.1 设置 Gateway 模式
首次运行需要设置 gateway.mode=local：
pnpm openclaw config set gateway.mode local

5.2 启动 Gateway
export PATH="/opt/homebrew/opt/node@22/bin:$PATH"
cd /Users/evanzuo/Desktop/work/项目/openclaw
# --force 是防止端口冲突报错，--ws-log full 显示完整的消息内容（包括工具调用参数和返回值）。
nohup pnpm openclaw gateway run --port 18789 --force --ws-log full
重启Gateway
pkill -f "openclaw gateway run"
export PATH="/opt/homebrew/opt/node@22/bin:$PATH"
cd /Users/evanzuo/Desktop/work/项目/openclaw
nohup pnpm openclaw gateway run --port 18789 --force --ws-log full > /tmp/openclaw-gateway.log 2>&1 &

成功示例（关键日志行）：
🦞 OpenClaw 2026.2.25 (8a006a3)

03:55:00 [gateway] agent model: antchat/ling_v25_1t_6_2_c_20
03:55:00 [gateway] listening on ws://127.0.0.1:18789, ws://[::1]:18789 (PID 19306)
03:55:00 [feishu] starting feishu[main] (mode: websocket)
03:55:01 [feishu] feishu[main]: bot open_id resolved: ou_ff7ca1ec9bb1b19b1129241a7d8daff0
03:55:01 [feishu] feishu[main]: WebSocket client started
[ws] ws client ready     ← ✅ 飞书长连接建立成功

需要确认的关键信息：
● agent model: antchat/ling_v25_1t_6_2_c_20 — 模型配置正确
● listening on ws://127.0.0.1:18789 — Gateway 正在监听
● ws client ready — 飞书 WebSocket 长连接已就绪
5.3 后台运行 Gateway
如果需要让 Gateway 在后台持续运行：
# 方式一：在单独的终端窗口运行（推荐调试时使用）
pnpm openclaw gateway run --port 18789

# 方式二：使用 nohup 后台运行
nohup pnpm openclaw gateway run --port 18789 > /tmp/openclaw-gateway.log 2>&1 &

# 查看日志
tail -f /tmp/openclaw-gateway.log

⚠️ Gateway 必须持续运行。关闭终端或按 Ctrl+C 会导致 Bot 离线。

6. 完成配对与对话
6.1 在飞书中发起对话
1. 打开飞书 App（手机或电脑）
2. 在消息列表搜索你的应用名称（如 "evan_openclaw"）
⚠️ 不要跟"开发者小助手"对话，那是飞书官方的系统 Bot。要找你自己创建的应用。
3. 发送一条消息（如 "hi"）
Bot 会回复一条配对信息：
OpenClaw: access not configured.
Your Feishu user id: ou_d746994322dc718c72a747622d14e0cc
Pairing code: B9NT9KAW
Ask the bot owner to approve with:
openclaw pairing approve feishu B9NT9KAW

6.2 批准配对
在另一个终端窗口（不要关闭 Gateway 所在的终端）运行：
pnpm openclaw pairing approve feishu B9NT9KAW

⚠️ 配对码存在 Gateway 运行时的内存中。如果 Gateway 重启了，之前的配对码会失效，需要重新发消息获取新配对码。
6.3 开始对话
配对成功后，直接在飞书中发送消息，Bot 即可正常回复。
成功示例（飞书对话截图描述）：
用户: hi
Bot:  Hey! I see we're connected via Feishu... what's up?


7. 最终配置文件参考
以下是完成所有配置后的完整 ~/.openclaw/openclaw.json：
{
  "meta": {
    "lastTouchedVersion": "2026.2.25",
    "lastTouchedAt": "2026-02-28T08:14:32.474Z"
  },
  "models": {
    "providers": {
      "antchat": {
        "baseUrl": "https://antchat.alipay.com/v1",
        "apiKey": "cspDG6TrYWlGooeC6Wuu2P71LIGO7SWm",
        "api": "openai-completions",
        "models": [
          {
            "id": "ling_v25_1t_6_2_c_20",
            "name": "Ling v25",
            "reasoning": false,
            "input": [
              "text"
            ],
            "cost": {
              "input": 0,
              "output": 0,
              "cacheRead": 0,
              "cacheWrite": 0
            },
            "contextWindow": 65536,
            "maxTokens": 8192
          }
        ]
      }
    }
  },
  "agents": {
    "defaults": {
      "model": {
        "primary": "antchat/ling_v25_1t_6_2_c_20"
      },
      "models": {
        "antchat/ling_v25_1t_6_2_c_20": {}
      },
      "compaction": {
        "mode": "safeguard"
      }
    }
  },
  "tools": {
    "web": {
      "search": {
        "kimi": {
          "apiKey": "sk-tiCPjxId7drQzJiHMWMdGIxaUjvZUEt4l7SnbSLveeKLiN4B"
        }
      }
    }
  },
  "commands": {
    "native": "auto",
    "nativeSkills": "auto",
    "restart": true,
    "ownerDisplay": "raw"
  },
  "channels": {
    "feishu": {
      "enabled": true,
      "dmPolicy": "pairing",
      "accounts": {
        "main": {
          "appId": "cli_a92bae28f638dcb5",
          "appSecret": "lpt5o8qyNALyO0b95dh58bunOJIASgo1",
          "botName": "AI助手"
        }
      },
      "allowFrom": [
        "ou_d746994322dc718c72a747622d14e0cc"
      ]
    }
  },
  "gateway": {
    "mode": "local",
    "auth": {
      "mode": "token",
      "token": "d4291f2489b18bd555026aa5ccd14b1c48b2328baabf311a"
    }
  },
  "plugins": {
    "entries": {
      "feishu": {
        "enabled": true
      }
    },
    "installs": {
      "feishu": {
        "source": "path",
        "sourcePath": "/Users/evanzuo/Desktop/work/项目/openclaw/extensions/feishu",
        "installPath": "/Users/evanzuo/.openclaw/extensions/feishu",
        "version": "2026.2.25",
        "installedAt": "2026-02-27T03:32:13.062Z"
      }
    }
  }
}

8. 常见问题排查
8.1 Gateway start blocked: set gateway.mode=local
原因：首次运行 Gateway 未设置模式。
解决：
pnpm openclaw config set gateway.mode local

8.2 飞书事件订阅保存时提示"应用未建立长连接"
原因：飞书要求在保存长连接配置前，应用必须已经建立了 WebSocket 连接。
解决：
1. 先在 OpenClaw 中配置好飞书凭证
2. 启动 Gateway，等待看到 ws client ready 日志
3. 再回到飞书开放平台保存事件订阅配置
8.3 Bot 显示 user604938 而不是真实用户名
原因：缺少 contact:contact.base:readonly 权限。
解决：
1. 飞书开放平台 → "权限管理" → "批量导入" → 确保包含 contact:contact.base:readonly
2. "版本管理与发布" → 创建新版本并发布
8.4 配对码失效 / No pending pairing request found
原因：配对码存储在 Gateway 运行时内存中，Gateway 重启后失效。
解决：确保 Gateway 持续运行，在飞书中重新发送消息获取新配对码，然后在另一个终端窗口运行 pairing approve（不要中断 Gateway）。
8.5 Telegram 连接超时 / Network request failed
原因：api.telegram.org 在中国大陆无法直接访问。
解决方案：
● 配置代理：pnpm openclaw config set gateway.httpProxy "http://127.0.0.1:7890"（替换为你本机代理端口）
● 改用国内可访问的频道：飞书、钉钉等
8.6 duplicate plugin id detected 警告
原因：从源码运行时，飞书插件同时存在于项目源码目录和 ~/.openclaw/extensions/ 安装目录中。
影响：不影响使用，只是警告信息。可以忽略。
8.7 Gateway 运行后如何查看状态
# 查看 Gateway 进程
ps aux | grep openclaw-gateway

# 查看频道状态
pnpm openclaw channels status

# 查看日志
tail -f /tmp/openclaw/openclaw-$(date +%Y-%m-%d).log

8.8

附录：常用命令速查
命令
说明
pnpm openclaw --help
查看所有命令
pnpm openclaw gateway run --port 18789
启动 Gateway
pnpm openclaw gateway run --port 18789 --force
强制启动（杀掉占用端口的进程）
pnpm openclaw config set <path> <value>
设置配置项
pnpm openclaw models set <provider/model>
设置默认模型
pnpm openclaw models status
查看模型状态
pnpm openclaw channels list
列出已配置的频道
pnpm openclaw channels status
查看频道运行状态
pnpm openclaw plugins install <path>
安装插件
pnpm openclaw pairing list feishu
查看待批准的配对请求
pnpm openclaw pairing approve feishu <CODE>
批准飞书配对
pnpm openclaw doctor
诊断和排查问题
pnpm openclaw agent --message "你好"
通过 CLI 直接与 Agent 对话

本文档基于 OpenClaw 2026.2.25 版本、macOS arm64 环境编写。
配置成功日期：2026 年 2 月 27 日。
python ling_2d5_auto_eval_submit_local.py --model-output-path /mnt/exp/moe-lite/yuchen.fyc/agentic_rl/exp/ling25_off_segpo_multiturn_stable_add_all_general_fc_2d6_data --eval-save-dir /input/yfzuo/output/ling_2d6_eval/ling25_off_segpo_multiturn_stable_add_all_general_fc_2d6_data
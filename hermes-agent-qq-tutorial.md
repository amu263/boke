# 🔗 零基础教程：把 Hermes Agent 接入 QQ 群聊

> **最终效果**：你的 AI 助手（背后是 DeepSeek / Claude / GPT 等任意大模型）能像真人一样在 QQ 群聊里聊天、写代码、画图、部署项目——还能记住上下文！

---

## 📡 整体架构

```
 ┌──────────┐      WebSocket       ┌─────────────┐     HTTP      ┌──────────────┐
 │  NapCat   │ ←─────────────────→ │  NoneBot2    │ ←──────────→ │ Hermes Agent │
 │ (宿主机)   │   onebot v11 协议    │ (Docker容器)  │   OpenAI API  │   (容器内)    │
 │ QQ客户端   │                     │  桥接插件     │              │  大模型后端    │
 └──────────┘                      └─────────────┘              └──────────────┘
```

简单说就是三样东西串起来：

| 组件 | 作用 | 装在哪里 |
|------|------|---------|
| **NapCat** | 像个"遥控器"，操控你的 QQ 号收发消息 | 宿主机（装 QQ 的电脑） |
| **NoneBot2** | 翻译官，把 QQ 消息转成 API 请求喂给大模型 | Docker 容器 |
| **Hermes Agent** | 真正的大脑，对接各种大模型干活 | Docker 容器（和 NoneBot2 同一台） |

---

## 🧰 前置准备

你需要准备以下东西：

### 软件清单

| 软件 | 用途 | 获取方式 |
|------|------|---------|
| **Docker** | 运行容器 | `curl -fsSL https://get.docker.com \| sh` |
| **NapCat** | QQ 机器人框架 | [NapCat 文档](https://napneko.github.io/) |
| **Python 3.11+** | 运行 NoneBot2 | 系统自带或用 `uv` 管理 |
| **Git** | 拉代码 | `apt install git` 或 `brew install git` |
| **一个 QQ 小号** | 当机器人 | 注册一个新的 QQ 号即可 |

### 你需要知道的信息

| 项目 | 示例值 | 怎么获取 |
|------|--------|---------|
| 机器人 QQ 号 | `3999365644` | 你的机器人小号 |
| 目标群号 | `142766765` | 群聊设置里看 |
| 你的 QQ 号 | `3471905205` | 管理员用 |
| 容器 IP | `172.17.0.5` | 进容器后 `hostname -I` |
| 宿主机 IP | `192.168.2.10` | `ip addr` 看局域网 IP |

---

## 🐳 第零步：部署 Hermes Agent（Docker）

Hermes Agent 是整个系统的大脑。官方提供 Docker 镜像，一行命令就能跑。

### 0.1 首次运行：创建数据目录 + 初始化配置

Hermes 的所有数据（配置、API Key、会话、技能）都存在宿主机 `~/.hermes/` 下，挂载进容器的 `/opt/data`。

```bash
# 创建数据目录
mkdir -p ~/.hermes

# 进入交互式初始化向导（配置 API Key、选择模型）
docker run -it --rm \
  -v ~/.hermes:/opt/data \
  nousresearch/hermes-agent setup
```

> 💡 如果你用 Nous Portal 账号，容器里跑 `hermes setup --portal` 一次即可，登录 token 会持久化到 `~/.hermes/`。

### 0.2 以后台 Gateway 模式运行

配好 API Key 后，以后台服务方式跑：

```bash
docker run -d \
  --name hermes \
  --restart unless-stopped \
  -v ~/.hermes:/opt/data \
  -p 8642:8642 \
  nousresearch/hermes-agent gateway run
```

| 参数 | 作用 |
|------|------|
| `-d` | 后台运行 |
| `--restart unless-stopped` | 崩溃/重启后自动恢复 |
| `-v ~/.hermes:/opt/data` | 挂载数据目录 |
| `-p 8642:8642` | 暴露 API 端口给宿主机 |
| `gateway run` | 启动消息网关 + API 服务 |

> 🔒 容器内 gateway 由 **s6-overlay** 自动守护——崩溃秒级重启，不掉容器。

### 0.3 暴露 API 给外部调用（NoneBot2 需要）

默认 API 只监听 `127.0.0.1`。要在其他容器调用，需设环境变量：

```bash
docker run -d \
  --name hermes \
  --restart unless-stopped \
  -v ~/.hermes:/opt/data \
  -p 8642:8642 \
  -e API_SERVER_ENABLED=true \
  -e API_SERVER_HOST=0.0.0.0 \
  -e API_SERVER_KEY="$(openssl rand -hex 32)" \
  nousresearch/hermes-agent gateway run
```

| 环境变量 | 说明 |
|----------|------|
| `API_SERVER_ENABLED=true` | 开启 API 服务（必需） |
| `API_SERVER_HOST=0.0.0.0` | 监听所有网卡 |
| `API_SERVER_KEY=...` | API 认证密钥（≥8 字符） |

> ⚠️ `API_SERVER_KEY` 就是 NoneBot2 调用 API 时用的 `HERMES_API_KEY`。用 `openssl rand -hex 32` 生成一个随机密钥，记下来。

### 0.4 验证

```bash
# 检查健康状态
curl -s http://127.0.0.1:8642/v1/health
# → {"status":"ok"}

# 从其他容器测试（假设 Hermes 容器 IP 为 172.17.0.5）
curl -s http://172.17.0.5:8642/v1/health \
  -H "Authorization: Bearer 你的API_SERVER_KEY"
```

### 0.5 Docker Compose 方式（可选）

如果喜欢 Compose 管理，建一个 `docker-compose.yml`：

```yaml
services:
  hermes:
    image: nousresearch/hermes-agent:latest
    container_name: hermes
    restart: unless-stopped
    command: gateway run
    ports:
      - "8642:8642"
    volumes:
      - ~/.hermes:/opt/data
    environment:
      - API_SERVER_ENABLED=true
      - API_SERVER_HOST=0.0.0.0
      - API_SERVER_KEY=${API_SERVER_KEY}
```

```bash
API_SERVER_KEY="你的密钥" docker compose up -d
```

### 0.6 常用命令

```bash
docker logs -f hermes            # 实时日志
docker exec -it hermes bash      # 进入容器
docker exec hermes hermes setup  # 重新配置
docker restart hermes            # 重启
docker rm -f hermes              # 删除容器（数据在 ~/.hermes 不受影响）
```

### 0.7 挂 Dashboard（可选）

如果想让宿主机访问 Web 管理面板：

```bash
docker run -d \
  --name hermes \
  --restart unless-stopped \
  -v ~/.hermes:/opt/data \
  -p 8642:8642 \
  -p 9119:9119 \
  -e HERMES_DASHBOARD=1 \
  -e API_SERVER_ENABLED=true \
  -e API_SERVER_HOST=0.0.0.0 \
  -e API_SERVER_KEY="你的密钥" \
  nousresearch/hermes-agent gateway run
```

打开 `http://宿主机IP:9119` 就能看到仪表盘。

> ❗ Dashboard 暴露 API Key，不要开在公网。生产环境走 SSH 隧道或反向代理加认证。

### 0.8 升级

```bash
docker pull nousresearch/hermes-agent:latest
docker rm -f hermes
# 重新执行 0.2 的 docker run 命令（数据在 ~/.hermes，不会丢）
```

---

## 🚀 第一步：安装 NapCat（宿主机）

NapCat 是 QQ 机器人的"驱动程序"，它通过 WebSocket 把你的 QQ 号变成可编程接口。

### 1.1 使用 Docker 安装 NapCat

```bash
# 拉取 NapCat Docker 镜像
docker run -d \
  --name napcat \
  --restart always \
  -p 6099:6099 \
  -v ~/napcat/data:/app/.config/QQ \
  -v ~/napcat/config:/app/napcat/config \
  mlikiowa/napcat-docker:latest
```

### 1.2 配置 WebSocket 服务端

打开 NapCat WebUI：`http://你的宿主机IP:6099/webui`

进入 **网络配置** → **添加 WebSocket 服务端**：

```
名称：NoneBot
URL：ws://172.17.0.5:8080/onebot/v11/ws
```

> ⚠️ `172.17.0.5` 是你 Docker 容器的 IP。如果容器 IP 不同，改成实际的。

### 1.3 开启 HTTP 服务（给 NoneBot 用）

同样在 WebUI 里，确保勾选：
- ✅ HTTP 服务
- ✅ WebSocket 服务端

保存后 NapCat 会尝试连接，此时还没人接——等后面 NoneBot2 跑起来就好了。

### 1.4 登录 QQ 与查看令牌

如果你用的是扫码登录，NapCat WebUI 首页会显示一个二维码。用机器人 QQ 号扫码登录即可。

**查看登录令牌（Token）**：

登录成功后，令牌保存在容器内的配置文件中。两种方式查看：

**方式一：通过 WebUI（推荐）**

打开 `http://你的IP:6099/webui` → **网络配置**，在 HTTP 服务配置里能看到 `access_token`（用于 NoneBot / HTTP API 调用）。

**方式二：命令行**

```bash
# 进入 NapCat 容器
docker exec -it napcat sh

# 令牌在 onebot11 配置里
cat /app/napcat/config/onebot11_你的QQ号.json | grep token
```

**方式三：宿主机直接看**

```bash
# 因为你映射了配置目录
cat ~/napcat/config/onebot11_*.json | grep token
```

> 💡 这个 token 是 NapCat HTTP API 的认证令牌，如果你需要通过 HTTP（而非 WebSocket）调用 NapCat API，就需要它。NoneBot2 用的是 WebSocket 方式，一般不需要这个 token。

---

## 🔑 第二步：确认 Hermes API 可用

在第一步启动 Hermes 容器时，已经通过 `-e API_SERVER_KEY=...` 设置了 API 密钥。现在验证一下：

```bash
# 检查健康
curl -s http://127.0.0.1:8642/v1/health
# → {"status":"ok"}

# 用 API Key 测试调用
curl -s http://127.0.0.1:8642/v1/chat/completions \
  -H "Authorization: Bearer 你的API_SERVER_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model":"default","messages":[{"role":"user","content":"你好"}]}'
```

记好你的 `API_SERVER_KEY`，下一步 NoneBot2 要用。

---

## 🏗️ 第三步：创建 NoneBot2 桥接项目（容器内）

NoneBot2 是 Python 写的 QQ 机器人框架，我们用它接收 QQ 消息并转发给 Hermes。

> 💡 以下命令全部在 **Hermes 容器内**执行。先进容器：`docker exec -it hermes bash`

### 3.1 创建目录结构

```bash
# 进容器
docker exec -it hermes bash

# 创建工作目录
mkdir -p /opt/data/qq-bot/plugins
mkdir -p /opt/data/qq-bot/prompts
mkdir -p /opt/data/qq-bot/state
cd /opt/data/qq-bot
```

### 3.2 安装依赖

```bash
# 用 uv 安装（Hermes 容器自带）
uv pip install nonebot2 nonebot-adapter-onebot httpx pyyaml
```

### 3.3 创建启动文件 `bot.py`

```bash
cat > bot.py << 'PYEOF'
import nonebot
from nonebot.adapters.onebot.v11 import Adapter as ONEBOT_V11Adapter

nonebot.init(host="0.0.0.0", port=8080)
driver = nonebot.get_driver()
driver.register_adapter(ONEBOT_V11Adapter)
nonebot.load_plugins("plugins")

if __name__ == "__main__":
    nonebot.run()
PYEOF
```

### 3.4 创建环境变量 `.env`

```bash
cat > .env << 'ENVEOF'
HERMES_API_URL=http://127.0.0.1:8642/v1/chat/completions
HERMES_API_KEY=你在第零步设的API_SERVER_KEY
HERMES_QQ_ALLOWED_GROUPS=你的群号
HERMES_QQ_BOT_QQ=机器人QQ号
HERMES_QQ_ADMIN_QQ=你的QQ号(管理员)
ENVEOF
```

### 3.5 创建系统提示词 `prompts/niku.txt`

```bash
cat > prompts/niku.txt << 'TXLEOF'
你是一个在 QQ 群里帮助用户的 AI 助手。请遵循以下原则：

1. 默认使用中文回复
2. 保持简洁，群聊不需要长篇大论
3. 如果被要求画图/生图，调用生图能力，只返回图片
4. 如果被问及 API 状态或服务状态，如实回答
5. 不知道就说不知道，不要编造
6. 友好但不过度热情
TXLEOF
```

### 3.6 创建 `pyproject.toml`

```bash
cat > pyproject.toml << 'TOMLEOF'
[project]
name = "qq-bot"
version = "0.1.0"
requires-python = ">=3.11"
dependencies = [
    "nonebot2",
    "nonebot-adapter-onebot",
    "httpx",
    "pyyaml",
]
TOMLEOF
```

### 3.7 验证目录结构

```bash
tree /opt/data/qq-bot
```

应该看到：

```
/opt/data/qq-bot/
├── bot.py
├── .env
├── pyproject.toml
├── plugins/
├── prompts/
│   └── niku.txt
└── state/
```

> ✅ 项目骨架搭好了，接下来写核心插件。

---

## 🧠 第四步：编写桥接插件（核心）

这是整个项目的灵魂——`plugins/hermes_bridge.py`。

> 完整的生产级代码在 [GitHub](#)，以下按功能模块讲解。

### 4.1 基础骨架

```python
import os, re, time, httpx, asyncio
from nonebot import on_message
from nonebot.adapters.onebot.v11 import Bot, GroupMessageEvent, MessageSegment

# ── 读取配置 ──
HERMES_API_URL = os.getenv("HERMES_API_URL",
    "http://127.0.0.1:8642/v1/chat/completions")
HERMES_API_KEY = os.getenv("HERMES_API_KEY", "")

# 如果 .env 里没设 API Key，从 Hermes 配置里读
if not HERMES_API_KEY:
    import yaml
    with open("/opt/data/config.yaml") as f:
        HERMES_API_KEY = yaml.safe_load(f)["gateway"]["api_key"]

# ── 消息匹配 ──
matcher = on_message(priority=1, block=False)
```

### 4.2 提取 @消息

QQ 的 @消息有特殊格式 `[CQ:at,qq=机器人QQ]`，需要手工提取：

```python
def extract_prompt(event, self_id: str) -> str | None:
    raw = event.raw_message  # ⚠️ 用 raw_message，不能用 str(event.message)
    self_at = f"[CQ:at,qq={self_id}]"
    if self_at not in raw:
        return None  # 没人 @机器人，忽略
    text = raw.replace(self_at, "").strip()
    return text or None
```

### 4.3 调用 Hermes API

```python
async def ask_hermes(prompt: str, group_id: str) -> str:
    headers = {"Authorization": f"Bearer {HERMES_API_KEY}"}
    payload = {
        "model": "default",
        "messages": [
            {"role": "system", "content": system_prompt},
            {"role": "user", "content": prompt},
        ],
        "stream": False,
    }
    async with httpx.AsyncClient(timeout=7200) as client:
        r = await client.post(HERMES_API_URL, headers=headers, json=payload)
        r.raise_for_status()
        return r.json()["choices"][0]["message"]["content"]
```

> ⚠️ `timeout=7200` 是 2 小时——部署类长任务不会超时断连。

### 4.4 主消息处理

```python
@matcher.handle()
async def handle(bot: Bot, event: GroupMessageEvent):
    prompt = extract_prompt(event, str(event.self_id))
    if not prompt:
        return  # 没人 @，不处理

    # 检查冷却
    cd = check_cooldown(group_id, user_id)
    if cd:
        await bot.send(event, cd)
        return

    # 识别长任务，立刻告诉用户"在处理"
    if any(kw in prompt for kw in ["部署", "安装", "构建", "下载"]):
        await bot.send(event, "⏳ 收到，正在处理中，可能需要一些时间...")

    # 调用 Hermes
    answer = await ask_hermes(prompt, group_id)

    # 分段发送（QQ 单条消息有长度限制）
    for chunk in split_text(answer, 1800):
        await bot.send(event, chunk)
```

### 4.5 对话记忆（防遗忘）

```python
MAX_HISTORY = 100       # 最多存 100 条
RECENT_WINDOW = 20      # 最近 20 条原文发给模型
SUMMARY_TRIGGER = 10    # 每新增 10 条重新生成摘要

_chat_history = {}   # {群号: [{role, content}, ...]}
_chat_summary = {}   # {群号: "摘要文本"}

async def build_messages(prompt, group_id):
    history = _chat_history.get(group_id, [])
    recent = history[-RECENT_WINDOW:]  # 最近 20 条原文
    old = history[:-RECENT_WINDOW]     # 更早的压缩成摘要

    # 定期自动摘要旧消息（调用 Hermes 自己生成）
    if len(old) >= SUMMARY_TRIGGER:
        await generate_summary(old, group_id)

    messages = [{"role": "system", "content": system_prompt}]
    if old:
        messages.append({"role": "system", "content": f"[对话摘要] {_chat_summary[group_id]}"})
    messages.extend(recent)
    messages.append({"role": "user", "content": prompt})
    return messages
```

> 🧠 **这样设计的好处**：最近的对话一字不差，更早的自动压缩。既不会忘事，也不浪费 token。

---

## 🔌 第五步：连接 NapCat + NoneBot2

### 5.1 启动 NoneBot2

```bash
cd /opt/data/qq-bot
.venv/bin/python bot.py
```

看到以下日志说明成功：

```
[SUCCESS] nonebot | NoneBot is initializing...
[SUCCESS] nonebot | Succeeded to load plugin "hermes_bridge"
[SUCCESS] nonebot | Loaded adapters: OneBot V11
[INFO] uvicorn | Application startup complete.
```

### 5.2 NapCat 自动连接

NapCat 会不停尝试连接 `ws://172.17.0.5:8080/onebot/v11/ws`。NoneBot2 起来后，NapCat WebUI 里会显示：

```
✅ WebSocket Client: 已连接
```

### 5.3 测试第一条消息

在群里 **@机器人 你好**，你应该收到：

```
收到啦！有什么可以帮你的？
```

---

## ✅ 第六步：测试与调试

### 确认功能清单

| 测试项 | 操作 | 预期 |
|--------|------|------|
| @触发 | @机器人 你好 | 回复打招呼 |
| 管理员命令 | @机器人 /状态 | 显示运行状态 |
| 长回复分段 | @机器人 列出所有 Linux 命令 | 分多条发送 |
| 画图 | @机器人 画一只猫 | 回复图片 |
| 上下文记忆 | 先问"我叫小明"，再问"我叫什么" | 能答出"小明" |
| 长任务 | @机器人 部署 xxx | 先回"处理中"，完成后给结果 |
| 冷却 | 连续 @两次 | 第二次提示冷却 |

### 调试技巧

```bash
# 查看实时日志
tail -f nonebot.log

# 只看错误
grep -i error nonebot.log

# 检查插件是否加载
grep "load plugin" nonebot.log
```

---

## 🔧 第七步：生产级功能

### 7.1 冷却机制

```python
COOLDOWN_GROUP_S = 3   # 群冷却 3 秒
COOLDOWN_USER_S = 10   # 每人冷却 10 秒
```

防止刷屏和 API 费用爆炸。

### 7.2 管理员命令

| 命令 | 功能 |
|------|------|
| `/状态` | 查看运行状态、记忆数 |
| `/查图` | 最近生成的图片链接 |
| `/清记忆` | 清空群聊对话历史 |
| `/帮助` | 显示所有命令 |

### 7.3 自动重启

用 systemd 或 supervisor 保活：

```ini
# /etc/systemd/system/qq-bot.service
[Unit]
Description=QQ Bot (NoneBot2)
After=network.target

[Service]
Type=simple
WorkingDirectory=/opt/data/qq-bot
ExecStart=/opt/data/qq-bot/.venv/bin/python bot.py
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

```bash
systemctl enable qq-bot --now
```

### 7.4 每日群聊总结

在 `crontab -e` 里加上：

```bash
0 22 * * * curl -s "http://127.0.0.1:8642/v1/api/chat/completions" \
  -H "Authorization: Bearer $HERMES_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model":"default","messages":[{"role":"user","content":"总结今天群里的聊天内容，不超过500字"}]}'
```

---

## 🐛 附录：常见踩坑

### Q1: NapCat 连不上 NoneBot2

**症状**：WebUI 显示"连接失败"或"等待重连"

**排查**：

```bash
# 1. 确认 NoneBot2 在运行
ps aux | grep bot.py

# 2. 确认端口监听
cat /proc/net/tcp | grep ':1F90'  # 1F90 是 8080 的十六进制

# 3. 确认容器间网络互通（从容器里 ping 宿主机）
ping 192.168.2.10
```

### Q2: @机器人 没反应

**原因**：QQ 的 @消息格式有两种，插件用了 `raw_message` 不会漏。

**确认**：看日志里有没有收到消息：

```bash
grep "message.group.normal" nonebot.log
```

### Q3: API Key 认证失败 (401)

**症状**：日志里报 `401 Unauthorized`

**解决**：确保 API Key 和 Hermes Gateway 配置里的一致。建议让插件直接从 Hermes 配置文件读，避免 `.env` 转义问题：

```python
import yaml
with open("/opt/data/config.yaml") as f:
    api_key = yaml.safe_load(f)["gateway"]["api_key"]
```

### Q4: 长任务没回复

**症状**：部署/安装类请求，什么都没回

**原因**：默认超时太短（300 秒），Hermes 还在干活就断开了。

**解决**：把 `httpx.AsyncClient(timeout=7200)` 设为 2 小时。

### Q5: 机器人忘了前面说的

**症状**：连续对话中突然"失忆"

**原因**：没有保存对话历史，每次都是全新会话。

**解决**：参考第 4.5 节的对话记忆机制——存历史 + 自动摘要。

---

## 📂 完整项目结构

```
/opt/data/qq-bot/
├── bot.py                 # 启动入口
├── .env                   # 环境变量（敏感信息）
├── pyproject.toml         # Python 项目配置
├── plugins/
│   └── hermes_bridge.py  # 核心桥接插件（~330 行）
├── prompts/
│   └── niku.txt           # 系统提示词
├── state/                 # 运行状态缓存
└── nonebot.log            # 运行日志
```

---

## 🎉 总结

通过这套架构，你只需要：

1. **宿主机跑 NapCat** — 操控 QQ
2. **容器里跑 NoneBot2** — 翻译消息
3. **容器里跑 Hermes Agent** — 调用大模型

三样东西一串联，AI 就出现在了你的 QQ 群聊里。而且因为背后是 Hermes Agent，你可以随意换模型（Claude / DeepSeek / GPT / Qwen……），不用改任何代码。

---

*写于 2026 年 5 月 · 基于实际部署过程整理*

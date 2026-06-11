<div align="center">

<img src="docs/screenshots/banner.svg" alt="qwen2api-rs — Qwen Web → OpenAI · Anthropic · Gemini" width="100%"/>

# qwen2api-rs

把通义千问（Qwen）Web 端能力转换成 **OpenAI / Anthropic Claude / Gemini** 兼容接口的自托管网关。

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Rust](https://img.shields.io/badge/Rust-1.80%2B-dea584?logo=rust&logoColor=white)](https://www.rust-lang.org)
[![axum](https://img.shields.io/badge/axum-0.8-000000)](https://github.com/tokio-rs/axum)
[![tokio](https://img.shields.io/badge/tokio-1.x-3776ab)](https://tokio.rs)
[![Docker](https://img.shields.io/badge/docker-ready-2496ed?logo=docker&logoColor=white)](https://www.docker.com)

**语言**：[English](README.md) · [繁體中文](README.zh-TW.md) · **简体中文**

</div>

> 💡 **想直接体验服务效果？** 我自己部署了一份公开可用的实例，可从这里试用 → **<https://x.com/i/status/2063271049199046672>**
>
> （本仓库是给想自建的人；只想用一下不必自己跑。）

> **本项目是 [YuJunZhiXue/qwen2API](https://github.com/YuJunZhiXue/qwen2API)（Python + React）的 Rust 后端 + 纯原生前端重写版**，并非原作者，原始设计与协议分析归功于上游。基准上游版本与同步流程见 [`dev/UPSTREAM.md`](dev/UPSTREAM.md)。

- **后端**：Rust（`axum` + `tokio` + `reqwest` + `serde`），单一静态二进制，低内存、高并发。
- **前端**：纯 `HTML + CSS + JS` 三档（`web/`），零框架、零构建、可离线。
- **协议**：在同一个 binary 内同时提供 OpenAI / Anthropic / Gemini 三套 API 表面。

---

## 功能

- ✅ OpenAI Chat Completions（`/v1/chat/completions`）流式 + 非流式
- ✅ OpenAI Responses（`/v1/responses`）typed SSE events
- ✅ Anthropic Messages（`/v1/messages`、`/anthropic/v1/messages`）流式 + 非流式 + `count_tokens`
- ✅ Gemini `generateContent` / `streamGenerateContent`
- ✅ OpenAI Images（`/v1/images/generations`）— 驱动 Qwen 图像生成
- ✅ OpenAI Embeddings（占位，确定性向量）
- ✅ 文件上传（`/v1/files`）+ 对话附件（自动阿里云 OSS V4 上传 / 小文本内联）
- ✅ 工具/函数调用：工具定义注入 prompt + 从输出解析 `tool_call`（Qwen Web 无原生工具支持）
- ✅ 思考模式（reasoning）流式输出，**usage 采用上游真实 token 数**
- ✅ 账号池：4 层并发控制、最少负载选号、限流指数退避、跨账号重试
- ✅ chat_id 预热池（规避上游 `/chats/new` 0.5~6s 握手；对上万个账号设有覆盖数上限保护）
- ✅ 管理台 WebUI：运行状态、账号管理、API Key、接口测试、图片生成、系统设置
- ✅ `/healthz`、`/readyz` 探针

## 界面预览

<table>
  <tr>
    <td align="center" width="50%">
      <img src="docs/screenshots/stats.png" alt="统计面板：请求量、Tokens、TTFT、按模型/接口拆分"/>
      <sub><b>数据统计</b> · 请求 / Tokens / TTFT 分桶，按模型 + 接口拆分</sub>
    </td>
    <td align="center" width="50%">
      <img src="docs/screenshots/images.png" alt="图片生成：批量提交与本地画廊"/>
      <sub><b>图片生成</b> · 批量提交、比例切换、本地永久保存</sub>
    </td>
  </tr>
  <tr>
    <td align="center" colspan="2">
      <img src="docs/screenshots/videos.png" alt="视频生成：异步任务队列 + 智能跳过无 t2v 权限账号" width="70%"/>
      <br/><sub><b>视频生成</b> · 异步任务队列 + 自动重试 + 智能跳过无 t2v 权限账号</sub>
    </td>
  </tr>
</table>

## 快速开始

环境要求：Rust 1.80+（已测 1.93）。

```bash
cp .env.example .env          # 设置 ADMIN_KEY、PORT 等
mkdir -p data
# 放入账号：data/accounts.json = [{"email","token", ...}, ...]
#   token = 在 chat.qwen.ai 登录后，localStorage 里的 token 原始值
#   模板见 dev/accounts.example.json
cargo run --release
```

启动后：
- WebUI：`http://127.0.0.1:7860/`（系统设置页粘贴 `ADMIN_KEY` 或任一 API Key 作为会话密钥）
- API Base：`http://127.0.0.1:7860`

调用示例（OpenAI 兼容）：

```bash
curl http://127.0.0.1:7860/v1/chat/completions \
  -H "Authorization: Bearer <你的 API Key 或 ADMIN_KEY>" \
  -H "Content-Type: application/json" \
  -d '{"model":"gpt-4o","messages":[{"role":"user","content":"你好"}],"stream":true}'
```

Anthropic 兼容：

```bash
curl http://127.0.0.1:7860/v1/messages \
  -H "x-api-key: <你的 API Key 或 ADMIN_KEY>" \
  -H "anthropic-version: 2023-06-01" \
  -H "Content-Type: application/json" \
  -d '{"model":"claude-3-5-sonnet-20241022","max_tokens":1024,"messages":[{"role":"user","content":"你好"}]}'
```

模型名可用任意 OpenAI / Claude / Gemini 名称（自动映射到 Qwen，未知名称回退到 `DEFAULT_MODEL`），或直接用 `qwen3.7-plus`、`qwen3.7-plus-thinking` 等（`/v1/models` 可查全部）。

## 部署

两种方式皆可（单一静态 binary，rustls 无需系统 OpenSSL）。

### Docker（推荐，尤其从原 Python 版迁移者）

与原项目相同的 docker-compose 工作流；镜像约 145MB（debian-slim 基底，比原 Python 版含 camoufox 小很多；可改 distroless/musl 再瘦身）。

```bash
# data/ 可直接沿用原版（放 accounts.json 等）
mkdir -p data
vim docker-compose.yml      # 修改 ADMIN_KEY 等
docker compose up -d --build
docker compose logs -f
```

- 数据持久化：`./data` 挂载到容器 `/app/data`。
- 内置 `HEALTHCHECK`（打 `/healthz`）。
- 更新：`git pull && docker compose up -d --build`。

### Binary（最轻量，单机 / VPS）

```bash
cargo build --release          # 产出 target/release/qwen2api-rs
cp .env.example .env && vim .env
mkdir -p data                  # 放 accounts.json
WEB_DIR=web ./target/release/qwen2api-rs
```

建议用 systemd 常驻（`/etc/systemd/system/qwen2api-rs.service`）：

```ini
[Unit]
Description=qwen2api-rs gateway
After=network.target

[Service]
WorkingDirectory=/opt/qwen2api-rs
EnvironmentFile=/opt/qwen2api-rs/.env
ExecStart=/opt/qwen2api-rs/qwen2api-rs
Restart=always

[Install]
WantedBy=multi-user.target
```

> **Docker vs Binary 取舍**：Docker = 可复现、隔离、跨发行版可移植、与原版同流程、易更新 / 重启；Binary = 启动最快、占用最小、无需 docker，但需自行用 systemd 常驻且跨机器需注意 glibc 版本（或用 musl 静态编译）。

## 环境变量

完整列表见 [`.env.example`](.env.example)（含风控 / 账号池 / 上下文等 20+ 项）。变量名与原 Python 版兼容，可直接指向同一份 `data/`。

| 变量 | 用途 | 默认 |
|---|---|---|
| `PORT` | 服务端口 | `7860` |
| `ADMIN_KEY` | 管理台密钥 | `change-me-now` |
| `MAX_INFLIGHT_PER_ACCOUNT` | 每账号同时在途请求数 | `2` |
| `MAX_RETRIES` | 跨账号重试次数 | `3` |
| `ACCOUNT_MIN_INTERVAL_MS` | 同账号最小请求间隔（风控） | `3000` |
| `CHAT_ID_PREWARM_TARGET_PER_ACCOUNT` | chat_id 预热池每账号目标数 | `5` |
| `DEFAULT_MODEL` | 未知下游模型回退 | `qwen3.7-plus` |
| `DATA_DIR` | 数据目录 | `./data` |

## 认证

- 下游请求：`Authorization: Bearer <key>`、`x-api-key`、或 `?key=`。
- 若 `data/api_keys.json` 有配置 key，则必须使用 `ADMIN_KEY` / 已创建的 key；否则放行任意 key。
- 管理台 `/api/admin/*`：`Bearer` 须等于 `ADMIN_KEY` 或已创建的 key。

## 架构

技术栈与 Python→Rust 模块对应见 [`dev/ARCHITECTURE.md`](dev/ARCHITECTURE.md)；实测捕捉的上游协议（含 SSE 格式）见 [`dev/PROTOCOL.md`](dev/PROTOCOL.md)。

```
src/
  main.rs            入口 / 路由
  config.rs state.rs db.rs error.rs util.rs auth.rs
  account/           账号池（account.rs / pool.rs）
  upstream/          上游传输（client / payload / sse / executor / chat_id_pool）
  request/           标准请求构建（model_modes / prompt_builder / client_profiles / model_catalog）
  toolcall/          工具调用（注入 + 解析 + 名称混淆）
  execution/         编排 + 流式翻译（translator / presenter / formatters）
  context/           附件 / OSS V4 上传 / 本地文件库
  api/               各协议端点（openai / anthropic / gemini / responses / images / videos / files / embeddings / admin / probes）
  stats.rs           SQLite 统计子系统
  media.rs           媒体任务队列（图片 / 视频）
web/                 纯前端三档（index.html / app.js / style.css）
dev/                 开发笔记（上游版本追踪、架构、协议捕捉、部署）
```

## 与原项目 ([YuJunZhiXue/qwen2API](https://github.com/YuJunZhiXue/qwen2API)) 的刻意差异

1. **移除浏览器自动注册**（camoufox / Playwright + 临时邮箱）→ 仅手动贴 token；额外提供 `chat.qwen.ai` 纯 HTTP signin + 自动 refresh worker。
2. **usage 改用上游真实 token 数**（原版用字符数估算）。
3. **默认旗舰模型**更新为 `qwen3.7-plus`。
4. **工具调用**采用单一稳定的文本格式（`<tool_call>{json}</tool_call>`）注入 + 解析（brace-balanced，无 regex 失稳）。
5. **媒体子系统**：图片 / 视频重构为异步任务队列（SQLite 持久化）+ 自动重试 + 智能跳过无 t2v 权限账号。
6. **可观测性**：新增 `stats.rs` 统计面板（请求量 / Tokens / TTFT / 按模型 + 接口拆分）。

详见 [`dev/UPSTREAM.md`](dev/UPSTREAM.md)。

## 致谢

- 上游原项目 [**YuJunZhiXue/qwen2API**](https://github.com/YuJunZhiXue/qwen2API) — 原始协议分析、多协议转换思路、整套账号池 / 风控设计皆源自此。
- Qwen / 通义千问 by 阿里巴巴 — 模型与 Web 界面。

## 许可

[MIT License](LICENSE)

> 仅供学习与自托管研究使用。Qwen 为阿里巴巴商标，使用须遵守其服务条款；本项目不保证上游 Web 接口行为的稳定性。

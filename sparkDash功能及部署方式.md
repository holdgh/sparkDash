# sparkDash 功能及部署方式

> 基于当前仓库（v2.0.0，Apache-2.0，Mia'a AI Lab）的分析整理，供后续实操查阅。
> 分析日期：2026-10-10 · 来源：`README.md` / `package.json` / `Dockerfile` / `docker-compose.yml` / `.env.example`

---

## 一、项目是做什么的

sparkDash 是一个**多机实时监控 Web 仪表盘**，面向 NVIDIA DGX Spark（GB10 平台），也可监控任意带 NVIDIA GPU 的 Linux 主机（作为 `host` 类型单元）。

- 一个浏览器窗口同时查看任意数量的 Spark / GPU 主机
- 实时推送 GPU、CPU、统一内存（GB10 128GB LPDDR5X）、存储、网络、LLM 推理指标（WebSocket + sparkline 历史）
- 支持从 UI 直接增删改排监控单元（热配置，无需重启）

---

## 二、核心功能

### 1. 监控采集
| 能力 | 说明 |
|------|------|
| 多单元 | 每个单元有独立详情页 + 共享 Overview；角色 Head / Worker / Standalone |
| 本地采集 | 通过 sysfs / proc / `nvidia-smi` 读取宿主指标（常经 nsenter 进入宿主命名空间） |
| 远程采集 | SSH（密钥或密码）执行同样的命令；连接复用（`SSH_CONTROL_PERSIST_SECONDS`，默认 60s）避免 SSH/PAM 会话风暴 |
| 非 Spark 主机 | Linux + 独立 NVIDIA GPU 工作站可作为 `kind: host` 单元，**RAM 与 VRAM 分开显示**，多 GPU 主机逐卡展示 |
| 实时流 | WebSocket `/ws` 快照推送；轮询间隔可配（默认 2000ms，最小 1000ms；`nvidia-smi dmon` 带宽项不低于 2s） |
| 优雅降级 | 采集出错返回零值/默认值，持续失败标记 offline，不崩溃 |

### 2. LLM 相关
- **LLM 探测**：自动识别 llama.cpp、vLLM、sglang、ds4-server、EXL3、TensorFold、FreeToken、q27；实时 decode/prefill tok/s、TTFT、KV cache、队列、p95 延迟（vLLM/q27 走 Prometheus `/metrics`）
- **多端口**：同一单元可监控多个 LLM 服务，各自独立探测
- **启停 LLM**：注册 start.sh / stop.sh 启动器，从 UI 启停并流式查看 shell 输出
- **Token 统计 / 每日峰值**：累计 token 按模型统计 + 历史曲线

### 3. 基准测试（Bench）
- **Decode**：多并发流式 decode tok/s，文本类型选择（Structured / Prose / Code / JSON）
- **Prefill**：上下文 1k–300k 扫描 prefill tok/s 与 TTFT
- **Quality**：固定种子题库（QA、推理、算术、状态跟踪、GSM8K、MMLU 等），temperature 0，保留最近 30 次，支持配对比较 + McNemar p 值
- **Tool Eval Bench**：工具调用评测（trials、并排比较）
- 远程目标：LAN HTTP 或 SSH 隧道到 loopback；一次性远程主机需列入 `SPARKDASH_BENCH_HOSTS`

### 4. 可选集成
| 集成 | 默认 | 说明 |
|------|------|------|
| ComfyUI | 关 | 队列/任务/进度/取消、模型清单、Overview 芯片；`comfyMonitoring` + `comfyPort`(8188) |
| Hermes Agent | 关 | 后台更新检查（10 分钟）、状态徽章、一键/批量 `hermes update` |
| Tailnet | 关 | 标记"局域网通但脱离 tailnet"的单元 |
| Prometheus | 关 | `/metrics` 暴露 |
| Glance | — | 集成看板 |
| Wake-on-LAN | — | 仅支持的专用 GPU 主机网卡；DGX Spark 不响应魔术包 |

### 5. 其他
- 电源控制：SSH 优雅关机
- 4 套主题（White / Light / Dark / OLED），localStorage 持久化
- 命令面板 Ctrl/⌘ K、侧边栏、页面过渡
- Prompt Showcase：整页多终端 LLM 流式演示（最多 32 条 prompt）
- Fleet energy：整机功耗/能耗估算（GPU + CPU 模型 + 23W 基础开销，为**估算值**非墙插实测）
- 活动事件日志（保留 2000 条）

---

## 三、技术栈与目录结构

| 层 | 技术 |
|----|------|
| 前端 | React 19、TypeScript、Vite 8、Tailwind CSS v4（`src/`） |
| 后端 | Node.js（ESM）、Express 5、`ws`（`server/`，纯 JS） |
| 平台 | ARM64 — DGX Spark GB10（Neoverse V2） |
| 部署 | Docker 多阶段构建（arm64）、Compose |
| 密钥 | AES-256-GCM 加密 SSH 密码 |
| 端口 | **5555** 面板/API；**5173** Vite dev |
| 测试 | 后端 `node --test`，前端 `vitest` |

```txt
sparkDash/
├── src/                 React + TypeScript SPA（api / components / hooks / styles）
├── server/              Express + WebSocket
│   ├── sparks/          SparkRegistry、SparkMonitor
│   ├── collectors/      SystemCollector、LlmProbe、ssh
│   ├── llmtokens/ llmlaunch/ tooleval/ metrics/ energy/ health/ events/
│   ├── secretsStore.js  加密密码持久化
│   └── validate.js      Host/user 校验（SSRF 防护）
├── config/              运行时状态（volume；secrets 已 gitignore）
├── docs/                PROGRAM.md、REMOTE-ACCESS.md
├── scripts/             deploy / rollback / watchdog 等
├── Dockerfile           生产多阶段构建（arm64）
├── docker-compose.yml   生产
├── docker-compose.dev.yml
└── deploy.sh            重建/重创建容器辅助脚本
```

数据流：`Browser ↔ WebSocket /ws ↔ SparkMonitor.snapshot() ↔ collectors`；`Browser ↔ REST /api/* ↔ SparkRegistry + SparkMonitor`。后台轮询循环常驻（即使无客户端），保证速率类指标（tok/s、网络字节/s、磁盘 IO）准确。

---

## 四、部署方式

### 方式 A：Docker 生产部署（推荐，DGX Spark / Linux arm64）

```bash
git clone https://github.com/MiaAI-Lab/sparkDash.git
cd sparkDash
npm run build            # 必须先构建前端 → dist/（compose 会挂载 ./dist）
docker compose up --build -d
```

- 打开 **http://127.0.0.1:5555**（仅监听 loopback）
- 容器：`restart: always`、`privileged: true`、`network_mode: host`、`pid: host`
- 挂载 `/proc`、`/sys`、`/`、`nvidia-smi` 与 NVIDIA 库用于本机指标
- `server/` 挂载进容器，`node --watch` 热重载；改前端只需在宿主 `npm run build` 后刷新浏览器
- 辅助脚本：`./deploy.sh`（可加 `--build`、`--frontend`）

**前置条件**：Linux + **arm64** + 已装 NVIDIA 驱动 + Docker/Compose。

> x86 机器：Dockerfile/compose 内的 `platforms: linux/arm64` 和 `aarch64` 库挂载需自行修改，且官方标注 x86 Docker 构建**未测试**；建议改用方式 B。

**Docker 下的 SSH 密钥**：容器内 OpenSSH 使用 `/root/.ssh`，宿主 `~/.ssh` 不会自动生效，需在 `docker-compose.yml` 取消注释：

```yaml
- ${HOME}/.ssh/id_ed25519:/root/.ssh/id_ed25519:ro
```

自定义名（如 `id_ed25519_shared`）需挂载**为** `id_ed25519`，或设 `SSH_IDENTITY_FILE` 指向容器内路径；密钥 mode 须为 600。运行 sparkDash 的这台机器在 UI 里用 **This host (local collectors — no SSH for metrics)** 添加。

### 方式 B：裸机 npm 部署（开发 / x86 / 任意 Linux）

```bash
npm install
npm run build
npm start                # 生产：node server/index.js → 5555

# 开发模式（热更新）
npm run dev              # Vite 5173 + Express 5555 并行
```

- **需要 Node 22**
- 其他可选命令：`npm run typecheck`、`npm test`（后端 node --test + 前端 vitest）、`npm run docker:dev`（源码挂载 + HMR 的开发 Compose）

### 方式 C：非 Spark 机器运行 dashboard

dashboard 不必跑在 Spark 上，可以是任意 Linux 小主机 / NAS / VM / x86 mini PC：

- 运行 dashboard 的机器**无需 GPU**，也可不作为监控单元添加
- 所有 Spark 通过 **Add Spark** 用 LAN IP + SSH 用户作为远程单元接入
- x86 下推荐方式 B（Docker 需改造且未测试）

### 当前 Windows 环境提示

官方 compose/Dockerfile 面向 Linux arm64，Windows 下建议：
- **WSL2** 内跑 Docker 或 Node；或
- 直接 `npm install && npm run dev` 用开发模式体验（本机 collectors 依赖 sysfs/proc/`nvidia-smi`，Windows 原生上**本机采集不可用**，只能监控远程 Linux 单元）

---

## 五、配置（`.env`）

复制 `.env.example` 为 `.env`：

| 变量 | 默认 | 说明 |
|------|------|------|
| `PORT` | 5555 | 服务端口 |
| `BIND_HOST` | `127.0.0.1` | 监听地址；非 loopback 必须配 token 或显式允许 |
| `SPARKDASH_TOKEN` | 空 | 设置后远程访问/写操作需 bearer token |
| `SPARKDASH_ALLOW_OPEN_REMOTE` | 1 | 设 0 则无 token 的远程绑定**拒绝启动**（fail closed） |
| `SPARKDASH_ALLOWED_HOSTS` | 空 | 允许的目标主机白名单 |
| `LLM_PORT` | 8888 | 新单元默认 LLM 端口 |
| `POLL_INTERVAL_*` | 跟随 UI 设置 | 显式设置则钉死对应采集域的轮询间隔 |
| `POLL_INTERVAL_STORAGE` | 5000 | 存储采集间隔 |
| `SSH_CONTROL_PERSIST_SECONDS` | 60 | 复用已认证 SSH 连接；0 = 禁用 |
| `SSH_IDENTITY_FILE` | 空 | 容器内非默认名的私钥路径 |
| `SPARKDASH_BENCH_HOSTS` | 空 | 允许的一次性远程基准测试主机（逗号分隔） |
| `SPARKDASH_SECRETS_KEY` | 自动 `config/.secrets-key` | **不要删除密钥文件**，否则加密密码不可读 |
| `VITE_HISTORY_HOURS` | 8 | 前端历史保留小时数（构建期打入） |

UI 内全局设置（齿轮图标 / `GET|PUT /api/settings`）：轮询间隔、默认 LLM 端口、离线自动隐藏等。

---

## 六、远程访问与安全

**默认只监听 127.0.0.1，HTTP/WS API 无应用层鉴权**，因此：

1. **推荐**：SSH 隧道
   ```bash
   ssh -N -L 5555:127.0.0.1:5555 user@sparkdash-host
   # 然后本机打开 http://127.0.0.1:5555
   ```
2. 或：带认证的 TLS 反向代理 / Tailscale Serve（见 `docs/REMOTE-ACCESS.md`）
3. 或：`BIND_HOST=0.0.0.0` **且** `SPARKDASH_TOKEN`（否则任何人可读数据、改配置、关机；header 会显示 Open access 警告）

**安全要点**
- SSH 密码不入 `sparks.json`、不回 API；AES-256-GCM 存于 `config/sparks-secrets.json`，密钥 `config/.secrets-key`
- 优先用 SSH 密钥而非密码
- 目标校验拒绝 link-local / `0.0.0.0/8` / 组播保留段 IPv4；私网、回环、公网地址放行
- 探测短超时（SSH 连接约 5s、HTTP 约 3s）
- `GET /api/health` 的 `authMode` 报告当前模式：`loopback-open` / `bearer` / `open-remote` / `required-missing`
- 实测运维容量：**12 个单元**

---

## 七、常用 REST API 速查

| 方法 | 路径 | 用途 |
|------|------|------|
| GET | `/api/sparks` | 单元列表（密码脱敏） |
| POST / PATCH / DELETE | `/api/sparks[/:id]` | 增改删（热生效） |
| PUT | `/api/sparks/order` | 保存标签顺序 |
| GET | `/api/sparks/:id/metrics` | 单次指标快照 |
| GET | `/api/fleet-energy[\/history]` | 整机能耗估算及历史 |
| GET | `/api/llm-token-totals[\/history]` | Token 累计与历史 |
| GET | `/api/sparks/:id/gpu-history` | GPU 历史（最长 8h） |
| GET | `/api/events` | 事件日志（最多 2000 条） |
| GET/POST/PUT/DELETE | `/api/sparks/:id/llm-launchers…` | LLM 启停脚本管理与执行 |
| GET/POST/DELETE | `/api/sparks/:id/tool-eval/…` | Tool Eval Bench |
| POST | `/api/sparks/test` · `/api/sparks/:id/test` | 连通性测试（不持久/可存密码） |
| POST | `/api/sparks/:id/comfy/cancel` | 取消 ComfyUI 任务 |
| PUT | `/api/sparks/:id/password` | 保存 SSH 密码 |
| PUT/POST/DELETE | `/api/sparks/:id/llm-ports…` | LLM 端口管理（热） |
| POST | `/api/sparks/:id/llm/bench` · `/prefill-bench` · `/quality-bench` | 三类基准测试（202，轮询/取消） |
| GET | `/metrics` | Prometheus 输出（opt-in） |
| GET/PUT | `/api/settings` | 全局设置 |
| WS | `/ws` | 实时指标流 |

---

## 八、实操 checklist

- [ ] 确认部署方式：DGX Spark/arm64 → Docker（方式 A）；x86/开发 → npm（方式 B）；Windows → WSL2 或 dev 模式
- [ ] `npm install`（方式 B 必须）；Docker 方式也要先 `npm run build` 生成 `dist/`
- [ ] 准备 `.env`（从 `.env.example` 复制），确认 `BIND_HOST` / token 策略
- [ ] 启动后访问 `http://127.0.0.1:5555`（dev 模式前端为 `http://localhost:5173`）
- [ ] UI 中 **Add Spark**：本机选 local collectors；远程填 LAN IP + SSH 用户（密钥优先）
- [ ] 用 **Connectivity Test** 验证 SSH / LLM / ComfyUI 连通
- [ ] 按需开启 ComfyUI / Tailnet / Hermes / Prometheus 等可选探测
- [ ] 需要外网/局域网访问时，按「远程访问与安全」配置 SSH 隧道或 token
- [ ] 备份 `config/` 目录（含 `sparks.json`、`sparks-secrets.json`、`.secrets-key`），**切勿丢密钥文件**

---

## 九、故障排查 FAQ

### Q1：SSH 连接报错 `sshpass is not installed. Install it with: sudo apt-get install sshpass`

**现象**：添加远程单元或做 Connectivity Test 时，提示 `sshpass is not installed...`，并伴随 "Check the host address, SSH user, key/password, and network route" 排查提示。

**原因**：该错误来自 `server/collectors/ssh.js`（密码认证分支）。sparkDash 用 `sshpass -e` 把密码传给 ssh（密码不进 argv/进程列表），**只有填写了 SSH 密码才会检查 sshpass**；裸机/容器 Linux 环境若未安装即报此错。

**解决方式（二选一）**：

1. **在部署 sparkDash 的 Linux 服务器上安装 sshpass**：
   ```bash
   # Debian / Ubuntu
   sudo apt-get update && sudo apt-get install -y sshpass
   # RHEL / CentOS / Fedora
   sudo yum install -y sshpass
   which sshpass && sshpass -V   # 验证
   ```
   **装完必须重启 sparkDash**（`docker compose restart` 或重启 `npm start`）：`ssh.js` 对 `_sshpassAvailable` 做了进程内缓存，首次检测后不再探测，已运行进程不会发现新装的 sshpass。apt 装到 `/usr/bin/sshpass`，正好在检测路径内（`/usr/bin`、`/usr/local/bin`、`/bin`、`/opt/homebrew/bin` + PATH 扫描），无需额外配置。

2. **改用 SSH 密钥认证（推荐，绕开 sshpass）**：
   ```bash
   # 在 sparkDash 服务器上
   ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519 -N ""   # 已有密钥可跳过
   ssh-copy-id -i ~/.ssh/id_ed25519.pub 用户名@目标IP  # 输入一次密码完成分发
   ssh 用户名@目标IP                                   # 验证免密登录
   ```
   然后在 **Add Spark** 中**密码留空**（留空即走密钥分支，不触发 sshpass 检查）。自定义名密钥需设 `SSH_IDENTITY_FILE`，权限须 600；Docker 部署需挂载 `${HOME}/.ssh/id_ed25519:/root/.ssh/id_ed25519:ro`。

**补充**：Windows 原生不支持 sshpass（检测路径均为 Unix 路径，PATH 扫描也不匹配 `sshpass.exe`），Windows 上部署只能用密钥认证，或在 WSL2 内运行 sparkDash。

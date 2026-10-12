# sparkDash 用户使用手册

> 适用版本：sparkDash v2.0.0 · 面向日常操作者的中文使用手册
> 配套文档：`sparkDash功能及部署方式.md`（分析与部署）、`README.md`（官方原文）、`docs/REMOTE-ACCESS.md`（远程访问）
> 整理日期：2026-10-10 · 更新：2026-10-12（新增第十五、十六章：指标解读 + 实操演练）

---

## 目录

- [使用须知：适用范围与边界限制](#使用须知适用范围与边界限制)

1. [快速上手](#一快速上手)
2. [界面总览](#二界面总览)
3. [添加与管理监控单元](#三添加与管理监控单元)
4. [Overview 总览页](#四overview-总览页)
5. [Spark 详情页](#五spark-详情页)
6. [LLM 服务与模型管理](#六llm-服务与模型管理)
7. [基准测试（Benchmarks）](#七基准测试benchmarks)
8. [Prompt Showcase](#八prompt-showcase)
9. [统计页面：Token / 能耗 / 活动](#九统计页面token--能耗--活动)
10. [Settings 设置](#十settings-设置)
11. [快捷键与移动端](#十一快捷键与移动端)
12. [状态、告警与常见提示](#十二状态告警与常见提示)
13. [安全与远程访问](#十三安全与远程访问)
14. [故障排查 FAQ](#十四故障排查-faq)
15. [指标解读：监测入门必读](#十五指标解读监测入门必读)
16. [实战演练：两台 vLLM 单元上手路线](#十六实战演练两台-vllm-单元上手路线)

---

## 使用须知：适用范围与边界限制

> sparkDash 是**面向 NVIDIA DGX Spark / GPU Linux 主机的轻量级实时监控面板**，不是通用 APM，也不是云厂商级可观测平台。它有明确的**硬件、软件、数据保真度与规模边界**——本章内容全部来自官方文档与 v2.0.0 源码实现，**首次使用前务必通读**，避免把"sparkDash 没显示"误判为"服务出问题"，或把估算值当成实测值。

### 0.1 硬件与平台边界

| 限制项 | 具体边界 | 说明 |
|--------|----------|------|
| **GPU 厂商：仅 NVIDIA** | AMD（ROCm）、Intel Arc、Apple Silicon **完全不支持** | 全部 GPU 指标（利用率/显存/温度/功耗/节流/进程表）来自 `nvidia-smi` / NVML；无 NVIDIA 驱动的机器 GPU 面板无数据 |
| **本机采集：仅 Linux** | Windows / macOS **不能作为本机采集单元** | 本机指标依赖 sysfs / proc / nsenter；Windows 原生部署只能监控**远程 Linux 单元**，本机单元无指标（WSL2 内部署可解） |
| **产品定位：DGX Spark 优先** | 通用 Linux + NVIDIA 主机可用，但部分视图按硬件呈现 | DGX Spark 显示 128GB **统一内存**堆叠条；普通主机（`host` 类型）为 **RAM 与 VRAM 分开展示**、多卡逐卡一块面板 |
| **CPU 温度语义** | GB10 上是 **ACPI/SoC 区域温度**，不是核心温度 | 跨机型比较 CPU 温度无意义 |
| **Docker 镜像：arm64** | x86 Docker 构建官方**未测试**，需自行改造 | x86 Linux 请用裸机 npm（方式 B）；容器需 `privileged` + nsenter 才能读宿主命名空间的指标 |
| **Wake-on-LAN** | **DGX Spark 不响应魔术包**；仅部分 host 类型专用 GPU 网卡支持 | 唤醒按钮对 DGX Spark 无效；host 类型还需在 Edit 里填对 MAC Address |
| **远程关机** | 依赖目标机存在 `/usr/local/bin/spark-shutdown` 脚本 | 没有该脚本则关机功能不可用；关机不可从面板撤销 |
| **温度告警阈值固定** | Warm > 65°C、Hot > 85°C（摄氏，写死在界面） | 是显示层标注，不能按机型/散热条件自定义 |

### 0.2 能监测什么服务（LLM 支持范围）—— 最重要的边界

sparkDash **不是对所有模型服务都能出完整指标**。探测逻辑（`server/collectors/LlmProbe.js`）分三层：

**第一层：自动识别且指标完整的引擎（9 种）**

| 引擎 | 识别方式 | 备注 |
|------|----------|------|
| llama.cpp | `/slots` 探测 | 含 timings 的 prefill/decode 细分 |
| **vLLM** | `/v1/models` + Prometheus `/metrics` | KV pool、TTFT 直方图、队列等深度指标**依赖 `/metrics` 可达**；`/metrics` 被关或被挡 → 深度指标缺失 |
| SGLang | `/server_info` 等原生端点 | Prometheus 可选（`--enable-metrics`） |
| ds4-server | `/metrics` 序列（`ds4_*`） | |
| EXL3 | `/health` 健康端点 | |
| q27 | `/metrics` 序列（`q27_*`） | |
| TensorFold | `/health` | **不上报累计计数器时 tok/s 保持 0，绝不猜** |
| FreeToken | `/stats` | |
| （OpenAI 兼容） | `/v1/models` 的 `owned_by` 分类 | |

**第二层：OpenAI 兼容但不在清单内 → 按 vLLM 兜底显示**

- **Ollama、TGI（HTTP）、Triton（HTTP）、LM Deploy、LocalAI、Jan** 等只要暴露 `/v1/chat/completions` 就能被"识别"，但源码中分类失败的**默认兜底就是 `return "vllm"`**（`LlmProbe.js` `_classifyOpenAIBackend`）
- 后果：**流式期间仍能推算 decode tok/s**（从响应流估算），但 **KV cache、Running/Waiting 队列、TTFT 直方图、Preempts 等 vLLM 专属指标将缺失、恒为 0 或张冠李戴**；模型显示的"引擎标签"也不可信
- **结论：这类引擎 = "能看有没有在跑、跑多快"，但"跑得健不健康"看不了**

**第三层：完全无法探测**

- **非 OpenAI 兼容、非 llama.cpp 的服务**（gRPC-only Triton、私有协议、未暴露 HTTP 端点的推理框架）→ 探测失败，LLM 卡**无有效指标**
- 引擎在**第二/三层**时，Bench、Showcase、Tool Eval 也**无法使用**（它们都通过 OpenAI 兼容接口发请求）

**探测的可达性前提**（任一不满足 → *LLM is unavailable*）

1. 端口从 **sparkDash 所在机器**可达：本机单元走 `127.0.0.1`，远程单元走 **LAN IP**
2. 远程服务**只绑 `127.0.0.1`** 时会自动走 **SSH 隧道**转发（`llmTunnel.js`）——**前提是该单元 SSH 认证必须成功**；SSH 密码认证又依赖 sshpass（见 0.6）
3. 目标机防火墙 / 容器网络 / 安全组需放行 sparkDash → LLM 端口
4. 端口填错、服务未启动、服务重启换了端口 → 卡片显示 *LLM is unavailable*，恢复后自动重连
5. **Worker 角色永不探测 LLM**；Standalone 角色受 *LLM monitoring* 开关控制；Head 始终探测

**其他被监测对象的前置依赖**

| 对象 | 前置条件 |
|------|----------|
| ComfyUI | 勾选 *Monitor ComfyUI* + 端口（默认 8188）可达 |
| Hermes 更新检查 | 目标机已安装 `hermes`（`~/.local/bin` 或 `/usr/local/bin`） |
| Tailnet 状态 | 目标机运行 `tailscaled` |
| Tool Eval Bench | 需在**目标 Spark 上安装** `tool-eval-bench` CLI |
| 一次性远程 Bench 目标 | 必须列入 `.env` 的 `SPARKDASH_BENCH_HOSTS` |
| Prometheus 抓取 | Settings 打开 *Prometheus metrics*（默认关，关闭时 `/metrics` 返回 404） |

### 0.3 采集机制与数据保真度限制

| 限制 | 说明 |
|------|------|
| **无自动发现** | 不会扫描局域网找主机，所有单元必须**手动 Add Spark** 添加 |
| **轮询频率有下限** | UI 可选 1 / 2 / 5 / 10s（底层钳制 0.5–60s）；**带宽采集域（`nvidia-smi dmon` 本身阻塞约 1s）硬性不低于 2s**，设置更小也会被抬到 2s |
| **SSH 负载随频率放大** | 远程单元每个轮询周期都通过 SSH 执行一批命令；轮询越快、单元越多，SSH 会话与目标机 CPU 开销越大 —— **多台远程不建议 1s 轮询** |
| **规模实测容量 12 单元** | 官方实测运维容量为 **12 个单元**；更多单元需自行压测验证（注意 0.6 的 SSH/HTTP 超时） |
| **速率指标需要 1–2 个周期稳定** | tok/s、网络字节/s、磁盘 IO 是**差分速率**，刚打开页面、刚重启服务的前几个数值偏低是正常现象 |
| **后台轮询常驻** | 即使没有浏览器打开，服务端也在持续采集（为保证速率准确）—— 这是特性也是资源消耗，无法暂停 |
| **失败降级 ≠ 服务停了** | 采集失败时返回**零值/默认值**，持续失败才标 *offline*。所以"tok/s 突然为 0"有三种可能：请求真的结束了、SSH 抖动采不到、服务挂了 —— **结合 Activity 事件日志和状态点区分** |
| **探测超时很短** | SSH 连接约 **5s**、HTTP 约 **3s**；高延迟/拥塞的跨网段链路可能被判离线 |
| **stale 判定** | 数据超过 `max(10s, 3×轮询间隔)` 未更新即显示 stale 变暗 —— 卡顿的不只是网络，也包括目标机过载 |

### 0.4 数据保留期限（过期即丢，无法追溯）

| 数据 | 保留期 / 上限 |
|------|----------------|
| 实时快照（WebSocket） | 仅内存，断开即止 |
| **GPU 历史图表** | 最长 **8 小时**（服务器端 2s 采样持久化窗口） |
| 前端 sparkline 历史 | `VITE_HISTORY_HOURS` 默认 **8h**（**构建期写死**，改需重新 `npm run build`） |
| **Token 统计** | 小时粒度 **72 小时**、天粒度 **35 天**；按 **UTC 天**分组；**引擎重启后计数器会重新播种**，曲线可能跳变 |
| 活动事件日志 | 最多 **2000 条**（超出滚动丢弃最早的） |
| Quality Bench 结果 | 每单元仅保留**最近 30 次**；**每单元同时只能跑 1 个** Quality |
| Showcase 历史 | 可手动 Clear，清空不可恢复 |
| 能耗历史 | 可长期聚合，但**增删单元后需重启 sparkDash 才能建立新的统计口径** |
| Bench 调试细节 | 默认不保存 prompts/HTTP ID/GPU 采样，需 Settings 开 *Save benchmark debug traces* |

### 0.5 数值性质：哪些是估算、哪些不是

| 数值 | 性质 | 细节 |
|------|------|------|
| **GPU 使用率** | 采样语义，**≠ 算力利用率** | "周期内有 kernel 在跑"的时间占比；90% 利用率可能实际只用一半算力，必须结合 tok/s 判断 |
| **Fleet energy 能耗** | **模型估算值，非墙插实测** | = GPU 功耗 + CPU 模型 + **23W 基础开销**（且按节点做了上下限钳制）；只适合横向对比，不能当电费账单 |
| **Token 计数** | 视引擎而定 | vLLM / SGLang / ds4 / q27 / llama.cpp 上报 usage 或计数器 → **精确**；不上报的后端按**文本估算（约 4 字符 ≈ 1 token）** → **跨引擎对比 Token 总量需谨慎** |
| **TTFT / ITL / p95** | 窗口统计 | 反映**被探测请求**的分布，不是目标机所有客户端的全局统计 |
| **CPU 温度（GB10）** | 区域温度 | ACPI/SoC 区域，非核心温度（见 0.1） |
| 缺失字段 | 显示 N/A / 0，**不编造** | 引擎不上报的指标宁可空着（如 TensorFold 无计数器时 tok/s = 0） |

### 0.6 访问、安全与规模边界

| 限制 | 说明 |
|------|------|
| **默认无鉴权** | 只监听 `127.0.0.1` 且**应用层无认证**；`BIND_HOST=0.0.0.0` 时**必须**配 `SPARKDASH_TOKEN`，否则任何能连到端口的人都能改配置、关机 |
| **密码认证依赖 sshpass（仅 Linux）** | SSH 密码模式需要 `sshpass`；**Windows 上无法使用密码认证**（只能用密钥），Linux 装完 sshpass 还要**重启服务**（检测结果有进程内缓存） |
| **目标地址白名单（SSRF 防护）** | 拒绝 link-local、`0.0.0.0/8`、组播保留段 IPv4 —— 这些特殊地址填了也连不上 |
| **凭据安全** | 密码 AES-256-GCM 存 `config/sparks-secrets.json`；**`config/.secrets-key` 丢失则已存密码全部作废**，务必备份且勿删 |
| **无外部告警推送** | **没有**邮件 / Webhook / Slack / 钉钉等通知渠道；所有告警只在 **UI 内展示**（告警条、事件日志）—— 关键告警需人工盯面板 |
| **Bench 会打满 GPU** | Decode/Prefill/Quality 都是**真实流量**，跑分期间该实例上的正常请求会明显变慢；**不要在承载生产流量的实例上跑 Bench** |
| **运维容量** | 单面板实测 **12 单元**；超过需自行验证 SSH 并发与轮询负载 |

### 0.7 一分钟自检：我的场景适用吗？

| 你的场景 | 是否适用 | 备注 |
|----------|----------|------|
| NVIDIA DGX Spark 集群 | ✅ 完整支持 | 产品定位目标 |
| Linux + NVIDIA 独立 GPU 工作站/服务器 | ✅ 支持 | 以 `host` 类型添加 |
| AMD / Intel / Apple GPU 机器 | ❌ 不支持 | 无 nvidia-smi 即无 GPU 指标 |
| 在 Windows 上监控远程 Linux 机器 | ✅ 支持 | 本机采集不可用而已；密码认证需改用密钥 |
| 部署 vLLM / llama.cpp / SGLang 等 9 种引擎 | ✅ 指标完整 | 见 0.2 第一层 |
| 部署 Ollama / TGI / LM Deploy 等 OpenAI 兼容服务 | ⚠️ 部分支持 | 能看 tok/s，健康指标缺失（按 vLLM 兜底） |
| gRPC / 私有协议推理服务 | ❌ 不支持 | 无法探测 |
| 需要邮件/IM 告警通知 | ❌ 不支持 | 仅 UI 内展示 |
| 需要墙插级功耗计量 | ❌ 不支持 | 能耗为模型估算 |
| 监控超过 12 个单元 | ⚠️ 超出实测容量 | 需自行压测 |
| 长期存储（>8h）GPU 曲线、（>35 天）Token 曲线 | ❌ 不支持 | 到期滚动丢弃，需自行对接外部时序库（可开 Prometheus `/metrics` + Grafana） |

---

## 一、快速上手

### 1. 启动

```bash
# 生产（Docker，DGX Spark / Linux arm64）
npm run build && docker compose up --build -d

# 开发 / 裸机（Node 22）
npm install
npm run dev        # Vite 5173 + Express 5555
```

| 模式 | 访问地址 |
|------|----------|
| 生产 / Docker | `http://127.0.0.1:5555` |
| 开发（推荐用这个调试前端） | `http://localhost:5173`（自动代理 API/WS 到 5555） |

### 2. 三步跑通

1. 打开页面 → 侧边栏点 **`+`**（或命令面板 → *Add a Spark or GPU host*）
2. 走完三步向导 **Connect → Role → Services**（详见第三章）
3. Overview 上看到该单元的卡片，绿点 = 在线，实时数据自动推送

> **本机就是被监控机器**时：在第一步勾选 *This host (local collectors, no SSH for metrics)*，无需填 SSH。

---

## 二、界面总览

```
┌─────────────┬──────────────────────────────────────────────┐
│ 侧边栏       │  页面主体                                     │
│ · Logo      │                                              │
│ · Jump to…  │   Overview / Spark 详情 / Bench / 统计页        │
│ · Sparks 列表│                                              │
│ · Workspace │                                              │
│ · Benchmarks│                                              │
│ · Settings  │                                              │
│ · 主题/连接态│                                              │
└─────────────┴──────────────────────────────────────────────┘
```

### 侧边栏（`AppSidebar`）

| 区域 | 内容 |
|------|------|
| 顶部 | Logo（点击回 Overview）、折叠按钮（Ctrl+B） |
| **Jump to…** | 打开命令面板，带 `Ctrl K` 键帽提示 |
| **Sparks** | 每单元一行：状态点（绿=在线 / 灭=离线 / 黄=热降频）+ 名称 + 副标题（LLM 跑着时显示 decode tok/s，否则 `off` / `worker`）。**可拖拽排序**；`+` 添加 |
| **Workspace** | Overview · Token totals · Fleet energy · Activity · Showcase |
| **Benchmarks** | Decode · Prefill · Quality · Tool Eval Bench（可折叠） |
| 底部 | Settings 按钮、主题切换、连接状态（`Live · 2 s poll` / `Reconnecting…`）、**Open access** 芯片（有风险时） |

### 三个 URL 概念

- 每个页面都有独立 URL（`/`、`/tokens`、`/energy`、`/activity`、`/spark/:id`、`/bench/:type`、`/showcase/:id`），**可直接收藏、右键新标签打开**。
- 浏览器前进/后退正常工作；链接到已删除的 Spark 会自动跳回 Overview。

---

## 三、添加与管理监控单元

### 3.1 添加单元（三步向导）

**入口**：侧边栏 Sparks 区 `+`，或命令面板 → *Add a Spark or GPU host*。

#### Step 1 — Connect（连接）

| 字段 | 说明 |
|------|------|
| **Unit type** | `NVIDIA DGX Spark` 或 `Dedicated GPU host`（任意 Linux + 独立 NVIDIA 卡） |
| ☐ **This host** | 勾选 = 本机用 local collectors，**不需要 SSH** |
| **LAN IP** | 远程必填；本机可留空（留空则"打开链接"类按钮不可用，且无 Wake-on-LAN） |
| **CX7 IP** | 可选，仅 DGX Spark |
| **SSH User / Port** | 端口默认 22，sshd 改过才需要改 |
| **SSH Auth** | `Key` 或 `Password` |
| → Key 模式提示 | SSH 在 **sparkDash 服务器**上执行（不是浏览器）；Docker 下需把私钥挂到 `/root/.ssh/id_ed25519` 或设 `SSH_IDENTITY_FILE` |
| → Password 模式提示 | 密码以 AES-256-GCM 加密存服务端，**不进 `sparks.json`、不回 API**，容器重启不丢 |
| **Test 按钮** | 可选。临时测试 SSH / 硬件 / LLM 连通性，**不写入配置**；逐项显示 pass / failed / skipped 及修复建议 |

#### Step 2 — Role（角色）

三选一卡片：

| 角色 | 含义 |
|------|------|
| **Standalone** | 独立运行自己的 LLM（默认） |
| **Head** | 集群主节点，始终监控本地 LLM |
| **Worker** | 只做分片，**没有本地 LLM API**（不探测端口） |

**Name 必填**（显示在侧边栏和 Overview；ID 由名称自动生成）。

> **❓ 角色只是一个标签/属性吗？**
>
> 不是。角色（`role`）是一个**行为开关**，直接决定 sparkDash 怎么对待这台机器，而不只是界面上的一行小字。它影响：
>
> | 影响面 | 具体行为 |
> |--------|----------|
> | **是否探测 LLM** | **Worker：永不探测**（不建 LLM 探针、不轮询端口）；**Head：始终探测**（忽略 LLM monitoring 开关）；**Standalone：由 *LLM monitoring* 复选框决定**（默认开）。这是最核心的差异，代码见 `server/sparks/SparkMonitor.js` 的 `_llmMonitoringEnabled()` |
> | **详情页是否显示 LLM 面板** | Worker 的 Spark 详情页**不渲染** LLM server 卡（也没有 "+ Add LLM port"）；Head / Standalone 正常显示 |
> | **Overview 卡片内容** | Worker 卡显示 *"Worker of {head}"* 链接 + **head 的模型与 decode tok/s**（不是自己的）；且**不显示** ModelLauncher 启动器。Head / Standalone 卡显示自己的模型、tok/s 和启动器 |
> | **模型标签来源** | Worker 可在 Edit 里填 **Worker label**（如 `DeepSeek V4 Flash`）手动指定；留空则自动镜像其 head 正在服务的模型 id；再没有则显示 `distributed` |
> | **能耗统计口径** | Fleet energy 的 **Wh/output-token 只统计 head 与 standalone**，worker 被跳过 —— 因为 worker 只是 head 引擎的前端，计入会导致同一份 token 被算两遍（见 `server/energy/FleetEnergyTracker.js`） |
> | **Prometheus 标签** | 每条指标带 `role="head|worker|standalone"` 标签，可在 Grafana 里按角色筛选 |
> | **UI 呈现** | 卡片/页头的角色徽章、侧边栏副标题（worker 显示 `worker`）、命令面板 hint |
>
> **记忆口诀**：Head = 主节点，永远盯 LLM；Worker = 纯算力分片，一切 LLM 展示都"借用" head 的；Standalone = 单机自洽，LLM 监控可自己开关。
>
> **怎么改**：Edit Spark → Role 下拉即可热生效（无需重启）。注意 Worker 需要额外选 **Head Spark** 才能把 head 的模型显示出来；旧版本只有 `workerNode` 布尔字段，现在会自动归一化为 `worker`。

#### Step 3 — Services（服务）

- **LLM ports**：逗号分隔可填多个，默认取 Settings 的 *Default LLM port*（8888）；Worker 不显示此项
- ☐ **Monitor ComfyUI**：任务队列与实时进度，默认端口 8188
- ☐ **Monitor Hermes updates**：每 10 分钟后台检查更新，并出现 *Update Hermes* 按钮

最后点 **Add Spark** / **Add GPU host** 完成。

> **❓ LLM ports 是什么？是 GPU 服务器上模型 API 服务的端口吗？支持多个吗？**
>
> **是的，你的理解正确。** LLM ports 填的是**被监控机器上、对外提供模型推理服务的 HTTP API 端口**（OpenAI 兼容风格的接口），**不是** GPU 硬件端口，也和 SSH 端口无关。
>
> **它如何工作**（`server/collectors/LlmProbe.js`）：
> ```
> sparkDash ──HTTP──> http://{host}:{port}/v1/models 、/slots 、/metrics …
> ```
> - 目标主机：本机单元走 `127.0.0.1`（引擎常默认 `--host 127.0.0.1`）；远程单元走该 Spark 的 **LAN IP**；也可用 `llmHost` 字段固定目标
> - 每个端口**独立创建一个 LlmProbe 实例并行轮询**，自动识别后端（llama.cpp / vLLM / sglang / ds4-server / EXL3 / q27 …），从而读出 decode tok/s、prefill、KV cache、队列、TTFT 等
>
> **支持多个端口 —— 而且是完整的一等能力**：
>
> | 层面 | 说明 |
> |------|------|
> | 添加时 | 输入框里用**逗号分隔**，如 `8888,30000,30001` |
> | 详情页 | 每个端口一张独立的 **LLM server 卡**，互不干扰；卡片右上可 **+ Add LLM port** 增加、也可删除（热生效，无需重启） |
> | API key | 每个端口**各自保存**一份 Bearer token；增删端口时服务端会自动迁移/清理对应密钥（`syncLlmApiKeysToPorts`） |
> | 统计 | Token totals、Prometheus、能耗的 Wh/token 都**按端口分别记账** |
> | 运行时 | 后端接口：`PUT /api/sparks/:id/llm-ports`（整体替换）、`POST …/llm-ports`（新增一个）、`DELETE …/llm-ports/:port`（删一个），全部热更新 |
>
> **典型场景**：同一台 Spark 上同时跑 vLLM(8888) 和 llama.cpp(8080)，或一台主机起多个推理实例 —— 各填一个端口即可同屏监控。
>
> **注意**：
> - **Worker 角色不显示也不会探测** LLM ports（它没有本地 API）
> - 端口必须是**从 sparkDash 所在机器能访问到的**；远程服务若只绑 `127.0.0.1`，探测会失败（需绑 `0.0.0.0` 或走隧道）
> - 端口填了但服务没起，卡片会显示 *LLM is unavailable*，服务起来后自动恢复
> - 单元已添加后改端口：Edit Spark 里改，或详情页 LLM 卡的 *+ Add LLM port*，均为热更新

### 3.2 编辑单元

**入口**：Spark 详情页右上角 **Edit**（铅笔），或命令面板 → *Edit {当前 Spark}*。

单页长表单，主要字段：

- 基本：Unit type、Name、LAN IP、CX7 IP、**MAC Address**（仅 host 类型，WoL 覆盖，占位符显示自动检测值）、This host
- 角色：Role 下拉 + **LLM monitoring**（Standalone 才有）+ **Worker label** / **Head Spark**（Worker 才有）
- 可选服务：**ComfyUI monitoring**（+ 端口）、**Hermes Agent**、**Tailnet monitoring**
- SSH：User / Port / Auth / Password
  - 已存密码显示 *(leave blank to keep stored secret)*
  - 未设密码显示警示：*"Enter once and Save (or Test). Stored encrypted — host does not need to be online."*
  - **修改了 SSH 目标（host/user/isLocal）后旧密码会被丢弃，需重新输入**
- **Test**：表单与已存配置一致时走已保存配置测试；有未保存修改时自动改为"测眼前所见"的临时测试
- **Remove**（危险）：弹 `confirm("Remove Spark …? This cannot be undone.")`

> 保存按钮在缺名称 / 远程缺 LAN IP / 缺密码时禁用。

### 3.3 排序与隐藏

- **拖拽排序**：侧边栏拖动即可，保存到服务端；失败提示 *"Could not save the Spark order… The previous order was restored."*
- **键盘排序**：聚焦条目 → 按 **M** 拾起 → 方向键移动 → **Enter/Space** 放下 → **Esc** 取消
- **隐藏 Worker**：Settings → *Hide worker nodes*（直链与批量操作仍包含它们）

---

## 四、Overview 总览页

URL：`/`

### 4.1 页头

- 元信息：`N Sparks`、`x/y online`、统一内存总量、`N workers hidden`
- 操作按钮（按条件出现）：

| 按钮 | 出现条件 | 行为 |
|------|----------|------|
| **Update Hermes** | 有单元开启 Hermes | 批量 `hermes update`，带进度条 *Updating Hermes — done/total*；有待更新时琥珀色 + 数字徽标 |
| **Wake all** | 舰队中有 host 类型 | 批量 Wake-on-LAN（**DGX Spark 不响应魔术包**） |
| **Shut down all** | 始终（危险区） | 见下方确认流程 |

**关机确认（Danger zone — Shutdown All）** 必须同时：
1. 勾选 *I understand this cannot be undone from the dashboard.*
2. 手动输入 `poweroff`

弹窗会列出将被中断的工作：进行中的 LLM 请求、运行中的 ComfyUI 任务、Hermes 更新。结果形如 `3 shut down, 1 failed, 2 skipped`。

### 4.2 告警条与过滤

- **Fleet alert strip**（Settings 的 *Show fleet exceptions* 开启时显示全部，否则只显示热降频）：离线 / GPU 热降频·HW slowdown / 磁盘 ≥90% / LLM 不可用 / tailnet 不可用，每条带 `Open {name}` 按钮
- **搜索与状态过滤**（Settings 的 *Show search and status filters*，默认关）：`Search up to 12 units` + All / Online / Offline / Issues（Issues = 在线但磁盘 ≥90%）

### 4.3 Spark 卡片

| 卡片元素 | 内容 |
|----------|------|
| 卡头 | 状态点、名称（点击进详情）、角色标签、LAN IP |
| 健康 chips | 热降频 / HW slowdown / 磁盘告警等 |
| 离线卡 | *Host unreachable*；host 类型另有 **Wake up** 按钮（*Wake packet sent*） |
| 指标条 | VRAM/统一内存（可拆 引擎/系统/空闲）、GPU 使用率、host 的 RAM、GPU 温度、GPU 功耗 |
| **ModelLauncher** | 无 LLM 在跑且非 worker 时显示：一键 **Start** 已注册的 start.sh 启动器，或 **Add model** 注册第一个（最多 3 行 + `+N more`） |
| Worker 卡 | *Worker of {head}* 链接 + head 的模型与 decode tok/s；未设置 head 时提示 *Pick its head in Edit Spark…* |
| 大数字 | decode tok/s、prefill tok/s（进行中显示 `prefilling…`） |
| 芯片行 | Throttle / LLM 队列（`2 running · 1 queued`、`3/4 slots`）/ ComfyUI 状态 / Hermes / **Offline** |
| 底部 | 根盘 Storage used/total 与 available、模型行（模型 id + `128k ctx`） |

### 4.4 右侧栏 KPI

- **4 个 KPI 磁贴**：总 decode tok/s、总功耗、显存占用、今日 tokens（带趋势线）
- **Fleet energy 卡**：24h 逐时功率条、`x/y fresh`、*Open details* → `/energy`
- **Token totals 卡**：按模型累计 tokens → `/tokens`
- **Recent activity**：最近 6 条事件 → `/activity`

### 4.5 空状态

| 提示 | 含义 |
|------|------|
| *No Sparks registered* | 还没添加单元，点 *Add a DGX Spark or GPU host…* |
| *Worker nodes are hidden* | Settings 里关掉 *Hide worker nodes* 可恢复 |
| *All Sparks are offline* | 全部离线且开了自动隐藏 |

---

## 五、Spark 详情页

URL：`/spark/:id` — **所有信息在一页**（宽屏双列，窄屏按 GPU → 内存 → LLM 堆叠）。

### 5.1 页头与操作

- 第一行：状态点（离线时 title 显示 *Offline — {原因}*）、名称、角色/worker/local/后端标签、**uptime**（`up 3h 20m`）、Hermes 标签
- 第二行：LAN IP、tailscale IP、设备型号 + GPU 芯片 + 内存
- 在线时下方显示 **HealthList** 健康问题清单
- 右上操作：

| 按钮 | 说明 |
|------|------|
| **Update Hermes** | 有更新时琥珀色 + 落后 commit 数徽标；点开对话框可看发行说明并 *Update now*；运行中 *Hermes updating…* |
| **Edit** | 打开编辑对话框 |
| **Shut down** | 危险色，确认弹窗（勾选 + 输入 `poweroff`）；需宿主存在 `/usr/local/bin/spark-shutdown` |
| **Wake** | 仅离线的 host 类型；离线 DGX 显示 *No Wake-on-LAN* |

### 5.2 面板一览

| 面板 | 内容 |
|------|------|
| **GPU** | 使用率环、温度（Hot>85 / Warm>65，°C/°F）、功耗 draw/limit、SM clock、节流标签（Thermal / Power cap / HW slowdown / No throttle）、VRAM 条（可拆 引擎/系统/空闲）、**GPU 进程表**（按 VRAM 排序，含进程名）、NVRM 内核错误计数；**多卡主机每卡一块**；另有 GPU 历史图表（可选时间窗） |
| **Unified memory**（DGX Spark） | GB10 128GB 堆叠条：模型权重 / KV cache / 系统 / 空闲 + `used of total · %` |
| **CPU** | 使用率 + 温度 + CPU 功耗（GB10 上为 ACPI/SoC 区域温度，非核心温度） |
| **RAM**（仅 host 类型） | 系统内存使用条 —— 与 VRAM 分开展示 |
| **Models** | 管理自己的 start.sh / stop.sh 启动器，见 6.1 |
| **LLM server** | 每端口一卡，见 6.2 |
| **ComfyUI** | 端口 `:8188` + **Open** 外链；运行中任务（标题、进度%、预计剩余、*Cancel*）；空闲 *Idle — no running job*；不可达提示 *Ensure ComfyUI is running…* |
| **Storage** | 合并读写吞吐、**Refresh**、齿轮可逐盘启停 + **Auto-refresh** 开关；每盘 label·device、用量条、↑写↓读速率、free |
| **Network** | Download / Upload 大数字 + 趋势线、Primary 网卡（链路速率）、每网卡 IP 与 tx/rx；**Settings** 可逐网卡开关 |
| **Tailnet**（开启时） | `online` / **`OFF TAILNET`**（红）/ `unknown`、backendState、健康消息、tailscale IP |

---

## 六、LLM 服务与模型管理

### 6.1 Models 面板（启停你自己的模型）

#### 概念：什么是"Model"？

Models 面板里的一个 "model" **不是模型文件本身**，而是一个**启动器（launcher）**：指向 Spark 上的一个**文件夹**，该文件夹里放着你写的 `start.sh` 和 `stop.sh` 两个脚本。点击 Start 时，sparkDash 通过 **SSH 以该单元的 SSH 用户身份**在目标机上执行 `start.sh`，并把输出实时流回面板终端。

关键行为（源码 `server/llmlaunch/commands.js`）：

- **start 是"脱钩"运行的**（setsid 分离进程）：关浏览器、重启 sparkDash、SSH 断开，**模型都不会被杀**，之后还能用 *Output* 重新接入看日志
- **stop 是"挂靠"运行的**：stop 脚本执行期间连接断了会被终止
- 脚本输出写到目标机 `~/.cache/sparkdash/llm/<id>.log`（日志超 32MB 自动裁到 8MB）；面板终端最多显示**最新 5000 行**
- 脚本**可以没有执行位**：有 `x` 权限就直接执行，否则用 `bash` 解释执行
- 每个 Spark 最多注册 **24 个**启动器

**入口**：Spark 详情页 **Models** 面板 → *Add model*（空列表时为 *Add your first model*）；Overview 卡片的 ModelLauncher 也能注册/启动第一个。面板状态每 20s 自动刷新。

#### Add model 对话框：逐字段说明

> 对话框顶部提示语：*"Point sparkDash at the folder on {Spark} that holds your model's start and stop scripts. It runs those scripts as the SSH user and shows their output here."*

| 字段 | 必填 | 用途 | 填写规则与校验 | 示例 |
|------|------|------|----------------|------|
| **Name** | ✅ 必填 | 显示在列表、Overview 卡片上的名称；同时用于生成内部 ID（名称 + 随机后缀，所以改名不会串到旧进程的日志） | 非空，**≤ 80 字符**；纯展示用，不参与任何路径 | `Qwen3-8B vLLM`、`GLM 5.3 EXL3` |
| **Directory on {Spark}** | ✅ 必填 | **start/stop 脚本所在的文件夹**（在目标机上的路径），运行时会 `cd` 进去执行 | ① 必须以 **`/` 或 `~/`** 开头（绝对路径或家目录相对路径）② **不允许空格** ③ 服务端还会强制检查：该目录**必须解析在 SSH 用户的家目录之内**，且**不是全局可写**（否则拒绝执行） | `/home/you/llms/qwen3-8b` 或 `~/llms/qwen3-8b` |
| **Start script** | 选填（默认 `start.sh`） | 点 **Start** 时执行的脚本**文件名**（不是路径，相对于上面的目录） | 必须是**文件名**且以 **`.sh` 结尾**，只允许字母、数字、`. _ -`；执行前服务端校验：**必须是普通文件、不能是软链接、属主必须是 SSH 用户本人、不能全局可写** | `start.sh`、`run-vllm.sh` |
| **Stop script** | 选填（默认 `stop.sh`） | 点 **Stop** 时执行的脚本文件名；也用于清理**在面板之外手动启动**的同款模型 | 规则同 Start script | `stop.sh`、`kill-vllm.sh` |
| **Port** | 选填 | 填了之后，当 sparkDash 探测到**该端口上有 LLM 在应答**时，列表项会亮绿色 **`Serving :端口`** 标签（并显示模型 id） | 数字，**1–65535**；留空则只按脚本运行状态显示 Running/Stopped。**重要：此端口必须也出现在该单元的 LLM ports 列表里**（Add Spark / Edit Spark 里配的），否则探测不到、不会显示 Serving | `8000`、`8888` |
| **Notes** | 选填 | 备注，显示在列表项名称下方 | ≤ 300 字符，纯展示 | `2-node cluster, needs ~100 GB` |

按钮：*Cancel* 取消 / ***Add model*** 提交（编辑时为 *Save*）。校验错误会以红字显示在表单底部（如 *"The directory must start with / or ~/"*、*"Spaces are not supported in the directory path"*）。

**修改限制**：模型**运行中时不能改 Directory / Start / Stop 三个字段**，服务端返回 *"This model is still running. Stop it before changing its directory or scripts."* —— 先 Stop 再改。

#### 一个可用的最小示例（vLLM 场景）

目标机 `~/llms/qwen3-8b/` 下放两个文件：

```bash
# start.sh —— 前台方式：脚本活着 = 模型在跑
#!/usr/bin/env bash
cd "$(dirname "$0")"
exec vllm serve ./Qwen3-8B-FP8 \
  --host 0.0.0.0 --port 8000 \
  --served-model-name qwen3-8b
```

```bash
# stop.sh —— 按端口收拾干净（也可以写 pkill -f "vllm serve"）
#!/usr/bin/env bash
fuser -k 8000/tcp 2>/dev/null || true
```

对话框填写：Name=`Qwen3-8B vLLM`，Directory=`~/llms/qwen3-8b`，Start/Stop 留默认，**Port=`8000`**，并确认该单元的 LLM ports 里包含 8000。

> `start.sh` 写成"后台启动后立即退出"（`vllm serve ... &` + `exit 0`）也可以：状态靠**进程组存活检测**判断，脚本退出后只要模型进程还活着就仍显示 Running。

#### 保存后的列表操作

每行按状态显示不同按钮：

| 状态标签 | 含义 |
|----------|------|
| **`Serving :端口`**（绿） | 端口上有 LLM 正在应答（仅在填了 Port 且该端口被探测时） |
| **`Running`**（蓝） | start 脚本（或它拉起的进程组）还在目标机上运行 |
| **`Stopped`** | 进程组已不存在 |
| **`Stopping…`** | stop 脚本执行中 |

| 按钮 | 行为 |
|------|------|
| **Start** | 执行 start 脚本（脱钩）；若**已有另一个模型在跑**，弹出 *"… is running. Stop it first?"* → *Stop it, then start* 会先跑旧模型的 stop 脚本、**成功后自动接续 start**（stop 失败则不会启动新模型） |
| **Stop** | 执行 stop 脚本（挂靠，有确认）；即使状态是 Stopped 也保留此按钮，用于**清理在别处手动启动的模型** |
| **Output** | 重新接入查看正在运行的 start 脚本输出 |
| **Edit**（铅笔） | 打开同一对话框修改（运行中禁止改路径/脚本） |
| **Remove**（×） | 从列表移除，确认语明确写着 *"Files stay untouched."* —— **只删注册记录，不删目标机上的任何文件** |

**终端区**（面板底部，Start/Output 后出现）：状态徽标 + 标题 + 已运行时长；工具按钮 **Stop watching / Cancel**（停止观看，**脚本继续跑**；stop 场景则中止 stop 脚本）、**Copy**、**Expand**（全屏弹窗）、**Show/Hide output**；偏好勾选 *Show the output when I start or stop a model*（存本浏览器）。日志被截断时提示 *"Only the newest 5,000 lines are kept."*；跟随超过 **6 小时**会显示 *Stopped watching*（*detached*，模型照常运行）。

**空态文案**：*"Add the folder on {name} that holds a model's start.sh and stop.sh. Then start and stop it from here and watch it load."* ；单元离线时提示 *"{name} is offline, so models cannot be started or stopped."*

> **与 LLM 面板的关系**：Models 面板管**进程生命周期**（启/停/看日志），LLM server 面板管**运行指标**（tok/s、TTFT）。两者通过 **Port** 串联：Port 填对 + LLM ports 配对 → 启动后 LLM 卡自动亮起，形成"一键启动 → 立即看指标"的闭环。

### 6.2 LLM server 面板

每个端口一张卡，可 **+ Add LLM port** 增删端口。

**指标**（各带 ⓘ 解释）：

| 指标 | 含义 |
|------|------|
| model id + 引擎 | 自动识别 llama.cpp / vLLM / sglang / ds4-server / EXL3 / q27 等 |
| **Decode tok/s** | 实时生成速度 |
| **Prefill tok/s** | 预填充速度，进行中显示 *prefilling* |
| Cached / Uncached prefill | 缓存与未缓存预填充 |
| KV-cache 池 | 剩余 token 与内存、使用率 |
| Running / Waiting | 请求队列 |
| TTFT mean/p95、E2E p95、ITL p95 | 延迟分位 |
| Preempts、Prefix cache、MTP accept | 引擎健康度 |
| **PostureBadge** | 端点暴露/鉴权姿态 ok / warn / danger |
| Port + **API key** | 若端点需要 Bearer token；已存显示 `•••••••• (saved — leave blank to keep)` |
| 趋势图 / Daily peak / Total tokens | tok/s 曲线、每日峰值、按模型累计 |

**Bench 启动器行**：`Decode` `Prefill` `Quality` `Tool Eval` 四个按钮（跳转对应基准页并**自动预选该 Spark**）+ **Showcase**（新窗口打开）+ **Remote**（展开填 Host URL / Port / HTTPS，可对任意 OpenAI 兼容端点跑 Decode/Prefill/Quality，*nothing is probed until you run*）。

---

## 七、基准测试（Benchmarks）

侧边栏 **Benchmarks** 区，或从 LLM 面板按钮进入。每页顶部有 **Run on** Spark 选择 chips（记住上次选择；默认取第一个在线且有 LLM 的非 worker 单元）。

### 先懂概念：Prefill 与 Decode（读本章前的 3 分钟科普）

本章的 Decode / Prefill 基准名字就来自**一次模型推理的两个阶段**。任何一次请求（比如发一条聊天消息）都固定分成首尾相接的两段：

```text
你的输入: "写一篇短文"
     │
     ▼
┌─────────────┐   阶段一：Prefill（预填充）
│ 并行读入全部 │   一次前向传播，把整段输入 token 同时算完
│ 输入 token   │   → 生成第一个输出 token
│             │   计算密集：所有输入一起过 Transformer，GPU 算力吃满
└─────────────┘
     │  ← 这里产生 TTFT（首字延迟）
     ▼
┌─────────────┐   阶段二：Decode（解码）
│ 一个一个往外 │   自回归：每生成 1 个 token，都要拼回输入再算一次
│ 吐 token     │   循环 N 次直到结束
│ "写""一""篇" │   访存密集：每次只算 1 个 token，算力大量闲置，
└─────────────┘   瓶颈在读权重 / KV cache 的显存带宽
     │
     ▼
生成结束
```

**两阶段对比**

| | **Prefill（预填充）** | **Decode（解码）** |
|---|---|---|
| 计算方式 | 输入 token **并行**一次算完 | 输出 token **串行**逐个生成 |
| 耗时主导 | 输入越长越慢（线性增长） | 输出越长越慢（次数多） |
| 硬件瓶颈 | **算力**（GPU 算得多） | **显存带宽**（反复读权重和 KV cache） |
| 用户体感 | 等第一个字出现（**TTFT**） | 打字速度（**tok/s、ITL**） |
| 对应指标 | Prefill tok/s、TTFT、Cached prefill | Decode tok/s、ITL、KV cache、队列 |

**与本章及监测的关联**：

- 粘贴 10 万字长文档 → **prefill 主导**：TTFT 飙升、GPU 利用率高，但 decode 还没开始
- 让它写 2000 字长文 → **decode 主导**：首字很快，但要持续输出，KV cache 逐渐填满
- 一次完整请求 `E2E 延迟 ≈ TTFT + 输出token数 × 每token间隔`
- 因此 **Prefill 基准测的是"读题有多快"**（各上下文长度下的 prefill tok/s 与 TTFT），**Decode 基准测的是"打字有多快"**（各并发下的生成吞吐）—— 一个是算力/长上下文能力，一个是访存/批量吞吐能力，**两者都要测才完整**
- 日常监测中同理：GPU 忙但 tok/s 低 = 可能卡在 prefill（prefill-bound）；Waiting 堆积、KV 满 = decode 阶段的容量不够用（见第十五章 15.3）

### 各基准一览

| 基准 | 用途 |
|------|------|
| **Decode** | 不同并发下的生成速度（tok/s） |
| **Prefill** | 各上下文长度的预填充速度与 TTFT |
| **Quality** | 固定种子题库打分，比较模型/量化/KV 格式 |
| **Tool Eval Bench** | 69 个确定性场景的工具调用质量 |

### 7.1 Decode

- **输入**：Model（多模型时）、**Type**（Structured / Prose / Code / JSON，temp 0、thinking off）、**Concurrency** 多选（1–32，默认 1、2）、**Max tokens/stream**（默认 400，范围 64–2048）
- **运行**：*Run benchmark*（运行中变 *Cancel run*）；进度条 `x/y levels` + 实时 tok/s
- **结果**：每并发一行 —— ×N 徽标、TTFT、成功/失败流数、**Aggregate tok/s**、Stream tok/s、比例条
  - *Aggregate* = 所有并发流的总 decode tok/s；*Stream* = 单流平均
- **收尾**：*Clear history* / *Copy results*（开启 Benchmark share image 后可 *Copy as image*）/ *Run again*；页底 History 表可回看

#### 任务类型（Type）：Structured / Prose / Code / JSON

**先明确一个关键点**：Type 只是**提示词模板的标签**，**不是 guided decoding / 语法约束**（源码注释原文：*"prompt labels only — not guided decoding"*）。选了 JSON 不会启用 JSON grammar 强制解码，模型仍是自由生成——它只决定"让模型写什么样的内容"。测试固定 **temp 0、thinking off**，保证可复现。

| 类型 | 提示词内容（实际发送） | 输出特征 | 适用场景 |
|------|------------------------|----------|----------|
| **Structured**（默认） | *Count from 1 to 200. Output only the numbers, separated by spaces.* | 纯数字序列，格式极简、无歧义 | 实验室标准协议（对应官方 `bench_decode.py --structured`）；**结果最稳定**，适合快速横向对比 |
| **Prose** | *Write a detailed step-by-step explanation of how a hash map works…* | 自然语言长文（哈希表逐步讲解） | 实验室默认基准 prompt；**最贴近聊天/文档生成**的真实负载 |
| **Code** | 每路并发发送**不同的真实 Python 任务**（binary_search、merge_sort、lru_cache、dijkstra……共 32 个任务的目录），要求"只输出 Python 源码，无注释、无 docstring、无 markdown 围栏"，并附带测试 | 代码：缩进、标识符、符号密集 | **代码助手类负载**；各路前缀天然不同，专为并发测试设计（详见下方并发行为） |
| **JSON** | *Emit only a JSON array of fake GPU metrics rows…*（大量引号/括号/逗号的符号密集型输出） | 紧凑 JSON 数据流 | **结构化 API / 数据导出类负载**；注意仍是普通提示词，非 guided JSON |

**并发时的提示词行为**（`pickDecodeBenchPrompts`）：

- **Code**：每路取目录中**不同的任务**（第 1 路 binary_search、第 2 路 merge_sort……），前缀天然不同；跑之前还有一次**不计分的 JIT warmup**，避免首路的前缀缓存命中让后续并发"占便宜"
- **Structured / Prose / JSON**：各路共用同一提示词，但自动追加 `(stream i/N)` 后缀——**确保各路不共享前缀缓存块**，测的是真并发而非缓存红利
- 所有类型都会自动追加*"继续生成直到达到最大输出长度，不要提前停止"*的指令，防止模型提前 EOS 导致实际生成远少于 Max tokens

**使用建议**：

1. **横向对比（两台机器/两次运行）务必用同一 Type** —— 不同 Type 的 tok/s 不可直接比较（数字流、散文、代码的分词密度和生成模式不同，天然有快慢差）
2. 按自己的业务选：聊天/文档 → **Prose**；代码补全 → **Code**；结构化数据 → **JSON**；只求稳定基准 → **Structured**
3. 常规做法：用 **Structured** 做可复现的基线对比，再用自己业务对应的 Type 复测一遍看真实表现

#### 结果分析：线性度判读与"甜点并发"

Decode bench 看的不是一个分数，而是**三个数字的联动**：Aggregate（聚合吞吐）、Stream（单流速度）、TTFT。核心问题是：**加并发换来的总吞吐，是不是以牺牲单流体验为代价？**

**实测示例**（某单元，Max tokens 400，从 ×1 一路测到 ×32）：

```text
×1   38.5 agg   38.5/str  · TTFT  61ms
×2   76.1 agg   38.1/str  · TTFT 143ms
×3  112.7 agg   37.7/str  · TTFT  87ms
×4  149.9 agg   37.6/str  · TTFT  95ms
×5  183.1 agg   36.7/str  · TTFT 164ms
×6  219.3 agg   36.7/str  · TTFT 119ms
×8  290.7 agg   36.5/str  · TTFT 131ms
×10 344.1 agg   34.7/str  · TTFT 169ms
×12 412.8 agg   34.6/str  · TTFT 205ms
×16 542.5 agg   34.2/str  · TTFT 214ms
×24 753.9 agg   31.7/str  · TTFT 290ms
×32 948.6 agg   30.4/str  · TTFT 480ms
```

判读步骤：

**① 算线性效率 = 实际加速比 ÷ 理想加速比**

| 并发 | Aggregate | 加速比 | 线性效率 | 单流变化 | 判定 |
|------|-----------|--------|----------|----------|------|
| ×2 | 76.1 | 1.98× | 99% | -1% | ✅ |
| ×4 | 149.9 | 3.89× | 97% | -2% | ✅ |
| ×8 | 290.7 | 7.55× | 94% | -5% | ✅ 仍接近线性 |
| ×12 | 412.8 | 10.72× | 89% | -10% | ✅ |
| ×16 | 542.5 | 14.09× | 88% | -11% | ✅ 边际开始缓降 |
| ×24 | 753.9 | 19.58× | 82% | **-17.7%** | ⚠️ 逼近警戒线 |
| ×32 | 948.6 | 24.64× | 77% | **-21%** | ❌ 单流跌破 -20% |

- **效率 ≥ 90%**：vLLM continuous batching 正常，**GPU 远未饱和**，每加一路并发稳定贡献约一路的 tok/s
- **效率 70–90%**：开始出现带宽/调度竞争（本例 ×10 起进入此区间，每路边际贡献从 35–38 逐步降到 24–26），仍可用
- **效率 < 70% 或 Aggregate 走平**：吞吐到顶，再加并发只增加排队

**② 看 Stream 是否跳水**

- 本例单流从 38.5 缓降到 ×16 的 34.2（-11%），到 **×32 才跌到 30.4（-21%），首次突破 -20% 警戒线**
- **警戒线**：单流比 ×1 掉 20% 以上说明带宽挤爆，甜点已过
- **注意绝对值**：业务若要求单流 ≥35 tok/s，本例最多开到 **×10**（34.7）；对吞吐型批量业务，单流 -20% 可以接受

**③ 看 TTFT 的涨幅**

- bench 中所有流**同时发起**，×N 的 TTFT 包含排队 + prefill 交错开销；本例 61ms → 480ms，短 prompt 下绝对值仍 <0.5s，正常
- **只看趋势，别抠单点**：注意 ×2=143ms 反而比 ×3=87ms 高——同发 burst 下 TTFT 抖动很大，单个点的异常不代表问题
- **不要**把它当生产 TTFT 预期（真实长输入的 TTFT 由 prefill 决定，见 7.2）
- 若 TTFT 出现**数量级跳变**（如毫秒级 → 秒级），才需要配合 Running/Waiting 确认是否严重排队

**④ 定位两个拐点，得出甜点并发**

先明确定义这两个"拐点"——它们不是官方术语，而是本手册为方便判读给**曲线开始变质的位置**起的名字：

> **体验拐点（单流体验拐点）**
> 继续加并发时，**单流速度（Stream tok/s）首次跌破基线（×1 单流）的 80%**（即下降超过 20%）的那一档。
> *含义*：加并发带来的总吞吐增长，开始**明显挤压每一个用户的生成速度**——从"人人够用"变为"人人变慢"。它回答的问题是：**再加一路，现有用户的体验还撑得住吗？**

> **吞吐拐点（聚合吞吐拐点）**
> 继续加并发时，**Aggregate（聚合 tok/s）不再显著增长、曲线趋于走平**（相邻两档的边际增量明显萎缩，趋近于零甚至倒退）的那一档。
> *含义*：GPU 算力/显存带宽/调度已**饱和**，再加并发不会产出更多 token，只会让请求在队列里越排越长（表现为 TTFT 暴涨、Running/Waiting 堆积）。它回答的问题是：**这台机器的总产能天花板在哪？**

两者关系与取舍：

- **甜点并发 = 两个拐点中先出现的那个的前一档**（取更保守者）——先撞上哪个，就由哪个说了算
- **只有吞吐拐点**：还早着呢，但单流可能已经在慢慢变差，仍要对照 -20% 警戒线
- **只有体验拐点**：产能还有富余，但面向交互式用户时**体验比吞吐更重要**，同样以它为准
- **测试范围内两个都没出现**：说明拐点在更高并发处，可加档补测；**不要**把"没测到拐点"当成"无限可加"

| 拐点 | 本例位置 | 现象 |
|------|----------|------|
| **体验拐点**（先到） | **×32** | 单流 30.4，跌破 ×1 的 80%（-21%） |
| **吞吐拐点**（未到） | >×32 | Aggregate 仍在涨（×24→×32 还 +25.8%），未走平；但边际贡献已从 ~36/路降到 ~24/路 |

**甜点并发 = 先出现的拐点的前一档 → 本例 ×24**；结合业务目标微调：

- 交互式业务（保单流体验）：**×10 以内**（单流 ≥34.7）
- 吞吐/体验平衡：**×16–24**（推荐对外服务上限）
- 纯吞吐最大化：可补测 ×48/×64 找 Aggregate 真正走平的位置，但超过 ×24 后每路用户都在明显变慢，交互式业务无意义

**横向对比要点**：两台机器同参数各测一遍，**拐点更靠右（仍线性的档位更高）的那台调度/带宽余量更大**，而不是只比 ×1 的单流数字。另外，本例与另一轮同参数测试在 ×1/×2/×4/×8 各点的 Aggregate 差异 **<0.5%** —— temp 0 + 固定 prompt 下这套 bench 可复现性极佳，数据可以放心用于严格对比。

### 7.2 Prefill

- **输入**：**Context size** 按钮网格（超模型上下文的自动禁用）+ 自定义 token 数 + Add
- 提示：*Unique-prefix prompts；128k–300k can take tens of minutes.*
- **结果**：每上下文尺寸一行 prefill tok/s + TTFT；支持清历史 / 复制 / 重跑

#### 结果分析：为什么输入越长，prefill tok/s 越小？

**实测现象：上下文从 1k 加到 32k/128k，prefill tok/s 单调下降 —— 这是预期行为，不是测错了。**原因按主次：

1. **Attention 计算量随序列长度超线性增长（主因）**
   - Prefill 虽是所有输入 token 并行一次算完，但**每个 token 都要和前面所有 token 做 attention**
   - 每 token 的计算量 = 常数部分（过权重矩阵）+ `O(序列长度)` 部分（attention）
   - 输入 1k 时第二项可忽略，tok/s 接近满速；输入 128k 时每处理一个 token 都要 attend 128k 个位置，**单 token 成本变高 → 每秒能处理的 token 数下降**
   - **下降曲线的斜率就是这台机器的"长上下文能力"**

2. **KV cache 读写带宽压力**
   - 长 prompt 的 prefill 要边算边把 KV 写入显存，attention 还要读回之前所有层的 KV
   - 序列越长，**显存带宽消耗越大**；长上下文段 prefill 会从"算力受限"逐渐转向"带宽受限"，吞吐进一步降低

3. **引擎调度因素（vLLM 尤其相关）**
   - **Chunked prefill**：超长 prompt 被切成固定大小的 chunk 串行处理，chunk 间有调度开销
   - prefill 与在线 decode 请求**共享调度器**：测试时实例上若有其他流量，长 prefill 会更频繁地被切分、让路
   - 接近 `max_model_len` 时 KV 分配压力增大，可能再掉一档

4. **数据可信度保证**：bench 刻意使用 **unique-prefix prompts（唯一前缀）绕开前缀缓存**，测的是真实的"从零读题"速度；若用重复前缀，长上下文数字会被缓存命中虚高，反而测不出真实曲线

#### 曲线判读

| 曲线形态 | 判断 |
|----------|------|
| 平缓下降（如不低于 1k 时的 80%） | ✅ 健康，符合预期 |
| 中后段加速下滑 | ⚠️ 带宽/调度可能到瓶颈，对照详情页 Throttle 标签 |
| 某一档**断崖式**下跌或直接失败 | ❌ 可能撞到 `max_model_len`、KV OOM 或量化配置问题 |
| TTFT 增长略快于线性 | 正常（含 O(L²) 分量）；若快得离谱，看 Running/Waiting 是否在排队 |

**实际用途**：这条曲线直接回答**"这台机器处理多长的文档会明显变慢"**。两台对比时，若 16k 前差不多、32k 后拉开差距，说明差异出在长上下文段的显存带宽或 KV 量化配置上——这比单看一个点的 tok/s 有信息量得多。

### 7.3 Quality

- **Categories 多选**：QA(150)、Reasoning(40)、Arithmetic chain(40)、State tracking(40)、GSM8K(200)、MMLU(285) 默认勾选；可选 Instruction following(40)、Long-context recall（8k–256k）
- 每卡显示条目数、thinking 标注、**How it works** 展开说明（测什么/发什么/怎么评分/成本/示例）
- **其他**：Concurrency（默认 4，上限 16）、**Run label**（如 `fp4 KV`）、**Compare with**（与历史运行对比）
- **结果**：总分 + 分类得分 + 历史表；配对比较给出 Run A / Run B、相同回复数、单边正确题数、**McNemar p 值**（p ≥ 0.05 显示 *difference within noise*）
- 约束：每单元同时只能跑一个 quality；保留最近 30 次；默认全量约 755 个请求，thinking 类目较慢

#### 八类场景（Categories）详解

**总览**（✔ = 默认勾选）：

| # | 类别 | 条目数 | Thinking | 单题上限 | 耗时 | 默认 | 测什么 |
|---|------|--------|----------|----------|------|------|--------|
| 1 | **QA** | 150 | 关 | 64 tok | 快 | ✔ | 短题直答的**基础准确率**（算术/字符串/日期） |
| 2 | **Reasoning** | 40 | 开 | 8192 tok | 慢 | ✔ | 故事中多个量的**多步推理** |
| 3 | **Arithmetic chain** | 40 | 开 | 12288 tok | 慢 | ✔ | 10 步**长链精确算术**（一步错全盘错） |
| 4 | **State tracking** | 40 | 开 | 12288 tok | 慢 | ✔ | 25 次转移后的**状态跟踪**（类比变量追踪） |
| 5 | **GSM8K** | 200 | 开 | 8192 tok | 慢 | ✔ | 公开基准：小学应用题（可与文献粗对照） |
| 6 | **MMLU** | 285 | 关 | 1024 tok | 快 | ✔ | 公开基准：57 学科四选一**广度知识** |
| 7 | **Instruction following** | 40 | 关 | 1024 tok | 快 | ✘ | **格式指令遵循**（代码解析输出的可靠性） |
| 8 | **Long-context recall** | 按选档×条数 | 关 | 512 tok | 很慢 | ✘ | 长文档中**找最新值**（抗"旧值干扰"） |

> 默认 6 类 = 150+40+40+40+200+285 = **755 个请求**。

**1️⃣ QA —— 基础准确率（快速体检）**

- **测什么**：短、可验证问题的直接正确率。*"a bad quantisation or KV-cache format shows up here first"* —— **换量化/KV 格式后最先在这里暴露问题**
- **怎么测**：150 题 = 30 道固定题 + 120 道固定种子生成题（乘法、加减、字符串反转、字母计数、排序、二进制、日期偏移、最小公倍数），thinking 关
- **怎么判分**：**回复必须"承诺"答案，不能只是提到它** —— 数字题取回复中**最后一个数字**（写 "391 or 5" 判错）；日期取最后一个日期；排序取最后一组数；单词/反转串需在短回复中整体命中或以它收尾。**"先答对又自我纠正"、"not Canberra" 这类绕法都判错**
- **示例**：`What is 17 * 23?`
- **成本**：快，每题 ≥64 token

**2️⃣ Reasoning —— 多步推理**

- **测什么**：在故事中同时理清多个量、按顺序做运算的能力
- **怎么测**：40 道词题——三人各持若干物品，互给、丢失、翻倍，最后合并均分入箱、余数放罐；**要求同时答出两个数**；thinking 开
- **怎么判分**：末行必须是 `Answer: <余数>, <每箱>`，**两个数字都必须完全正确**
- **成本**：慢（思考模型更慢），每题上限 8192 token

**3️⃣ Arithmetic chain —— 长链算术**

- **测什么**：长算术链的**精确性**——链中任何一步出错，最终数字就错
- **怎么测**：40 条链：从 3 位数起，依次做 10 步运算（乘、加、减、取余+1000、向下取整除法），全程保持整数；thinking 开
- **怎么判分**：末行 `Answer: <数字>` 必须等于真值（接受 markdown/千分位逗号）
- **成本**：慢，每题上限 12288 token

**4️⃣ State tracking —— 状态跟踪**

- **测什么**：长序列小变更中保持状态正确——**类比程序里的变量追踪**
- **怎么测**：40 个故事：5 人各持初始 token，随后 25 个事件（转赠、捡到、给一半向下取整），问某人最后有多少；thinking 开
- **怎么判分**：末行 `Answer: <数字>` 完全正确
- **成本**：慢，每题上限 12288 token

**5️⃣ GSM8K —— 公开基准·小学应用题**

- **测什么**：课堂数学应用题能力；**公开数据集，结果可与文献粗对照**
- **怎么测**：从 GSM8K test 集（MIT 协议）用固定种子**一次性抽取 200 题**（全集 1319 题），逐步求解；thinking 开
- **怎么判分**：末行 `Answer: <数字>` 等于数据集答案；允许 `$`、千分位逗号、`%`、单位（如 "18 dollars"），**不允许第二个数值或含糊表述**
- **⚠️ 误差提示**：200 题抽样的单次得分 **95% 置信区间约 ±4–7 分**——**小差异请用配对比较（McNemar）**，且**不要**与文献公布的 GSM8K 全集分数直接比

**6️⃣ MMLU —— 公开基准·57 学科四选一**

- **测什么**：法律、医学、数学、历史、计算机等 **57 个学科的广度知识与应试推理**
- **怎么测**：从 MMLU test 集（MIT 协议）固定种子**每科抽 5 题 = 285 题**，四选一 A–D；thinking 关
- **怎么判分**：末行 `Answer: <字母>`（大小写不限；"B. Paris" 可以，"B or C" 不行）或回复只有字母
- **⚠️ 误差提示**：285 题单次得分 **约 ±5 分（95%）**；同样**勿与文献 few-shot MMLU 分数直接比较**

**7️⃣ Instruction following —— 格式指令遵循（IFEval 风格）**

- **测什么**：按**精确格式要求**输出的能力——*"the kind of thing that breaks pipelines that parse a model's output"*，直接关系生产解析管线的稳定性
- **怎么测**：40 个短写作任务，每题 2–4 条**可程序化验证**的规则，从这些组随机组合：字数上下限、恰好 N 个 `* ` 项目符号、恰好 N 段（`***` 分隔）、整段包在双引号里、`<<标题>>`、必须/禁止出现的词、禁用逗号、全小写/全大写、以指定句子收尾、以 `P.S.` 结尾
- **怎么判分**：**全部规则由代码检查（不是再用一个模型打分）**，全部满足才 pass
- **示例**：*"Write a short piece about how bees make honey. 1. Use exactly 4 bullet points… 2. Do not use any commas."*
- **默认不勾选的原因**：*"so older overall scores stay comparable"*——加进来会改变总分口径，历史对比就断了

**8️⃣ Long-context recall —— 长上下文"针"召回**

- **测什么**：**在长文档深处找到"最新值"而非过期值**的能力——正是 RAG/长对话场景的核心风险（模型抄了旧答案）
- **怎么测**：生成一篇长填充文本，散布 **16 条** `Remember this: the code for <动物> is <5位数字>` 事实，其中 **4 条在文末附近被更正**；然后问每只动物的**当前**编码。上下文档位自选（**8k / 16k / 32k / 64k / 128k / 256k**），每档条数默认 2（最多 5）
- **怎么判分**：**16 个最新编码全对才 pass**；被更正过却答了旧值 = stale，判错
- **约束**：串行逐条执行（不并发）、每题 512 token、大档位**可能耗时数分钟**；超出模型上下文的档位自动跳过

**使用说明（怎么选、怎么读）**

1. **新手首次跑**：保持默认 6 类全选 → 得到一条基线（约 755 请求，thinking 类较慢，留足时间）
2. **换量化/换 KV 格式**：先看 **QA**（最灵敏的体检项），再看 Reasoning/Arith 确认推理没退化
3. **生产解析管线用户**：额外勾选 **Instruction following**（但注意总分口径变化，勾了就一直勾，别和历史裸分混比）
4. **长文档/RAG 场景**：勾选 **Long-context recall**，第一轮选默认 32k 即可；模型宣称支持多长，就用对应档位实测
5. **两台对比 / 前后对比**：**类别组合必须完全相同**；差异不大时用 **Compare with** 配对比较看 **McNemar p 值**（p ≥ 0.05 = *difference within noise*，别把噪声当提升）
6. **提交值约定**：temperature 0、固定 `seed=1234`、题目由固定种子确定（suite v1）——同一套题可逐题配对；**请求错误/超时不计分**（排除出对比表）
7. **成本预期**：QA/ MMLU / Instruction / Long 快；Reasoning / Arith / Track / GSM8K 是 thinking + 高 token 上限的"慢类"；同时只能跑一个 Quality，跑之前确认实例上没有真实流量

### 7.4 Tool Eval Bench

- **SetupCard**：检查 `tool-eval-bench` CLI 是否装在该 Spark（含安装指引）
- 三个页签：**Configure and run / Results / History**
- 配置视图：**Simple**（预设模板）/ **Advanced**（服务端 spec 全量选项）
  - 模板（固定 seed 42）：*Regular run*（69 场景，10–20 min）、*Regular + Hard*（92，15–30 min）、*Short*（15，约 2 min）、*Hard only*（23，5–10 min）+ **Trials**（重复试验次数）
- 测试目标：**Model on {spark}**（`127.0.0.1:{port}`）或 **Custom URL**（Base URL + Model + API key；*key 只发给该 Spark 一次，不存储*）
- 无模型在跑时提示先从 Models 面板启动
- **RunCard** 预览命令 → **Start**，运行中显示 done/total、分数、**Stop**
- 离线警告：*The tool runs on the Spark, so runs and checks need it online.*

> 页脚署名：*Powered by tool-eval-bench by SeraphimSerapis.*

---

## 八、Prompt Showcase

URL：`/showcase/:id`（可带 `?port=&model=`），**全屏、无侧边栏**的多终端 LLM 流式演示，最多 32 路。

**配置区**：

| 项 | 默认 / 范围 |
|----|-------------|
| Port | 该单元的 LLM 端口 |
| Terminals | 1–32（自动排成近方形网格） |
| Prompt type | 类型选择 |
| Max tokens | 512 |
| Temp | 0–2，默认 0.7 |
| Thinking | 开/关 |
| Show prompts | 开/关 |

- **Run** 启动；**Stop** 需确认 *Stop all showcase streams?*
- 每个终端卡实时显示提示词与生成内容（reasoning 可折叠）、token 数、TTFT、实时/峰值 tok/s
- 顶部聚合：Model / Aggregate / Avg / Tokens / Server / Streams
- **Past runs** 可恢复；**Clear** 需确认 *Clear all saved showcase history for this Spark?*
- 轮询间隔 300ms；仪表盘内版入口在 Workspace → Showcase（带 *Run on* chips）

---

## 九、统计页面：Token / 能耗 / 活动

### 9.1 Token totals（`/tokens`）

- 时间范围：`Last 24 h / 7 d / 14 d / 30 d / All time`
- **Reset…** 菜单（全舰队或单个单元；警告 *Counting restarts from zero… This can't be undone.*）
- **Copy CSV** 导出
- 内容：汇总磁贴（Generated / Cached prefill / Computed prefill + 环比）、堆叠柱状图（**Group by Type / Model / Spark**）、**Insights**、**By model** 表（可点列排序）、**By Spark and endpoint** 表（可跳转）、**About this data**（UTC 天、引擎重启后计数器重播种、保留期 小时 72h / 天 35 天）

### 9.2 Fleet energy（`/energy`）

- 范围：`24 h / 7 d / 14 d / 31 d`；**Clear…**（>7 天 / >24 小时 / 全部）；**Copy CSV**
- 内容：含**电费成本**的汇总磁贴（价格在 Settings 设置）、按小时/天的堆叠柱状图、Average fleet power 折线、Efficiency、**By node** 可排序表、Insights
- 状态提示：
  - *Warming up — no complete energy interval recorded yet.*
  - *Fleet membership changed. Restart sparkDash to establish a truthful new accounting scope.*（改了舰队成员后重启生效）
  - *Partial coverage: x/N nodes currently fresh.*
- ⚠️ 数据是**估算值**（GPU + CPU 模型 + 23W 基础开销），非墙插实测

### 9.3 Activity（`/activity`）

- 实时事件时间线，按天分组（Today / Yesterday…），每条可展开
- 顶部按天堆叠柱状图（Errors / Warnings / Success / Info）
- 过滤：范围 `24 h / 7 d / All`、严重度 chips、类别、Spark 下拉、关键词搜索、清空
- **Copy** 导出可见事件；**Clear**（>7 天 / >24 小时 / 全部，带确认）
- 滚离顶部时显示未读计数，可一键回到顶部

---

## 十、Settings 设置

**入口**：侧边栏底部 **Settings**、命令面板 *Open settings*、手机标签栏 Settings。Esc 或点遮罩关闭；无改动时 Save 禁用。

| 分组 | 设置项 | 说明 |
|------|--------|------|
| **Appearance** | 4 色板 | White / Light / Dark / OLED，立即生效，**仅存本浏览器** |
| **Refresh rate** | 1s / 2s / 5s / 10s | 轮询间隔；远程走 SSH，1s 开销最大 |
| **Temperature unit** | °C / °F | |
| **Default LLM port** | 1–65535 | 新增单元预填（默认 8888） |
| **显示开关** | Hide offline Sparks | 自动隐藏离线单元 |
| | Hide worker nodes | 从 Overview 和侧边栏移除 Worker |
| | Show search and status filters | Overview 搜索框 + 状态下拉（默认关） |
| | Show Fleet Energy | Overview 能耗卡 |
| | Show LLM Token Totals | Overview Token 卡 |
| | Show active fleet exceptions | Overview 告警条（默认关） |
| **Benchmark** | Benchmark share image | Copy 按钮增加 *Copy as image* |
| | Save benchmark debug traces | 历史里存 prompts/HTTP IDs/预览/GPU 采样（默认关） |
| **GPU** | Detailed VRAM breakdown | VRAM 拆为 引擎/系统/空闲（默认开） |
| **Prometheus** | Prometheus metrics | 开启后 `{origin}/metrics` 可被抓取（默认关） |
| **能耗** | Electricity price | 货币符号 + 每 kWh 价格；留空隐藏成本 |
| **Access token** | Set/Not set + Add/Change/Clear | 连接设置了 `SPARKDASH_TOKEN` 的服务器；**仅存浏览器** |

底部：版本号、mia-ai.net、𝕏、GitHub 链接。

---

## 十一、快捷键与移动端

### 桌面快捷键

| 快捷键 | 作用 |
|--------|------|
| **Ctrl / ⌘ + K** | 打开/关闭命令面板 |
| **Ctrl / ⌘ + B** | 显示/隐藏侧边栏（输入框中不触发） |
| **M**（聚焦侧边栏条目时） | 键盘拾起重排，方向键移动，Enter/Space 放下，Esc 取消 |
| **Esc** | 关闭对话框/面板 |
| 修饰键点击链接 | 保留浏览器原生行为（新标签等） |

### 命令面板（Ctrl/⌘ K）能做什么

- **Go to**：Overview、Token totals、Fleet energy、Showcase、Activity、每个 Spark（hint 显示 online/offline）
- **Benchmarks**：各基准页
- **Actions**：Add a Spark or GPU host；在新窗口打开某 Spark 的 showcase；Edit 当前 Spark；Switch theme（`white · light · dark · oled`）；显示/隐藏侧边栏；打开设置

### 移动端

底部标签栏：**Overview** / **Sparks**（弹出单元选择 sheet）/ **Stats**（Token totals、Fleet energy、Activity，sheet 内含 Showcase、Benchmarks 与 *Shut down all*）/ **Settings**。

---

## 十二、状态、告警与常见提示

### 连接与数据新鲜度（顶部横幅）

| 文案 | 含义与处理 |
|------|------------|
| *Connecting to live telemetry…* | 首次连接中，稍候 |
| *Telemetry is stale. Last valid update was 42s ago.* | 超过 `max(10s, 3×轮询间隔)` 未更新，页面整体变暗 |
| *Live telemetry disconnected. Showing data from 2m 5s ago.* | WS 断开，展示的是旧数据 |
| 侧边栏 `Live · 2 s poll` | 正常；`Reconnecting…` = 正在重连 |

### 关键告警

| 提示 | 处理 |
|------|------|
| **Open access** 芯片 | 服务端非 loopback 且无 token，*任何人可改设置*。在服务器设 `SPARKDASH_TOKEN`；或点 *Don't show again*（按浏览器记忆） |
| **Host unreachable** | 检查网络/SSH；host 类型可点 **Wake up** |
| **Thermal throttle / HW slowdown** | GPU 过热或硬件降速，检查散热 |
| **storage is at or above 90%** | 清理磁盘 |
| **LLM is unavailable** | 端口无服务或探活失败，检查模型是否在跑 |
| **OFF TAILNET**（红） | LAN 通但脱离 tailnet，检查 `tailscaled` 会话 |
| *Hermes not found* | 目标机未装 hermes（`~/.local/bin` 或 `/usr/local/bin`） |
| *Benchmark interrupted — server restarted during the run* | 服务端重启打断了运行，重跑 |
| *No models are currently served by this target.* | 先从 Models 面板启动模型 |

### 关机与唤醒的限制

- 关机是**优雅关机**，依赖宿主 `/usr/local/bin/spark-shutdown`
- **DGX Spark 没有 Wake-on-LAN**（板载网卡不支持魔术包）；只有 host 类型且 NIC 支持才可 Wake
- 关机必须勾选确认 + 输入 `poweroff`，无法从 dashboard 撤销

---

## 十三、安全与远程访问

**默认只监听 `127.0.0.1`，HTTP/WS API 无应用层鉴权。**

### 推荐访问方式

```bash
# 1) SSH 隧道（最简单安全）
ssh -N -L 5555:127.0.0.1:5555 user@sparkdash-host
# 本机打开 http://127.0.0.1:5555
```

其他：带认证的 TLS 反向代理、Tailscale Serve，或 `BIND_HOST=0.0.0.0` **+** `SPARKDASH_TOKEN`。

### Token 使用流程

- 服务端设置了 `SPARKDASH_TOKEN` 时，首次请求/WS 被拒会弹 **Access token** 对话框
- 输入一次 → 校验 → 仅存本浏览器 → live 连接自动带 token 重连（无需刷新）
- **Settings → Access token** 可查看/修改/清除

### 密钥与凭据

- SSH 密码 AES-256-GCM 存 `config/sparks-secrets.json`，密钥在 `config/.secrets-key` —— **切勿删除密钥文件**
- **优先用 SSH 密钥**；Docker 下挂载到 `/root/.ssh/id_ed25519`，mode 600
- 备份 `config/` 目录即可保留全部单元配置与凭据

### Prometheus 抓取

Settings 开启 *Prometheus metrics* 后 `GET /metrics` 生效（关闭时 404）。鉴权与 REST 相同：loopback 开放；远程绑定 + token 时需 `Authorization: Bearer …`。

```yaml
scrape_configs:
  - job_name: sparkdash
    scrape_interval: 15s
    static_configs:
      - targets: ["sparkdash.lan:5555"]
```

抓取读的是内存快照，**不额外产生 SSH / nvidia-smi 调用**。

---

## 十四、故障排查 FAQ

**Q1：页面打不开 / 一直 Reconnecting？**
确认服务在跑（`curl http://127.0.0.1:5555/api/health`）；开发模式确认用 5173 端口；有 token 时看是否弹了 Access token 对话框。

**Q2：添加远程 Spark 时 SSH 测试失败？**
- 确认从 **sparkDash 服务器**能 `ssh user@ip` 通（SSH 不在浏览器里跑）
- 报 `sshpass is not installed`：**密码认证**需要 sshpass，`sudo apt-get install sshpass` 后**重启 sparkDash**（检测结果有进程内缓存）；或改用密钥认证（`ssh-copy-id` 后 Add Spark 密码留空）
- Docker：私钥要挂到 `/root/.ssh/id_ed25519` 或设 `SSH_IDENTITY_FILE`，mode 600
- 密码模式：先输入密码再点 Test（*Enter the SSH password first*）
- 改过 host/user/isLocal 后旧密码被丢弃，需重输

**Q3：本机单元没有指标？**
本机采集依赖 sysfs/proc/`nvidia-smi`，**只在 Linux 上可用**；Windows 上只能监控远程 Linux 单元（或用 WSL2/远程容器部署）。

**Q4：LLM 卡显示不可用？**
- 确认模型服务在该 Spark 的配置端口上监听（默认 8888）
- 端点需要鉴权时，在 LLM 卡填 **API key**
- 远程探测用的是 Spark 的 **LAN IP**，确认填对了

**Q5：Worker 卡看不到模型？**
在 Edit Spark 里给 Worker 选 **Head Spark**；Worker 本身没有本地 LLM API。

**Q6：Wake 按钮不生效？**
DGX Spark 不支持 Wake-on-LAN；host 类型需在 Edit Spark 填 **MAC Address**。

**Q7：能耗页提示 Fleet membership changed？**
增删过单元后需要**重启 sparkDash** 才能建立新的统计口径。

**Q8：数据变灰/变暗？**
顶部横幅会说明是 stale 还是断线；多为网络/服务端问题，稍等自动恢复。

**Q9：排序/设置没保存？**
看是否有 ErrorBanner（如 *Could not save the Spark order…*）；主题与 Settings 部分项**只存本浏览器**，换浏览器需重设。

**Q10：如何验证安装是否正常？**
```bash
npm run typecheck   # 前端类型检查
npm test            # 后端 node --test + 前端 vitest
```

**Q11：npm 启动后，一关终端 sparkDash 进程就没了，怎么办？**

**原因**：直接 `npm start` 时，node 进程挂在当前 shell 的会话下，终端关闭 / SSH 断开 → 进程收到 SIGHUP 被杀。三种解决方式按场景选：

**方案一：screen（最直观，推荐快速上手）**

```bash
sudo apt-get install -y screen        # 已装可跳过

screen -S sparkdash                   # 创建并进入名为 sparkdash 的会话
cd /path/to/sparkDash
npm start                             # 正常启动，日志照常滚动

# 按 Ctrl+A 然后按 D 脱离会话（d = detach）—— 此时关闭终端也没关系
```

之后随时回来：

```bash
screen -ls                            # 列出会话（显示 Attached / Detached）
screen -r sparkdash                   # 重新接入，页面数据一直保持
```

- 终端关闭、SSH 掉线，会话里的 sparkDash **继续运行**
- `screen -r` 提示 *There is no screen to be resumed* 说明会话已结束（进程退出了），重新 `screen -S sparkdash` 即可
- 局限：**服务器重启不会自动拉起，进程崩溃也没人拉起**

**方案二：nohup（不装任何工具的一行命令）**

```bash
cd /path/to/sparkDash
nohup npm start > sparkdash.log 2>&1 &
tail -f sparkdash.log                 # 看日志；Ctrl+C 只退出 tail，不杀进程
```

- 后续查看日志用 `tail -f sparkdash.log`；停止服务用 `kill $(pgrep -f "node server/index.js")`
- 局限：同 screen —— 不自动重启、不开机自启

**方案三：systemd（长期生产的正确姿势，自动重启 + 开机自启）**

```ini
# /etc/systemd/system/sparkdash.service
[Unit]
Description=sparkDash
After=network.target

[Service]
Type=simple
User=你的用户名
WorkingDirectory=/path/to/sparkDash
ExecStart=/usr/bin/npm start
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now sparkdash
sudo systemctl status sparkdash        # 查状态
journalctl -u sparkdash -f             # 看日志
```

**建议**：日常试用用 **screen**（立刻可用）；确认要长期运行后切换 **systemd** —— 它额外解决 screen/nohup 不管的三件事：**开机自启、崩溃自动重启（Restart=always）、日志统一进 journal**。仓库 `scripts/` 下只有 Docker 部署辅助脚本，没有现成 systemd 单元，需自行创建上面的文件。

---

## 十五、指标解读：监测入门必读

> 本章面向"第一次做模型监测"的读者：解释 sparkDash 上每个数字**是什么、怎么读、什么算异常**。看懂了这章，前面各章的界面才有意义。

### 15.1 硬件指标

| 指标 | 含义 | 怎么读 |
|------|------|--------|
| **GPU 使用率（GPU Util）** | 采样周期内**至少有一个 kernel 在 GPU 上运行**的时间占比 | 闲时 0–5%，推理时应明显拉高。**注意：它不等于算力利用率** —— 90% 利用率也可能只用了一半算力（batch 太小、访存瓶颈）。要结合 tok/s 一起看 |
| **显存 / 统一内存（VRAM）** | 模型权重 + KV cache + 框架缓冲占用的内存 | DGX Spark 是 128GB 统一内存，条形图可拆为 **引擎 / 系统 / 空闲**。空闲长期 <10% 要警惕：新请求可能被拒绝或触发 KV 驱逐 |
| **GPU 温度** | GPU 芯片温度 | sparkDash 标注 **Warm > 65°C、Hot > 85°C**。长期 Hot 会触发热降频（Thermal throttle），tok/s 掉就是这个原因 |
| **GPU 功耗（draw / limit）** | 当前功耗 / 功耗上限 | 推理打满时应接近上限；明显偏低 + 利用率低 = 没有真正的负载 |
| **节流标签（Throttle）** | Thermal（过热）/ Power cap（功耗墙）/ HW slowdown（硬件降速）/ No throttle（正常） | 出现前三种都是**硬件层告警**，优先处理（散热、供电），否则模型速度永远上不去 |
| **CPU / RAM / Storage / Network** | 主机侧资源 | RAM 与 VRAM **分开展示**（仅 host 类型有独立 RAM 条）；磁盘 ≥90% 出告警；网络给出 ↑上传 ↓下载速率 |

### 15.2 LLM 推理指标（核心重点）

一次 LLM 请求分两个阶段：**prefill（预填充，读你的输入）** → **decode（解码，一个字一个字往外吐）**。sparkDash 的 LLM 卡围绕这两阶段给指标：

| 指标 | 阶段 | 含义 | 直觉解释 |
|------|------|------|----------|
| **Decode tok/s** | decode | 每秒生成多少 token | **打字速度**。人阅读约 5–10 tok/s，20+ 就很流畅；单流 50–150 tok/s 属常见范围（随模型/硬件差异大） |
| **Prefill tok/s** | prefill | 每秒处理多少输入 token | **读题速度**。决定长文档/长提示词要等多久 |
| **TTFT**（Time To First Token） | prefill→decode | 从发请求到吐出第一个 token 的耗时 | **首字延迟**。聊天体验的关键；输入越长 TTFT 越高。p95 = 95% 请求都快过这个值 |
| **ITL**（Inter-Token Latency） | decode | 相邻两个 token 之间的间隔 | **打字均匀度**。ITL p95 高 = 偶尔卡顿一下（如 KV 重分配、被抢占） |
| **E2E p95** | 全程 | 整个请求端到端耗时的 95 分位 | 一次完整请求最坏情况的量级 |
| **KV-cache 池** | decode | 保存已算过 token 的内存池：剩余量 + 使用率 | **草稿纸**。满了新请求排队或驱逐旧缓存；`Prefix cache` 表示公共前缀复用命中情况 |
| **Running / Waiting** | — | 正在生成的请求数 / 排队等待数 | **队列**。Waiting 持续 >0 说明并发超出服务处理能力；对应 Overview 卡片上的 `2 running · 1 queued`、`3/4 slots` 芯片 |
| **Preempts** | — | 请求被引擎抢占（通常是 KV 不够）次数 | 持续增长 = 显存吃紧，batch 开太大或模型+上下文塞太多 |
| **Cached / Uncached prefill** | prefill | 命中前缀缓存的预填充量 | cached 占比高 = 省算力，同一批重复前缀的场景收益明显 |

**vLLM 特有**：vLLM（和 q27）的指标走其 **Prometheus `/metrics` 端点**读取（其余引擎为本地探测），所以 vLLM 卡上的队列、KV、延迟数据来自 vLLM 自己的统计，是权威值。

### 15.3 组合判断：几个典型场景

| 现象组合 | 可能原因 | 处理方向 |
|----------|----------|----------|
| GPU 利用率高、decode tok/s 也不错 | 正常满载 | 无需处理 |
| GPU 利用率低、Waiting 堆积 | 瓶颈不在 GPU：CPU 预处理、网络、或单流场景 batch 太小 | 看 ITL/E2E 是否网络问题；并发压上去再看 |
| GPU 利用率高、但 tok/s 很低 | 可能是**长上下文 prefill 占主导**（prefill-bound），或模型过大/量化不当 | 区分看 Prefill tok/s；用 Prefill bench 确认 |
| TTFT 平时低、随上下文变长暴涨 | 正常物理规律 | 对比 Prefill bench 曲线判断是否超预期 |
| Running 满、Waiting 持续 >0、Preempts 增长 | 显存/KV 不足 | 降并发、减上下文长度、或换更小模型 |
| tok/s 突然掉一档 | 热降频 / 功耗墙 / 被其他任务抢卡 | 看详情页 Throttle 标签和 GPU 进程表 |
| 数字全 0、卡显示 *LLM is unavailable* | 服务没起或端口不对 | 确认 vLLM 还活着、端口监听、绑定了可达地址 |
| 数据长时间不动 | 面板与服务器失联 | 看顶部 stale / Reconnecting 横幅（第十二章） |

### 15.4 读数的好习惯

1. **先看基线，再看波动**：服务空闲时记下各指标的"安静值"，之后对比才有意义
2. **速率类指标依赖持续采样**：sparkDash 后台轮询常驻（即使你没开浏览器），tok/s、网络字节/s 才准确；所以刚打开页面前几秒的数字可能偏低，等 1–2 个轮询周期
3. **p95 比平均值可信**：平均值掩盖长尾，p95 才是"用户最差体验"的代表
4. **历史曲线看趋势**：详情页 GPU 历史（最长 8h）和 LLM 趋势图，比瞬时值更能说明问题
5. **能耗是估算值**：Fleet energy = GPU + CPU 模型 + 23W 基础开销，不是墙插实测（第九章已标注）

---

## 十六、实战演练：两台 vLLM 单元上手路线

> **适用场景**：裸机 npm 部署，已添加两台跑着 vLLM 的服务器单元。按阶段逐步操作，每步都写明**做什么、应看到什么、学到什么**。总时长约 2–3 小时（可分几天完成）。

### 阶段一：巡检两台单元（约 15 分钟）

- [ ] **Overview（`/`）**：确认两张卡片状态点为绿色（在线），指标条在每 2s 跳动；页头显示 `2 Sparks · 2/2 online`
- [ ] **逐台点进详情页**：核对 GPU 面板（使用率/温度/功耗/进程表）、内存面板（host 类型看 RAM 与 VRAM 分开）、存储、网络
- [ ] **确认 LLM 卡**：应显示你的**模型 id**、引擎标签 **vLLM**、端口号；Decode tok/s 空闲时为 0 或很低是正常的
- [ ] **跑一次 Connectivity Test**（Edit Spark → Test）：SSH / 硬件 / LLM 逐项应 pass
- [ ] （可选）侧边栏拖动两台单元排个顺序，体会热配置

> **学到**：知道每个面板"正常时长什么样"——这是以后发现异常的基准。

### 阶段二：制造负载，观察实时指标（约 30 分钟，最有价值）

**1. 先拿到模型 id 和端口**：LLM 卡上直接有；或在目标机上：

```bash
curl http://127.0.0.1:<端口>/v1/models
```

**2. 打一条流式请求**（在任意能访问该服务器的机器上执行；用你部署时的端口，vLLM 默认 8000）：

```bash
curl http://<目标IP>:<端口>/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model":"<模型id>","messages":[{"role":"user","content":"写一篇500字的科幻短文"}],"max_tokens":800,"stream":true}'
```

**3. 同时盯着 sparkDash，依次观察**：

- [ ] LLM 卡 **Decode tok/s** 从 0 跳到实际生成速度
- [ ] 趋势图出现一个"驼峰"，请求结束后回落
- [ ] Overview 卡片出现 `1 running` 队列芯片
- [ ] **GPU 使用率拉高、显存中 KV cache 部分上涨**
- [ ] 多开 2–3 个终端同时打请求 → **Waiting 开始堆积、Aggregate tok/s 上升、单流速度下降**（并发换吞吐）
- [ ] 活动页（`/activity`）可能记录了相关事件

> **学到**：建立"请求 ↔ 指标"的因果直觉：什么在涨、涨多快、什么时候回落。这是监测人员的核心手感。

### 阶段三：Decode 基准 —— 两台横向对比（约 20 分钟）

- [ ] 侧边栏 **Benchmarks → Decode**，顶部 **Run on** 选第一台
- [ ] Type 选 **Prose**（或 Structured），Concurrency 勾 **1、2、4、8**，Max tokens 保持 400 → *Run benchmark*
- [ ] 跑完 **Copy results** 保存结果；换第二台**同样参数**再跑
- [ ] 对比两列结果：**Aggregate tok/s**（总吞吐）与 **Stream tok/s**（单流速度）

**怎么读结果**：

| 观察 | 结论方向 |
|------|----------|
| 单台的 Aggregate 随并发增长 | 并行批处理生效，硬件还没到顶 |
| 并发 8 时 Aggregate 反而降 | 过载：KV/调度成为瓶颈，记下"甜点并发" |
| 两台单流速度差很多 | 模型/量化/硬件不一致，找出更快的那台 |

> **注意**：bench 会真实打满 GPU，**别在正在给别人提供服务的实例上跑**；bench 期间的正常流量体验会变差。

### 阶段四：Prefill 基准 —— 长上下文能力（约 15 分钟，大上下文耗时更久）

- [ ] **Benchmarks → Prefill**，选单元
- [ ] 第一轮先勾小档位：**1k、4k、8k、16k** → 运行（每档输出 prefill tok/s + TTFT）
- [ ] 换第二台跑同样档位
- [ ] 两台曲线对比：TTFT 随上下文的**增长斜率**谁更平缓

> **学到**：判断"这台机器适合处理多长的文档/对话"。提示：128k–300k 档位可能要跑几十分钟，初期别勾。

### 阶段五：Quality 基准 —— 回答质量（约 30 分钟起）

- [ ] **Benchmarks → Quality**，选单元
- [ ] **新手建议先勾 2–3 个小类**（如 QA(150) + Arithmetic(40)），避免全量 755 请求的时长
- [ ] **Run label** 填能区分这次运行的标签（如 `baseline`），Concurrency 默认 4 → 运行
- [ ] 第二台用**相同类别**跑，Run label 填 `另一台的名字`
- [ ] 用 **Compare with** 做配对比较：看总分差 + **McNemar p 值**（p ≥ 0.05 显示 *difference within noise* = 差异只是噪声）

> **学到**：**速度 ≠ 质量**。两台速度相当时，用 Quality 才知道谁的答案更对；p 值告诉你差异是否"算数"。

### 阶段六：统计与演示页面（约 20 分钟）

经过阶段二的压测后：

- [ ] **Token totals（`/tokens`）**：应看到刚才生成的 token 计入 *Generated*；试 **Group by Model / Spark** 切换、**Copy CSV** 导出
- [ ] **Fleet energy（`/energy`）**：若在 Settings 填了电价，看能耗成本；记住**估算值**的定位
- [ ] **Activity（`/activity`）**：按严重度过滤，回看刚才各阶段发生过什么
- [ ] **Showcase**：LLM 卡 → **Showcase** 按钮，Terminals 设 4–8、Run，体验多路并发流式演示的视觉效果（适合投屏展示）

### 阶段七：收尾与日常巡检（长期习惯）

**一次性收尾**

- [ ] Settings：按需调整 Refresh rate（两台远程 SSH 建议保持 **2s**，1s 会显著加大 SSH 负载）、主题、显示开关
- [ ] 确认 `.env` 与 `BIND_HOST`/token 策略符合你的访问方式（见第十三章）
- [ ] 备份 `config/` 目录（含 `sparks.json`、`sparks-secrets.json`、`.secrets-key`）

**每日巡检清单（30 秒）**

1. Overview 两个状态点都是绿？
2. 告警条（Settings 开 *Show active fleet exceptions*）没有新条目？
3. 空闲时段 GPU 温度/显存是否正常？
4. Token 曲线是否在按预期增长？

**出问题时**：先看顶部横幅（stale / 断线）→ 对照第十二章告警表 → 仍不行翻第十四章 FAQ。

---

## 附：日常操作速查

| 我想… | 怎么做 |
|-------|--------|
| 快速验证指标是否正常 | 发一条流式 curl 请求，盯 LLM 卡 Decode tok/s（第十六章·阶段二） |
| 对比两台机器谁更快 | Decode / Prefill 基准，同参数各跑一遍 |
| 添加一台机器 | 侧边栏 `+` → 三步向导 |
| 快速跳到某页/某单元 | Ctrl/⌘ K → 搜索 |
| 换主题 | 侧边栏主题按钮循环 / ⌘K *Switch theme* / Settings 色板 |
| 一键启动模型 | Overview 卡片 ModelLauncher → **Start** |
| 测某模型生成速度 | LLM 卡 → **Decode** → *Run benchmark* |
| 比较两个量化质量 | Quality 基准 → 勾选类别 → *Compare with* 历史运行 |
| 停掉单元上的 ComfyUI 任务 | Spark 详情 → ComfyUI 卡 → **Cancel** |
| 批量升级 Hermes | Overview → **Update Hermes** |
| 关机整舰队 | Overview → **Shut down all** → 勾选 + 输入 `poweroff` |
| 远程访问面板 | SSH 隧道，或 `BIND_HOST` + `SPARKDASH_TOKEN` |
| 导出统计数据 | Token/Energy/Activity 页的 **Copy CSV** / **Copy** |

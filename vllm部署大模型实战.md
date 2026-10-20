# vLLM 部署大模型实战：命令参数详解

## 一、示例命令

```bash
CUDA_VISIBLE_DEVICES=0,3 vllm serve /nas_data/models/Qwen/Qwen3-32B \
  --served-model-name Qwen3-32B \
  --tensor-parallel-size 2 \
  --port 6663 \
  --trust-remote-code
```

## 二、环境变量部分

### `CUDA_VISIBLE_DEVICES=0,3`

- 限制进程只能看到 0 号和 3 号 GPU（物理卡），其他卡对本进程不可见。
- 作用：**指定用哪两块卡来部署**，同时避免抢占机器上其他任务的 GPU。
- 注意：设置后 vLLM 内部看到的卡编号会重映射为 `0,1`，与后面 `--tensor-parallel-size 2` 配合。

## 三、命令主体

### `vllm serve <模型路径>`

- 启动 OpenAI 兼容的 API 服务（默认提供 `/v1/chat/completions`、`/v1/completions` 等端点）。
- `/nas_data/models/Qwen/Qwen3-32B`：本地模型目录，需包含 `config.json`、权重文件（safetensors）、`tokenizer` 等；也支持 HuggingFace 模型 ID（如 `Qwen/Qwen3-32B`）联网下载。

## 四、各选项详解

### `--served-model-name Qwen3-32B`

- 对外暴露的模型名（API 请求里 `model` 字段填这个名字）。
- 不设则用模型路径末尾目录名；设置后客户端调用时写 `model: "Qwen3-32B"`，与本地路径解耦，便于迁移路径后接口不变。

### `--tensor-parallel-size 2`（简称 `-tp`）

- **张量并行**：把模型每一层的权重切分到 2 张 GPU 上协同计算。
- 必须等于实际可用 GPU 数（这里是 2），否则启动报错。
- 32B 模型 FP16 约需 ~64GB 显存，单卡放不下或为了降低单卡延迟，故切到 2 张卡。
- 与之对应还有 `--pipeline-parallel-size`（流水线并行，按层切分），本命令未用。

### `--port 6663`

- 服务监听端口，默认 8000；这里改为 6663。
- 完整接口如：`http://<主机IP>:6663/v1/chat/completions`。

### `--trust-remote-code`

- 允许执行模型仓库 `tokenizer.py` / `modeling_*.py` 等自定义代码。
- Qwen 系列部分 tokenizer/模型实现依赖自定义代码，**不加通常会报错或 tokenizer 加载失败**。
- 安全提示：只对可信来源的模型开启（等于信任作者代码在你机器上执行）。

## 五、常用可选参数（按需追加）

| 参数 | 作用 |
|---|---|
| `--max-model-len` | 最大上下文长度，超长显存不够时调小它 |
| `--gpu-memory-utilization 0.9` | 显存占用比例上限（默认 0.9） |
| `--dtype auto` | 计算精度（auto/fp16/bf16/fp32；量化格式用 `--quantization` 指定） |
| `--api-key xxx` | 为 API 加鉴权 |
| `--enable-prefix-caching` | 前缀缓存，多轮对话/同系统提示加速 |
| `--served-model-name` 可重复传 | 一个服务暴露多个别名 |
| `--max-num-seqs 256` | 最大并发请求数，按吞吐/延迟调 |
| `--host 0.0.0.0` | 监听地址；默认即绑定所有网卡（0.0.0.0），限本机访问需显式 `--host 127.0.0.1` |

## 六、执行效果与验证

启动后服务监听 6663 端口，用 curl 测试：

```bash
curl http://127.0.0.1:6663/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model":"Qwen3-32B","messages":[{"role":"user","content":"你好"}]}'
```

启动日志中两个关键指标（用来判断资源是否够用）：

- `GPU KV cache size: 643,232 tokens` —— GPU KV 缓存可存储的 token 总数。
- `Maximum concurrency for 40,960 tokens per request: 15.70x` —— 并发请求数估计（token 数取自 `max_model_len`）；低于业务吞吐需求就加 GPU 或节点。

---

## 七、部署场景总览：并行策略如何选

> 参考：[vLLM 官方文档 - 并行与扩展](https://docs.vllm.com.cn/en/latest/serving/parallelism_scaling/)

vLLM 的分布式推理策略选型（官方推荐口径）：

| 场景 | 策略 | 参数设置 |
|---|---|---|
| 单 GPU | 无需分布式 | 模型放得下单卡即可 |
| 单节点多 GPU | 张量并行（TP） | `--tensor-parallel-size <GPU数>` |
| 多节点多 GPU | TP + 流水线并行（PP） | TP = 每节点 GPU 数，PP = 节点数 |
| 单节点但 GPU 数无法整除模型 | PP（按层切分） | `--tensor-parallel-size 1 --pipeline-parallel-size <GPU数>` |
| 单节点且卡间无 NVLink（如 L40S） | PP 优于 TP | 通信开销更低、吞吐更高 |

核心原则：**不断加 GPU/节点，直到显存足以容纳模型 + KV 缓存**。

- **张量并行（TP）**：每层权重按列/行切到多张卡，卡间每层都有集合通信（算法源自 Megatron-LM），要求高速互联（NVLink）。
- **流水线并行（PP）**：按模型层切分，不同卡算不同层，通信只发生在层边界，通信量小但有流水线气泡。
- **MoE 模型**：vLLM 支持「数据并行注意力 + 专家并行（EP）或 TP 的 MoE 层」组合，详见[数据并行部署](https://docs.vllm.com.cn/en/latest/serving/data_parallel_deployment/)。

**分布式运行时默认规则**：多节点默认用 Ray，单节点默认用 Python `multiprocessing`；可用 `--distributed-executor-backend` 显式覆盖（`mp` = multiprocessing，`ray` = Ray）。

## 八、场景一：单机单卡

模型单卡放得下时最简部署，不需要任何并行参数：

```bash
vllm serve /path/to/model \
  --served-model-name my-model \
  --gpu-memory-utilization 0.9 \
  --max-model-len 32768
```

要点：

- 显存不够优先调：`--max-model-len`（上下文长度直接吃 KV 显存）、`--gpu-memory-utilization`；或改用**已量化好的权重**（AWQ/GPTQ 等格式的 checkpoint，vLLM 自动识别，一般无需手动传 `--quantization`）。
- 用启动日志里的 `GPU KV cache size` 和 `Maximum concurrency` 确认能支撑的并发。

## 九、场景二：单机多卡

### 9.1 张量并行（TP）—— 首选

```bash
# 离线推理（Python API）
from vllm import LLM
llm = LLM("facebook/opt-13b", tensor_parallel_size=4)
output = llm.generate("San Francisco is a")
```

```bash
# 在线服务
vllm serve facebook/opt-13b \
  --tensor-parallel-size 4
```

对应本文开头的示例即此场景：`CUDA_VISIBLE_DEVICES=0,3` + `--tensor-parallel-size 2`。

### 9.2 TP + PP 组合（共 8 卡）

```bash
vllm serve gpt2 \
  --tensor-parallel-size 4 \
  --pipeline-parallel-size 2
```

### 9.3 单节点多卡场景的运行时选择

- 单节点默认用 `multiprocessing` 后端，无需额外安装。
- 如需显式指定：`--distributed-executor-backend mp` 或 `ray`（Ray 为可选依赖，需 `pip install "ray[cgraph]"`）。

## 十、场景三：多机多卡

### 10.1 前提条件

- 各节点执行环境必须一致（模型路径、Python 包版本），推荐用**统一容器镜像**屏蔽主机异构。
- 模型建议提前在每节点相同路径下载，或放共享分布式文件系统；否则需给容器传 `-e HF_TOKEN=<TOKEN>` 联网拉取。

### 10.2 方案一：Ray 集群（官方默认多节点方案）

**（1）搭建 Ray 集群**（使用官方辅助脚本 `examples/ray_serving/run_cluster.sh`）：

头节点：

```bash
bash run_cluster.sh \
  vllm/vllm-openai \
  <HEAD_NODE_IP> \
  --head \
  /path/to/the/huggingface/home/in/this/node \
  -e VLLM_HOST_IP=<HEAD_NODE_IP>
```

每个工作节点：

```bash
bash run_cluster.sh \
  vllm/vllm-openai \
  <HEAD_NODE_IP> \
  --worker \
  /path/to/the/huggingface/home/in/this/node \
  -e VLLM_HOST_IP=<WORKER_NODE_IP>
```

注意事项：

- 每个工作节点的 `VLLM_HOST_IP` 必须唯一。
- **安全提示**：`VLLM_HOST_IP` 必须使用私有网段地址——该网络流量未加密。
- 任一终端关闭即整个集群终止；节点间需 IP 互通。

**（2）验证集群**（进入任意容器执行）：

```bash
ray status
ray list nodes
```

**（3）在 Ray 集群上运行 vLLM**：集群资源对 vLLM 全部可见，只需在**单个节点**执行一条命令：

```bash
docker exec -it <container_name> /bin/bash   # 若 Ray 在容器内，先进容器
```

通用做法（2 节点 × 8 卡 = 16 卡，TP=每节点 GPU 数，PP=节点数）：

```bash
vllm serve /path/to/the/model/in/the/container \
  --tensor-parallel-size 8 \
  --pipeline-parallel-size 2 \
  --distributed-executor-backend ray
```

也可以 TP 直接等于集群 GPU 总数：

```bash
vllm serve /path/to/the/model/in/the/container \
  --tensor-parallel-size 16 \
  --distributed-executor-backend ray
```

### 10.3 方案二：multiprocessing 多节点（不依赖 Ray）

2 节点 × 8 卡，头节点（node-rank 0）：

```bash
vllm serve /path/to/the/model/in/the/container \
  --tensor-parallel-size 8 --pipeline-parallel-size 2 \
  --nnodes 2 --node-rank 0 \
  --master-addr <HEAD_NODE_IP>
```

工作节点（node-rank 1，headless）：

```bash
vllm serve /path/to/the/model/in/the/container \
  --tensor-parallel-size 8 --pipeline-parallel-size 2 \
  --nnodes 2 --node-rank 1 \
  --master-addr <HEAD_NODE_IP> --headless
```

两端参数必须一致（`--nnodes`、`--tp`、`--pp`），只有 `--node-rank` 不同；工作节点加 `--headless`。

## 十一、多机部署的网络优化

### 11.1 InfiniBand 高速互联

跨节点 TP 依赖快速节点间通信，首选 InfiniBand 等高速网卡。配置方式：给 `run_cluster.sh` 追加额外参数，例如：

```text
--privileged -e NCCL_IB_HCA=mlx5
```

（具体标志咨询系统管理员。）

### 11.2 启用 GPUDirect RDMA

GPUDirect RDMA 让网卡绕过 CPU/系统内存直读 GPU 显存，降低延迟与 CPU 开销。需要两项配置：

1. 容器安全上下文加 `IPC_LOCK` 能力（锁内存页、防换出到磁盘）；
2. 挂载 `/dev/shm` 提供 IPC 共享内存。

Docker 启动示例：

```bash
docker run --gpus all \
  --ipc=host \
  --shm-size=16G \
  -v /dev/shm:/dev/shm \
  vllm/vllm-openai
```

Kubernetes pod 规范片段：

```yaml
spec:
  containers:
    - name: vllm
      image: vllm/vllm-openai
      securityContext:
        capabilities:
          add: ["IPC_LOCK"]
      volumeMounts:
        - mountPath: /dev/shm
          name: dshm
      resources:
        limits:
          nvidia.com/gpu: 8
        requests:
          nvidia.com/gpu: 8
  volumes:
    - name: dshm
      emptyDir:
        medium: Memory
```

### 11.3 验证 RDMA 是否生效

用详细 NCCL 日志启动：

```bash
NCCL_DEBUG=TRACE vllm serve ...
```

判定日志关键字：

- `[send] via NET/IB/GDRDMA` —— GPUDirect RDMA 生效（高效）。
- `[send] via NET/Socket` —— 走原始 TCP，跨节点 TP 不高效。

## 十二、场景选型速查

| 场景 | 首选方案 | 关键命令/参数 |
|---|---|---|
| 单卡装得下 | 单卡 + 量化 + 调 `max-model-len` | `vllm serve <model>` |
| 单机多卡（有 NVLink） | TP | `--tensor-parallel-size N` |
| 单机多卡（无 NVLink / GPU 数不整除） | PP | `--tensor-parallel-size 1 --pipeline-parallel-size N` |
| 单机超大 | TP + PP 组合 | 如 `TP=4 --pipeline-parallel-size 2` |
| 多机多卡 | Ray 集群：机内 TP + 机间 PP | `run_cluster.sh` + `--distributed-executor-backend ray` |
| 多机多卡（不想装 Ray） | multiprocessing | `--nnodes N --node-rank R --master-addr IP` |
| MoE 大模型 | DP attention + EP/TP | 参见官方「数据并行部署」「专家并行部署」文档 |

其他参考文档：

- [分布式部署故障排查](https://docs.vllm.com.cn/en/latest/serving/distributed_troubleshooting/)
- [数据并行部署](https://docs.vllm.com.cn/en/latest/serving/data_parallel_deployment/)
- [专家并行部署](https://docs.vllm.com.cn/en/latest/serving/expert_parallel_deployment/)

---

## 十三、专有名词详解

### 13.1 并行与分布式

#### 张量并行（TP, Tensor Parallelism）

- **一句话**：把模型**每一层内部**的大矩阵（张量）切开，分到多张 GPU 上同时算。
- **原理**：神经网络的核心运算是矩阵乘法。例如一个 `[hidden_size × intermediate_size]` 的权重矩阵，可以按列切成两半，两张卡各算一半结果，最后把输出拼起来（或做一次 all-reduce 求和）。
- **特点**：
  - 切分粒度最细（层内），**每层算完都要跨卡同步一次**，对卡间通信带宽极其敏感。
  - 因此 TP 几乎总是用在**同一台机器内**（走 NVLink），跨机器做 TP 会被网络延迟拖死。
  - 要求 GPU 数能被注意力头数（num_attention_heads）等整除，否则无法均分。
- **对应参数**：`--tensor-parallel-size N`。
- **类比**：8 个人一起抄一本书，每人负责每一页的 1/8，但每抄完一页都要互相核对一遍——核对（通信）非常频繁。

#### 流水线并行（PP, Pipeline Parallelism）

- **一句话**：把模型**按层切开**，不同的 GPU 负责不同的层，像工厂流水线一样接力计算。
- **原理**：例如 48 层模型切到 2 张卡，卡 0 算第 1~24 层，卡 1 算第 25~48 层。数据先在卡 0 前向传播，把中间结果传给卡 1 继续算。
- **特点**：
  - 通信只发生在**层与层的边界**（切分点），频率远低于 TP，对网络带宽要求低，**适合跨机器**。
  - 缺点是**流水线气泡**（见下文）：前半段时间只有前段的卡在忙，后段空转，GPU 利用率打不满。
  - 常与 TP 组合：机内 TP + 机间 PP，把高频通信留在机内。
- **对应参数**：`--pipeline-parallel-size N`。
- **类比**：做菜流水线，一人切菜一人炒菜——切菜时炒菜的闲着，炒菜时切菜的闲着，除非两道工序完全咬合。

#### 流水线气泡（Pipeline Bubble）

- PP 的固有浪费：流水线启动阶段（前段卡先算、后段卡等待）和收尾阶段（前段卡空闲、后段卡还在算），这部分时间 GPU 没有满负荷工作，称为"气泡"。
- 节点越多、每卡层数越少，气泡占比越大。缓解方法包括 micro-batch（把一个大请求拆成多个小批次在流水线上错峰流动）等，vLLM 内部自动处理。

#### 专家并行（EP, Expert Parallelism）

- 专用于 **MoE（混合专家）模型**的切分方式。MoE 模型的 FFN 层由多个"专家"（独立的小 FFN 网络）组成，每个 token 只激活其中少数几个专家。
- EP 把不同的专家放到不同的 GPU 上：token 被路由到哪个专家，就发到对应 GPU 上计算，卡间用 all-to-all 通信交换 token。
- 优点是利用了专家之间天然的独立性，通信模式与模型结构匹配；大规模 MoE（如 DeepSeek-V3、Qwen3-MoE）部署时常用「DP attention + EP」组合。
- 详见官方「专家并行部署」文档。

#### 数据并行（DP, Data Parallelism）与 DP attention

- **经典 DP**：每张卡都持有**完整的模型副本**，各自处理不同的请求，只在反向传播时同步梯度（推理场景不需要同步梯度，所以推理的 DP 几乎零通信开销）。用途是**提高吞吐**，不是解决单卡放不下的问题。
- **DP attention**：vLLM 对 MoE 模型的优化组合——注意力部分用数据并行（每个 DP 副本独立处理自己的请求序列），MoE 专家部分用 EP 或 TP。这样注意力的 KV Cache 可以在副本间独立分配，扩展性更好。
- 与 TP 的本质区别：DP 是"多份完整模型各干各的"，TP 是"一份模型拆开一起干"。

#### MoE（Mixture of Experts，混合专家模型）

- 一种模型架构：FFN 层由多个并行的"专家"网络组成，前面加一个路由器（router/gate），为每个 token 动态选择少数几个专家（如 8 选 2）来激活。
- 代表模型：DeepSeek-V3/R1、Qwen3-MoE、Mixtral、DBRX 等。
- 特点：总参数量很大（知识容量大），但每个 token 实际计算量（激活参数）相对小。部署时显存要装下全部专家，但计算和通信模式稀疏，因此需要 EP 等专门的并行策略。

#### Megatron-LM

- NVIDIA 2019 年提出的超大模型训练框架/论文（arXiv:1909.08053），首次系统性给出了张量并行（层内矩阵切分）的高效实现；流水线并行的系统方案来自其后续工作。vLLM 的 TP 算法直接源自该工作，是业界事实标准。

#### NCCL（NVIDIA Collective Communications Library）

- NVIDIA 的 GPU 集合通信库，提供跨 GPU/跨机器的 all-reduce、all-gather、all-to-all、broadcast 等通信原语。
- vLLM 的 TP/PP 底层通信依赖 NCCL；跨节点时 NCCL 走 TCP 或 RDMA。
- 相关环境变量：`NCCL_IB_HCA`（指定用哪块 RDMA 网卡）、`NCCL_DEBUG=TRACE`（打印详细通信日志）、`NCCL_SOCKET_IFNAME`（指定走哪块网卡）等。
- 排查多机性能问题时，看 NCCL 日志里数据走的是 `NET/IB/GDRDMA`（高效）还是 `NET/Socket`（低效）。

#### 集合通信（Collective Communication）

- 多个 GPU 协同完成的通信操作统称，常见原语：
  - **all-reduce**：每张卡都得到所有卡数据的聚合结果（如求和）。TP 中每层结束时最常用。
  - **all-gather**：每张卡都收集到所有卡上的分片数据（如拼接完整的输出）。
  - **all-to-all**：每张卡把不同数据发给不同卡（EP 中路由 token 用）。
  - **broadcast**：一张卡把数据发给所有卡。
- TP 每层都要做集合通信，所以带宽直接决定 TP 效率。

#### 分布式运行时 / executor backend

- vLLM 需要把"调度器的指令"分发到多个 GPU/多个节点的 worker 进程上执行，这个底层执行框架就是"分布式运行时"。
- vLLM 支持两种：
  - **`mp`（multiprocessing）**：用 Python 多进程实现，零额外依赖，单节点和简单多节点可用。
  - **`ray`**：用 Ray 分布式框架实现，多节点的默认方案，支持跨机调度、容错。
- 通过 `--distributed-executor-backend mp|ray` 显式指定；默认规则：单节点用 mp，多节点用 ray。

### 13.2 硬件与互联

#### NVLink

- NVIDIA 私有的 GPU 间高速直连互联技术，GPU 之间可以互相直接读写显存。带宽远高于 PCIe，例如 H100 SXM 的 NVLink 4 每 GPU 双向共 900 GB/s（单向 450 GB/s，由 18 条 25 GB/s 链路经 NVSwitch 聚合）。
- 作用：TP 的每层 all-reduce 都要跨卡传数据，NVLink 提供足够带宽才能让 TP 接近线性加速；没有 NVLink（只有 PCIe）时 TP 性能会大幅下降。
- 服务器内 8 卡高端卡的 **SXM 版**（A100/H100/H200 SXM）通常经 NVSwitch 全互联 NVLink；PCIe 版、消费卡和部分推理卡（如 L40S）卡间没有 NVLink，此时官方建议用 PP 替代 TP。

#### PCIe（Peripheral Component Interconnect Express）

- 通用高速总线标准，GPU 与 CPU、GPU 与 GPU 之间通信的"普通公路"。带宽远低于 NVLink（PCIe 4.0 x16 约 32 GB/s/方向，全双工双向各享带宽）。
- 跨 GPU 大量数据交换时 PCIe 是瓶颈，所以 TP 尽量走 NVLink。

#### InfiniBand（IB）

- 面向数据中心的高性能低延迟网络，专为 HPC/AI 集群设计，支持 RDMA。
- 多机多卡训练/推理的首选互联网络；带宽常见 200/400 Gbps。跨节点 TP/PP 的性能很大程度取决于 IB 网络。
- 对应的网卡称为 **HCA**（Host Channel Adapter），驱动设备名常为 `mlx5`（Mellanox 系列），所以 NCCL 配置里常见 `NCCL_IB_HCA=mlx5`。

#### RDMA（Remote Direct Memory Access）

- 远程直接内存访问：允许一台机器的网卡**绕过对方的 CPU 和操作系统内核**，直接读写对方的 GPU 显存/内存。
- 效果：延迟极低（微秒级）、CPU 零开销、带宽打满。
- 与普通 TCP 网络的区别：TCP 收发数据要经过双方 CPU 的协议栈处理；RDMA 把这些全部卸载到网卡硬件上。

#### GPUDirect RDMA

- RDMA 的扩展：网卡不仅绕过 CPU 内存，还能**直接读写 GPU 显存**，数据路径为「GPU 显存 ⇄ RDMA 网卡」，完全不经过主机内存。
- 收益：跨节点传输大块数据（如 TP 的激活值、PP 的中间结果）延迟更低、CPU 开销更小。
- 启用条件：
  1. 容器加 `IPC_LOCK` capability（锁定内存页防止换出）；
  2. 挂载 `/dev/shm`。
- 验证方法：`NCCL_DEBUG=TRACE` 启动，日志出现 `[send] via NET/IB/GDRDMA` 即生效；出现 `[send] via NET/Socket` 说明还在走低效 TCP。

#### NCCL 环境变量速查

| 变量 | 作用 |
|---|---|
| `NCCL_IB_HCA` | 指定使用哪块（哪些）RDMA 网卡，如 `mlx5` |
| `NCCL_DEBUG=TRACE` | 打印详细通信日志，排查走 IB 还是 Socket |
| `NCCL_SOCKET_IFNAME` | 指定 TCP 通信使用的网络接口 |
| `NCCL_P2P_DISABLE=1` | 禁用 GPU 间 P2P 直连（排查问题时用） |
| `NCCL_IB_DISABLE=1` | 禁用 IB，强制走 TCP（排查问题时用） |

#### IPC_LOCK

- Linux capability（能力）之一：允许进程锁定内存页（mlock），防止其被换出到磁盘。
- GPUDirect RDMA 要求锁定通信缓冲区，保证网卡访问的显存/内存始终在物理内存中，所以容器需要加上这个 capability。

#### `/dev/shm`（共享内存）

- Linux 的内存文件系统（tmpfs），存在于 RAM 中，是进程间共享大块数据最快的方式。
- NCCL/vLLM 的多进程通信会用到共享内存；容器默认 `/dev/shm` 只有 64MB，多卡场景必须扩大（Docker：`--shm-size=16G`；K8s：emptyDir 挂载 Memory 介质）。
- 太小会导致 NCCL 初始化失败或通信变慢。

#### HCA（Host Channel Adapter）

- InfiniBand 网卡的正式名称（对应以太网的 NIC）。软件中枚举为 `mlx0`、`mlx5_0` 等设备名，`NCCL_IB_HCA=mlx5` 即指定使用 Mellanox/Mellanox-NVIDIA 系列 IB 网卡。

### 13.3 显存与性能

#### KV 缓存（KV Cache）

- **是什么**：LLM 生成时，之前 token 的 Key/Value 注意力结果会被缓存下来，后续生成新 token 时无需重算历史部分——这是自回归推理加速的核心机制。
- **代价**：极其吃显存。每生成一个新 token，就要为它存一份 KV；上下文越长、并发请求越多，KV Cache 占用越大，往往**远超模型权重本身**。
- **vLLM 的创新**：PagedAttention——像操作系统虚拟内存分页一样管理 KV Cache，消除碎片、支持跨请求共享前缀（配合 `--enable-prefix-caching`）。
- **文档中的 `GPU KV cache size: 643,232 tokens`**：权重加载后，剩余显存最多能缓存约 64 万个 token 的 KV，这就是实际能支撑的上下文×并发总量的上限。

#### 上下文长度（max_model_len）

- 单个请求能处理的最大 token 数（输入 + 输出）。
- 直接影响 KV Cache 需求：`max_model_len` 设为 32768 时，每个请求最坏情况要占 32768 个 token 的 KV 空间；设小可以腾出显存给更多并发。
- 显存不够时的第一调优手段：`--max-model-len` 调小。

#### 显存占用比例（gpu-memory-utilization）

- vLLM 启动时按这个比例预分配显存（默认 0.9），即给模型权重 + KV Cache 共留出 90% 的显存，剩下 10% 留给 CUDA 上下文、激活值等。
- 单独一台机器只跑一个 vLLM 实例时可以保持默认；同卡跑多个实例时需调小各实例的比例。

#### 量化（Quantization）

- 把模型权重（有时也包括激活值/KV Cache）从高精度（FP16/BF16，16 位）压缩到低精度（INT8/INT4/FP8 等），从而：
  - **显存减半甚至减到 1/4**，小卡能跑大模型；
  - 计算更快（新卡对低精度有硬件加速）；
  - 代价是精度轻微损失。
- 常见格式：
  - **AWQ**：激活感知权重量化，INT4 权重 + FP16 激活，主流 4bit 方案。
  - **GPTQ**：训练后量化，同样 INT4 权重为主。
  - **FP8**：H100 等新卡原生支持，精度损失更小。
  - **BitsAndBytes（bnb）**：INT4/INT8，适合实验，推理性能一般。
- vLLM 对已量化模型通常自动识别（`--quantization auto`），无需手动指定。

#### 吞吐（Throughput）与延迟（Latency）

- **吞吐**：单位时间能处理多少请求/token（如 tokens/s、请求/s）。衡量系统"总共干了多少活"。
- **延迟（首 token 延迟 TTFT / 每 token 间隔 TPOT）**：单个请求从发出到收到响应要多久。衡量"单个用户体验"。
- 两者通常矛盾：vLLM 用连续批处理（continuous batching）在高并发下最大化吞吐，但单请求延迟可能上升。调优时先明确业务指标是吞吐优先还是延迟优先。
- 文档中 `Maximum concurrency ... 15.70x`：当前配置下约能同时跑 15~16 路并发请求，这是吞吐能力的直观指标。

#### 并发（Concurrency）

- 同时在服务中处理的请求数。vLLM 会把多个请求的 token 拼进同一个 batch 里 GPU 并行计算（连续批处理），只要有空位就动态插入新请求。
- 上限由 KV Cache 容量决定：`GPU KV cache 总 token 数 ÷ 平均每请求 token 数 ≈ 最大并发`。
- `--max-num-seqs` 可以手动设置并发上限（默认 256，通常让 KV Cache 自然决定即可）。

#### Prefill 与 Decode（预填充与解码）

- LLM 推理的两个阶段：
  - **Prefill（预填充）**：把输入 prompt 一次性并行读入，计算所有输入 token 的 KV Cache。计算密集（算力 bound），决定首 token 延迟（TTFT）。
  - **Decode（解码）**：逐个生成新 token，每步都要读全部 KV Cache。访存密集（带宽 bound），决定生成速度（TPOT）。
- 两个阶段特性完全不同，所以衍生出「分离式部署」（Prefill 和 Decode 用不同实例，分别优化）等进阶方案。
- TP 在 Prefill 阶段效率高（矩阵大），Decode 阶段效率低（每步算的矩阵小、通信占比高）——这也是多卡部署时需要权衡的点。

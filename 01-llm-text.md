# 01 · LLM 文本生成

全部为 llama.cpp（CUDA13 预编译，`-ngl 99 -fa on`）或 FreeToken 实测。llama-bench 口径：pp512 / tg128。

## Qwen3.8-27B（稠密 27.3B，Mamba+线性注意力混合架构）

### 量化档对比（GPU 空闲，q8 KV，-c 8192）

| 档位 | 文件大小 | 显存占用 | pp512 | tg128 | tg512 | 备注 |
|------|---------|---------|-------|-------|-------|------|
| UD-Q3_K_XL | 12.52GB | 13.29GB | 666 t/s | 28.3 t/s | 28.4 t/s | |
| **UD-IQ3_XXS** ✅ | 11.10GB | 11.96GB | **865 t/s** | 28.8 t/s | 29.6 t/s | prefill 快 30%，质量盲评无差异 |
| Uncensored IQ2_M fused | 10.6GB | 13.5GB | 777 t/s | 33.7 t/s | — | tg 比 IQ3_XXS 快 17%（省带宽），PPL 最糊档 |
| Uncensored i1-IQ3_M ✅日用 | 11.89GB | 12.9GB（含 mmproj） | — | 28.4 t/s 基线 | — | MTP 头内置 blk.64 |

+mmproj 视觉投影器（0.87–0.93GB）后显存各 +1.14GB 左右；Q3_K_XL+视觉 14.43GB 接近顶格。

### MTP 投机解码（--spec-type draft-mtp）

| 配置 | code | chat | prose | 说明 |
|------|------|------|-------|------|
| IQ2_M + MTP n=2 | 55.5 t/s (+70%) | 47.1 (+44%) | 35.1 (+8%) | 基线 32.6 |
| i1-IQ3_M + MTP n=2 | 47.8 (+68%) | 40.1 (+41%) | 31.7 (+11%) | 基线 28.4 |

- 拆分 draft 模型（noMTP + draft-Q8_0）在 16GB 卡上**反而比 fused 内置慢**——448GB/s 带宽受限卡多载一个 3.2GB drafter 不划算（H200 上结论相反）。
- n=3 退化；n=1 时 prose 最好（39.8）。思考型生成 MTP 收益最低（prose 场景）。

### 长上下文

- 解码掉速真实存在：短对话 27 t/s → 1169-token 上下文 ~13–14 t/s（混合架构在 llama.cpp 上的固有现象）。
- KV 缓存 q8→q4_0：长文本检索无质量损失、速度相当、省 0.5GB → 长文本直接 q4 KV。
- 128k ctx（q4 KV）：加载后 14586 MiB，6.5 万 token 输入跑通；256k 预计 ~15.8GB 超显存未试。

### llama.cpp 版本影响（i1-IQ3_M，同机同参）

| 版本 | tg128 | pp512 |
|------|-------|-------|
| b10092 | 29.02 | 931.9 |
| b10472 | 29.05 | 932.1 |
| **b10566（=v0.2.0）** | 29.11 | 935.5 |

普通路径 ±0.5% 全是噪声——"v0.2.0 提速 42%"不成立；**MTP 路径确有小幅真实改进**：b10435 ≈35.4 → b10566 ≈38.4 t/s（+6~9%）。

## 其他 GGUF 模型（llama.cpp）

| 模型 | 量化/大小 | 速度 | 备注 |
|------|----------|------|------|
| Muse-Glimmer-30B（Meta agentic 多模态） | Q3_K_XL 13.36GB | 文本 ~25.6 t/s | DFlash 投机解码 **15.4 t/s 净负收益**——Meta 官方 3.1x 加速是在 5090(1792GB/s) 上测的，448GB/s 卡验证开销 > 收益 |
| Ornith-1.5-9B（Qwen-VL 架构） | Q4_K_M 5.63GB | 66 t/s | 16s 视频 38 帧抽帧推理 38.8s |
| Ornith-1.5-35B-A3B-APEX-MTP-I-Mini | 14.4GB MoE | 挂视觉 45–73 t/s；**纯文本卸载 mmproj 后 56–77 t/s** | I-Mini 档 MTP 头随主体降级，接受率仅 10–28%；纯文本别挂 mmproj（更快且知识问答更准） |
| Qwen3-Omni-30B-A3B Q4（音频理解） | --n-cpu-moe 16 分流，15.6GB 显存 | 13 t/s | 音乐听感分析主力 |

## FreeToken v0.1.2（WSL2，MoE offload 后端）

FlashML 的边缘原生 MoE 引擎，专家驻留内存 + GPU LRU 缓存，OpenAI/Anthropic 双协议。

| 配置 | 结果 |
|------|------|
| Qwen3.6-35B-A3B FP8（35.8GB 权重） | 解码稳态 **27–29 t/s**（即本机 FP8 上限）；prefill ≈1500–1600 tok/s 线性 |
| 同款换 NVFP4 权重（22.4GB） | hybrid 后端解锁（CPU 算 58% + PCIe 取 42%）→ 30.1 t/s，仅 **+7%**（miss 占比小），省 13.4GB 内存 |
| Gemma-4-26B-A4B NVFP4（18GB，A4B 无思考链直出） | ~29 t/s 持平 |
| FTW 预转换权重 | 装载 2.5min → **15s** |
| 上下文扩展（KV reserve 131072） | 解码 16K=28.2 / 60K=26.2 / 100K=25.5 t/s（深上下文仅 -10%，GDN 架构 KV 只有 ~2.7GB） |
| radix 前缀缓存（默认开） | 20K prompt 冷 13.3s → 重发 4.8s（cached 99%） |

FreeToken 不支持投机解码/MTP；稠密 27B 模型上不了 FreeToken（fused 全驻显存，NVFP4 检查点 18.8–21.9GB > 16GB 卡预算）。

### 平台瓶颈标定（ft bench bw）

- CPU STREAM 内存带宽 23 GB/s
- PCIe H2D 6.8 GB/s —— 正好是 **Gen3 x8 平台上限**（i10400 无 Gen4）
- 结论：MoE offload 路线的解码天花板就在 ~29–30 t/s，换更大模型不再有速度空间

## 关键避坑

1. **显存被占时别测速**：部分层落 CPU 曾测出假 8.1 t/s（正常 28+）；跑 llama-bench 前确认 GPU 空闲。
2. cu128 下同配置两次跑可差 2.5 倍（allocator 分配运气），比较前先重启清显存。
3. Windows 服务加载期 `/health` 假阳性返回 200 但请求 503，等日志出现 `listening on` 再打请求。

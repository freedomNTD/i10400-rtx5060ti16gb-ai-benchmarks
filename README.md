# AI 模型本地实测基准 · i5-10400 + RTX 5060 Ti 16GB

> 仓库即数据集名：`i10400-rtx5060ti16gb-ai-benchmarks`。消费级单卡 Windows / Linux 双系统平台上各类本地 AI 模型的**实测性能数据集**：LLM 文本生成、文生图、视频生成、TTS/音乐。所有数字均为本机真实跑测记录（2026-08 起；2026-10 增补 Windows/Linux 同条件对比），覆盖 llama.cpp、FreeToken、ComfyUI（MiniMax-H3 / LTX / Qwen-Image 等）、多家 TTS 引擎。

## 测试平台

| 部件 | 规格 |
|------|------|
| CPU | Intel Core i5-10400（6C12T，仅 PCIe Gen3） |
| 内存 | 128GB DDR4-2400 |
| GPU | **NVIDIA GeForce RTX 5060 Ti 16GB**（Blackwell，sm_120，约 448GB/s 显存带宽） |
| GPU 驱动 | NVIDIA 621.01（Windows）/ 615.71.09（Mint，615 分支） |
| 系统 | Windows 11 IoT LTSC 24H2 + Linux Mint 22.3（Ubuntu 24.04.5，kernel 7.0）双系统；Windows 侧另有 WSL2（Ubuntu 22.04） |
| 推理框架 | llama.cpp（CUDA13 官方预编译，b10373/b10435/b10566）· FreeToken v0.1.2（WSL）· ComfyUI v0.42.5 多实例（torch 2.9.1+cu130 / comfy-kitchen 0.2.39；旧数据含 v0.33.x + torch 2.8+cu128） |

## 目录

| 文件 | 内容 |
|------|------|
| [01-llm-text.md](01-llm-text.md) | LLM 文本生成：Qwen3.8-27B 各量化档、MTP 投机解码、MoE（35B-A3B FreeToken）、长上下文衰减、投机解码翻车案例 |
| [02-image-gen.md](02-image-gen.md) | 文生图：Qwen-Image-2512 步数/量化 A/B、Qwen-Image-2.1 Windows/Linux 40 步对比、Z-Image-Turbo 兼容性、Anima-2.9B、注意力后端收益 |
| [03-video-h3-ltx.md](03-video-h3-ltx.md) | 视频生成：MiniMax-H3 速度基准与分辨率×帧数边界地图、六版 LoRA 横评、量化路线对比；LTX-2.5 |
| [04-tts-audio.md](04-tts-audio.md) | TTS/音频：MOSS/IndexTTS/FireRedTTS3 RTF 对比、音乐分析管线耗时横评 |

## 总览速查（亮点数字）

| 模型/配置 | 指标 |
|-----------|------|
| Qwen3.8-27B i1-IQ3_M + MTP2（llama.cpp） | chat 40.1 t/s（基线 +41%） |
| 同款关闭思考 UD-IQ3_XXS | tg ≈ 29 t/s / pp512 865 t/s |
| Qwen3.6-35B-A3B FP8（FreeToken offload） | 解码稳态 27–29 t/s，prefill ≈1500 t/s |
| Z-Image-Turbo bf16 9步 1024² | **28s/张**（质量盲评输给 Qwen-2512） |
| Qwen-Image-2512 fp8 1344×896 16步 | 101s/张（盲评最佳平衡点） |
| Qwen-Image-2.1 int8 40步 1024²（Win/Mint 同 150W） | **22.645s vs 22.64s（打平）**；Mint 默认 180W 21.55s |
| MiniMax-H3 R2V 640×640×124帧×3参考图 | 11.77 s/it |
| MiniMax-H3 640p×209帧 25步原生 | 37min/段（生产上限帧数） |
| LTX-2.5 distilled Q4_K_M 768×512×97帧 3步 | 4.7 s/it，峰值可低至 6.4GB |
| FireRedTTS3 Base 克隆 | RTF 3.08 / 显存 14.1GB |
| IndexTTS-2.5 克隆 | RTF 1.3–1.5（比 MOSS 快一倍） |

## 方法与口径说明（重要）

- 单卡家用环境、日常生产中的真实记录，**不是严格控制变量的实验室基准**；A/B 测试一般同 prompt 同 seed，速度为单次或少量重复计时。
- 画质结论来自「VLM 盲评」（qwen3.8-27B 本地判图）或用户人眼终判，标注在各表内。
- 视频帧数边界数据的关键指标是**换页窒息判别**（功耗掉到 ~34W 平底 + 显存顶满 + 进度停滞），本卡的有效边界大量依赖该签名。
- llama-bench 数字均在 GPU 空闲时采集——显存被占时部分层落 CPU 会测出假速度（实测过假 8.1 t/s）。
- Windows/WDDM 显存不自动释放：每次跑分前 taskkill 所有推理进程并确认空闲显存，否则同配置波动可达 2.5 倍。
- 跨系统图像对比：同一批模型文件、同一份 ComfyUI 源码/脚本、同 prompt/seed、同 150W 功耗墙；主指标为 ComfyUI 服务端 execution 时间，墙钟含 1s 轮询误差；同 seed 输出相关 1.0000 时视为同管线。

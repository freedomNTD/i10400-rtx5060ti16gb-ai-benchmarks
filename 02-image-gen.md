# 02 · 文生图 / 图像编辑

ComfyUI v0.33.x–v0.42.5（torch 2.8+cu128 / 2.9.1+cu130），RTX 5060 Ti 16GB。画质判定 = 本地 VLM（Qwen3.8-27B 视觉）盲评或人眼终判。

## Qwen-Image-2512 fp8 —— 步数阶梯 A/B

条件：1344×896，Lightning LoRA，shift13，同 prompt 同 seed。**步数决定一切：速度和质量都只由步数决定，4-step/8-step LoRA 身份完全无影响**（交叉组合盲评两两平局证实）。

| 步数 | 单张耗时 | 峰值显存 | 盲评结论 |
|------|---------|---------|---------|
| 25（默认档） | 135s avg（portrait 199s） | 15890 MiB | 质量基准 |
| **16 ✅日用最佳平衡** | **101s**（比 25 步快 25%） | ~15700 MiB | 盲评 4:0 全胜 8 步 |
| 8 | 54s（再快一倍） | ~15700 MiB | 细节糊、结构僵，只配出草图 |

## fp8 safetensors vs GGUF

同 prompt 同 seed，1344×896 / 16 步 / Lightning：

| 加载方式 | 耗时 | 峰值显存 | 质量 |
|----------|------|---------|------|
| fp8 原生（UNETLoader） | 117s | 15718 MiB | 持平 |
| GGUF Q5_K_M（15GB，UnetLoaderGGUF） | 187s（**慢 60%**） | 15049 MiB（-0.7G） | 盲评持平 |

结论：质量无差但 fp8 走原生 tensor core 快得多；GGUF 只在小显存或多模型同挂场景有价值。

## Qwen-Image-2.1 int8 —— Windows 11 vs Linux Mint（原生 40 步 1024p）

条件：Qwen-Image-2.1 base int8（`qwen_image_2.1_int8_convrot` + `qwen3vl_8b_int8_convrot`），原生完整管线，**不挂 BlockCache/Spectrum**；1024×1024、40 步、cfg 1.0、euler/simple、`--use-ck-attention`；同 prompt、同 seed 43–46；预热 1 次不计。两边使用同一批模型文件、同一份 ComfyUI 源码与 bench 脚本。

| 环境 | 功耗墙 | 4 次热态（墙钟） | 服务端中位数 | GPU 采样 |
|------|-------:|-----------------|-------------:|----------|
| Windows 11 IoT LTSC 24H2 / 驱动 621.01 | 150W | 23.26–23.33s | **22.645s** | p95 150.2W / SM 2797MHz / 69.2°C |
| Mint 22.3 / 驱动 615.71.09（默认） | 180W | 22.05s ×4 | **21.545s** | p95 180.1W / SM 2797MHz / 74°C |
| Mint 22.3 / 驱动 615.71.09（限 150W） | 150W | 23.05s ×4 | **22.64s** | p95 150.1W / SM 2632MHz / 70°C |

软件：ComfyUI v0.42.5（两边同源）、torch 2.9.1+cu130、comfy-kitchen 0.2.39；Windows 驱动 621.01，Mint 驱动 615.71.09（Ubuntu 源内 615 分支；NVIDIA 官网 NFB 615.78.08 未进源）。

**结论**

- **同 150W 打平**：Windows 服务端中位数 22.645s，Mint 22.64s，差 0.02%，在噪声内。墙钟 Mint 看起来快 0.25s，主要是脚本 1s 轮询/队列误差；服务端 execution 时间更准。
- **默认 180W 的优势来自功耗墙**：Mint 21.545s 比 Windows 150W 的 22.645s 快 4.86%，不是 OS 本身快 5%。
- **升级没有提速**：0.39.1→0.42.5 后 Windows 基本不变（-0.3%）；Mint 从 0.39.1+595.99.02 的 22.47s@150W 变成 0.42.5+615.71.09 的 22.64s@150W，慢约 0.76%；180W 下也从 21.41s 变成 21.545s（-0.63%）。从结果看，0.42.5 的改动没有改善这条 int8/ck-attention 热路径。
- **质量一致**：两边同 seed 输出均 1024×1024，跨系统相关系数 1.0000、平均绝对差 0.11–0.14/255，基本逐像素一致，说明对比的是同一模型/同一管线。
- 冷启动参考：Windows 50.3s、Mint 47.1s（含模型加载，单次，不作主指标）。

## Z-Image-Turbo 在 sm_120 上的兼容性实录

| 版本 | 结果 |
|------|------|
| bf16 基线 | ✅ 9步/1024²/cfg=0 = **28s/张**（cpu-offload）；但盲评 4:0 输给 Qwen-2512 fp8 的 135s 档 |
| unsloth FP8.pt | ❌ torchao rowwise FP8 序列化，sm_120 `torch._scaled_mm` 不支持 rowwise scaling；强改 tensorwise 可跑但 10min/张（21 倍慢的回退路径） |
| unsloth INT8.pt | ❌ pickle 依赖 torchao 0.18 已删除的类，无降级路径 |

快速模型用速度换质量的典型样本：28s vs 101s，质量差肉眼可辨。

## 其他图像模型

| 模型 | 配置 | 耗时 | 备注 |
|------|------|------|------|
| Qwen-Rapid-AIO v23 fp8（指令改图） | 8187 环境 | ~78s/张 | 改图默认；Q3_K GGUF 130s 档弃用 |
| Anima-2.9B 动漫 T2I（fp16） | 30 步 CFG4 euler+sgm_uniform | 832×832=31.4s ｜ 812×1216=45.5s ｜ 1152×1536=87.1s | ~2.2 万像素/秒线性；显存仅模型 5.57GB + TE 1.14GB |

## 注意力后端收益（comfy kitchen attention，int8 attention）

| 场景 | baseline（热） | + kitchen | 提速 |
|------|--------------|-----------|------|
| boogu 2real 图像管线 1536px/8步 | 72s/张 | 66s/张 | +8.3% |
| Qwen-Edit GGUF 896×1536/12步 | 130s/张 | 115s/张 | +11.6% |

图像场景收益（+8~12%）大于视频场景（约 +3%）。A/B 时注意 ComfyUI 输入哈希缓存——同 seed 同输入会 3 秒返回假结果。

## 平台注意事项

- RTX 5060 Ti 是 sm_120（Blackwell）：FP8 量化权重是否可用取决于序列化格式（rowwise scaling 不支持、tensorwise 可以），买卡前按此检查社区量化版。
- WDDM 显存不自动释放，换任务前 taskkill 确认空闲（跑分口径统一的前提）。

# 04 · TTS / 音频

RTX 5060 Ti 16GB，Windows。RTF = 生成耗时 / 音频时长（<1 为实时倍速以上）。

## 声音克隆 TTS 对比

| 引擎 | RTF | 显存 | 备注 |
|------|-----|------|------|
| **IndexTTS-2.5**（B站） | **1.3–1.5** ✅最快 | ~10GB 级 | 408 字 → 67–75s 音频；自带按 token 分段；长文案情绪参数丰富 |
| MOSS-TTS-Realtime | ~3.0（基准） | ~9.5GB | 音色克隆自然度当前最优（人耳评定）；Realtime 服务化 |
| FireRedTTS3 Base 克隆 | 3.08（与 MOSS 持平） | **14.1GB** | 杀手锏 = 24 语言+21 方言克隆 |
| FireRedTTS3 Instruct | 4.5 | 同上 | 文字直造音色 + 语义/声学编辑 |

听感结论：IndexTTS-2.5 快一倍但停顿/自然度细节未达 MOSS（多版 A/B 后保留备用）；FireRedTTS3 与 MOSS 无速度优势，定位方言/造音色特种兵。

- IndexTTS-2.5 的 torch 2.8.0+cu128 在 sm_120（Blackwell）正常推理。
- FireRedTTS3 踩坑警示：上游 autocast(bfloat16) 装饰器被误删后会落 fp16（上限 65504），激活值 ~38 万直接 step0 NaN → device-side assert，报错位置完全不相干。

## 音乐生成

| 模型 | 任务 | 实测 |
|------|------|------|
| MiniMax Music 3（词曲唱一体） | 30s 歌曲冒烟 | 全程 362s，峰值显存 9.9GB（8191 独立环境，cfg 必须=1） |

## 音乐风格分析（听歌反推 caption）管线横评

同一 30s 样本、ground truth 已知：

| 方案 | 耗时/首 | 准确度 |
|------|--------|--------|
| Qwen3-Omni-30B-A3B Q4 llama.cpp + librosa 硬指标组合 | **27s** ✅主力 | BPM/人声/语言/乐器/段落基本全对 |
| Step-Audio-2-mini transformers | 71 分钟 ❌深度挂机模式 | 六项全对但太慢 |
| Qwen2.5-Omni llama.cpp 单打 | 3s | 流派/乐器/BPM 全错（llama.cpp 音频实现差）→ 只作退路 |

速度差三个数量级，取舍就是准确度和实时性：生产用 Omni 组合方案，抽查处挂机跑 Step-Audio。

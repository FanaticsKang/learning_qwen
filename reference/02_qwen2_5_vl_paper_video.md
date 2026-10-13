# Qwen2.5-VL 技术报告:视频处理(原文摘录)

出处:**Qwen2.5-VL Technical Report**(arXiv:2502.13923)
章节:§2.1.1(视频编码器)、§2.1.2(原生动态分辨率与帧率)、§2.1.3(绝对时间对齐 M-RoPE)、§2.2.1(视频数据)、§3.3.4(视频理解评测)。

## §2.1.1 — 3D patch 与窗口注意力

> "we extend our approach to 3D patch partitioning. Specifically, we use 14×14 image patches
> as the basic unit ... **For video data, two consecutive frames are grouped together**,
> significantly reducing the number of tokens fed into the language model."

> "only four layers employ full self-attention, while the remaining layers utilize windowed
> attention with a maximum window size of **112×112 (corresponding to 8×8 patches)** ...
> windowed attention ... ensures that computational cost scales linearly with the number of
> patches rather than quadratically."

视频逐帧进入 ViT 时,窗口注意力把逐帧成本控制为线性 —— 这是长视频可行的关键之一。

## §2.1.2 — 原生动态分辨率 + 动态帧率

> "Qwen2.5-VL incorporates **dynamic frame rate (FPS) training** and **absolute time
> encoding**."

> "we introduce a novel and efficient strategy that **aligns MRoPE IDs directly with the
> timestamps** ... the model can understand the tempo of time through the **intervals
> between temporal dimension IDs**, without necessitating any additional computational
> overhead."

## §2.1.3 — 绝对时间对齐的 M-RoPE(相对 Qwen2-VL 的核心改进)

> "...**aligning the temporal component of MRoPE with absolute time** ... by leveraging the
> **intervals between temporal IDs**, the model is able to learn **consistent temporal
> alignment across videos with different FPS sampling rates**."

对 Qwen2-VL 局限的批评:

> "...the temporal position IDs in MRoPE were **tied to the number of input frames**, which
> did not account for the speed of content changes or the absolute timing of events."

即:视频时间位置的间隔 = 真实秒数间隔(由 `second_per_grid_ts × tokens_per_second` 换算,
实现见 [03_transformers_video_mrope.md](03_transformers_video_mrope.md))。
Qwen-Drive 基座(Qwen3.5)的 M-RoPE 即属于这一支的延续。

## §2.2.1 — 训练时的视频数据

> "we **dynamically sampled FPS during training** to achieve a more evenly distributed
> representation of FPS."

## §3.3.4 — 评测时的帧数与 token 上限

> "we capped the maximum number of frames analyzed per video at **768**, with the total
> number of video tokens not exceeding **24,576**."

与 [qwen_vl_utils](qwen_vl_utils_vision_process.py) 源码常量对应:
`FPS_MAX_FRAMES = 768`、`FPS = 2.0`(默认采样帧率)、`VIDEO_MAX_TOKEN_NUM = 768`(每帧 token 上限)。

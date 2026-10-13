# Qwen3-VL 模型卡:视频相关特性(摘录)

出处:Hugging Face 官方模型卡 `Qwen/Qwen3-VL-8B-Instruct`(Qwen-Drive 基座 Qwen3.5 的直接同系)。

## 三项与视频直接相关的架构更新

**① Interleaved-MRoPE:**
> "In the distribution of positional encoding across the frequency spectrum, larger
> intervals are allocated to the **time** component than to h and w. This enhances the
> model's ability to capture long temporal dependencies, leading to **better long-video
> reasoning**."

相比 Qwen2.5-VL 固定三段切分(如 Qwen-Drive 的 `mrope_section=[11,11,10]`,t 段固定占低频),
Qwen3-VL 让 (t, h, w) 三轴**交错占用全频段**,时间轴不再被锁死在最低频。

**② DeepStack:**
> "Multi-level visual features are injected into the LLM across multiple layers, providing
> fine-grained ... details" —— 视觉 token 不再只在输入层注入,多层都补充视觉细节,
> 长视频/多帧场景下细节保留更好。

**③ Text–Timestamp Alignment(训练目标):**
> "trained ... with **text–timestamp alignment**, which strengthens ... **long video
> reasoning and precise video grounding**" —— 模型学会把文本事件锚定到视频的秒级时间戳。

## 视频输入的使用方式(模型卡代码,与 qwen_vl_utils 一致)

```python
messages = [
    {
        "role": "user",
        "content": [
            {"type": "video", "video": "file:///your/video.mp4",
             "fps": 2.0,           # 采样帧率;或 "nframes": 128
             "max_pixels": ...},
            {"type": "text", "text": "Describe this video."},
        ],
    }
]
```

## 与 Qwen-Drive 的关系

Qwen-Drive-1.0 的 VLM 基座(Qwen3.5-4B,`vision_config` 标识 `qwen3_5_vision`)保留
`temporal_patch_size=2` 与 M-RoPE 时间轴,说明视频通路是基座自带的;
但其 `mrope_section=[11,11,10]` 固定分段属于 Qwen2.5-VL 风格(而非 Interleaved 版),
规划流水线也没有接视频解码 —— 视频能力主要体现在 VQA/感知评测与训练数据(见
[06_qwen_drive_repo_video.md](06_qwen_drive_repo_video.md))。

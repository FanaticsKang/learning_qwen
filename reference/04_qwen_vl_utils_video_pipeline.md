# qwen_vl_utils 视频处理流水线导读(带注释)

配合同目录完整源码 [qwen_vl_utils_vision_process.py](qwen_vl_utils_vision_process.py) 阅读。
该库是 Qwen2.5-VL / Qwen3-VL 官方推理预处理库,`process_vision_info()` 产出的
`videos` 张量直接喂给 `processor(..., videos=...)`。

## 0. 输入格式

```python
messages = [{
    "role": "user",
    "content": [
        {"type": "video", "video": "file:///path/drive.mp4",
         "fps": 2.0,            # 可选:采样帧率(默认 2.0);也可改传 "nframes": 20
         "max_frames": 768,     # 可选:fps 模式下的帧数上限
         "total_pixels": ...},  # 可选:整段视频总像素预算
    ],
}]
```

## 1. 关键常量(源码 24–38 行)

```python
SPATIAL_MERGE_SIZE   = 2
VIDEO_MIN_TOKEN_NUM  = 128    # 每帧最少 token
VIDEO_MAX_TOKEN_NUM  = 768    # 每帧最多 token
FPS                  = 2.0    # 默认采样帧率(Qwen2-VL 论文即 2fps)
FRAME_FACTOR         = 2      # ★ 帧数必须是 2 的倍数 —— 对应 temporal_patch_size=2
FPS_MIN_FRAMES       = 4      # 最少 4 帧(凑满 2 个时间步)
FPS_MAX_FRAMES       = 768    # 最多 768 帧
MODEL_SEQ_LEN        = 128000 # 整段视频的序列长度预算(可用环境变量覆盖)
```

## 2. 解码:三种后端自动择优(389–400 行)

`torchcodec`(首选,快)→ `decord` → `torchvision` 兜底。
都把整个视频(可用 `video_start`/`video_end` 截段)解成帧,得到:

```python
video  # shape (T, C, H, W),T = 原始总帧数(如 30fps×60s = 1800)
video_fps  # 原始帧率
```

## 3. 采样:smart_nframes(144–181 行)—— 决定"取几帧"

```python
fps = ele.get("fps", FPS)                       # 默认 2.0
nframes = total_frames / video_fps * fps        # ★ 按时长换算:60s 视频取 120 帧
nframes = min(min(max(nframes, min_frames), max_frames), total_frames)
#            ↑ 夹在 [4, 768] 区间(再夹进总帧数内)
nframes = floor_by_factor(nframes, FRAME_FACTOR)   # ★ 向下取偶数!temporal_patch_size=2
# 若用户直接给 "nframes": 也会 round_by_factor 到偶数
```

两个"×2"都源自时间 patch 大小:
- **帧数必须为偶数** —— 否则最后 1 帧凑不满一个时间步(processor 会 pad,官方干脆在采样时就取偶);
- `max_frames=768` 上限本身也写成偶数。

## 4. 均匀取帧(317 行)

```python
idx = torch.linspace(start_frame, end_frame, nframes).round().long()
video = vr.get_batch(idx)      # (nframes, C, H, W) —— 等间隔抽帧,保留时序覆盖
```

## 5. 每帧预算:整段摊销(448–475 行)

与图像路径最大的差异之一:**一段视频的所有帧 resize 到同一个 (H,W)**,预算按帧摊:

```python
min_pixels    = VIDEO_MIN_TOKEN_NUM * factor²            # 每帧下限
total_pixels  = MODEL_SEQ_LEN * factor² * 0.9            # 整段预算(默认巨大,可被覆盖)
max_pixels    = max(
    min(VIDEO_FRAME_MAX_PIXELS,                          # 每帧上限(768 token 对应像素)
        total_pixels / nframes * FRAME_FACTOR),          # ★ 总预算÷帧数×2
    int(min_pixels * 1.05))
resized_h, resized_w = smart_resize(h, w, factor, min_pixels, max_pixels)
video = resize(video, [resized_h, resized_w], BICUBIC)   # 一次 resize 全部帧
```

`× FRAME_FACTOR(=2)` 的含义:预算虽按"每帧"表述,但一个 ViT 输入单元覆盖 2 帧,
等价于"每个时间步"的预算翻倍。帧数越多 → 每帧允许的分辨率越低(自动权衡)。

## 6. 进入 processor 之后的 patch 化(3D tube)

qwen_vl_utils 输出的 `(nframes, C, H, W)` 交给 transformers 的 image processor,
与图像同一条 patch 化代码,只是多一个时间维:

```
(T=nframes, 3, H, W)
  → reshape(3, T//2, H//16, 2, W//16, 2, 16, 16)      # (C, t, h块, t内, w块, w内, ph, pw)
  → 每个时空 patch = (3, 2, 16, 16) 展平 = 1536 维       # ★ 与图像路径 expand 后同维!
  → video_grid_thw = (T//2, H/16, W/16)
  → 模型内 2×2 空间合并 → token = (T//2) × (H/32) × (W/32)
```

**对照 `scene.py:240`**:`unsqueeze(1).expand(-1, 2, -1, -1)` 做的事,就是把"缺时间维的
单帧"补成与视频张量相同的 `(C, T=2, H, W)` 布局 —— 后续 reshape/permute 完全同一套代码。
区别只在 T 维的内容:图像 = 同一帧复制 2 份(零拷贝视图);视频 = 真实相邻 2 帧
(默认相隔 0.5 s,含运动信息)。

## 7. fps 元数据去哪了

`fetch_video` 返回 `sample_fps`,processor 换算 `second_per_grid_ts = temporal_patch_size / fps`,
传给 `get_rope_index` 把时间位置对齐到绝对时间(见 [03_transformers_video_mrope.md](03_transformers_video_mrope.md))。

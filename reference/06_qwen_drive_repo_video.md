# Qwen-Drive-1.0 仓库自身的"视频线索"(本地查证)

出处:本地 `2609.00111v1.pdf`(论文)+ `Qwen-Drive-1.0/` 仓库代码与文档。

## 论文里的证据

1. **图 2(输入示意)明确画了 "Driving videos"** 作为四类输入之一 —— 模型对外宣称支持视频输入。
2. **训练数据(§数据配比)**:Stage-2 混合数据中 *"Multi-view Video temporal 4.4%"*;
   留存统计:*"3.7% video samples"* + *"10.1% single-view temporal, 4.4% multi-view temporal"*。
   即模型确实用视频/时序样本训练过(VQA 与感知能力),但占比很小。
3. 论文附录示例的提问方式:*"What you are seeing are frames from a video sequence..."*
   —— 官方对"视频"的操作化定义就是**帧序列**。

## 代码里的证据(规划流水线 = 帧序列,不是视频)

`src/qwen_drive/scene.py`:

```python
# _patchify,240 行 —— 单帧复制凑时间维(视频接口的退化用法)
pixels = pixels.unsqueeze(1).expand(-1, self.temporal_patch_size, -1, -1)   # temporal_patch_size = 2

# encode_images,267 行 —— 每一帧都是独立"图像",时间 grid 恒为 1
grids.append((1, rows, cols))          # image_grid_thw 的 T=1
```

- 整个仓库**没有任何视频解码代码**(无 decord/torchcodec/ffmpeg 调用);
- 12 帧 = 3 视图 × 4 时刻(t-1.5s/‑1.0s/‑0.5s/0s),作为 12 张独立图片编码,
  时序关系靠 prompt 里的 `frame: k` 文本标签 + M-RoPE 的帧内恒定 t 表达;
- 每帧分辨率独立控制(历史 320p 档、当前 720p 档)—— 视频通路做不到这一点
  (qwen_vl_utils 强制整段视频统一分辨率)。

## data.md 的证据:视频在数据准备阶段就被拆成帧

`docs/data.md` §"Obtaining the camera frames" → NVIDIA PhysicalAI:

> "This dataset **ships videos rather than images**, so the frames have to be decoded.
> 1. Each clip has one video per camera, named `<clip_id>.<camera>.mp4`.
> 2. **Decode every frame of each video** ... and write frame n (zero-based) as `<clip_id>/<camera>/<n:08d>.jpg`"

官方给的命令就是 ffmpeg 逐帧出 jpg:

```bash
ffmpeg -i "$clip.$camera.mp4" -qscale:v 2 -start_number 0 "$root/$clip/$camera/%08d.jpg"
```

即:**基准工程链路里,"视频处理"发生在数据准备阶段(ffmpeg 前置拆帧),模型侧只吃帧序列**。
与 qwen_vl_utils 的"模型侧解码+采样"是两条互补的工程选择。

## 结论

Qwen-Drive 的视频能力 = 基座 VLM 自带(时间 patch=2 + M-RoPE 时间轴 + 训练数据里的视频样本),
但发布仓库的规划/评测流水线刻意走**多视图帧序列**路径;真要喂一个 mp4 文件,走的是
[04_qwen_vl_utils_video_pipeline.md](04_qwen_vl_utils_video_pipeline.md) 那条官方视频通路。

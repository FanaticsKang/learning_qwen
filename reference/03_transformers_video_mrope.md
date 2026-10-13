# transformers 源码:视频时间维的 RoPE 位置计算(get_rope_index)

出处:`transformers` v4.57.1 `src/transformers/models/qwen2_5_vl/modeling_qwen2_5_vl.py` → `Qwen2_5_VLModel.get_rope_index`(Qwen2-VL 同款结构,Qwen3-VL 演进版见 05 篇)。

## 视频分支 vs 图像分支(对照)

```python
if ed_image < ed_video:                # ---- 图像分支 ----
    t, h, w = image_grid_thw[image_index]      # 图像的 t 恒为 1
    second_per_grid_t = 0                       # ★ 图像:时间步长 = 0
    ...
else:                                  # ---- 视频分支 ----
    t, h, w = video_grid_thw[video_index]       # 视频:t = nframes / temporal_patch_size(=2)
    if second_per_grid_ts is not None:
        second_per_grid_t = second_per_grid_ts[video_index]   # ★ 一个 grid 步 = 多少秒
    else:
        second_per_grid_t = 1.0                                # 默认 1 秒
    ...
llm_grid_t, llm_grid_h, llm_grid_w = (
    t.item(),
    h.item() // spatial_merge_size,     # 空间合并 2×2 只作用于 h、w
    w.item() // spatial_merge_size,
)
```

## 时间位置 id 的计算(视频)

```python
range_tensor = torch.arange(llm_grid_t).view(-1, 1)          # 第 0 个时间步、第 1 个、…
expanded_range = range_tensor.expand(-1, llm_grid_h * llm_grid_w)
second_per_grid_t = torch.as_tensor(second_per_grid_t, ...)
time_tensor = expanded_range * second_per_grid_t * self.config.vision_config.tokens_per_second
time_tensor_long = time_tensor.long()
t_index = time_tensor_long.flatten()          # 视频时间位置 = 网格序号 × 秒/步 × 每秒 token 数

llm_pos_ids_list.append(torch.stack([t_index, h_index, w_index]) + text_len + st_idx)
```

图像分支同一公式里 `second_per_grid_t = 0` → **t_index 全 0**:同帧所有 token 时间号相同
(Qwen-Drive `image_grid_thw=(1, rows, cols)` 走的正是这一支)。

## `second_per_grid_ts` 是什么

前向签名中的说明:视频沿时间维**每个 3D 位置网格步对应的时间间隔(秒)**。
预处理器用 `temporal_patch_size / fps` 算出:默认 fps=2.0 时,一步 = 2 帧 / 2fps = **1 秒**。

`tokens_per_second` 来自 `vision_config`(Qwen2.5-VL 发布值为 25)。
`get_rope_index` 文档字符串中的原始例子:

> "with fps=1, tokens_per_second=25, temporal_patch_size=2, the interval is 25×2/1 = 50,
> so the video temporal positions advance as `[0, 0, 0, 0, 50, 50, 50, 50, 100, ...]`"

(每个时间步内 4 个 token = 1 个时间步 × 2×2 空间合并的 4 个 token。)

## 结论(与 Qwen-Drive 对照)

| | Qwen-Drive 图像路径(scene.py) | 视频路径(qwen_vl_utils + transformers) |
| --- | --- | --- |
| grid | `image_grid_thw = (1, rows, cols)` | `video_grid_thw = (nframes/2, rows, cols)` |
| 时间位置 t | 帧内恒定(步长 0 → 全 0) | `arange(grid_t) × second_per_grid_t × tokens_per_second`,沿绝对时间递增 |
| token 数 | `(rows/2) × (cols/2)` | `grid_t × (rows/2) × (cols/2)` |
| 语义 | "同一时刻" | 每步间隔真实秒数,模型能读出事件节奏 |

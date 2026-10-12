# 02 — 图2 中图像的尺寸、token 对应关系与 ViT 编码

> 问题:图2 中的图像多少尺寸,一个 token 对应多大,如何对图像进行的编码(ViT)?

## 1. 结论速览

| 项 | 值 | 依据 |
| --- | --- | --- |
| 原始相机图像 | 972×1079(W×H,demo 实测) | `data/demo/frames/*.jpg` |
| 尺寸策略 | **历史帧 → 320p 级,当前帧 → 720p 级**(显式 `target_size` 或像素预算) | 论文 2.3.1;`CameraFrame` 注释 scene.py:90 |
| 历史帧实际 ViT 输入 | **384×416**(W×H)→ **156 token** | demo 元数据 384×416 = 预算 174080px 的 smart_resize 结果(已验证) |
| 当前帧实际 ViT 输入 | **704×800**(W×H,由 720×799 吸附到 32 的倍数)→ **550 token** | `_patchify` + `smart_resize`,已用代码验证 |
| **1 个视觉 token = 32×32 像素** | patch 16×16 × 空间合并 2×2 | `image_patch_size=16`, `image_spatial_merge_size=2`(config 默认) |
| 整个 12 帧场景 | 9 历史×156 + 3 当前×550 = **3054 个视觉 token** | 推算并验证 |
| ViT 单个 patch 输入维度 | 3 通道 × 时间 2 × 16×16 = **1536** | `image_temporal_patch_size=2` |

## 2. 尺寸:动态分辨率 + 双档预算

论文 2.3.1:"We resize historical images to 320p and current images to 720p. This allocation
retains more spatial detail in the current observation while representing the motion history
with fewer visual tokens."(历史帧省 token,当前帧保细节)

代码实现两级机制(`QwenDriveProcessor._patchify`,scene.py:215):

1. **显式 `target_size`**(评测/基准场景必带,如 demo 的 384×416 与 720×799):先 bicubic 缩放到该尺寸,再吸附到 32 的倍数(720×799 → **704×800**,因为 720/32=22.5 取整到 22,799/32≈25 取整到 25);
2. **像素预算兜底**(自建场景无 target_size 时):`history_image_pixels=174080`、
   `current_image_pixels=921600`,由 `smart_resize` 把分辨率等比缩进预算并取整到 32 的倍数。
   用 demo 原图 972×1079 验证:历史预算 174080 → **恰好 384×416**(与基准元数据一致);
   当前预算 921600 → 896×992(比 demo 元数据更大,说明基准是按 720p 档刻意定的 target_size)。

## 3. 一个 token 对应多大:16×16 patch + 2×2 合并 = 32×32 像素

流程(scene.py:237-253):

- 缩放后图像按 **16×16** 切 patch,得 rows×cols 网格(384×416 → 24×26;704×800 → 44×50);
- ViT 输出后做 **2×2 空间合并(pixel-unshuffle 风格)**,相邻 4 个 patch 拼成一个 LLM token:
  `token_count = rows × cols // 4`;
- 即 **1 token ↔ 32×32 像素**。

| 帧 | ViT 输入 W×H | patch 网格 | 视觉 token |
| --- | --- | --- | --- |
| 历史(9 帧) | 384×416 | 24×26 | 24×26/4 = **156** |
| 当前(3 帧) | 704×800 | 44×50 | 44×50/4 = **550** |
| 合计 12 帧 | — | — | **3054** |

## 4. ViT 编码全流程

ViT 属于 Qwen3.5-4B VLM 原生组件(`vlm_config`,经 `AutoModelForImageTextToText` 加载,
modeling_qwen_drive.py:105),仓库不改它。完整链路:

```
PIL 图像 (RGB)
  │ ① resize:target_size 或 smart_resize 吸附到 32 倍数      [scene.py:215]
  │ ② 归一化:像素→[0,1],再 (x-0.5)/0.5 → [-1,1]             [scene.py:235]
  │ ③ 时间维复制:单帧沿 temporal 维 expand 成 2 份
  │    (image_temporal_patch_size=2;视频时则取相邻两帧)       [scene.py:240]
  │ ④ 切 patch 并展平:每 patch = 3×2×16×16 = 1536 维,
  │    排列为 (rows/2, cols/2, 2, 2, C, t, 16, 16) → [rows*cols, 1536]
  ▼
原生 ViT(Qwen3.5-VL vision tower,含 2D 位置编码,freeze/finetune 视训练阶段)
  │ ⑤ 每 16×16 patch → 一个 ViT 输出向量(共 rows×cols 个)
  ▼
空间合并 2×2(④的逆排列思想:相邻4 patch 通道维拼接)→ [rows*cols/4, 4*D_vit]
  ▼
投影层 → LLM hidden size
  │ ⑥ 替换 prompt 中对应位置的 <image> 占位 token
  │    (<vision_start> img…img <vision_end>,占位数量 = token_count)  [scene.py:296]
  ▼
LLM(Qwen3.5-4B,GDN+GQA 混合层):文本 token + 视觉 token 一起做多模态 RoPE
  (mrope_section [11,11,10],时间/高/宽三段位置)做自回归生成 / 缓存 K,V
```

关键点:

- **无固定分辨率**:Qwen-VL 系原生动态分辨率,输入尺寸跟随图像长宽比吸附到 32 网格,
  只有像素预算上限;所以"图2 的图像尺寸"不是单一值,而是 320p/720p 两档动态尺寸。
- **占位 token 数在预处理时就已确定**(`encode_images` 返回 `token_counts`,scene.py:255),
  prompt 中 `<image>` 的个数与 ViT 实际输出严格相等。
- **BEV 感知头复用两路特征**(论文 2.1):ViT 输出的浅层特征 F^v(进 LLM 之前)和
  图像 token 穿过整个 LLM 后的特征 F^m;Planning Expert 则只消费 LLM 各 GQA 层的
  post-RoPE K/V 缓存(每 4 层取 1 份,共 8 份,`layers_per_kv=4`)。
- 配置出处:`configuration_qwen_drive.py:126-130`(预算与 patch 参数)、
  `docs/model.md` "Images" 表;`grid_pixel_limit = 12800×32² ≈ 13.1M px` 是
  target_size 场景的硬上限(scene.py:204)。

## 5. 复算脚本

验证脚本(与本仓库 `smart_resize` 同逻辑,已跑通):

```
历史帧 972x1079 按预算174080  → ViT输入 384x416, 网格 26x24, token 156
当前帧 demo target 720x799    → ViT输入 704x800, 网格 50x44, token 550
每视图 3×156+1×550 = 1018;全场景 12 帧 = 3054 token;patch 输入维度 3×2×16×16 = 1536
```

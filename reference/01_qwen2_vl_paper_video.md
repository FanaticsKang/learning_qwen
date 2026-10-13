# Qwen2-VL 论文:图像与视频统一处理(原文摘录)

出处:**Qwen2-VL: Enhancing Vision-Language Model's Perception of the World at Any Resolution**(arXiv:2409.12191)
章节:§2.1.1(3D patches)、§2.1.2(M-RoPE)、§2.2(Unified Image and Video Understanding)、表 5。

## §2.1.1 — 3D patch(时间维从哪来)

> "For video inputs, Qwen2-VL extends the 2D patch processing to 3D, enabling the joint
> processing of pairs of consecutive frames.
> 3D patches extend the spatial patching to include a temporal dimension:
>
> Patch_t,i,j = Patch_{t:i×P+h:(i+1)×P, j×P:(j+1)×P}, t ∈ [1, ⌊2T_t/2⌋]
>
> where **two consecutive frames are considered jointly**. This extension enables the model
> to handle video data, adding a temporal dimension T to the patch sequence."

## §2.2 — 关键原句:图像复制成两帧(即 `expand(-1, 2, -1, -1)` 的出处)

> "To preserve video information as completely as possible, we sampled each video at
> **two frames per second**. Additionally, we integrated 3D convolutions (Carreira and
> Zisserman, 2017) with a depth of two to process video inputs, allowing the model to handle
> **3D tubes instead of 2D patches**, thus enabling it to process more video frames without
> increasing the sequence length (Arnab et al., 2021). **For consistency, each image is
> treated as two identical frames.** To balance the computational demands of long video
> processing with overall training efficiency, we dynamically adjust the resolution of each
> video frame, **limiting the total number of tokens per video to 16,384**."

三个要点:
1. 视频 ViT 用 **3D 卷积(时间深度=2)** 切 patch —— 即 ViViT 式 "tube" embedding,每个输入向量覆盖相邻 2 帧 × patch×patch 像素。
2. **"each image is treated as two identical frames"** —— 单张图复制两份凑齐时间维,与视频共用同一套权重。这正是 Qwen-Drive `scene.py:240` 那行 `unsqueeze(1).expand(-1, 2, -1, -1)` 的论文出处。
3. 视频按 **2 fps 采样**,token 预算按整段视频控制(16384)。

## §2.1.2 — M-RoPE 对视频的时间编号

> "For videos, which are treated as sequences of frames, **the temporal ID increments for
> each frame**, while the height and width components follow the same ID assignment pattern
> as images."

图像(grid_t=1)帧内时间号不变;视频每个时间 grid 步(= 2 帧)时间号 +1。
注:Qwen2-VL 的时间 id 只数"帧序号",不与真实秒数对齐 —— 这一局限由 Qwen2.5-VL 修复(见 02 篇)。

## 表 5 — 评测时的视频设置(数字)

| 设置 | 值 |
| --- | --- |
| Max Number of Frames per Video | **768** |
| Max Number of Tokens per Video | **16,384** |
| 采样方式 | 2 fps(每秒 2 帧) |

## 论文出处

- ViViT(tube embedding 原型):Arnab et al., "ViViT: A Video Vision Transformer", ICCV 2021。
- 3D 卷积(时间深度 2):Carreira and Zisserman, "Quo Vadis, Action Recognition? A New Model and the Kinetics Dataset"(I3D), CVPR 2017。

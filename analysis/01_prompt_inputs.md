# 01 — 图2 中的 "Prompt" 里输入了什么

> 问题:图2 显示输入包括 Observed Images(观测图像),那么对应的 prompt 输入了什么?

## 1. 结论概览

图2 的输入分为两条路径,**prompt(VLM 的文本输入)只覆盖其中一部分**:

| 图2 输入 | 去向 | 形式 |
| --- | --- | --- |
| Observed Images | VLM | `<image>` 占位 token,嵌入在 prompt 序列中 |
| Prompt(指令文本) | VLM | ChatML user turn 中的文本 |
| Noisy Trajectory | Planning Expert | 连续张量(不经过 VLM) |
| Time(flow time) | Planning Expert | 连续标量(AdaLN 条件) |
| Ego state | Planning Expert | 连续向量(速度/加速度/导航指令 one-hot) |

Planning 路径下,自车历史轨迹**同时**出现在两处:文本 prompt 里以数字列出(供 VLM 推理用),又以张量形式直接喂给 Planning Expert(`history` / `history_velocity` / `history_acceleration`)。

## 2. prompt 的完整结构(规划模式)

来源:`src/qwen_drive/scene.py` 的 `QwenDriveProcessor.build_input_ids()`(scene.py:280)。
序列是标准 ChatML 格式,实际 token 序列为:

```
<|im_start|>user
<FRONT VIEW>frame: 0<vision_start><img×N0><vision_end>frame: 1<vision_start><img×N1><vision_end>
            frame: 2<...>frame: 3<...>
<FRONT LEFT VIEW>frame: 0<...>frame: 1<...>frame: 2<...>frame: 3<...>
<FRONT RIGHT VIEW>frame: 0<...>frame: 1<...>frame: 2<...>frame: 3<...>
{instruction 文本}
<|im_end|>
<|im_start|>assistant
<|im_end|>            ← 规划模式:助手轮立刻闭合(训练时的格式);推理模式:留空让模型先写 reasoning
```

要点:

- **序列化顺序是 view-major(视图优先)**:先把一个相机视角的 4 个时间帧连续放完,再放下一个视角。论文 2.1 节解释:这样同一视角的连续观测在 token 序列中相邻,暴露时间变化,对动态环境下的控制很重要(对应论文公式中 planning 样本的布局 ℓ)。
- 视图标签 `<FRONT VIEW>` / `frame: 0` 等都是**普通词表 token**,不引入新的特殊 token、不改架构(论文 2.1 节 "Both tag types use ordinary vocabulary tokens")。全量 8 个视图方向标签:`<FRONT VIEW> <FRONT RIGHT VIEW> <RIGHT VIEW> <BACK RIGHT VIEW> <BACK VIEW> <BACK LEFT VIEW> <LEFT VIEW> <FRONT LEFT VIEW>`;本仓库 demo/评测用前 3 个(`CAMERA_VIEWS`,scene.py:45)。
- `<vision_start> ... <vision_end>` 之间填 `token_count` 个 `<image>` 占位 token(数量 = 该图 ViT 输出的视觉 token 数,见笔记 02)。

## 3. instruction 文本逐段拆解

模板 `_INSTRUCTION_HEADER`(scene.py:58)+ 历史轨迹行 + 导航指令。demo 数据
(`data/demo/planning_scenes.jsonl` 第 1 条)中的真实文本:

```
The input images are organized by camera view. Each view contains 4 temporal frames captured
at 0.5s intervals (frame 0 at t-1.5s, frame 3 is the current frame at t=0s).
1. Historical trajectories (x, y, heading) in the current frame's ego coordinate system.
   Positive x points forward, positive y points left, and a positive heading indicates a left turn：
 -t-1.5s: (0.0000, 0.0000, 0.0000);
 -t-1.0s: (0.0000, 0.0000, 0.0000);
 -t-0.5s: (0.0000, 0.0000, 0.0000);
 -t-0s: (0.0000, 0.0000, 0.0000);
2. Active navigation command: [GO STRAIGHT]
```

三段内容:

1. **图像组织说明**:每个视角含 4 个时间帧,间隔 0.5s,frame 3 是当前帧;
2. **历史轨迹**:4 个时刻的 `(x, y, heading)`(单位见轨迹约定,x 前进 / y 向左 / heading 左转为正),保留 4 位小数——`__post_init__` 特意不转 float32,以免打印的第 4 位小数和训练数据对不上(scene.py:132);
3. **导航指令**:三选一 `GO STRAIGHT / TURN LEFT / TURN RIGHT`(`NAV_COMMANDS`,scene.py:48)。

若为 REASONING_PLANNING 模式,还会追加一句(`REASONING_REQUEST`,scene.py:54):
`"\n\nGive a one-sentence brief reasoning of the ego's future driving decision ONLY."`

## 4. 不在 prompt 里、直接进 Planning Expert 的输入

`QwenDriveProcessor.__call__`(scene.py:368)返回的张量里,以下绕过 VLM 直接给专家网络:

| 张量 | 内容 | 维度/含义 |
| --- | --- | --- |
| `history` | 16 个历史位姿 (x,y,heading) | 归一化后经 Fourier/MLP 融入每个 waypoint token |
| `history_velocity` / `history_acceleration` | 历史速度/加速度 | 编码后拼进 waypoint token |
| `ego_status` | 速度 + 加速度 + 驾驶指令 one-hot(共 8 维,`ego_status_dim=8`) | 与 flow time、导航指令求和后经 AdaLN 注入每层 |
| `nav_command` | 整数指令 id | AdaLN 条件 |
| (噪声 + flow time) | 50 个带噪 waypoint、扩散时间步 | DiT 输入与 AdaLN 条件 |

## 5. VQA 模式的 prompt(对照)

`encode_vqa`(scene.py:318):图片直接排在问题前,无视图/帧标签、无历史轨迹,
可选 system 段:`<|im_start|>user [图][图]...问题<|im_end|><|im_start|>assistant`。
即通用 VQA 走 Qwen3.5 原生接口,不携带任何驾驶专用结构。

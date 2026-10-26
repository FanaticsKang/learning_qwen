# 01 · Lift-Splat-Shoot 事实底稿

> 本文件为 BEV 投影对比笔记的 LSS 分册，数字与结论的唯一来源是下方标注的 `path:line`（相对于 `bev_projection/code/lift-splat-shoot/`）与论文原文。后续 HTML 各节只引用本底稿，不重查代码。

## 1. 元信息

| 项 | 值 |
|---|---|
| 仓库 | https://github.com/nv-tlabs/lift-splat-shoot |
| commit hash | `2903467c91ee9c12f0917a12c22ab1f04e607ae0`（2024-10-14 "Make model weights publicly accessible"，已用 `git rev-parse HEAD` 复核，与 manifest 一致） |
| 克隆日期 | 2026-10-26（`--depth 1` 浅克隆，见 `00_manifest.md`） |
| 论文 | Philion & Fidler, *Lift, Splat, Shoot: Encoding Images from Arbitrary Camera Rigs by Implicitly Unprojecting to 3D*, ECCV 2020；arXiv:2008.05711**v1**（`papers/lss_2008.05711.pdf`）。v1 无附录——深度离散化仅有 §3.1（定义）与 §4.1（取值）两处描述 |
| nuScenes 配置口径 | **无 config 文件**。`main.py` 是 fire 入口（`main.py:13-20`），全部默认值在 `src/train.py::train()` 函数签名 `src/train.py:18-43`。LSS 为 nuScenes-native（`src/data.py:13` 直接用 nuscenes devkit），故 train.py 默认值即 nuScenes 口径：`H=900, W=1600`（`train.py:23`）、`resize_lim=(0.193, 0.225)`（`train.py:24`）、`final_dim=(128, 352)`（`train.py:25`）、`ncams=5`（`train.py:29`）、`xbound=[-50.0, 50.0, 0.5]`（`train.py:34`）、`ybound=[-50.0, 50.0, 0.5]`（`train.py:35`）、`zbound=[-10.0, 10.0, 20.0]`（`train.py:36`）、`dbound=[4.0, 45.0, 1.0]`（`train.py:37`）、`outC=1`（`train.py:67`） |
| 关键派生量（nuScenes 口径） | 深度格数 **D=41**（`torch.arange(4.0, 45.0, 1.0)` 左闭右开，中心 4.0–44.0 m）；图像特征网格 **fH×fW=8×22**（`ogfH//downsample, ogfW//downsample`，downsample=16）；上下文通道 **C=camC=64**（`models.py:144`）；BEV 网格 **200×200×1**（nx=(200,200,1)）；depthnet 输出通道 **D+C=105**（`models.py:46`）；每相机视锥点数 41×8×22=7216，6 相机共 43296 |

## 2. 机制流水线（每步标注【固定几何】/【可学习】）

输入：N 环视图 `(B,N,3,128,352)` + 每相机标定量 `intrins(K,3×3)`、`rots(3×3)`、`trans(3,)`（cam→ego，取自 `calibrated_sensor`，`src/data.py:135-138`）与增广校正量 `post_rots(3×3)`、`post_trans(3,)`（`src/data.py:132-133,151-154`）。

| # | 步骤 | 形状流（B·N 合并视角） | 属性 | 出处 |
|---|---|---|---|---|
| 0 | 标定/增广量准备：K、rot、trans 来自 nuScenes 标定表；post_rots/post_trans 由 resize/crop/flip/rotate 增广解析算出（训练期随机、验证期确定，但始终是几何量而非学习量） | — | 【固定几何】（逐样本数据） | `src/data.py:135-138`、`src/data.py:96-119`、`src/tools.py:120-144` |
| 1 | 视锥构造：`ds=arange(*dbound)` 得 41 个固定深度；`xs/ys=linspace(0, ogf−1, f)` 得 8×22 像素格；stack 成 `frustum(41,8,22,3)`，以 `nn.Parameter(requires_grad=False)` 冻结注册 | (41,8,22,3) | 【固定几何】（论文 §3.1 原文："there are no learnable parameters in this transformation"） | `src/models.py:153-164` |
| 2 | 图像主干：EfficientNet-B0（ImageNet 预训练），取 /16 端点（112ch, 8×22）与 /32 端点（320ch, 4×11） | (B·N,3,128,352)→(B·N,112,8,22)+(B·N,320,4,11) | 【可学习·2D 侧】 | `src/models.py:43`、`src/models.py:61-82` |
| 3 | 特征融合 `up1=Up(320+112,512)`：/32 上采样×2 与 /16 拼接（432ch）→卷积到 512ch @ /16 | →(B·N,512,8,22) | 【可学习·2D 侧】 | `src/models.py:45`、`src/models.py:15-34` |
| 4 | depthnet：1×1 卷积 512→105，前 41 通道为深度 logits、后 64 通道为上下文 c | →(B·N,105,8,22) | 【可学习·2D 侧】（α 与 c 均在此产生） | `src/models.py:46`、`src/models.py:51-59` |
| 5 | 深度分布：对 41 维 logits 做 `softmax(dim=1)` 得 α（每像素一个 41 类离散分布） | α:(B·N,41,8,22) | 【可学习·2D 侧】（分布是预测值；41 个格中心本身固定，见步骤 1） | `src/models.py:48-49`、`src/models.py:56` |
| 6 | Lift 外积：`new_x = depth.unsqueeze(1) * x[:,D:D+C].unsqueeze(2)`，即论文 Eq.(1) `c_d = α_d·c` | →(B·N,64,41,8,22) | 【桥·参数自由】（逐元素乘） | `src/models.py:57` |
| 7 | 几何反投影 `get_geometry`：先撤增广（`frustum − post_trans`、`inverse(post_rots)`），再 `K⁻¹` 反投影（`(u·d, v·d, d)`），最后 `combine = rots @ K⁻¹` 并 `+trans` 变换到 ego 系 | frustum(41,8,22,3)→(B,N,41,8,22,3) | 【固定几何·参数自由】（纯矩阵运算，无学习参数） | `src/models.py:166-186` |
| 8 | Splat `voxel_pooling`：体素量化 `((geom−(bx−dx/2))/dx).long()`→越界过滤（nx 范围内保留）→rank 编码+argsort→QuickCumsum 求和池化→griddify `(B,C,Z,X,Y)`→Z 折叠 | (B,N,41,8,22,64)→(B,64,1,200,200)→(B,64,200,200) | 【桥·参数自由】（scatter 求和，`use_quickcumsum=True`） | `src/models.py:200-242`、`src/tools.py:193-219` |
| 9 | BEV 编码器 BevEncode（resnet18 式：7×7 s2 conv → layer1-3 → 上采样×4 拼接 → ×2）：产出可行驶性/分割 logits | (B,64,200,200)→(B,1,200,200) | 【可学习·3D/BEV 侧】 | `src/models.py:90-126`、`src/models.py:252-255` |

监督仅在步骤 9 之后（BEV 二值分割图，vehicle 框栅格化，`src/data.py:171-193`；BCEWithLogits，`src/tools.py:222-229`）。深度 α **无任何直接监督**，只靠下游损失回传（论文称 implicit/latent depth）。

## 3. 可学习参数清单表（nuScenes 口径）

> 张量形状按 B=1、单相机列写（N 维在 batch 维合并）；"侧"指参数位于投影桥的哪一端。全模型可训练参数论文计 14.3M（§4.1）。

| 参数（组） | 所属模块 | 形状/通道（nuScenes 口径） | 侧 | 代码出处 `path:line` |
|---|---|---|---|---|
| `camencode.trunk`（EfficientNet-B0 全部卷积，ImageNet 预训练初始化） | CamEncode | 输入 (3,128,352)；本模型消费 /16 端点 112ch×8×22 与 /32 端点 320ch×4×11 | 2D 侧（lift 之前） | `src/models.py:43`、`src/models.py:61-82` |
| `camencode.up1`（Up(320+112, 512)：两组 3×3 conv+BN+ReLU） | CamEncode | 432→512 @ /16，输出 (512,8,22) | 2D 侧 | `src/models.py:45`、`src/models.py:22-29` |
| `camencode.depthnet`（1×1 conv，无 padding） | CamEncode | Conv2d 512→**105**（=D41+C64），输出 (105,8,22)：前 41ch→α logits，后 64ch→c | 2D 侧（α、c 的唯一来源） | `src/models.py:46`、`src/models.py:54-57` |
| `bevencode.conv1`+`bn1`（7×7 conv, stride 2 + BN） | BevEncode | 64→64 @ 200×200→100×100 | 3D/BEV 侧（splat 之后） | `src/models.py:95-97` |
| `bevencode.layer1/2/3`（resnet18 前三段，`pretrained=False`） | BevEncode | 64→64→128→256，空间 100→100→50→25 | 3D/BEV 侧 | `src/models.py:94`、`src/models.py:100-102` |
| `bevencode.up1`（Up(64+256, 256, ×4)：上采样×4 拼接 x1 + resnet block） | BevEncode | 320→256 @ 25×25→100×100 | 3D/BEV 侧 | `src/models.py:104`、`src/models.py:15-34` |
| `bevencode.up2`（上采样×2 + 3×3 conv 256→128 + 1×1 conv 128→outC） | BevEncode | 256→128→1，100×100→200×200 | 3D/BEV 侧 | `src/models.py:105-112` |

要点：**没有任何可学习参数位于投影桥上**——外积（`src/models.py:57`）、`get_geometry`（`src/models.py:166-186`）、`voxel_pooling`（`src/models.py:200-242`）三者均零参数。可学习量全部在 2D 侧（α 的产生与 c 的产生，`src/models.py:46`）与 BEV 侧（splat 后的 CNN，`src/models.py:90-126`）。

## 4. 固定几何清单表

| 项 | 具体值 | 出处 |
|---|---|---|
| 深度分箱 dbound | [4.0, 45.0, 1.0)→**D=41** 个格中心 4.0–44.0 m（步长 1.0 m；arange 左闭右开，45.0 不含） | `src/train.py:37`；`src/models.py:157` |
| 输入图像 final_dim | (128, 352)（ogfH=128, ogfW=352） | `src/train.py:25`；`src/models.py:155` |
| 特征下采样倍率 downsample | 16 → 特征像素格 fH×fW=**8×22** | `src/models.py:143`、`src/models.py:156`、`src/models.py:159-160` |
| 视锥 frustum | (41, 8, 22, 3)，`nn.Parameter(…, requires_grad=False)` 冻结 | `src/models.py:153-164` |
| BEV x 范围 xbound | [-50.0, 50.0, 0.5]→nx_x=200，dx=0.5，bx_x=−49.75（bin 中心对齐） | `src/train.py:34`；`src/tools.py:174-179`；`src/models.py:135-141` |
| BEV y 范围 ybound | [-50.0, 50.0, 0.5]→nx_y=200，dy=0.5，bx_y=−49.75 | `src/train.py:35`；`src/tools.py:174-179` |
| BEV z 范围 zbound | [-10.0, 10.0, 20.0]→**nx_z=1**（单层 pillar，dz=20 m，bz=0；z∈[−10,10) 全落同一格） | `src/train.py:36`；`src/tools.py:174-179`；`src/models.py:208`、`src/models.py:215-217` |
| dx/bx/nx 注册方式 | 三个均 `nn.Parameter(…, requires_grad=False)`——以参数形式注册但冻结，不进优化器梯度更新 | `src/models.py:139-141` |
| 相机内参 K | 每相机 3×3，取自 nuScenes `calibrated_sensor['camera_intrinsic']`（逐样本数据） | `src/data.py:136` |
| 相机外参 E | rot 3×3 + trans 3（cam→ego），同取自 `calibrated_sensor` | `src/data.py:137-138` |
| 增广校正 post_rots/post_trans | 2D 仿射（resize/crop/flip/rotate 的解析合成），嵌入 3×3 齐次形式；在 `get_geometry` 中先被逆掉 | `src/data.py:132-133,151-154`；`src/tools.py:120-144`；`src/models.py:175-176` |
| 体素量化与池化 | 索引 `((geom−(bx−dx/2))/dx).long()`、越界过滤、rank=x·(ny·nz·B)+y·(nz·B)+z·B+batch、argsort、cumsum-trick 求和、Z 折叠 `cat(unbind(dim=2))`——全部参数自由 | `src/models.py:208`、`src/models.py:215-217`、`src/models.py:222-227`、`src/models.py:230-240`；`src/tools.py:182-219` |
| QuickCumsum 开关 | `use_quickcumsum=True`（解析梯度的 sum-pooling 自定义 autograd） | `src/models.py:151`、`src/tools.py:193-219` |
| 相机配置 | cams 6 个（`CAM_FRONT_LEFT/FRONT/FRONT_RIGHT/BACK_LEFT/BACK/BACK_RIGHT`）；训练默认 `ncams=5` 随机抽 5/6，验证用全 6 | `src/train.py:57-59`、`src/train.py:29`；`src/data.py:195-201` |
| 数据增广（只影响 post_rots，不引入学习参数） | 原图 H=900,W=1600；resize_lim (0.193,0.225)；rot_lim (−5.4,5.4)；rand_flip=True；bot_pct_lim (0.0,0.22) | `src/train.py:23-28`；`src/data.py:96-119` |
| 输出头通道 outC | 1（BEV 二值分割） | `src/train.py:67` |
| camC（上下文通道 C） | 64（硬编码常量，非 config） | `src/models.py:144` |

## 5. 2D-3D 关系小结

- **推/拉方向：推（push）**。LSS 从 2D 图像侧出发，把每像素特征沿固定视锥射线"抬"到 3D（lift），再以纯 scatter 落进 BEV 网格（splat）。不存在 3D→2D 的采样/拉取路径，无任何 cross-attention。
- **深度：显式分布、隐式监督**。每个像素显式输出 41 维离散深度分布 α（softmax，`src/models.py:48-49`），格中心固定（4–44 m）；但 α 无深度标签，仅由 BEV 任务损失端到端回传（论文 §3.1 "latent depth"、§5.1 frozen-encoder 消融证明其学到了有效深度）。one-hot α 退化为 pseudo-lidar、均匀 α 退化为 OFT 式全程铺展（论文 §3.1 原文指出）。
- **几何与学习的分工**：全部可学习参数位于投影桥两端——2D 侧（EfficientNet-B0 + up1 + depthnet，负责"什么内容、放在多深"）与 BEV 侧（resnet18 式编码器）；中间的投影核（视锥分箱、K/E 反投影、体素 scatter 求和）零参数、纯固定几何。即"学习决定特征与深度权重，几何决定落点"。
- **query 有无及形态：无 query**。不存在可学习的 3D query / BEV embedding / position embedding；3D 位置全部来自固定视锥网格点（每相机 41×8×22=7216 个），BEV 格内容由落入该 pillar 的点求和得到（`src/models.py:236-240`）。这是与 BEVFormer（可学习 BEV query 拉取）对比时最根本的形态差异。

## 6. 论文 vs 代码差异点 + 待画图要点

### 6.1 差异与勘正记录

| # | 项 | 论文说法 | 代码事实 | 处理 |
|---|---|---|---|---|
| 1 | 机制图编号 | 任务线索称"LSS 图 2" | 论文 **Fig.2 是动机图**（传统图像帧预测 vs BEV 帧预测，素材引自 SegNet/NMP）；lift 机制图是 **Fig.3**，端到端流程图是 **Fig.4** | 重绘对象取 Fig.3 + Fig.4，不以 Fig.2 为机制底图 |
| 2 | 深度区间端点 | §4.1："D between 4.0 meters and 45.0 meters spaced by 1.0 meters" | `torch.arange(4.0, 45.0, 1.0)` 左闭右开→中心 4.0–44.0 m 共 41 个，45.0 **不含**（`src/models.py:157`） | 论文表述含糊而非矛盾；以代码为准：41 格、中心 4–44 m |
| 3 | 深度集示例公式 | §3.1 示例 `{d0+Δ, …, d0+|D|·Δ}`，按 d0=4、|D|=41、Δ=1 会含 45.0 | 同上，不含 45.0 | 同上，以代码为准 |
| 4 | 相机数量 | §5："camera rig … comprised of 6 cameras" | 训练默认随机用 5（`ncams=5`，`src/train.py:29`；`src/data.py:196-198`），验证用全 6（`src/data.py:199-201`） | 属训练策略差异，画图按 6 相机环视表达、脚注注明训练随机 5/6 |
| 5 | pillar"无限高度" | §3.2："Pillars are voxels with infinite height" | 用 zbound [-10,10,20] 单 z 格（nx_z=1）近似"无限高"，z∈[−10,10) 落同一格 | 画 BEV 网格时标 1 层 pillar（20 m 单格）而非无限柱 |
| 6 | 附录深度离散化 | （任务预期附录或有补充） | arXiv v1 到参考文献即止，**无附录**；深度离散化全部信息即 §3.1+§4.1 | 不再引用不存在的附录 |
| 7 | 可训练参数量 | §4.1："14.3M trainable parameters" | 仓库无复现脚本给出该数；结构一致（EffB0+resnet18 前三段）但未逐参数复算 | 底稿引用论文 14.3M，标注"论文值、未在代码侧复算" |
| 8 | 正样本权重 pos_weight | §5：对象分割 1.0、lane 5.0 | 代码默认 `pos_weight=2.13`（`src/train.py:31`），与论文任一任务值都不同 | 记录为任务/口径差异，不影响投影机制 |
| 9 | "nearest pillar"表述 | §3.2："assign every point to its nearest pillar" | bin 中心对齐的 floor 量化 `((x−(bx−dx/2))/dx).long()`（`src/models.py:208`），数学上等价（格中心即 bx+k·dx） | 非差异，绘图按"落入 0.5 m 格即求和"表达 |
| 10 | 骨干网络 | §4.1：EfficientNet-B0（ImageNet 预训练）；BEV 网 ResNet-18 前三层 + ×4 上采拼接 + ×2 上采 | `src/models.py:43`（from_pretrained）、`src/models.py:94-112` 逐层吻合 | 一致，无差异（列入表以示已核对） |

其余核对一致项：输入 128×352、BEV ±50 m/0.5 m/200×200、sum-pooling+cumsum trick（§4.2 ↔ `src/tools.py:193-219`）、Eq.(1) 外积（§3.1 ↔ `src/models.py:57`）。

### 6.2 待画图要点（Fig.3 + Fig.4 重绘规范）

- **Fig.3（lift 机制图）重绘**：
  - 保留构图元素：单像素出发的一条射线；沿射线的 41 个固定深度格（论文原图为点列）；α 的柱状分布（原图左侧）；context 向量 c（原图左上）；外积 α_d·c 后各深度点的特征强度（原图右侧，长度随 α 变化）。
  - 叠加编码（本笔记新增，原图没有）：① 固定几何元素用虚线/灰色（41 个格中心 4–44 m、8×22 像素格）；② 可学习元素用实色（α 分布、c）；③ 注记 D=41、C=64、格宽 Δd=1.0 m、特征格 8×22@/16。
  - 一句话构图意图：让读者一眼看到"格中心是死的、α 是学的"。
- **Fig.4（端到端流程图）重绘**：
  - 保留构图元素：n 幅输入图 + 每相机 K/E →（lift）每相机一个视锥点云 →（splat）公共 BEV 栅格 200×200 → BEV CNN → 分割/规划输出。
  - 叠加编码：三段颜色带——2D 侧可学习（image encoder+depthnet）、固定几何桥（frustum/K/E/scatter，零参数）、BEV 侧可学习（BevEncode）；形状流注记 (B,N,3,128,352)→(B·N,64,41,8,22)→(B,64,200,200)→(B,1,200,200)。
  - 一句话构图意图：呈现"可学习参数全部在两端、中间投影核零参数"的三明治结构——这正是四模型对比中 LSS 的定位。

# 03 · TPVFormer 事实底稿

> 本文件为 BEV 投影对比笔记的 TPVFormer 分册，数字与结论的唯一来源是下方标注的 `path:line`（相对于 `bev_projection/code/TPVFormer/`）与论文原文（arXiv:2302.07817 **v2** PDF，含附录 A–E）。后续 HTML 各节只引用本底稿，不重查代码。范围限定 nuScenes LiDAR 分割口径（`config/tpv_lidarseg.py`，occupancy=False）；SemanticKITTI SSC（`config/tpv04_occupancy.py`、`kitti_ssc/`）不在口径内，仅差异块引用。TPVFormer 为单帧模型，无时序模块。

## 1. 元信息

| 项 | 值 |
|---|---|
| 仓库 | https://github.com/wzzheng/TPVFormer |
| commit hash | `459bc060901c9c4920f802252f04b290a449e4a1`（2024-09-07 "Update readme.md"，已用 `git rev-parse HEAD` 复核，与 manifest 一致） |
| 克隆日期 | 2026-10-26（`--depth 1` 浅克隆，见 `00_manifest.md`） |
| 论文 | Huang, Zheng et al., *Tri-Perspective View for Vision-Based 3D Semantic Occupancy Prediction*, CVPR 2023；`papers/tpvformer_2302.07817.pdf`（arXiv v2，2023-03-02；正文 8 页+附录 A–E）；表示定义 §3.1 Eq.1–5 p.3–4、TPVFormer §3.2 Eq.6–12 p.4–5、应用 §3.3 p.5、实现 §4.2 p.7 + 附录 B p.9、消融 Table 4/5 p.8、Fig.2 p.2、Fig.3 p.3、Fig.4 p.5 |
| nuScenes 配置口径 | **`config/tpv_lidarseg.py`**（README 的 A100 主训练命令，`README.md:78`；`_dim_=128` 与论文 TPVFormer-Base（TPV 200×200×16、特征 128，附录 B p.9 + Table 4 p.8）一致，故取此为主口径）。同目录 `tpv_lidarseg_dim64.py`（`_dim_=64`，`tpv_lidarseg_dim64.py:13`）、`tpv_lidarseg_dim96.py`（96）为轻量变体，不引用。核心量：`point_cloud_range=[-51.2,-51.2,-5.0,51.2,51.2,3.0]`（`tpv_lidarseg.py:11`）、`_dim_=128`（`tpv_lidarseg.py:13`）、`num_heads=8`（`tpv_lidarseg.py:14`）、`_pos_dim_=[48,48,32]`（`tpv_lidarseg.py:15`）、`_num_levels_=4`（`tpv_lidarseg.py:17`）、`_num_cams_=6`（`tpv_lidarseg.py:18`）、`tpv_h_=200, tpv_w_=200, tpv_z_=16`（`tpv_lidarseg.py:20-22`）、`tpv_encoder_layers=5`（`tpv_lidarseg.py:26`）、`num_points_in_pillar=[4,32,32]`（`tpv_lidarseg.py:27`）、`num_points=[8,64,64]`（`tpv_lidarseg.py:28`）、`hybrid_attn_anchors=16`/`hybrid_attn_points=32`（`tpv_lidarseg.py:29-30`）、`num_points_in_pillar_cross_view=[16,16,16]`（`tpv_lidarseg.py:150`）、`nbr_class=17`（`tpv_lidarseg.py:34`） |
| 插件代码目录 | **`tpvformer10/`**——`occupancy=False`（`tpv_lidarseg.py:7`）→ `train.py:73-76` 选 `tpv_lidarseg_builder` → `from tpvformer10 import *`（`builder/tpv_lidarseg_builder.py:1`）。仓内另存旧版 `tpvformer04/`（被 SemanticKITTI 口径 `builder/tpv_occupancy_builder.py:1` 使用），两版模块同名不同实现；`encoder.py:150-151` 自注 "self attention in tpvformer04 … an older version. Now we use get_cross_view_ref_points instead"。本底稿一律引 `tpvformer10/` |
| 关键派生量（nuScenes 口径） | TPV query 数 **200×200 + 16×200 + 200×16 = 40000+3200+3200 = 46400**（@128 维，共 5.94M 参数，仅为同范围稠密 voxel 200×200×16=640000 格的 **7.25%**）；hw 格边长 102.4/200=**0.512 m**、z 格边长 8/16=**0.5 m**；输入 900×1600（nuScenes 原生，无 resize，`dataloader/dataset_wrapper.py:28-51`）pad 到 32 整除→928×1600（`dataset_wrapper.py:45` + `dataloader/transform_3d.py:19,38`）；FPN 四层 128ch @ /8、/16、/32、/64（`tpv_lidarseg.py:125-132`）；ICA 每查询每命中相机采样位置数：hw 8head×4level×8point=**256**、zh/wz 8 子查询×256=**2048**（见 §2 步骤 8）；CVHA 每查询 8head×3plane×32point=**768**（见 §2 步骤 7） |

## 2. 机制流水线（每步标注【固定几何】/【可学习】）

输入：`(B,6,3,928,1600)`（B=1/GPU，`config/_base_/dataset.py:15`；原图 900×1600 无 resize，光度增广+caffe 归一化+pad /32，`dataloader/dataset_wrapper.py:10-11,28-51`；GridMask 在模型内做，`tpvformer10/tpvformer.py:47-49,62-63` + `tpv_lidarseg.py:100`）。lidar2img 不受任何 2D 几何增广改写。

| # | 步骤 | 形状流 | 属性 | 出处 |
|---|---|---|---|---|
| 0 | 标定准备：`lidar2img`（每相机 4×4）= 内参 K 左乘 lidar→cam 外机（`sensor2lidar` 求逆构造），随 `img_metas` 下发 | 6×(4,4) | 【固定几何】（逐样本标定量） | `dataloader/dataset.py:79-94`（82-90 矩阵构造） |
| 1 | 图像主干 ResNet-101（stage3/4 DCNv2，FCOS3D 初始化，BN 冻结）：取 C3/C4/C5 | (B·6,3,928,1600)→512ch@116×200、1024ch@58×100、2048ch@29×50（/8、/16、/32，由 pad 后尺寸推算） | 【可学习·2D 侧】 | `tpv_lidarseg.py:114-124`；`config/_base_/schedule.py:3`；`tpvformer10/tpvformer.py:65` |
| 2 | FPN：4 层输出（`add_extra_convs='on_output'` 补出 /64），全部 128ch | →128ch@116×200、58×100、29×50、15×25 | 【可学习·2D 侧】 | `tpv_lidarseg.py:125-132`；`tpvformer10/tpvformer.py:70-77` |
| 3 | 展平+嵌入注入：每层特征 flatten 后加 `cams_embeds`（6,128）与 `level_embeds`（4,128），拼成 `feat_flatten` | →(6, 30825, B, 128)（30825=23200+5800+1450+375） | 【可学习·桥侧嵌入】 | `tpvformer10/tpv_head.py:96-113`（103-104 注入） |
| 4 | 三平面 TPV query 组装：三张可学习嵌入表 + 逐平面位置编码（编码本平面两轴、被忽略轴槽位置零） | hw (40000,B,128)、zh (3200,B,128)、wz (3200,B,128)；pos 同形状 | 【可学习·3D 侧 query】 | `tpvformer10/tpv_head.py:53-55,84-94`；`tpv_head.py:141-143,147-170` |
| 5 | 三平面 3D 锚点射线生成（固定网格，register_buffer）：每平面查询在其**正交轴**上均匀取 `num_points_in_pillar` 个归一化锚点——hw 沿 Z 取 4 个、zh 沿 X 取 32 个、wz 沿 Y 取 32 个（轴重排 permute 落到世界 x/y/z） | ref_3d_hw (1,4,40000,3)、ref_3d_zh (1,32,3200,3)、ref_3d_wz (1,32,3200,3)，取值 [0,1]³ | 【固定几何】（无参数 linspace 网格；论文 Eq.6–7 "N_ref may change across planes"） | `tpvformer10/modules/encoder.py:36-47`；点列生成 `encoder.py:124-148`（139-144 linspace） |
| 6 | 投影 `point_sampling`（论文 Eq.8 Ppix）：锚点反归一到 pc_range 度量坐标→齐次→`lidar2img` 矩阵乘→深度>eps 判前向→uv 除以深度、再除以 pad 后 img_shape 归一化→`tpv_mask`（z>eps 且 0<u,v<1） | 每 plane →(6,B,Nq,Nref,2) uv + (6,B,Nq,Nref) mask | 【固定几何·参数自由】（纯标定投影，无学习深度） | `tpvformer10/modules/encoder.py:167-220`（176-181 反归一、196-197 矩阵乘、200-210 掩码、204-205 img_shape 归一）；逐平面调用 `encoder.py:250-256` |
| 7 | CVHA 跨视图混合注意力（论文 Eq.10–12；每层的第一步）：三平面自身为 key/value；每查询的参考点分 3 组（level=平面）——本平面 = 自身格位重复 16 次，两个邻平面 = 沿对方轴均匀 16 个格心（固定 buffer `cross_view_ref_points`，形状 (46400,3,16,2)）；`sampling_offsets`（32=16 锚×2 偏移/头/层）与 `attention_weights`（32 点 softmax × **额外 1 logit 在 3 平面间 softmax 的层级门控**，初始化偏置 10 偏向本平面）由 query 预测；偏移按各平面 (W,H) 归一化后加到锚点上，CUDA deformable 采样；三平面分别 output_proj+残差 | (B,46400,128)→同；每查询每层 8head×3plane×32point=768 采样位置 | 【固定几何给锚点 + 可学习偏移/权重/门控·3D 侧自注意力】 | 锚点 `tpvformer10/modules/encoder.py:49-51,54-121`；模块 `tpvformer10/modules/cross_view_hybrid_attention.py:36-50`（子模块）、`123-127`（层级门控）、`147-156,161-183`（采样）、`201-205`（投影+残差）；层内调用 `tpvformer10/modules/tpvformer_layer.py:171-195`（172-179 三平面 spatial shapes） |
| 8 | ICA 图像交叉注意力（论文 Eq.9；核心桥）：仅命中相机参与——按 `tpv_mask` 对 (Nq,Nref) 求 any（任一锚点入像即该相机参与该查询）逐平面逐相机挑出查询重排（rebatch）；`TPVMSDeformableAttention3D` 内 `value_proj` 投影 value，`sampling_offsets`/`attention_weights`（softmax 覆盖 level×point）由 query 预测；zh/wz 查询先复制为 8 个子查询（各管 4 个锚点，`points_multiplier=[1,8,8]`），每锚点每头每层 2 个学习偏移，采样位置=投影锚点+偏移/层尺寸，四层特征双线性采样（CUDA kernel），8 子查询结果求和还原；scatter 回原序后除以命中相机数（clamp≥1，论文 Eq.9 的 1/\|V_hit\|） | 每 plane →(B,Nq,128)；hw 每查询每命中相机 256、zh/wz 2048 个采样位置 | 【可学习·桥】（offsets/weights/门控是学习量；锚点射线、投影、hit 判定为固定几何） | `tpvformer10/modules/image_cross_attention.py:104-164`（116 any 判定、146-150 deformable 调用、157-160 相机数归一）；`TPVMSDeformableAttention3D` 同文件 242-254（三平面 offsets/weights+共享 value_proj）、280-298（预测+softmax）、300-314（子查询拆分/求和）、375-401（采样位置合成，`floor_sampling_offset=False` 见 `tpv_lidarseg.py:61`）、412-422（CUDA kernel） |
| 9 | 输出投影+残差：ICA 外壳 `output_proj`(128→128)+dropout+残差 | (B,46400,128) | 【可学习·3D 侧】 | `tpvformer10/modules/image_cross_attention.py:58,161-164` |
| 10 | 层堆叠（论文 HCAB/HAB）：HCAB = ('self_attn','norm','cross_attn','norm','ffn','norm')（CVHA→ICA→FFN，FFN 128→256→128）×3 层；HAB = ('self_attn','norm','ffn','norm')（仅 CVHA+FFN）×2 层；共 5 层 → 三平面 tpv_embed | 每 plane (B,Nq,128) | 【可学习·3D 侧】（层数/顺序为配置值；论文 Fig.4 与 Table 5 最优 3+2 一致） | `tpv_lidarseg.py:36-75,77-95,152-158`；`tpvformer10/modules/tpvformer_layer.py:165-221`；论文 Table 5 p.8 |
| 11 | TPV 聚合（论文 Eq.4–5、§3.3）：**点查询**——点坐标（训练时为监督格索引 `train_grid`）归一化 /grid×2−1 后在三平面各 `grid_sample` 双线性采样，三特征**直接求和**（无学习权重）；**体特征**——三平面沿各自正交轴广播成 (200,200,16,128) 后求和；scale_h/w/z=1 无上采样 | 点 (B,C,n)、体 (B,128,200,200,16) | 【固定几何】（投影/双线性/求和零参数）+ 求和后接可学习 MLP | `tpvformer10/tpv_aggregator.py:34-97`（69-71 归一化、73-77 grid_sample、79-84 广播+求和）；点来源 `train.py:186` |
| 12 | 轻量分割头：decoder Linear(128,256)+Softplus+Linear(256,128) → classifier Linear(128,17) | →(B,17,…) logits（vox+pts 两路输出） | 【可学习·3D 侧下游】 | `tpvformer10/tpv_aggregator.py:24-30,87-97`；`tpv_lidarseg.py:101-113` |

损失（桥之外，一句带过）：lovasz 作用点预测、CE 作用体预测（`tpv_lidarseg.py:8-9`；`train.py:191-202`；`builder/loss_builder.py:9-13`），与论文 §4.2 p.7 一致。

## 3. 可学习参数清单表（nuScenes 口径，config/tpv_lidarseg.py）

> "侧"指参数位于投影桥的哪一端。CVHA 出现在全部 5 层（每层 1 个）；ICA 出现在前 3 层 HCAB（每层 1 个）；FFN 每层 1 组。

| 参数（组） | 所属模块 | 形状（nuScenes 口径） | 侧 | 代码出处 `path:line` |
|---|---|---|---|---|
| `tpv_embedding_hw`（HW 平面 query 表，本模型的名片参数） | TPVFormerHead | `nn.Embedding(200×200,128)`=**(40000,128)**，5.12M | 3D 侧（桥的主动方） | `tpvformer10/tpv_head.py:53`（200×200 来自 `tpv_lidarseg.py:20-21`） |
| `tpv_embedding_zh` | TPVFormerHead | `nn.Embedding(16×200,128)`=**(3200,128)**，0.41M | 3D 侧 | `tpvformer10/tpv_head.py:54` |
| `tpv_embedding_wz` | TPVFormerHead | `nn.Embedding(200×16,128)`=**(3200,128)**，0.41M | 3D 侧 | `tpvformer10/tpv_head.py:55` |
| 位置编码 `h_embed`+`w_embed`+`z_embed`（CustomPositionalEncoding，本仓自研注册） | TPVFormerHead | `Embedding(200,48)`+`Embedding(200,48)`+`Embedding(16,32)`，输出拼 128 维（忽略轴槽位置零） | 3D 侧（逐层注入 CVHA，见 §6 差异 #5） | `tpvformer10/tpv_head.py:141-143,147-170`；配置 `tpv_lidarseg.py:159-165` |
| `cams_embeds` | TPVFormerHead | (6, 128)（每相机一个可学习向量，加到所有特征层） | 桥（2D 特征侧注入） | `tpvformer10/tpv_head.py:51-52,103` |
| `level_embeds` | TPVFormerHead | (4, 128)（每尺度一个） | 桥（2D 特征侧注入） | `tpvformer10/tpv_head.py:49-50,104` |
| CVHA `sampling_offsets` ×3 平面子模块 ×5 层 | TPVCrossViewHybridAttention | 每子模块 `Linear(128, 8×3×32×2=1536)`；权重零初始化、bias 为"本平面 2D 放射网格 ×(j+1)/2 + 邻平面球面网格（2 极角×4 方位角投影）×2(j+1)" | 3D 侧（跨平面自注意力的学习偏移） | `tpvformer10/modules/cross_view_hybrid_attention.py:42-44,59-65,67-76,94-105` |
| CVHA `attention_weights`（**含平面层级门控**：输出 33=32 点 + 1 门控 logit，门控在 3 平面间 softmax，初始偏置 10 偏向本平面） ×3 ×5 层 | TPVCrossViewHybridAttention | 每子模块 `Linear(128, 8×3×33=792)` | 3D 侧（平面间学习权重所在，见 §6 差异 #3） | `tpvformer10/modules/cross_view_hybrid_attention.py:45-47,107-110,123-127` |
| CVHA `value_proj`/`output_proj` ×3 ×5 层 | TPVCrossViewHybridAttention | 各 `Linear(128,128)` | 3D 侧 | `tpvformer10/modules/cross_view_hybrid_attention.py:39-41,48-50` |
| ICA `sampling_offsets` ×3 平面子模块 ×3 层 | TPVMSDeformableAttention3D | hw `Linear(128, 8×4×8×2=512)`；zh/wz `Linear(128, 8×4×64×2=4096)`；权重零初始化、bias 放射状网格（每锚点 2 偏移） | 桥（拉取路径上的学习量） | `tpvformer10/modules/image_cross_attention.py:247-249,261-274`（num_points=[8,64,64] 来自 `tpv_lidarseg.py:28`） |
| ICA `attention_weights` ×3 ×3 层 | TPVMSDeformableAttention3D | hw `Linear(128, 8×4×8=256)`；zh/wz `Linear(128, 8×4×64=2048)`；softmax 覆盖 level×point（hw 32、zh/wz 256 维） | 桥 | `tpvformer10/modules/image_cross_attention.py:251-253,275,290-293` |
| ICA `value_proj`（三平面共享 1 个） ×3 层 | TPVMSDeformableAttention3D | `Linear(128,128)` | 桥（2D 特征侧） | `tpvformer10/modules/image_cross_attention.py:254,368` |
| ICA 外壳 `output_proj` ×3 层 | TPVImageCrossAttention | `Linear(128,128)` | 桥（聚合后） | `tpvformer10/modules/image_cross_attention.py:58,162` |
| 每层 FFN+LN（HCAB 3 组 norm+FFN、HAB 2 组） ×5 层 | TPVFormerLayer | FFN 128→256→128（2 个 Linear，dropout 0.1）+LN(128) | 3D 侧 | `tpv_lidarseg.py:72-73,74,94`；`tpvformer10/modules/tpvformer_layer.py:112-130,216-219` |
| 聚合头 `decoder`+`classifier` | TPVAggregator | Linear(128,256)+Softplus+Linear(256,128)+Linear(128,**17**) | 3D 侧（桥之外下游；17 类=`tpv_lidarseg.py:34`，16 前景类+背景 ignore 0=`config/_base_/dataset.py:27,4`） | `tpvformer10/tpv_aggregator.py:24-30`；`tpv_lidarseg.py:101-113` |
| `img_backbone`+`img_neck`（ResNet-101-DCN+FPN，FCOS3D 预训练） | TPVFormer | 见 §2 步骤 1–2 | 2D 侧（拉取之前） | `tpv_lidarseg.py:114-132`；`config/_base_/schedule.py:3` |

要点：**投影桥上的学习量 = 三平面 query 表（46400×128=5.94M，名片参数）+ ICA 的 offsets/weights/value_proj（3 层）+ CVHA 的 offsets/含平面门控的 weights（5 层）+ 两侧注入嵌入（cams/level embeds）**；而"在哪采"的骨架——三平面正交轴锚点射线、跨视图 16 锚点网格、lidar2img 投影、hit 判定、聚合阶段的投影+双线性+求和——全部零参数（见 §4）。论文未给 nuScenes 口径参数量（SSC 口径 6.0M，§4.5 p.8，论文值、异口径不比较）。

## 4. 固定几何清单表

| 项 | 具体值 | 出处 |
|---|---|---|
| 感知范围 pc_range | [-51.2, -51.2, -5.0, 51.2, 51.2, 3.0]（X/Y ±51.2 m；Z −5~3 m），real 102.4×102.4×8.0 m | `tpv_lidarseg.py:11`；`tpvformer10/tpv_head.py:39-41` |
| 三平面网格 | hw 200×200（格 0.512×0.512 m）；zh 16×200（z 格 **0.5 m** × y 格 0.512 m）；wz 200×16（x 0.512 × z 0.5） | `tpv_lidarseg.py:20-22,11`；聚合 reshape `tpvformer10/tpv_aggregator.py:42-44` |
| hw 锚点射线（沿 Z） | 4 锚 `linspace(0.5,7.5,4)/8`→度量 **{−4.5, −2.167, 0.167, 2.5} m**，间距 2 m（与 BEVFormer z 锚同式同值） | `tpvformer10/modules/encoder.py:36` + `139-140` |
| zh 锚点射线（沿 X） | 32 锚 `linspace(0.5,101.9,32)/102.4`→度量 **−50.7 ~ +50.7 m**，间距 101.4/31=**3.27 m** | `tpvformer10/modules/encoder.py:38-40` + `139-140` |
| wz 锚点射线（沿 Y） | 32 锚，同 zh 数值（沿 Y） | `tpvformer10/modules/encoder.py:42-44` |
| 跨视图锚点网格 | 每查询每平面 16 锚：本平面=自身格心重复 16 次；邻平面=沿对方轴 16 个均匀格心（如 hw 查询在 zh/wz 平面沿 z 取 16 个 z 格心） | `tpvformer10/modules/encoder.py:49-51,54-121`（57-63 本平面、64-75 邻平面）；`tpv_lidarseg.py:150` |
| lidar2img 来源 | `lidar2img = K(4×4) @ (lidar→cam 外参)`，`sensor2lidar` 求逆构造，纯标定量，无增广改写、无学习修正 | `dataloader/dataset.py:79-94` |
| 投影与 hit 判定 | 反归一→齐次→4×4 矩阵乘→深度>eps=1e-5 判前向→uv/深度→除以 pad 后 img_shape→0<u,v<1；nan_to_num 防错 | `tpvformer10/modules/encoder.py:176-212` |
| hit 粒度 | 查询×相机级：Nref 个锚点任一命中即该相机参与该查询（mask any 语义）；count=命中相机数 clamp≥1，实现论文 Eq.9 的 1/\|V_hit\| | `tpvformer10/modules/image_cross_attention.py:116,157-160` |
| zh/wz 子查询拆分 | `points_multiplier=[1,8,8]`：zh/wz 查询的 32 锚分给 8 个子查询（各 4 锚），输出求和还原 | `tpvformer10/modules/image_cross_attention.py:242-244,300-307,309-314` |
| offset 归一化基准 | ICA：各图像特征层 (W,H)；CVHA：各 TPV 平面 (W,H)=[[200,200],[16,200],[200,16]] | `tpvformer10/modules/image_cross_attention.py:382-388`；`tpvformer10/modules/tpvformer_layer.py:172-179` + `cross_view_hybrid_attention.py:168-174` |
| 偏移取整开关 | `floor_sampling_offset=False`（采样位置不做 floor 回绕） | `tpv_lidarseg.py:61`；`tpvformer10/modules/image_cross_attention.py:399-400` |
| 聚合投影（点→平面） | 点（格索引）归一化 /grid×2−1 → 三平面 `grid_sample` 双线性；体特征沿正交轴广播；两路均为**直接求和**（论文 Eq.5 "bilinear interpolation and summation"） | `tpvformer10/tpv_aggregator.py:69-84`；点为格索引 `dataloader/dataset_wrapper.py:84-85` + `train.py:186` |
| 图像输入分辨率 | 900×1600（无 resize），pad 到 32 整除→928×1600（size_divisor=32）；caffe 归一化 mean=[103.530,116.280,123.675] std=1 | `dataloader/dataset_wrapper.py:10-11,28-51`；`dataloader/transform_3d.py:19,38`；论文附录 B p.9 "1600x900" |
| 多尺度特征层（推算） | /8:116×200、/16:58×100、/32:29×50、/64:15×25，各 128ch（out_indices=(1,2,3)+FPN num_outs=4） | `tpv_lidarseg.py:118,125-132` |
| 相机数 | 6（cams_embeds 6；`_num_cams_=6`） | `tpv_lidarseg.py:18`；`tpvformer10/tpv_head.py:51` |
| GridMask 增广 | use_grid_mask=True（prob 0.7，随机几何遮挡，非学习） | `tpv_lidarseg.py:100`；`tpvformer10/tpvformer.py:47-49,62-63` |
| 训练超参（供 HTML 脚注） | AdamW lr=2e-4、backbone lr_mult=0.1、wd=0.01、cosine（warmup 500 iter）、grad clip 35、24 epochs、每 GPU batch=1（6 视图） | `config/_base_/optimizer.py:1-12`；`train.py:114-121,209`；`config/_base_/schedule.py:2`；`config/_base_/dataset.py:15`（论文附录 B p.9：24ep、batch 8×A100、lr 2e-4 一致） |

## 5. 2D-3D 关系小结

- **推/拉方向：拉（pull）**。主动方是 3D 侧的三组可学习平面 query：每个平面查询沿本平面正交轴取固定锚点射线（hw 4 点沿 Z、zh/wz 32 点沿 X/Y），经 lidar2img 投到 6 相机，从命中相机的多尺度特征上"拉取"锚点邻域特征（deformable 采样）。与 BEVFormer 同为拉取式；差异在于锚点射线不在同一根 Z 轴上，而分散到三根正交轴。
- **深度：隐式、无深度分布**。不预测 per-pixel 深度（对比 LSS 的 41 维 α）；深度不确定性由每查询的锚点射线离散枚举（hw 沿 Z 4 档、zh/wz 沿水平轴 32 档），由 attention 权重隐式加权，无深度监督、无深度输出头。
- **几何与学习的分工**：固定几何决定"候选落点"——三平面网格（0.512/0.5 m 格）、正交轴锚点射线（4/32/32）、跨视图 16 锚网格、lidar2img 投影、hit 掩码、聚合阶段的投影+双线性+求和；学习决定"query 内容"（三张嵌入表 46400×128）、"采多远、权重多少"（ICA/CVHA 的 offsets 与 weights）。特别的：**平面之间的学习权重存在于 CVHA 的层级门控**（attention_weights 第 33 个输出在 3 平面间 softmax，`cross_view_hybrid_attention.py:123-127`），而**聚合阶段三平面是等权求和、无学习权重**（`tpv_aggregator.py:83-84`，与论文 Eq.5 一致）。
- **query 有无及形态：有，且是三张平面嵌入表**。TPV query = (40000,128)+(3200,128)+(3200,128) 共 46400 条，逐格与三平面对应（论文 §3.2 "initialize TPV queries as learnable parameters"，且 "TPV queries and TPV planes refer to the same set of feature vectors"——query 与平面同物异名）。对比：BEVFormer 单表 40000 条（仅 BEV），LSS 无 query；TPVFormer 用 **7.25% 于稠密 voxel 的 query 预算**（46400 vs 640000）覆盖全 3D 空间——体表征靠"广播+求和"零参数重建，任意分辨率可查（论文 Fig.6）。
- **三平面交互（BEVFormer 没有的）**：CVHA 让三平面 query 互为 key/value（跨平面 deformable 自注意力），是"3D 侧自注意力"；BEVFormer 的 self-attn 是 BEV 平面内+时序，TPVFormer 无时序。

## 6. 论文 vs 代码差异点 + 待画图要点

### 6.1 差异与勘正记录

| # | 项 | 论文说法 | 代码事实 | 处理 |
|---|---|---|---|---|
| 1 | 机制图编号 | 任务线索"TPVFormer 图 2" | 论文 **Fig.2 是任务总览**（相机输入→TPVFormer→稀疏监督→稠密占用预测，p.2）；**Fig.4 才是 TPVFormer 架构图**（Backbone→HCAB/HAB→TPV→预测头，p.5）；另 Fig.3（p.3）为 voxel/BEV/TPV 表示对比 | 重绘对象取 **Fig.4**（机制）+ Fig.3（表示概念）；Fig.2 仅作任务背景 |
| 2 | 三平面聚合权重 | 任务侧预期"可学习权重混合"；**本 PDF（arXiv v2）Eq.5/§3.3 明写 bilinear 采样 + 求和（sum）**，全文无 α | 代码同为**直接求和**，无任何学习权重（`tpvformer10/tpv_aggregator.py:83-84`；`tpvformer04` 版逐字节相同，diff 为空） | 以本仓+本 PDF 为准：**聚合=等权求和**。"可学习 α 加权"不属于本 commit 与 v2 论文，底稿不采用 |
| 3 | CVHA 平面门控 | Eq.12 仅 `CVHA(t)=DA(t, R, T)`，未提平面级权重 | `attention_weights` 每平面多输出 1 个 logit，在 3 平面间 softmax 成层级门控并与点权重相乘；初始偏置 10 使训练初期"本平面主导"（`cross_view_hybrid_attention.py:45-47,107-110,123-127`） | 代码侧补充机制；平面间的学习混合在此实现（不在聚合处，对照 #2）；画图在 CVHA 标注 3-way plane gate |
| 4 | 本平面参考点 | §3.2 CVHA："To collect reference points on the top plane, we simply sample a few **random** points in the neighborhood" | 本平面 16 锚=**自身格心重复 16 次**（`encoder.py:57-63`），邻域扩散交给可学习偏移（初始化为放射状网格，`cross_view_hybrid_attention.py:59-65`）；邻平面锚=沿轴 16 个均匀格心（与论文 Eq.11 的 {(d_i,h)}/{(w,d_i)} 一致） | 以代码为准：确定性锚点+学习偏移；画图按代码画 |
| 5 | 位置编码注入点 | §3.2 TPV Queries："add 3D positional embedding **before the first encoder layer**" | `tpv_pos` 在**每一层**的 CVHA 内部重新相加（`tpvformer_layer.py:185-192` 逐层传 query_pos → `cross_view_hybrid_attention.py:148-149`）；ICA 不加 pos | 代码更激进（逐层注入）；画图把 pos 标注在每个 CVHA 入口 |
| 6 | 层数 N1/N2 | 正文只定义 HCAB/HAB 结构，未定数目（Table 5 消融最优 3+2 为 SSC 口径，p.8） | nuScenes 配置 3 HCAB + 2 HAB（`tpv_lidarseg.py:152-158`） | 以代码为准；与 Table 5 最优组合一致，列入一致性项 |
| 7 | cams/level embeds | 论文未提 | `cams_embeds`(6,128)、`level_embeds`(4,128) 注入多尺度特征（`tpv_head.py:49-52,103-104`；BEVFormer 同名遗产） | 代码侧补充机制，画图可灰色小标 |
| 8 | zh/wz 子查询拆分 | 论文未提（Eq.9 只按相机聚合） | zh/wz 查询复制为 8 子查询各管 4 锚（`points_multiplier=[1,8,8]`，`image_cross_attention.py:242-244,300-314`），注意力 softmax 跨 level×point（hw 32 维、zh/wz 256 维）联一 | 代码侧实现细节，进流水线步骤 8 |
| 9 | 每锚点偏移数 | 论文未给数值 | 每锚点每头每层 2 个学习偏移（`num_points=[8,64,64]`/`num_z_anchors=[4,32,32]`，`tpv_lidarseg.py:27-28`） | 以代码为准；画图标 2 偏移/锚 |
| 10 | Eq.6 反投影公式 | `(x,y)=((h−H/2)·s, (w−W/2)·s)`（ego 居中） | linspace 格心反归一 `(i+0.5)/N×102.4−51.2`（`encoder.py:139-144,176-181`），取 i+0.5=h−0.5 时与论文式恒等 | 一致（列入以示已核对） |
| 11 | N_ref 平面间可变 | §3.2 ICA："N_ref may change across planes because of the different ranges of axes" | [4, 32, 32]（Z 8 m vs X/Y 102.4 m，`tpv_lidarseg.py:27`） | 一致 |
| 12 | 200×200×16、dim 128、1600×900、ResNet101-DCN(FCOS3D)、CE(voxel)+lovasz(points)、AdamW 2e-4/24ep | Table 4 p.8、附录 B p.9、§4.2 p.7 | `tpv_lidarseg.py:13,20-22`、`dataset_wrapper.py`（无 resize）、`tpv_lidarseg.py:114-124,8-9`、`config/_base_/optimizer.py:1-9` + `schedule.py:2` 逐项吻合 | 一致 |

其余核对一致项：hit-view 剔除机制（§3.2 "removing invalid sets if none of the reference points falls onto the image" ↔ `image_cross_attention.py:113-137`）、Eq.9 的 1/|V_hit| 归一（↔ `image_cross_attention.py:157-160`）、邻平面锚点 Eq.11（↔ `encoder.py:64-75,77-94,96-113`）、TPV 平面即 query（§3.2 ↔ `tpv_head.py:53-55`）。另记：仓内双插件目录 `tpvformer04`（旧，SemanticKITTI 用）/`tpvformer10`（nuScenes lidarseg 用），引用时必须带目录名。

### 6.2 待画图要点（Fig.4 重绘规范）

- **Fig.4 架构重绘**：
  - 保留构图元素：6 视图输入 → 图像 Backbone（多尺度特征，标 /8–/64）→ 堆叠块行（论文画了 5 块：前 3 块 HCAB=Hybrid-Attention+Cross-Attention、后 2 块 HAB=Hybrid-Attention）→ 三平面 TPV 表示（标 HW 200×200、ZH 16×200、WZ 200×16）→ 预测头 → 点/体语义占用输出。
  - 叠加编码（本笔记新增）：① 可学习元素实色——三张 query 嵌入表 (40000,128)/(3200,128)/(3200,128)、pos 编码、cams/level embeds、CVHA/ICA 的 offsets 与 weights（CVHA 标 3-way plane gate）、FFN、聚合头 MLP；② 固定几何元素虚线/灰——正交轴锚点射线（hw 沿 Z 4 锚 {−4.5,−2.167,0.167,2.5} m、zh/wz 沿水平 32 锚间距 3.27 m）、lidar2img 投影与 hit 掩码、跨视图 16 锚网格、聚合"广播/双线性+等权求和"；③ 注记 0.512/0.5 m 格、5 层=3+2、四尺度 128ch、query 预算 46400（7.25% of 稠密 voxel）。
  - 一句话构图意图：让读者看到"三平面 query 是学的、射线锚点与聚合几何是标的"。
- **Fig.3 表示概念重绘（可选并排）**：
  - 保留构图元素：voxel（立方网格）/ BEV（单平面+pillar 模糊）/ TPV（三正交平面）三者对比；TPV 画一个体素格 = 三平面投影点求和。
  - 叠加编码：三平面格标 0.512 m/0.5 m；求和符号标"等权（无学习权重）"；可加"任意分辨率点查询（grid_sample 双线性）"注记呼应论文 Fig.6。
  - 一句话构图意图：呈现"BEV 单平面 → TPV 三平面"的推广关系，并与 BEVFormer 分册的 pillar 图、LSS 分册的视锥图并排成第二组对照。

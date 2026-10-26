# 04 · VoxFormer 事实底稿

> 本文件为 BEV 投影对比笔记的 VoxFormer 分册，数字与结论的唯一来源是下方标注的 `path:line`（相对于 `bev_projection/code/VoxFormer/`）与论文原文（arXiv:2302.12251 PDF，正文 8 页+附录，13 页）。后续 HTML 各节只引用本底稿，不重查代码。**口径声明（重要）**：本 commit（`63da9241`）的 NVlabs/VoxFormer 与论文均**只有 SemanticKITTI 单目（cam2）SSC 口径，无 nuScenes 支持**（证据见 §6.1 差异 #1），manifest 中"qpn.py 为 nuScenes 配置"为误判。四分册里 VoxFormer 以 SemanticKITTI 口径入册，与 LSS/BEVFormer/TPVFormer 的 nuScenes 口径做异库对比时需在 HTML 注明数据集差异。

## 1. 元信息

| 项 | 值 |
|---|---|
| 仓库 | https://github.com/NVlabs/VoxFormer |
| commit hash | `63da92415447c2e97654bad977254c907f5ce703`（"update readme"，已用 `git rev-parse HEAD` 复核，与 manifest 一致；单分支 main） |
| 克隆日期 | 2026-10-26（`--depth 1` 浅克隆，见 `00_manifest.md`） |
| 论文 | Li, Yiming et al., *VoxFormer: Sparse Voxel Transformer for Camera-based 3D Semantic Scene Completion*, CVPR 2023；`papers/voxformer_2302.12251.pdf`。机制：§3.1–3.2 p.3、§3.3（voxel query/mask token）p.3–4、§3.4（stage-1 深度→proposal，Eq.1–2）p.4、§3.5（stage-2 DCA/DSA，Eq.3–5）p.5、§3.6 loss Eq.6 p.5、Fig.2 p.4；实现与数据集 §4.1 p.5–6；Table 1 p.6；消融 Table 4–7 p.8。**全文 0 次提及 nuScenes**（PDF 全文检索，13 页） |
| 主口径配置 | **`projects/configs/voxformer/voxformer-S.py`**（VoxFormer-S = 单帧单相机基准，论文 Table 1/Table 6/7 消融均以 S 为基准行；投影机制与 T 完全相同）。同目录其余四个：`voxformer-T.py`（唯一差异 `_num_cams_=5`、`_temporal_=[-12,-9,-6,-3]`，`voxformer-T.py:18-19`，即论文"当前+前 4 帧"时序版，README:48 IoU 44.15/mIoU 13.35=论文 headline）；`voxformer-S_deform3D.py`/`voxformer-T_deform3D.py`（stage-2 自注意力改真 3D 采样的增强变体，`voxformer-S_deform3D.py:97,103,109,112` 四处 type 替换，README:50-51）；`qpn.py`（**stage-1 QPN 训练配置，SemanticKITTI 口径，非 nuScenes**，见差异 #1）。核心量：`point_cloud_range=[0,-25.6,-2.0,51.2,25.6,4.4]`（`voxformer-S.py:20`）、`voxel_size=[0.2,0.2,0.2]`（`voxformer-S.py:21`）、`_dim_=128`（`voxformer-S.py:12`）、`_num_levels_=1`（`voxformer-S.py:15`）、`_num_cams_=1`（`voxformer-S.py:18`）、`_num_layers_cross_=3`/`_num_points_cross_=8`（`voxformer-S.py:8-9`）、`_num_layers_self_=2`/`_num_points_self_=8`（`voxformer-S.py:10-11`）、`bev_h=128, bev_w=128, bev_z=16`（`voxformer-S.py:51-53`）、位置编码表 512×512（`voxformer-S.py:128-132`）、`_nsweep_=10`/`_depthmodel_='msnet3d'`/`_query_tag_='query_iou5203_pre7712_rec6153'`（`voxformer-S.py:25-27`） |
| 插件代码目录 | `projects/mmdet3d_plugin/voxformer/`——`dense_heads/voxformer_head.py`（两阶段总装）、`modules/encoder.py`+`transformer.py`+`deformable_cross_attention.py`（stage-1 掩码 DCA）、`modules/encoder_3D.py`+`transformer_3D.py`+`deformable_self_attention_3D_custom.py`（deform3D 变体专用）、`modules/deformable_self_attention.py`（vanilla stage-2 DSA）、`modules/custom_base_transformer_layer.py`（层骨架）、`detectors/voxformer.py`（在线模型壳）、`detectors/lmscnet.py`（QPN）、`utils/header.py`（输出头）。离线管线在 `preprocess/`（深度→伪激光雷达→伪占用）。依赖 mmcv-full 1.4.0 + mmdet 2.14.0（`docs/install.md:27,33`），位置编码类 `LearnedPositionalEncoding` 即 mmdet 2.x 标准实现（插件内无同名注册） |
| 关键派生量（S 口径） | 输出体 **256×256×32=2,097,152 格 @0.2 m**（(51.2−0)/0.2×(25.6+25.6)/0.2×(4.4+2.0)/0.2）；query 体 **128×128×16=262,144 @0.4 m**（N_q，论文 §3.3 "Nq=h·w·z"）；图像 cam2 裁剪 **1220×370**（`semantic_kitti_dataset_stage2.py:64-65`）→ 特征 /16 → **24×77=1848 token**（ceil(370/16)×ceil(1220/16)）；每 query 每命中相机 8 头×1 level×8 采样点=**64 采样位置**；stage-2 每 voxel query 8 头×8 点 deformable 自采样；参数总量论文 Table 6（p.8，1/16 层）**57.90 M**（自算 ≈58.18 M，含 mmdet 无 fc 头 ResNet50 ≈23.5 M + bev_embed 33.55 M，numpy 复核）；训练显存论文 Table 4（p.8）occupancy 稀疏 query **14.6 G** vs dense 100% 18.5 G |

## 2. 机制流水线（每步标注【固定几何】/【可学习】）

VoxFormer 的"2D→3D"分**离线 stage-1**（深度→伪点云→伪占用→QPN proposal，全部预处理，不在训练图内）与**在线 stage-2**（掩码 deformable 交叉注意力 + MAE 式补全）。在线输入：`(B=1, N_cam=1, 3, 370, 1220)`（`voxformer-S.py:69-70` samples_per_gpu=1；`semantic_kitti_dataset_stage2.py:347-396`）。

| # | 步骤 | 形状流 | 属性 | 出处 |
|---|---|---|---|---|
| 0 | 立体深度估计（离线）：MSNet3D（MobileStereoNet3D，`MSNet3D_SF_DS_KITTI2015.ckpt`，逐序列基线 fx 388.18/389.63/381.83）对 image_2/3 立体对出深度图 | 图像对→深度图 | 【可学习·外部冻结模型】（论文 §3.4 "off-the-shelf"） | `preprocess/image2depth.sh:6-8`（ckpt/model）、`:20,24,28`（基线）；论文 §4.1 p.5 |
| 1 | 深度反投影（离线，论文 Eq.1）：`x=(u−c_u)·z/f_u`，`y=(v−c_v)·z/f_v`，`z=Z(u,v)`，rect→velo；过滤 `x≥0` 且 `z<80` | (u,v,Z)→点云 (N,4) | 【固定几何·参数自由】（纯标定反投影，论文 Eq.1 逐字对应） | `preprocess/utils/kitti_util.py:148-160`（154-155 公式）、`:162-164`（rect→velo）；`preprocess/utils/depth2lidar.py:9-17`（16 行过滤）、`:25`（max_high=80） |
| 2 | 10-sweep 累积+光线投射（离线）：历史帧按 poses 变换到当前帧拼接；在 256×256×32@0.2 m 网格上以 log-odds 光线投射（`compute_logodds_dp`，占据=1/自由=0）得**伪占用图** `.pseudo` | 点云→(256,256,32) 二值体 | 【固定几何·参数自由】（体素化+光线投射零参数） | `preprocess/utils/lidar2voxel.py:282-303`（位姿拼接）、`:314-316`（map_dims/voxel/范围）、`:318,323`（pc_range/log-odds）、`:329-337`（占据/自由/打包）；`preprocess/lidar2voxel.sh:12`（sequence_length=10）；范围值 `[[0,51.2],[-25.6,25.6],[-2,4.4]]` 与主口径 pc_range 一致 |
| 3 | QPN 占用预测（离线，论文 Φ_occ/"depth correction"）：LMSCNet_SS 以伪占用为输入（Z 当通道排成 (1,32,256,256)），2D U-Net 编码（f=32→48→64→80）+ 3D ASPP 分割头，输出 **128×128×16 二值占用**=query proposal 掩码，argmax 后 bit-pack 存 `.query_iou5203_pre7712_rec6153` 文件 | (1,32,256,256)→(2,128,128,16)→(128,128,16) | 【可学习·3D 侧深度门控】（**数据相关的稀疏化步骤**：哪些 voxel 获得 query 由深度+QPN 决定；论文 §3.4 Eq.2 `Qp=Reshape(Q[Mout])`） | 训练 `projects/configs/voxformer/qpn.py:41-44`（LMSCNet_SS/class_num=2/input [256,32,256]/out_scale 1_2）、`:60-61`（SemanticKittiDatasetStage1 './kitti/'）；模型 `projects/mmdet3d_plugin/voxformer/detectors/lmscnet.py:67-140`（编码器+头）、`:261`（GT 二值化）、`:267-268`（输入重排）、`:274`（BCE 损失）、`:304-317`（推理存 query）；GT 128×128×16 标签 `datasets/semantic_kitti_dataset_stage1.py:172-176` |
| 4 | 在线数据装配（stage-2 数据集）：读 `.query` proposal（bit 解包）、cam2 图像裁剪 370×1220、构造 `lidar2img = K(4×4) @ T_velo_2_cam`（P2 前 3×3 为内参，拼 4×4）随 img_metas 下发 | proposal (262144,)、img (1,3,370,1220)、lidar2img (1,4,4) | 【固定几何】（标定量；proposal 是步骤 3 的产物） | `datasets/semantic_kitti_dataset_stage2.py:64-65`（图像尺寸）、`:168-191`（扫描表：P、Tr、proj_matrix）、`:280-299`（lidar2img 构造，289 内参、294 矩阵乘）、`:332-343`（proposal 读入+meta） |
| 5 | 2D 主干+颈：ResNet50 取 C3（out_indices=(2,)，1024ch，stride 16，frozen_stages=1，BN 冻结）→ FPN 单层 128ch（1×1 降维+on_output 3×3 extra conv） | (1,3,370,1220)→1024ch@24×77→**128ch@24×77** | 【可学习·2D 侧】 | `voxformer-S.py:31-48`（backbone 33-40、neck 41-48）；`detectors/voxformer.py:52-77`（extract_img_feat）；论文 §4.1 p.5-6 "features in the 3rd stage…FPN…1/16" |
| 6 | 特征展平+嵌入注入：加 `cams_embeds`(1,128)（T 变体 (5,128)）与 `level_embeds`(4,128)（默认 num_feature_levels=4，仅 level 0 使用），成 value 序列 | →(N_cam=1, 1848, B, 128) | 【可学习·桥侧嵌入】 | `modules/transformer.py:39-49`（默认值）、`:65-70`（两张嵌入表）、`:114-134`（展平+120-123 注入） |
| 7 | query 选择：`proposal.reshape(-1)>0` 的 voxel 索引=unmasked（获得 query），`=0` 为 masked（后续填 mask token）；从 `bev_embed` 全表 (262144,128) 中**仅取命中行**；同时 `get_ref_3d` 生成全部 voxel 格心的归一化 3D 参考点（(i+0.5)/N∈[0,1]³，网格原点 [0,−25.6,−2]、体尺寸 51.2×51.2×6.4、格距 0.4 m） | 激活 query (N_p,B,128)（N_p 逐帧变）、ref_3d (262144,3) | 【可学习 query 内容 + 固定几何锚点】（表是学的，锚点格心是零参数 linspace 式网格） | `dense_heads/voxformer_head.py:53`（bev_embed）、`:82`（取 weight）、`:89-91`（unmasked/masked 索引）、`:108-109`（仅取命中行）、`:191-216`（get_ref_3d：199 scene_size、200 vox_origin、201 voxel_size=51.2/128=0.4、208-211 索引网格、214 归一化格心） |
| 8 | 投影 `point_sampling`（论文 P(p,t)）：voxel 格心反归一到 pc_range 度量坐标→齐次→`lidar2img` 矩阵乘→相机系深度>eps=1e-5 判前向→uv 除深度→除以 img_shape(370,1220) 归一→`bev_mask`（z>eps 且 0<u,v<1） | ref (1,1,262144,3)→每相机 uv (N_q,2)+mask | 【固定几何·参数自由】（每个 voxel 中心投到各图，判落点；无学习量） | `modules/encoder.py:94-149`（105-110 反归一、112-113 齐次、125-126 矩阵乘、129-131 除深度、133-134 img_shape、136-139 mask）；调用 `encoder.py:197-198` |
| 9 | 掩码 deformable 交叉注意力 DCA ×3（论文 Eq.3–4）：**只有 proposal 激活且落入某相机的 query 参与该相机**（按 bev_mask 逐相机 rebatch）；`MSDeformableAttention3D`（8 头、1 level、8 点）：`value_proj` 投影 2D 特征，`sampling_offsets`/`attention_weights`（softmax 覆盖 8 点）由 query 预测，采样位置=投影格心+偏移/(W,H)，双线性采样（CUDA kernel）；命中相机输出求和后除以命中数（clamp≥1，=论文 Eq.4 的 1/\|V_t\|）；外层 output_proj+dropout+残差；层序 ('cross_attn','norm','ffn','norm') | (N_p,B,128)→(N_p,B,128)；每 query 每命中相机 64 采样位置 | 【可学习·桥】（offsets/weights/value_proj/output_proj 是学习量；锚点投影与 hit 判定为固定几何） | 相机分组与归一 `modules/deformable_cross_attention.py:140-157`（rebatch）、`:159-168`（deformable 调用）、`:169-176`（计数归一）、`:71`（output_proj）；采样核 `deformable_cross_attention.py:208-253`（210 默认 8 头、249-252 三个 Linear、225 output_proj=None）、`:342-352`（预测+softmax）、`:361-371`（偏移归一+合成位置）；层配置 `voxformer-S.py:58-95`；论文 §3.5 Eq.4 p.5 |
| 10 | FFN ×3（每 DCA 层内）：Linear(128→**256**)→ReLU→Linear(256→128)，dropout 0.1（生效隐藏维 256=2×128，config 里 ffn_cfgs 的 1024 是被覆盖的死值，见差异 #4） | (N_p,B,128) | 【可学习·桥】 | `voxformer-S.py:85-95`；覆盖逻辑 `modules/custom_base_transformer_layer.py:81-92`（deprecated 参数回填 ffn_cfgs）、`:144-151`（建 FFN） |
| 11 | MAE 式补全：全网格容器 (128,128,16,128)；unmasked 位置写入 cross 输出（scatter 回原 voxel 序），masked 位置广播 `mask_embed`(1,128)（论文 §3.3 mask token） | →(262144,128) | 【可学习·3D 侧】 | `dense_heads/voxformer_head.py:110-113`（110-112 scatter、113 mask token 扩展）；论文 §3.3 p.4 |
| 12 | deformable 自注意力 DSA ×2（论文 Eq.5）：全体 262144 voxel 互采；vanilla S 版为 **2D 形式**——体网格按展平索引摆到 512×512 参考平面（`get_reference_points(512,512,'2d')`），每 voxel 在该平面做 deformable 采样（8 头×8 点，偏移/(512,512) 归一），value 复制 2 份（num_bev_queue=2 遗产）均值合并；层序 ('self_attn','norm','ffn','norm')+FFN(128→256→128)。deform3D 变体改为真 3D 参考点 (128,128,16) 与 3D 偏移（见差异 #6） | (262144,B,128)→(B,262144,128) | 【可学习·3D 侧自注意力】（参考平面/体的骨架为固定索引几何；位置编码在此注入） | vanilla：`dense_heads/voxformer_head.py:116-128`（119-120 传 512,512）、`modules/encoder.py:189-195`（512×512 ref_2d）、`:349-362`（self_attn 分支 query_pos=bev_pos 注入）、`modules/deformable_self_attention.py:61-111`（105-110 四个 Linear）、`:183-186`（value 复制）、`:212-235`（采样合成）、`:258-271`（均值+proj）；deform3D：`modules/encoder_3D.py:204-210`（'3dCustom' 参考点）、`modules/deformable_self_attention_3D_custom.py:182-183`（硬编码 [[128,128,16]]）、`:233-238`（3D 偏移归一）；论文 §3.5 Eq.5 p.5 |
| 13 | 输出头：三线性 ×2 上采样（128³体→256³体，align_corners=True）→ LayerNorm+Linear(128→20) 逐 voxel 分类 | (1,128,128,128,16)→(1,128,256,256,32)→(1,20,256,256,32) | 【可学习·3D 侧下游】（上采样本身零参数） | `utils/header.py:22-27`（MLP 头+上采样）、`:30-45`（前向）；`dense_heads/voxformer_head.py:58,129-134`；论文 §3.5 "upsampled and projected" p.5 |

损失（桥之外，一句带过）：stage-2 = 加权 CE（class_weights 按类频逆对数，`voxformer_head.py:61-63`、`utils/ssc_loss.py:111`）+ scene-class affinity（`sem_scal_loss`，`ssc_loss.py:63`，论文 §3.6 引 MonoScene）+ geo-scal affinity（`geo_scal_loss`，`ssc_loss.py:15`，论文未提，见差异 #9）；stage-1 = 二值 BCE（`lmscnet.py:274`、`ssc_loss.py:121`，论文 §3.6 "binary cross-entropy for occupancy"）。

## 3. 可学习参数清单表（SemanticKITTI 口径，projects/configs/voxformer/voxformer-S.py）

> "侧"指参数位于投影桥的哪一端。行内参数量经 numpy 复核。QPN 与深度模型离线训练后冻结产物供 stage-2 使用。

| 参数（组） | 所属模块 | 张量形状（S 口径） | 侧 | 代码出处 `path:line` |
|---|---|---|---|---|
| `bev_embed`（voxel query 表，本模型的名片参数） | VoxFormerHead | `nn.Embedding(128×128×16, 128)`=**(262144,128)**，33.55 M | 3D 侧（桥的主动方；每帧仅 N_p 行被激活） | `dense_heads/voxformer_head.py:53`（尺寸来自 `voxformer-S.py:51-53`） |
| `mask_embed`（mask token，论文 §3.3 m∈R^d） | VoxFormerHead | `nn.Embedding(1,128)`=**(1,128)**，128 | 3D 侧（补全端） | `dense_heads/voxformer_head.py:54`、填充 `:113` |
| 位置编码 `row_embed`+`col_embed`（mmdet LearnedPositionalEncoding） | VoxFormerHead.positional_encoding | 各 `Embedding(512,64)`，输出拼 **128** 维/位置（512×512 表） | 3D 侧（**仅 DSA 注入**，见差异 #3） | 配置 `voxformer-S.py:128-132`；构建 `dense_heads/voxformer_head.py:55`、使用 `:85-86,104,125`；注入点 `modules/encoder.py:349-361` |
| `cams_embeds` | PerceptionTransformer（cross & self 各一份） | **(1,128)**（T 变体 (5,128)） | 桥（2D 特征侧注入） | `modules/transformer.py:69-70`、注入 `:120-121`；num_cams `voxformer-S.py:63` |
| `level_embeds` | PerceptionTransformer ×2 | **(4,128)**（默认 num_feature_levels=4，实际 1 level，仅 [0] 行用） | 桥（2D 特征侧注入） | `modules/transformer.py:40,67-68`、注入 `:122-123` |
| ResNet50 主干（C3 出口，frozen_stages=1：stem+layer1 冻结） | VoxFormer.img_backbone | 1024ch@/16（mmdet 版无 fc 头，≈23.5 M） | 2D 侧 | `voxformer-S.py:31-40`；`detectors/voxformer.py:61` |
| FPN 颈（1×1 1024→128 + on_output 3×3 128→128） | VoxFormer.img_neck | 单层 **128ch**，≈0.28 M | 2D 侧 | `voxformer-S.py:41-48` |
| DCA `output_proj` ×3 层 | DeformCrossAttention | `Linear(128,128)`，每层 16,512 | 桥（聚合后） | `modules/deformable_cross_attention.py:71,177` |
| DCA `sampling_offsets` ×3 | MSDeformableAttention3D | `Linear(128, 8头×1level×8点×2=128)`，每层 16,512；零初始化+bias 放射状网格 | 桥（拉取路径的学习偏移） | `modules/deformable_cross_attention.py:249-250,259-271`；num_points `voxformer-S.py:80` |
| DCA `attention_weights` ×3 | MSDeformableAttention3D | `Linear(128, 8×1×8=64)`，每层 8,256；softmax 覆盖 8 点 | 桥 | `modules/deformable_cross_attention.py:251-252,272` |
| DCA `value_proj` ×3 | MSDeformableAttention3D | `Linear(128,128)`，每层 16,512 | 桥（2D 特征侧） | `modules/deformable_cross_attention.py:253`（注：`:225` output_proj=None） |
| DSA `sampling_offsets` ×2 | DeformSelfAttention | `Linear(256, 2queue×8头×1level×8点×2=256)`，每层 65,792 | 3D 侧（体间学习偏移） | `modules/deformable_self_attention.py:105-106,113-128`；num_points `voxformer-S.py:115` |
| DSA `attention_weights` ×2 | DeformSelfAttention | `Linear(256, 2×8×1×8=128)`，每层 32,896 | 3D 侧 | `modules/deformable_self_attention.py:107-108` |
| DSA `value_proj`/`output_proj` ×2 | DeformSelfAttention | 各 `Linear(128,128)`，每层 16,512×2 | 3D 侧 | `modules/deformable_self_attention.py:109-110,273` |
| 每层 FFN+LN（DCA 3 层+DSA 2 层共 5 组，每组 2×LN） | VoxFormerLayer | FFN Linear(128→256)+Linear(256→128)，每组 65,920（生效 256，见差异 #4） | 桥/3D 侧（按层所属） | `voxformer-S.py:85-95,117-127`；覆盖与构建 `modules/custom_base_transformer_layer.py:81-92,144-156` |
| 输出头 MLP | Header | LayerNorm(128)+Linear(128→**20**)，2,836 | 3D 侧（桥之外下游；20 类=`voxformer_head.py:51`，19 语义+empty） | `utils/header.py:22-25`；`dense_heads/voxformer_head.py:58` |
| QPN 全部卷积（离线，冻结供 stage-2） | LMSCNet_SS | 编码器 f=32：32→48→64→80 2D conv×2/块×4 块+1_8/1_4/1_2 解码 conv + SegmentationHead（Conv3d 1→8→ASPP(dil 1,2,3)→Conv3d 8→2） | 3D 侧深度门控（决定 N_p） | `projects/configs/voxformer/qpn.py:41-44`；`detectors/lmscnet.py:67-140`（79-108 编码、111-126 解码、134-140+327-366 头） |
| 立体深度模型 MSNet3D（外部，全冻结） | —（离线） | MobileStereoNet3D 预训练 ckpt | 2D 侧（深度的来源） | `preprocess/image2depth.sh:6-8` |

要点：**投影桥上的学习量 = voxel query 表（262144×128=33.55 M，名片参数）+ mask token + DCA/DSA 的 offsets/weights/value_proj（3+2 层）+ cams/level embeds + 两侧主干**；而"哪些 voxel 有 query"这一稀疏化本身也是学习量（QPN），但**逐帧、离线**；"在哪采"的骨架——0.4 m 格心锚点、lidar2img 投影、hit 判定、命中相机均值、×2 上采样——零参数（见 §4）。总量对账：论文 Table 6 p.8 记 VoxFormer-S（1/16 特征层）**57.90 M**；本表逐项加总（含 ≈23.5 M 无 fc ResNet50）≈58.18 M，差 ≈0.3 M（估计来自 FPN/统计口径），记为一致。

## 4. 固定几何清单表

| 项 | 具体值 | 出处 |
|---|---|---|
| 感知范围 pc_range | **[0, −25.6, −2.0, 51.2, 25.6, 4.4]**（车前方 51.2×51.2×6.4 m；ego 在体后沿中点） | `voxformer-S.py:20`；`dense_heads/voxformer_head.py:199-200`（scene_size/vox_origin 同值）；论文 §4.1 p.5 |
| 输出体网格 | **256×256×32 @0.2 m**（2,097,152 格；伪占用、GT、预测同网格） | `voxformer-S.py:21`（voxel_size 0.2）×pc_range 推导；伪占用 `preprocess/utils/lidar2voxel.py:314-316`；GT `datasets/semantic_kitti_dataset_stage2.py:410-423`；论文 §4.1 p.5 "256×256×32 since each voxel has a size of 0.2m" |
| query 体网格 | **128×128×16 @0.4 m**（N_q=262,144；为输出体的 1/8 降采样） | `voxformer-S.py:21,51-53`；`voxformer_head.py:201`（0.4=51.2/128）；论文 §3.3 "lower than output resolution" p.3 + §4.1 p.5（QPN 输出 128×128×16） |
| voxel 参考点锚点 | 全格心 `(i+0.5)/128,(j+0.5)/128,(k+0.5)/16` 归一化；每 voxel **1 个**参考点/相机（格心投影；config `num_points_in_pillar=8` 未用，见差异 #7） | `dense_heads/voxformer_head.py:208-214`；D=1 由 `modules/transformer.py:111-112`（仅 unmasked ref 传入）+ `modules/encoder.py:130-131`（permute 后 D 维=1） |
| lidar2img 来源 | `lidar2img = K(4×4) @ T_velo_2_cam`；K 取 calib P2 前 3×3（基线平移项被丢弃）；P2、Tr 来自 SemanticKITTI calib.txt，无增广改写 | `datasets/semantic_kitti_dataset_stage2.py:168-174,285-294` |
| 投影与 hit 判定 | 反归一 pc_range→齐次→4×4 矩阵乘→相机系 z>eps=**1e-5** 判前向→uv/z→除以 img_shape (370,1220)→0<u,v<1；nan_to_num 防错 | `modules/encoder.py:105-144` |
| 深度反投影（离线） | 论文 Eq.1（内参 (f_u,f_v,c_u,c_v) 由 P2 给）；过滤前向 x≥0、高程 z<80 m | `preprocess/utils/kitti_util.py:148-160`；`preprocess/utils/depth2lidar.py:9-17` |
| 伪占用光线投射（离线） | log-odds 光线投射（`compute_logodds_dp`），传感器原点 [0,0,0]，256×256×32@0.2 m；10 sweep 位姿累积 | `preprocess/utils/lidar2voxel.py:313-323,282-303`；`preprocess/lidar2voxel.sh:12` |
| 深度分箱 | **无**（连续深度→点云→占据格；对比 LSS 41 bins 外积、BEVFormer 无深度、TPVFormer 锚点射线枚举） | 步骤 1–2 全链路无分箱参数；论文 §3.4 |
| hit 粒度与聚合 | voxel×相机级：格心投影落入某相机即该相机参与该 voxel；输出按命中相机数求均值（clamp≥1）=论文 Eq.4 1/\|V_t\| | `modules/deformable_cross_attention.py:140-157,169-176`；论文 §3.5 p.5 |
| 每 query 采样数 | 每命中相机：8 头×1 level×(1 锚点+8 学习偏移点)；DSA 同为 8×8 | `voxformer-S.py:9,11,77-81,110-115`；论文 §4.1 p.6 "8 sampling points around each reference point" |
| DSA 采样骨架 | vanilla：展平索引 512×512 参考平面（偏移/(512,512) 归一）；deform3D 变体：(128,128,16) 3D 参考（偏移/(128,128,16)） | vanilla `modules/encoder.py:189-195`+`voxformer_head.py:119-120`+`deformable_self_attention.py:230-235`；deform3D `modules/encoder_3D.py:204-210`+`deformable_self_attention_3D_custom.py:182-183,233-238` |
| offset 归一化基准 | DCA：图像特征层 (W,H)=(77,24) | `modules/deformable_cross_attention.py:361-367` |
| 输出上采样 | ×2 三线性（align_corners=True），128×128×16→256×256×32，零参数 | `utils/header.py:27,35` |
| 图像口径 | cam2 裁剪 **1220×370**，ImageNet 归一化；特征 /16→24×77=1848 token | `datasets/semantic_kitti_dataset_stage2.py:64-65,73-80,363-370`；论文 §4.1 p.5 "crop RGB images of cam2 to size 1220×370" |
| 相机/时序 | S：1 帧 1 相机；T：当前+前 4 帧（`_temporal_=[-12,-9,-6,-3]`，num_cams=5），全部经同一 lidar2img 投影参与 DCA | `voxformer-S.py:18-19`；`voxformer-T.py:18-19`；`semantic_kitti_dataset_stage2.py:301-330`（参考帧 lidar2img 由 pose 链构造） |
| 训练超参（供 HTML 脚注） | stage-1（QPN）：AdamW lr=2e-4、24 ep（`qpn.py:100-107,117`）；stage-2：AdamW lr=2e-4、wd 0.01、cosine（warmup 500 iter）、grad clip 35、**20 ep**（`voxformer-S.py:187-202`）、每 GPU batch=1；两阶段分别训练（论文 §4.1 "train stage-1 and stage-2 separately with 24 epochs"，stage-2 代码为 20，见差异 #5） | 如左 |

## 5. 2D-3D 关系小结

- **推/拉方向：拉为主、推在门口**。在线主体是拉（pull）：3D voxel query 以自身格心经固定 lidar2img 投到图像、在命中相机上 deformable 采样"拉"特征（与 BEVFormer/TPVFormer 同族，均为 BEVFormer 式 MSDA 血统）。但**进入在线模型之前有一次决定性的"推"**：像素深度经 Eq.1 反投影成伪点云、光线投射成占用、QPN 修正后形成 proposal 掩码——2D 深度决定**哪些** 3D voxel 有资格成为 query（论文 §3.2 "2D features correspond to visible and occupied voxels"）。即：深度不参与在线特征提取，只做**深度门控稀疏化**。
- **深度：显式、离线、无分箱**。深度由外部立体模型（MSNet3D）一次性给出，连续值→离散占据，不存在 LSS 式 per-pixel 深度分布、不存在沿射线的多深度假设枚举（对比 TPVFormer 每查询 4/32/32 锚点射线）。在线注意力里每个 voxel 只有 1 个参考点（格心投影），深度不确定性被前置到 QPN 掩码里消化。代价即论文 Limitation：远距离深度不可靠 → 远距性能受限（§4.2.4 p.8）。
- **几何与学习的分工**：固定几何定"骨架"——0.2/0.4 m 双层网格、格心锚点、lidar2img、hit 判定、命中相机均值、×2 三线性上采样；学习定"内容与稀疏性"——query 嵌入表内容（262144×128）、**N_p 的大小与分布**（QPN，逐帧数据相关）、采样偏移与权重（DCA 3 层+DSA 2 层）、mask token、两侧主干。四分册中 VoxFormer 独有：**稀疏化本身是可学习的**（LSS 稠密 splat、BEVFormer/TPVFormer 全网格 query）。
- **query 有无及形态：有，全网格表+逐帧稀疏激活**。N_q=262,144 条 3D 网格形可学习参数（论文 §3.3 "3D-grid-shaped learnable parameters"），每帧仅 N_p（QPN 输出，数据相关；论文 Table 4 p.8 显示 100% dense 消融 IoU 34.6 vs occupancy 44.0、显存 18.5G vs 14.6G——稀疏既省显存又涨点）参与 DCA；stage-2 用 mask token 把其余 voxel 补齐后经 DSA 稠密化（MAE 式 sparse-to-dense，论文 Fig.2 "Mask Token"+"Updated Queries"）。对比：BEVFormer 40000 条 BEV 网格全激活、TPVFormer 46400 条三平面全激活、LSS 无 query。
- **体间交互（BEVFormer 没有的）**：DSA 是 voxel↔voxel 的 deformable 自注意力（3D 侧自注意力）；BEVFormer 的 self-attn 在 BEV 平面+时序，TPVFormer 的在平面间（CVHA）。VoxFormer 无时序 self-attn（T 变体的时序=把历史帧当"更多相机"进 DCA）。

## 6. 论文 vs 代码差异点 + 待画图要点

### 6.1 差异与勘正记录

| # | 项 | 论文说法 | 代码事实 | 处理 |
|---|---|---|---|---|
| 1 | **任务口径与 manifest 勘正** | 任务线索与 manifest（`00_manifest.md:10`）称 `qpn.py` 为"nuScenes 10 类检测、10 sweeps、camera-only 配置" | 本 commit **无任何 nuScenes 支持**：README:56-58 数据集勾选框 nuScenes 未实现；论文全文 0 次提及 nuScenes；`qpn.py:60-61` 用 `SemanticKittiDatasetStage1`+'./kitti/'（`datasets/semantic_kitti_dataset_stage1.py:37-41` 为 SemanticKITTI 序列划分），模型 `LMSCNet_SS`（`qpn.py:41`）训的是 128×128×16 二值占用；`qpn.py:19-23` 的 nuScenes class_names 是**死配置**（该 dataset 自带 ["empty","occupied"] 两类，`semantic_kitti_dataset_stage1.py:45`）。仓内"nuScenes"字样仅为 `encoder.py:43`/`encoder_3D.py:43` 的默认参数 `dataset_type='nuscenes'`（未使用）与 README:19 的 SSCBench 公告 | 底稿以 **SemanticKITTI 口径**为准（本仓唯一可运行口径）；manifest 该行建议后续修正 |
| 2 | stage-1 在线/离线 | §3.4 把深度→占用→proposal 画为 pipeline 组成（Fig.2 "Depth Prediction"框） | 代码中深度、反投影、光线投射、QPN **全部离线**（`preprocess/` 三脚本 + `qpn.py` 单独训练推理）；在线 stage-2 直接读 bit-pack 的 `.query_*` 文件（`voxformer-S.py:27` query_tag、`semantic_kitti_dataset_stage2.py:176-178,332`） | 画图把 stage-1 整体标"离线预处理"，虚线框；论文 Fig.2 未区分在线/离线 |
| 3 | 位置编码注入点 | §3.3 p.4 "learnable positional embeddings **will be added to voxel queries for attention stages**" | **cross-attn 分支实际不注入**：`bev_pos_cross_attn` 计算并传递（`voxformer_head.py:85,104`→`transformer.py:106-109,143`），但 `VoxFormerLayer` 的 cross 分支把 query_pos（None）传给 DCA（`encoder.py:371-386`，bev_pos 被 forward 签名截获后弃用）；只有 DSA 的 self 分支注入 `query_pos=bev_pos`（`encoder.py:349-361`） | 以代码为准：pos 编码只在 stage-2 DSA 生效；画图时 pos 标注在 DSA 入口，cross 入口不画 |
| 4 | FFN 隐藏维 | 论文未给 FFN 维度 | config `ffn_cfgs` 写 `feedforward_channels=1024`（`voxformer-S.py:88,120`）但被 deprecated 顶层参数 `feedforward_channels=_ffn_dim_=256` **覆盖**（`custom_base_transformer_layer.py:81-92` 回填 ffn_cfgs）→ 生效 **128→256→128** | 以生效值 256 入册；1024 为死配置 |
| 5 | 训练轮数 | §4.1 p.6 "train stage-1 and stage-2 separately with **24 epochs**" | stage-1 24 ep（`qpn.py:117`）✓；stage-2 **20 ep**（`voxformer-S.py:199`，README:48-51 Model Zoo 亦标 20ep） | 以代码 20 ep 为准，论文 24 为笔误或不一致 |
| 6 | DSA 的采样空间 | §3.5 Eq.5 `DSA(F3D,F3D)=DA(f,p,F3D)`，p=(x,y,z) **三维** | vanilla S/T：体网格按展平索引摆到 **512×512 二维参考平面**做 2D deformable（`encoder.py:189-195`+`voxformer_head.py:119-120`），"3D 位置"仅隐含在嵌入表/pos 表的索引里；真 3D 版在 `*_deform3D.py` 变体（`deformable_self_attention_3D_custom.py:182-183,233-238`，README:51 S-3D 44.42/12.86 > S 44.02/12.35） | 主口径记 vanilla 的 2D 实现；deform3D 作为代码侧增强入差异；画图按 vanilla 画并注明 |
| 7 | pillar 采样点 | 论文无此概念（BEVFormer 遗产） | config `num_points_in_pillar=8`（`voxformer-S.py:68,106`）仅存于 encoder 属性（`encoder.py:49`），参考点实际来自 head 的格心（每 voxel 1 点/相机）；`get_reference_points` 的 '3d' 分支无调用点 | 记为死配置；不计入机制 |
| 8 | QPN 推理脚本缺陷 | — | `lmscnet.py` `foward_test` 中 `save_query_path` 赋值被注释（`lmscnet.py:312-314`），`:317` 直接使用会 NameError；生成 `.query` 文件需手工恢复注释（路径示例 `./kitti/dataset/sequences_msnet3d_sweep10/.../queries/*.query_iou5203_...`） | 复现提示；不影响机制事实 |
| 9 | 损失组成 | §3.6 Eq.6：加权 CE + scene-class affinity（引 MonoScene） | 另有 **geo-scal loss**（`voxformer-S.py:24`、`ssc_loss.py:15`）与 CE/sem_scal 并开 | 代码侧补充；HTML 脚注可提 |
| 10 | query 比例 | Table 4 p.8 消融 10–100% 固定比例（dense/random 对照），未给实际 N_p | N_p 逐帧由 QPN argmax 决定（`lmscnet.py:304-305`），无固定比例；query_tag 中 iou5203/pre7712/rec6153 为 QPN 指标标注 | 不引用任何"固定比例"值；标注 N_p=数据相关 |
| 11 | level_embeds 尺寸 | 论文未提 | (4,128)（默认 num_feature_levels=4，`transformer.py:40`）但 num_levels=1 仅用 [0] 行（`voxformer-S.py:15`） | 记入清单；画图可灰色小标 |
| 12 | 一致性对账（列出以示已核对） | §4.1 p.5-6：MobileStereoNet 深度、LMSCNet 改造 Φ_occ（输入 256×256×32→输出 128×128×16）、cam2 裁 1220×370、ResNet50 C3+FPN /16、d=128、DCA 3 层+DSA 2 层、每参考点 8 采样点、Linear 128→20、lr 2e-4 | 逐一对应：`image2depth.sh:7`（MSNet3D）、`qpn.py:43-44`+`lmscnet.py:267-268`（256×256×32→128×128×16）、`semantic_kitti_dataset_stage2.py:64-65`、`voxformer-S.py:33-48`、`:12`、`:8-11`、`:9,11`、`header.py:24`、`voxformer-S.py:188` | 一致 |

### 6.2 待画图要点（Fig.2 p.4 重绘规范）

- **Fig.2 架构重绘**（原图构图：上排 Stage-1 "Depth-based Query Proposal"（Depth Prediction 框+Voxel Queries 3D 网格→Query Proposals Qp）；下排 Stage-2 "Feature Extractor→Image Features F2D"+"Q/KV 注意力记号（DCA 与 DSA 两个 Q-K-V 环）"+"Mask Token"+"Updated Queries Qp"+"Output UP & FC"）：
  - 保留构图元素：两阶段分栏；Stage-1 的深度→proposal 因果链与 3D voxel 网格（部分格子高亮=被选中的 Qp）；Stage-2 的两个 Q-K-V 环（DCA 连图像特征、DSA 连全体 voxel）；mask token 注入箭头（空格←m）；末端 UP & FC 输出 20 类体。
  - 叠加编码（本笔记新增）：① 可学习元素实色——voxel query 表 (262144,128)、mask token (1,128)、QPN（LC 网络）、DCA/DSA 的 offsets/weights/value_proj、cams/level embeds、ResNet50+FPN、Header MLP；② 固定几何元素虚线/灰——Eq.1 深度反投影（标定内参）、log-odds 光线投射（256×256×32@0.2 m）、0.4 m 格心锚点、lidar2img 投影射线（voxel 中心→命中相机）、hit 相机选择与 1/\|V_t\| 均值、×2 三线性上采样；③ 离线/在线边界——Stage-1 整体加"离线预处理"虚线框（差异 #2），.query 文件作为离线→在线的唯一接口。
  - 关键视觉点（本分册名场面）：**深度门控稀疏 query**——左图画满 128×128×16 网格但只有 N_p 个格子着色（N_p 数据相关），右侧图例给论文 Table 4 对照数字（occupancy 44.0 IoU/14.6 G vs dense 34.6/18.5 G）；Stage-2 的 DSA 处再画"mask token 填充空格→sparse-to-dense"。
  - 一句话构图意图：让读者看到"深度把 query 推稀、标定把特征拉准、学习填满其余"——四分册里唯一由深度显式决定 query 集合的模型。
- **与前三册并排的对照条（可选）**：同用"锚点→投影→deformable 拉取"骨架的三家（BEVFormer BEV 格 4 z 锚 / TPVFormer 三平面 4/32/32 锚 / VoxFormer voxel 格心 1 锚+深度预筛），建议以"query 数量 × 激活方式"为主轴做 4 列小表（LSS 0 query 稠密 splat / BEVFormer 40000 全激活 / TPVFormer 46400 全激活 / VoxFormer 262144 表、N_p 激活）。

# 06 · Qwen-Drive-1.0（感知头 UVTR 推流）事实底稿

> 本文件为 BEV 投影对比笔记的 Qwen-Drive 分册，数字与结论的唯一来源是下方标注的 `path:line`。Qwen-Drive-1.0 侧路径一律相对仓根 `Qwen-Drive-1.0/` 书写（如 `src/qwen_drive_perception/view_transform.py`、`docs/perception.md`）；**原版 UVTR 侧一律转引 `notes/05_uvtr.md` 已核事实**（其 path:line 相对 `bev_projection/code/UVTR/`），不重查原版代码。范围限定：感知头（perception 包）中 UVTR 推流的完整链路——ViT 特征→DepthNet→视锥反投影→voxel_pool_depth 散射→双路消费（BEV query 加成＋占用支路）；BEVFormer 式拉流侧（SCA/TSA/decoder）仅作消费语境引用，逐格展开属 02 分册。时序（TSA 单帧退化）、地图/占用下游为非目标，各一句带过。仓为 inference-only 发布（`docs/perception.md:14`），无训练代码与损失——所有"无监督"结论均按仓内证据书写（见 §1、§7 轴 2）。

## 1. 元信息

| 项 | 值 |
|---|---|
| 仓库 | https://github.com/QwenLM/Qwen-Drive-1.0.git（QwenLM 官方发布仓，Apache-2.0；`git remote -v` 实测） |
| commit hash | `28091c1532e869bc7aee91fc0aef6b3e6fd0b2e0`（2026-09-03 "Update README.md"，已用 `git rev-parse HEAD` 复核，与 manifest 一致） |
| 克隆日期 | 2026-10-28（完整克隆、非浅克隆，11 commits；位于项目根 `Qwen-Drive-1.0/`，不在 `bev_projection/code/`） |
| 论文/文档 | 无随仓论文；文档来源＝`docs/perception.md`（感知头官方说明）＋代码自述。基座 VLM＝Qwen-Drive-1.0-4B（Qwen3.5），感知权重约 0.5 GB 随目录单独发布（`docs/perception.md:35-36`；`src/qwen_drive_perception/modeling_perception.py:17-19`） |
| 配置口径 | 无 config 文件体系：全部超参冻结于 `src/qwen_drive_perception/configuration_perception.py`（"All values are fixed"，`configuration_perception.py:123-124`）。核心量：`embed_dim=256`（`configuration_perception.py:103`）、`llm_dim=2560`（`:101`）、`vit_dim=1024`（`:102`）、`det_pc_range=[-51.2,-51.2,-5.0,51.2,51.2,5.4]`（`:88`）、`det_voxel_size=[102.4/200,102.4/200,10.4]`（`:89`）、`frustum_range=[0,0,1.0,896,512,60.0]`（`:98`）、`frustum_size=[16,16,0.5]`（`:99`）、bev 200×200（`:106`）、`occ_pillar_h=16`（`:107`）、`num_query=900`（`:109`）、encoder/decoder 各 6 层（`:111-112`）、检测 7 类（`:27-35`）/占用 10 类（`:38-39`）/地图 6 类（`:41-48`）、`image_size=(896,512)`（`:120`） |
| 关键派生量 | 视锥 **56×32×118**（W=896/16、H=512/16、D=(60.0−1.0)/0.5，`view_transform.py:123,146-156`；每相机 211,456 点）；深度档中心 1.0,1.5,…,59.5 m；UVTR 体素 **[16,200,200]**（Z×Y×X：X=Y=int(102.4/0.512)=200、Z=occ_pillar_h=16，`modeling_perception.py:67-73`），XY 格 **0.512 m**、Z 格 10.4/16=**0.65 m**；BEV query **200×200=40000**（`heads.py:222`）；`uvtr_query_proj` Conv2d(256×16→256, 1×1)（`modeling_perception.py:82`）；adaptor 四尺度 112×64/56×32/28×16/14×8（推算：ViT patch 格 56×32、merge 2×2 后 LLM token 格 28×16，`dataset.py:85`＋`modeling_perception.py:271,277`，乘 `fpn.py:49` 的 scale_factors）；训练双口径 nuScenes 6 相机＋OpenScene/nuPlan 8 相机、固定 896×512（`docs/perception.md:110-112`；`configuration_perception.py:15-21`） |
| **深度监督形式（重点核对项）** | **无任何深度监督（按仓内证据）**。仓为 inference-only（`docs/perception.md:14`），全仓无损失代码（grep `loss` 于 `src/qwen_drive_perception/`、`scripts/` 零命中）；数据打包仅含 calib/检测框/占用/地图/可视化 lidar，无深度 GT（`dataset.py:66-80`）；深度经 `softmax(dim=1)` 后直入 voxel_pool kernel（`modeling_perception.py:158-161`）。训练期是否曾有辅助深度损失在仓内**不可考**——底稿按仓内证据记"无"，与原版 UVTR 同为"显式分布、无监督、下游梯度回传"（原版侧证据见 §7 轴 2） |

## 2. 机制流水线（每步标注【固定几何】/【可学习】）

输入：同一 VLM 前向内的**双流特征抽头**（merger 前向钩子，`modeling_perception.py:244-248`）——① LLM 图像 token：语言模型末层 hidden_states 经 final-norm 后取 image-token 掩码、留尾部 num_cams 个相机（`modeling_perception.py:262,273-277`），每相机 28×16×2560（merge 2×2 后格点，`modeling_perception.py:271,277`）；② ViT pre-merge patch：钩得 merger 输入经 merger.norm 后按 `image_grid_thw` 还原每相机 56×32×1024 格（`modeling_perception.py:263-267`；patch 16 px、`dataset.py:85` image_grid=(32,56)、`dataset.py:115`）。相机图 bicubic resize 到固定 896×512（`dataset.py:85,137-140`）；demo 推理 B=1（`modeling_perception.py:284`）；仓内无几何增广（推理仓）。

| # | 步骤 | 形状流 | 属性 | 出处 |
|---|---|---|---|---|
| 0 | 标定准备：`lidar2img`＝viewpad(内参)@lidar2cam_rt.T（`geometry.py:43-54`）再 `apply_image_scale` 把 896×512 resize 折入投影（`geometry.py:56-62`；`dataset.py:85-100`；`docs/perception.md:105-106`）；随 `img_metas` 附 `lidar2ego`、`img_shape`、`dataset_type`（`dataset.py:101-109`） | N×(4,4) | 【固定几何】（逐样本标定量＋解析 resize） | `src/qwen_drive_perception/geometry.py:43-62`；`dataset.py:85-109` |
| 1 | LLM 主流：adaptor `SimpleFPN(2560→256, scale_factors=(4.0,2.0,1.0,0.5))` → 4 尺度 256ch @112×64/56×32/28×16/14×8（推算），reshape 成 [B,N,256,h,w] 供 SCA 拉取 | (N,28,16,2560)→[B,N,256,h,w]×4 | 【可学习·2D 侧】（主流） | `modeling_perception.py:61-63,130-142`；`fpn.py:49-83` |
| 2 | ViT 流：vit_neck `SimpleFPN(1024→256, (1.0,))` 单 stage（1×1 Conv＋LN2d＋3×3 Conv＋LN2d）→ 256ch@56×32 | (N,32,56,1024)→[B,N,256,32,56] | 【可学习·2D 侧】（UVTR 流特征源） | `modeling_perception.py:66,144-156`；`fpn.py:44-46,67-68,73-78` |
| 3 | DepthNet：reduce Conv3×3(256→256)＋GN(32)＋ReLU → [BasicBlock(256)×3 → ASPP(mid 96) → Conv1×1(256→**118**)]，`softmax(dim=1)`＝显式 118 档深度分布；**无监督**（见 §1） | →(B·N,118,32,56) | 【可学习·2D 侧】（深度分布唯一来源） | `view_transform.py:79-100`；`modeling_perception.py:83-85,158-161`；BasicBlock `layers.py:104`；ASPP `view_transform.py:47-76` |
| 4 | 视锥格点 `frustum`：**普通 attribute**（刻意不注册 buffer，`to(bfloat16)` 不 cast）、fp32 `meshgrid(arange(0,896,16), arange(0,512,16), arange(1.0,60.0,0.5))` | (56,32,118,3) | 【固定几何】（无参数 arange 网格） | `view_transform.py:136-156`；fp32 注释 `:138-143`；docstring `:15-21` |
| 5 | `coord_preparing` 反投影：齐次化→uv×d→`inverse(lidar2img)` 矩阵乘→×`lidar2ego` 回 **ego 系**→(p−pc_range)/voxel_size 取 `.int()`→[16,200,200) 范围掩码 | (B,N,1,118,32,56,4)＋掩码 | 【固定几何·fp32】（lidar2img 随 fp32 视锥 cast，`view_transform.py:175-184`） | `view_transform.py:200-239`（齐次 `:211`、uv×d `:213`、inv `:214`、ego `:217-218`、.int() `:222`、掩码 `:231-238`） |
| 6 | `voxel_pool_depth` **视锥侧查表散射**（feat_sampling）：单尺度 assert（`view_transform.py:244`）；wrapper 按体素坐标算 rank（b·X·Y·Z＋x·Y·Z＋y·Z＋z；N_cam 槽=1、相机号不进 rank）并 argsort 分区间（`ops/__init__.py:123-161`，ranks `:152-158`、区间 `:91-99`）；CUDA kernel 逐命中视锥格读特征 (h,w)×深度档 (d,h,w)、同 (b,x,y,z) 区间内求和（跨相机合并）后写散入 (B,1,200,200,16,256) | →[B,1,256,16,200,200]（permute，`view_transform.py:248`） | 【桥·参数自由】（深度×特征融合在 kernel 内单遍完成，`ops/voxel_pool/src/voxel_pool_cuda.cu:92-135`，乘积 `:130`、写散 `:134`；出参 `ops/voxel_pool/src/voxel_pool.cpp:88`） | `view_transform.py:241-248`；`ops/__init__.py:80-161` |
| 7 | `feat_encoding`：对尺寸 1 的 sweep 槽 `sum(1)` → 3× [Conv3d(256,256,(3,3,3))＋BN3d＋ReLU] → UVTR 体素 | →(B,256,16,200,200) | 求和【参数自由】；Conv3d【可学习·3D 侧】 | `view_transform.py:250-258`；conv_layer `:126-134`（num_convs=3 默认 `:114`；实例化不覆盖 `modeling_perception.py:74-81`） |
| 8 | 消费 A（检测加成）：`_uvtr_voxel_to_bev_tokens`——reshape (B,4096,200,200)→`uvtr_query_proj` 1×1→(B,256,200,200)→**batch>1 取 mean**→压平 (40000,256)，加到 `bev_embedding`（`heads.py:244-245`）；`vit_bev_feat` 备用 elif 支路在（`heads.py:246-247`）但 forward 恒传 None（`modeling_perception.py:175`） | (B,256,16,200,200)→(40000,256) | 【可学习·3D 侧】（体素→BEV query 的加性先验；压缩参数量 4096×256） | `modeling_perception.py:82,117-122`；`heads.py:242-247` |
| 9 | 消费 B（占用）：同一**未压平**体素 `uvtr_occ_space`（`modeling_perception.py:168-169,177`）→ `grid_sample` 裁到数据集 occ 范围（nuScenes/nuPlan 切换 `perception_transformer.py:153-157`）→ `uvtr_occ_proj` Conv3d(256→16,1×1)＋`uvtr_occ_fuse` Conv3d(32→16,1×1)＋BN3d＋ReLU 残差融合进 BEV 体 | (B,16,16,200,200) | 【可学习·3D 侧】 | `perception_transformer.py:85-96,191-193,256-260`；`_adapt_volume_for_occ :177-183` |
| 10 | BEV 拉流 encoder：bev_queries＝`bev_embedding`(40000,256)＋UVTR 加成、bev_pos、`level_embeds`(4,256) 注入 4 尺度（`heads.py:242-250`；`perception_transformer.py:105-121`）；6 层 BEVFormerLayer（TSA→SCA→FFN）——SCA＝4 z 锚 ego 系投影（`lidar2img@inv(lidar2ego)`，fp32、禁 TF32）＋MSDeformableAttention3D **8 头×4 层×8 点**＋scatter_add rebatch；TSA 单帧退化为普通 BEV 自注意力（时序项一句带过） | (B,40000,256) | 【可学习·3D 侧＋桥】（offsets/weights 学习量；投影锚点固定） | 层数 `configuration_perception.py:111`；层定义 `bev_encoder.py:27,91-103`；投影 `bev_encoder.py:231-286,303-316`；SCA `attention.py:211-285`（构造 `:220`）；TSA 退化 `attention.py:71-73`、`bev_encoder.py:318-323` |
| 11 | 下游：900 object query 6 层 decoder（self-attn＋CustomMSDeformableAttention 1 层 4 点＋FFN＋迭代参考点）＋cls/reg ×6 → NMSFreeCoder top-300、±61.2 m；占用＝3D U-Net refiner＋softplus 头出 200×200×16×10；地图支路裁 60×30 m@0.15 m 出 6 类 | (B,900,256)→检测/占用/地图 | 【可学习·3D 侧下游】（投影桥之外） | `perception_transformer.py:225-248,262-264`；`bev_encoder.py:350-419`；cls/reg `heads.py:206-220,268-292`；NMSFreeCoder `heads.py:53-86`（topk `:64`）＋`modeling_perception.py:113-114`；refiner `occ_refiner.py:61-107`；地图 `heads.py:302-306`、`configuration_perception.py:95-96` |

监督：无（inference-only 仓，见 §1）——深度分布仅作乘性权重在 kernel 内（步骤 6）由训练期下游损失端到端回传。

## 3. 可学习参数清单表（UVTR 支路为主，消费头概览）

> "侧"指位于投影桥（步骤 5–7）的哪一端。**UVTR 桥零参数**（视锥格点、反投影、散射 kernel）；可学习量全部在 2D 侧（特征与深度）与 3D 侧（体素编码＋两路消费）。encoder/decoder 按 6 层计。

| 参数（组） | 所属模块 | 形状（本模型口径） | 侧 | 代码出处 `path:line` |
|---|---|---|---|---|
| `adaptor`（SimpleFPN，主流 4 尺度） | BEVFormerModelV2 | 4.0 档 ConvT×2(2560→1280→640)；2.0 档 ConvT(2560→1280)；1.0 档直通；0.5 档 MaxPool；各接 1×1＋LN2d＋3×3＋LN2d 出 256 | 2D 侧（供 SCA 拉取） | `modeling_perception.py:61-63`；`fpn.py:56-78` |
| `vit_neck`（SimpleFPN，(1.0,) 单 stage） | BEVFormerModelV2 | 1×1 Conv(1024→256)＋LN2d＋3×3 Conv(256,256)＋LN2d | 2D 侧（UVTR 流） | `modeling_perception.py:66`；`fpn.py:67-68,73-78` |
| `depth_net`（DepthNet，**深度分布唯一来源、本模型的机制名片**；重深度头） | BEVFormerModelV2 | reduce Conv3×3(256→256)＋GN；BasicBlock(256)×3；ASPP(mid 96，dilation 1/6/12/18)；Conv1×1(256→**118**)，softmax | 2D 侧（无监督） | `modeling_perception.py:83-85`；`view_transform.py:82-100`（ASPP `:47-76`） |
| `view_trans.conv_layer` ×3 | Uni3DVoxelPoolDepth | 各＝Conv3d(256,256,(3,3,3),pad1)＋BN3d(256)＋ReLU | 3D 侧（落格后） | `view_transform.py:126-134`（num_convs=3 默认 `:114`） |
| `uvtr_query_proj`（体素→BEV query 的唯一压缩点） | BEVFormerModelV2 | Conv2d(256×16=4096→256, 1×1) | 3D 侧（消费 A） | 定义 `modeling_perception.py:82`；消费 `:117-122`→`heads.py:244-245` |
| `uvtr_occ_proj`＋`uvtr_occ_fuse` | PerceptionTransformer | Conv3d(256→16,1×1)；ConvModule Conv3d(32→16,1×1)＋BN3d＋ReLU（残差融合） | 3D 侧（消费 B） | `perception_transformer.py:85-96`；消费 `:191-193,256-260` |
| `bev_embedding`（BEV query 表，被 UVTR 加成） | BEVFormerHead | `nn.Embedding(200×200,256)`＝**(40000,256)**，10.24M | 3D 侧（拉流主动方） | `heads.py:222`（200×200 来自 `configuration_perception.py:106`） |
| `query_embedding` | BEVFormerHead | `nn.Embedding(900, 512)`（拆 query＋query_pos） | 3D 侧（decoder，桥外） | `heads.py:223`（900 来自 `configuration_perception.py:109`） |
| `level_embeds` | PerceptionTransformer | (4, 256)（每尺度一个） | 桥（2D 特征侧注入） | `perception_transformer.py:74`；消费 `:111` |
| `positional_encoding` | BEVFormerHead | LearnedPositionalEncoding，row/col 各 Embedding(200,128) 拼 256 | 3D 侧（仅 TSA 注入） | `heads.py:202-204,250`；`layers.py:189` |
| `reference_points` | PerceptionTransformer | Linear(256, 3)（query_pos→初始参考点） | 3D 侧（decoder） | `perception_transformer.py:76,228` |
| SCA `sampling_offsets`/`attention_weights`/`value_proj` ×6 层 | MSDeformableAttention3D | Linear(256, 8×4×8×2=512)/Linear(256, 8×4×8=256)/Linear(256,256) | 桥（拉取路径学习量） | `attention.py:161-163`；头/层/点＝(8,4,8) `attention.py:220`；外壳 output_proj Linear(256,256) `:221` |
| TSA 四个 Linear ×6 层（时序项一句带过） | TemporalSelfAttention | Linear(512,128)/Linear(512,64)/Linear(256,256)×2 | 3D 侧（单帧退化） | `attention.py:83-86`；退化语义 `:71-73` |
| encoder 每层 FFN＋LN ×6 层 | BEVFormerLayer | FFN 256→512→256＋3×LN(256) | 3D 侧 | `bev_encoder.py:95-102`；`layers.py:141` |
| decoder 6 层＋cls/reg ×6 套 | DetectionTransformerDecoder／BEVFormerHead | 每层 MHA(8 头)＋CustomMSDeformableAttention(1 层 4 点)＋FFN；cls 2×(Linear256＋LN＋ReLU)＋Linear(256→**7**)、reg 2×(Linear256＋ReLU)＋Linear(256→**10**) | 3D 侧（桥外下游） | `bev_encoder.py:355-362,366-419`；`heads.py:206-220`；7 类 `configuration_perception.py:27-35` |
| `occ_decoder`＋`occ_pred_head` | PerceptionTransformer | OccVoxelUNetRefiner(16→32)（3D U-Net，仅 XY 下采样保 Z）＋Linear(32,64)＋Softplus＋Linear(64,**10**) | 3D 侧（桥外下游） | `perception_transformer.py:78-84`；`occ_refiner.py:61-107` |
| `feat_cropper`＋`seg_decoder` | BEVFormerHead | BevFeatureSlicer（固定 grid_sample，buffer）＋MapSegEncode(256→6)（ResNet18 式） | 3D 侧（桥外下游） | `heads.py:225-226,302-306`；`heads.py:96-141`；`map_seg.py:75-93` |

要点：与原版 UVTR 对照——桥同样零参数，但 2D 侧深度头从"单 1×1 Conv 跨尺度共享"（notes/05：`detectors/uvtr.py:53-55`）加重为 BasicBlock×3＋ASPP；3D 侧新增两路"体素→消费"压缩参数（`uvtr_query_proj`/`uvtr_occ_proj`＋`uvtr_occ_fuse`），原版体素是被 decoder 直接采样的被动容器、无此类压缩。

## 4. 固定几何清单表

| 项 | 具体值 | 出处 |
|---|---|---|
| frustum_range / frustum_size | (0,0,1.0,896,512,60.0) / (16,16,0.5) | `configuration_perception.py:98-99` |
| 视锥网格 | arange 三轴 meshgrid **56×32×118**；深度档中心 1.0,1.5,…,59.5 m（0.5 m 步长、名义至 60 m 不含端点）；16 px 格对应 896×512 | `view_transform.py:146-156`；docstring `:15-21` |
| depth_dim 推导 | int((60.0−1.0)/0.5)=**118** | `view_transform.py:123` |
| det_pc_range | [-51.2, -51.2, -5.0, 51.2, 51.2, **5.4**]（X/Y ±51.2 m；Z −5.0~5.4 m，高 10.4 m） | `configuration_perception.py:88` |
| UVTR 体素 | **[200,200,16]**（X,Y,Z；Z×Y×X 记法 [16,200,200]）：X=Y=int(102.4/0.512)=200、Z=occ_pillar_h=16；XY 格 **0.512 m**、Z 格 10.4/16=**0.65 m** | `modeling_perception.py:67-73`；`configuration_perception.py:89,107` |
| 反投影链 | 齐次→uv×d→inv(lidar2img)→×lidar2ego（回 ego 系）→(p−pc_range)/voxel_size `.int()`→[0,16)×[0,200)×[0,200) 范围掩码 | `view_transform.py:210-238` |
| fp32 防护 | frustum 为普通 attribute、不随 `to(bfloat16)` cast、反投影 fp32；SCA point_sampling 全程 fp32 并临时禁 TF32 | `view_transform.py:138-143`；`bev_encoder.py:232-235,262-263` |
| SCA 投影 | ego 系：`lidar2img @ inv(lidar2ego)`；z>eps=1e-5 且 0<u,v<1 掩码（uv 已除以 img_shape）；nan_to_num 防错 | `bev_encoder.py:258-263,264-279` |
| z 锚（SCA pillar） | 4 个：linspace(0.5,9.5,4)/10.4 → 度量 **{-4.5, −1.5, 1.5, 4.5} m**（Z=pc_range[5]−pc_range[2]=10.4） | `bev_encoder.py:198-220,303-307,188` |
| BEV 网格 | 200×200、格边长 102.4/200=**0.512 m**（real_w/real_h，grid_length） | `heads.py:199-200,258` |
| 散射 rank | b·X·Y·Z＋x·Y·Z＋y·Z＋z（N_cam 槽=1、相机号不进 rank → 跨相机在区间和内合并） | `ops/__init__.py:151-158`；出参 (B,N_cam,X,Y,Z,C) `ops/voxel_pool/src/voxel_pool.cpp:88` |
| 输入分辨率 | 固定 **896×512**（bicubic resize）；ViT patch 格 56×32、LLM token 格 28×16 | `dataset.py:85,137-140`；`configuration_perception.py:120`；`modeling_perception.py:271,277` |
| 相机数 | 训练双口径 **6**（nuScenes）/**8**（OpenScene、nuPlan）；推理取尾部 num_cams 相机 | `docs/perception.md:110-112`；`modeling_perception.py:269,276,278` |
| 占用双范围 | nuScenes [-40,40]×[-1,5.4] @0.4 m；nuPlan [-50,50]×[-4,4] @0.5 m；目标格恒 200×200×16 | `configuration_perception.py:90-93`；`perception_transformer.py:153-157,165-167` |
| 地图范围 | X [-30,30]、Y [-15,15] @0.15 m（60 m×30 m） | `configuration_perception.py:95-96`；`docs/perception.md:9` |
| 解码 | NMSFreeCoder top-**300**、post_center_range X/Y ±61.2 m、Z ±10 m | `heads.py:56,64,71-73`；`modeling_perception.py:113-114` |
| 单帧约束 | prev_bev 恒 None（接口保留、忽略）、shift=[0,0] | `perception_transformer.py:211`；`bev_encoder.py:318-323` |
| 常量镜像（死配置） | NUM_HEADS/FFN_CHANNELS/NUM_FEATURE_LEVELS/POINTS_IN_PILLAR/SCA/TSA/DECODER_NUM_POINTS（`configuration_perception.py:104-105,113-117`）全仓零消费（grep 证据）；生效值＝模块默认：SCA (8,4,8)（`attention.py:220`）、z 锚 4（`bev_encoder.py:188`）、decoder 4 点（`bev_encoder.py:359`）、FFN 512（`bev_encoder.py:95`）——值恰一致，引用须引消费处 | grep 全仓仅定义处命中；消费处如上 |

## 5. 2D-3D 关系小结

- **推/拉方向：UVTR 支路＝推（2D 视锥×显式深度→scatter 落 3D 体素），检测主桥＝拉（BEVFormer 式）——"推入体素＋拉取 BEV"双桥并联**。体素支路与 LSS 同族同实现范式（2D 侧枚举视锥格、按 (b,x,y,z) rank 区间散射，BEVDet 式 CUDA scatter，`ops/voxel_pool/src/voxel_pool_cuda.cu:92-135`）；但体素不直接出框——压平 4096→256 后仅作 BEV query 的**加性先验**（`heads.py:244-245`）与占用支路的特征底座（`perception_transformer.py:191-193`），检测内容由 6 层 SCA 从 LLM 四尺度特征"拉取"（§2 步骤 10）。对比原版 UVTR：体素在那里是检测 query 的直接采样对象（uni3d_detr 参考点 grid_sample，notes/05：`models/utils/uni3d_detr.py:270-320`），在 Qwen-Drive 则降格为 BEV query 的注入项。
- **深度：显式分布、无监督**。118 档 softmax（0.5 m/档、1.0–60 m），仓内无损失代码、无深度 GT（§1）；深度仅作 kernel 内乘性权重由下游梯度端到端回传——"显式分布、隐式（下游）监督"与原版 UVTR（64 档无监督，notes/05 §5）完全同构；BEVDepth 式 LiDAR 深度监督在两版中都不存在。
- **几何与学习的分工**：桥零参数——固定几何决定"落点骨架"（视锥 arange 网格、inv(lidar2img)×lidar2ego 反投影、[16,200,200] 索引与掩码、rank 散射）；学习决定"特征与深度概率"（vit_neck＋重深度头）与"落格后如何混合、如何被消费"（3×Conv3d＋两路压缩＋BEV/占用/地图头）。即"学习给内容与概率、几何给格点与定址"——与原版 UVTR、LSS 同构。
- **query 有无及形态：体素支路无 query；BEV 侧 (40000,256) 网格嵌入表被 UVTR 加成，另有 900 object query（桥外）**。体素格点是被动容器（与原版 UVTR 相同）；差异在消费端——原版"900 query 从体素体拉取"，Qwen-Drive"体素压平后加成 BEV query、检测拉流改在 LLM 特征上进行"。全篇对比中 Qwen-Drive＝四模型之外的第五种组合：LSS 式推流（实现层）＋BEVFormer 式拉流（检测层）并联，显式深度只服务推流支路。

## 6. 文档 vs 代码差异点

### 6.1 差异与勘正记录

| # | 项 | 文档/自述说法 | 代码事实 | 处理 |
|---|---|---|---|---|
| 1 | BEV 格边长 | `perception_transformer.py:19-20` 自述 "200x200 grid, **10.4 cm** cells" | 实际 102.4/200=**0.512 m**（`heads.py:199-200,258`；`configuration_perception.py:89`）——疑把 Z 高 10.4 m 误写进格边长 | 以代码为准：0.512 m；底稿与 HTML 不引 "10.4 cm" |
| 2 | "UVTR voxel pooling" 名义 | `docs/perception.md:18`："view_transform.py DepthNet and UVTR voxel pooling" | 实现方向＝LSS/BEVDet 式**视锥侧 scatter**（`view_transform.py:241-248`＋CUDA kernel）；原版 UVTR 是**3D 体素格点 gather**（F.grid_sample，notes/05 §5） | 行文避免"照搬 UVTR 投影"；对比表按 §7 轴 4 如实呈现 scatter/gather 差异 |
| 3 | 配置常量镜像 | `configuration_perception.py:104-105,113-117` 列有 NUM_HEADS/FFN_CHANNELS/NUM_FEATURE_LEVELS/POINTS_IN_PILLAR/SCA/TSA/DECODER_NUM_POINTS | 全仓零消费（grep）；生效值来自模块默认参数（`attention.py:154,220`；`bev_encoder.py:95,188,359`），数值恰好一致 | 引用一律引消费处；常量行标注"镜像死配置" |
| 4 | 深度监督可考性 | — | 仓为 inference-only（`docs/perception.md:14`），无损失代码（grep 零命中）、无深度 GT（`dataset.py:66-80`）；训练期是否曾有辅助损失**不可考** | §1/§7 按仓内证据记"无"，并明示"训练期不可考"限定语 |
| 5 | 深度范围精确化 | `view_transform.py:17-19` 自述 "0.5 m depth steps up to 60 m (118 bins)" | 起点 **1.0 m**（frustum_range z0=1.0，`configuration_perception.py:98`）、末档中心 59.5 m（`view_transform.py:123,149-151`） | 面板与行文写"118 档、0.5 m/档、1.0–60 m" |
| 6 | vit_bev_feat 支路 | `heads.py:246-247` 存在 elif 备用加成 | `modeling_perception.py:175` 恒传 None——未用代码路径 | 记录备查，不进图 |
| 7 | 单帧 TSA 退化 | `docs/perception.md:113-114`："temporal self-attention degenerates to plain BEV self-attention" | `attention.py:71-73`（BEV 队列 [query,query]）＋`bev_encoder.py:318-323`（prev_bev None、shift=[0,0]） | 一致（列入以示已核对；时序项一句带过） |
| 8 | lidar2img 折入 resize | `docs/perception.md:105-106`："lidar2img folds the 896x512 resize into the projection" | `geometry.py:56-62` apply_image_scale 左乘缩放；`dataset.py:88-100` 逐相机构造 | 一致 |
| 9 | 输出三项口径 | `docs/perception.md:5-9`：300 框/7 类、200×200×16/10 类、60×30 m/6 类 | NMSFreeCoder top-300（`heads.py:56,64`）、occ 格（`perception_transformer.py:165-167`）、map（`configuration_perception.py:95-96`）逐项吻合 | 一致 |

### 6.2 与 02/05 分册的引用边界

BEV 拉流侧（SCA/TSA/decoder/z 锚投影）的逐格展开引用 02 分册口径但**数字一律以本仓为准**（z 范围不同：Qwen-Drive Z −5.0~5.4、z 锚 {-4.5,−1.5,1.5,4.5} m，非 02 的 8 m/4 锚）；原版 UVTR 侧一律转引 notes/05。HTML 各节只引用本底稿与 notes/05，不重查代码。

## 7. 原版 UVTR vs Qwen-Drive 五轴对比素材

> 原版侧：repo JIA-Lab-research/UVTR@`8a24ec2`（notes/05 §1），path:line 转引自 notes/05（相对 `bev_projection/code/UVTR/`）。Qwen-Drive 侧：本仓 `28091c1`。

| 轴 | 原版 UVTR（notes/05 转引） | Qwen-Drive-1.0 |
|---|---|---|
| 1. 特征源（2D 侧） | ResNet-50＋FPN **4 尺度** 256ch（/8–/64），深度由**单个 1×1 Conv(256→64) 跨尺度共享**产生、每尺度一份（notes/05：`detectors/uvtr.py:53-55,204-209`；`uvtr_camera_base.py:40-58`） | UVTR 流＝VLM ViT pre-merge patch 1024→vit_neck **单尺度** 256ch@56×32（`modeling_perception.py:66,152`；`fpn.py:44-46`），深度由重深度头 BasicBlock×3＋ASPP(96)＋1×1 出 118 档（`view_transform.py:82-100`；`modeling_perception.py:83-85`）；另有原版没有的 LLM 主流 2560→4 尺度供 SCA 拉取（`modeling_perception.py:61-63`） |
| 2. 深度监督 | **无**：论文 p.4 "D_I is predicted without supervision"＋代码无深度 loss（形参穿透未用 `uvtr_head.py:407-408`）、管线不产深度 GT（`uvtr_camera_base.py:197-211`）（notes/05 §1/§6.1#2） | **无**（按仓内证据）：inference-only 无损失代码（`docs/perception.md:14`；grep 零命中）、无深度 GT（`dataset.py:66-80`）；softmax 直入 kernel（`modeling_perception.py:158-161`）。训练期不可考（§6.1#4）——两侧同为"显式分布、下游梯度回传" |
| 3. 离散化 | 深度 **D=64 档@1 m、0–64 m**（`uvtr_camera_base.py:62`；`uni3d_viewtrans.py:316-323`）；体素 **128×128×5**（XY 名义 0.8 m、实际间距 102.4/127≈0.806 m；Z linspace 含端点 {-5,-3,-1,1,3} m、间距 2 m）（`uvtr_camera_base.py:12,15-19`；`uni3d_viewtrans.py:199-202`） | 深度 **118 档@0.5 m、1.0–60 m**（`configuration_perception.py:98-99`；`view_transform.py:123`）；体素 **[16,200,200]**（XY 0.512 m、Z 0.65 m、范围 ±51.2 m/Z −5.0~5.4）（`modeling_perception.py:67-73`；`configuration_perception.py:88-89,107`）——更细深度×更粗 Z、更大 XY 覆盖 |
| 4. 消费方式＋实现 | **体素内检测**：900 query UniCrossAtten 参考点 grid_sample 从体素体拉取（num_points=1 sigmoid 门控；`uni3d_detr.py:270-320`；`uvtr_camera_base.py:80,103`）；视图变换实现＝**gather**（3D 体素格点投影→F.grid_sample 取特征与深度概率、按概率加权求和；`uni3d_viewtrans.py:279-359,215-226`；论文 p.15 naive grid sampling） | **压平加成 query＋占用支路**：体素 [B,256,16,200,200]→Conv2d(4096→256,1×1) 压平 [40000,256] 加到 bev_embedding（`heads.py:244-245`；`modeling_perception.py:82,117-122`）＋未压平体素进占用融合（`perception_transformer.py:85-96,191-193,256-260`）；检测头＝BEVFormer 式（6 层 SCA 8头×4层×8点、4 z 锚，从 LLM 特征拉取；`attention.py:220`；`bev_encoder.py:188,303-316`）；视图变换实现＝**scatter**（视锥侧查表：kernel 读特征 (h,w)×深度 (d,h,w)、rank 区间和写散入体素；`view_transform.py:241-248`；`ops/voxel_pool/src/voxel_pool_cuda.cu:92-135`） |
| 5. 工程细节 | fp16 桥内半精度（`uni3d_viewtrans.py:291-294`；`uvtr_camera_base.py:77`）；4 尺度并行；mmdet3d v0.17.3 依赖（notes/05 §1：`README.md:20`）；lidar 系＋uni_rot_aug 逆增广（`uni3d_viewtrans.py:275-276`）；输入 900×1600 pad 928×1600（`uvtr_camera_base.py:208`） | 视锥保 **fp32**（普通 attribute 防 bf16 cast，`view_transform.py:138-143`）＋SCA 投影 fp32 禁 TF32（`bev_encoder.py:232-235`）＋自研 bf16 deformable kernel（`attention.py:26-28,54-58`）；**单尺度 assert**（`view_transform.py:244`）；无 mmcv/mmdet 依赖（`docs/perception.md:14`）；**ego 系锚定**（`bev_encoder.py:192,258-263`；`view_transform.py:217-218`）、仓内无增广；batch>1 BEV token 取 mean（`modeling_perception.py:120-121`）；输入固定 896×512（`dataset.py:85`）；双数据集 6/8 相机（`docs/perception.md:110-112`） |

## 8. B 面板数字白名单（Qwen-Drive/UVTR 推流面板允许出现的全部机制数字，一律出自本底稿）

- 深度：**118 档、步长 0.5 m、范围 1.0–60 m**（末档中心 59.5 m）；
- 视锥：**56×32×118**、**16 px 格**、**896×512**；
- 体素：**[16,200,200]**（Z×Y×X）、XY 格 **0.512 m**、Z 格 **0.65 m**、范围 X/Y **±51.2 m**、Z **−5.0~5.4 m**；
- 特征与采样：**256 通道**、ViT **1024→256**、LLM **2560→256**、主流 **4 尺度**（112×64/56×32/28×16/14×8，推算）、**6 或 8 相机**（双口径，标明数据集）；
- 消费：**4096→256（1×1 Conv）**、bev **200×200=40000**、**900 query**、**6 层 encoder**、z 锚 **{-4.5,−1.5,1.5,4.5} m**、SCA **8 头×4 层×8 点**、**3×Conv3d 3³**、占用 **200×200×16、10 类**（nuScenes ±40 m / nuPlan ±50 m 两态可标）；
- 输入：**896×512**；
- 两态语义约束：「Qwen-Drive 码放」＝118 档深度加权的视锥点查表散射落 [16,200,200] 体素、压平后**加成** BEV query 并喂占用支路（不是 query 从体素采样）；散射态的体素网格用 Qwen-Drive 自己的 [16,200,200]@0.512 m，不得引入原版 UVTR 的网格数字；
- 禁入数字：**64、128×128×5、{-5,-3,-1,1,3}、0.8 m、1 m 步长、0–64 m、928×1600、900×1600**（原版 UVTR/BEVFormer 节口径）、**41、105**（LSS 节）——面板任何文案不得出现（两模型同屏对比时按对比节自身白名单另行约定）。

# 02 · BEVFormer（v1）事实底稿

> 本文件为 BEV 投影对比笔记的 BEVFormer 分册，数字与结论的唯一来源是下方标注的 `path:line`（相对于 `bev_projection/code/BEVFormer/`）与论文原文（arXiv:2203.17270，ECCV 2022 版 PDF，附录 A–D，**不含 v2 内容**）。后续 HTML 各节只引用本底稿，不重查代码。范围限定 v1：repo master 同时含有 BEVFormerV2（`projects/configs/bevformerv2/`、`projects/mmdet3d_plugin/bevformer/detectors/bevformerV2.py`），一律不引用。时序机制（TSA、can_bus 对齐）为笔记非目标，仅各一句带过。

## 1. 元信息

| 项 | 值 |
|---|---|
| 仓库 | https://github.com/fundamentalvision/BEVFormer |
| commit hash | `66b65f3a1f58caf0507cb2a971b9c0e7f842376c`（2024-05-03 "Update README.md"，已用 `git rev-parse HEAD` 复核，与 manifest 一致） |
| 克隆日期 | 2026-10-26（`--depth 1` 浅克隆，见 `00_manifest.md`） |
| 论文 | Li et al., *BEVFormer: Learning Bird's-Eye-View Representation from Multi-Camera Images via Spatiotemporal Transformation*, ECCV 2022；`papers/bevformer_2203.17270.pdf`（20 页版本；投影机制在 §3.2–§3.3 p.5、总架构 §3.1 p.4、取值 §4.2 p.7、SCA 附录 A.4 p.16） |
| nuScenes 配置口径 | `projects/configs/bevformer/bevformer_base.py`。核心量：`_dim_=256`（`bevformer_base.py:31`）、`_num_levels_=4`（`bevformer_base.py:34`）、`bev_h_=200`、`bev_w_=200`（`bevformer_base.py:35-36`）、`queue_length=4`（`bevformer_base.py:37`）、`point_cloud_range=[-51.2,-51.2,-5.0,51.2,51.2,3.0]`（`bevformer_base.py:11`）、encoder `num_layers=6`（`bevformer_base.py:80`）、`num_points_in_pillar=4`（`bevformer_base.py:82`）、SCA `num_points=8`、`num_levels=4`（`bevformer_base.py:97-98`）、`num_query=900`（`bevformer_base.py:66`）；`small.py`/`tiny.py` 同目录不引用 |
| 关键派生量（nuScenes 口径） | BEV query 数 **200×200=40000**、格边长 102.4/200=**0.512 m**（`bevformer_head.py:61-62,149-150`）；pillar 高 Z=3.0−(−5.0)=**8 m**、每 pillar 4 个 z 锚点（`encoder.py:189` + `bevformer_base.py:82`）；输入 900×1600（`bevformer_base.py:185`），pad 到 32 整除→928×1600（`bevformer_base.py:174`）；FPN 四层特征 256ch @ /8、/16、/32、/64（`bevformer_base.py:54-61`）；每 BEV query 每命中相机采样位置数 8head×4level×8point=**256**（见 §2 步骤 8） |

## 2. 机制流水线（每步标注【固定几何】/【可学习】）

输入：`(B,6,3,928,1600)`（B=1/GPU，`bevformer_base.py:198`；原图 900×1600 `bevformer_base.py:185`，`PadMultiViewImage` size_divisor=32 `bevformer_base.py:174`）。BEVFormer 不做任何会改写 lidar2img 的 2D 几何增广（pipeline 仅光度+归一化+pad，`bevformer_base.py:167-177`；`RandomScaleImageMultiViewImage` 存在但不在 base pipeline，`datasets/pipelines/transform_3d.py:291-324`），GridMask 在模型内做（`detectors/bevformer.py:52-53,82-83`）。

| # | 步骤 | 形状流 | 属性 | 出处 |
|---|---|---|---|---|
| 0 | 标定准备：`lidar2img`（每相机 4×4）由 `sensor2lidar` 外参求逆再左乘内参 K 构成，随 `can_bus`（18 维）一起进 `img_metas` | 6×(4,4)；18 | 【固定几何】（逐样本标定量） | `datasets/nuscenes_dataset.py:129-138,147`；`datasets/pipelines/transform_3d.py:246-254` |
| 1 | 图像主干 ResNet-101（stage3/4 带 DCNv2，FCOS3D 初始化）：取 C3/C4/C5 | (B·6,3,928,1600)→512ch@116×200、1024ch@58×100、2048ch@29×50（/8、/16、/32，由 pad 后尺寸推算） | 【可学习·2D 侧】 | `bevformer_base.py:43-53,249`；`detectors/bevformer.py:85` |
| 2 | FPN：4 层输出（含 `add_extra_convs='on_output'` 补出的 /64 层），全部 256ch | →256ch@116×200、58×100、29×50、15×25（/8、/16、/32、/64） | 【可学习·2D 侧】 | `bevformer_base.py:54-61`；`detectors/bevformer.py:90-91` |
| 3 | 展平+嵌入注入：每层特征 flatten 后加 `cams_embeds`（6,256）与 `level_embeds`（4,256），拼成 `feat_flatten` | →(6, 30825, B, 256)（30825=23200+5800+1450+375） | 【可学习·桥侧嵌入】 | `modules/transformer.py:164-184` |
| 4 | BEV query 组装：`bev_embedding`（40000,256）+ `can_bus_mlp(can_bus)` 注入（时序项，一句带过）+ 可学习位置编码 `bev_pos` | →(40000,B,256)；bev_pos:(B,256,200,200) | 【可学习·3D 侧 query】 | `dense_heads/bevformer_head.py:104-105,136-141`；`modules/transformer.py:119,159-162` |
| 5 | 3D 参考点生成（固定网格）：`linspace(0.5, N-0.5, N)/N` 得归一化格中心；每格 4 个 z 锚点 `linspace(0.5,7.5,4)/8` | ref_3d:(B,4,40000,3)，取值 [0,1]³ | 【固定几何】（无参数 linspace 网格） | `modules/encoder.py:46-71,188-189` |
| 6 | 投影 `point_sampling`：ref_3d 反归一到 pc_range 度量坐标→齐次→`lidar2img` 矩阵乘→uv 除以 img_shape 归一化→`bev_mask`（z>eps 且 0<u,v<1） | →reference_points_cam:(6,B,40000,4,2)、bev_mask:(6,B,40000,4) | 【固定几何·参数自由】（纯标定投影，无学习深度） | `modules/encoder.py:95-99,102-110,122-131,133-136,143-144` |
| 7 | TSA（时序自注意力，一句带过）：当前 query 与经位移/旋转对齐的历史 BEV 做 deformable attention；`bev_pos` 仅在此注入 | (B,40000,256) | 【可学习·时序】（非本笔记目标） | `modules/encoder.py:196-212,356-374`；`modules/temporal_self_attention.py` |
| 8 | SCA（空间交叉注意力，核心桥）：仅命中相机参与——按 bev_mask 逐相机挑出命中 query 重排（rebatch）；`MSDeformableAttention3D` 内 `value_proj` 投影 value，`sampling_offsets`（256→512）与 `attention_weights`（256→256，对 level×point=32 维 softmax）由 query 预测；offset 除以各层 (W,H) 归一化后按 4 个 z 锚点拆分（每锚点每头每层 2 个）；`采样位置=投影点+offset`，四层特征双线性采样（CUDA kernel）；结果 scatter 回原 query 序并除以命中相机数（论文 Eq.2 的 1/\|V_hit\|） | 每 query 每命中相机：8head×4level×8point=256 个采样位置；→(B,40000,256) | 【可学习·桥】（offsets/weights 是学习量；投影锚点与 hit 判定是固定几何） | `modules/spatial_cross_attention.py:134-141,143-153,162-164,165-172`；同文件 338-348,357-369,390-395 |
| 9 | 输出投影+残差+FFN：`output_proj`(256→256)+dropout+残差，随后 norm/FFN/norm（FFN 256→512→256） | (B,40000,256) | 【可学习·3D 侧】 | `modules/spatial_cross_attention.py:173-175`；`bevformer_base.py:102-105`；`modules/custom_base_transformer_layer.py:151-163` |
| 10 | 步骤 7–9 按 `operation_order=('self_attn','norm','cross_attn','norm','ffn','norm')` 堆叠 6 层 → `bev_embed` (B,40000,256) | (B,40000,256) | 【可学习·3D 侧】（层数为配置值） | `bevformer_base.py:80,104-105`；`modules/encoder.py:214-239` |
| 11 | 下游：decoder（900 object query，CustomMSDeformableAttention，BEV 单层）+ cls/reg 头出 3D 框与速度 | (B,900,256)→检测输出 | 【可学习·3D 侧下游】（投影桥之外） | `modules/transformer.py:263-289`；`bevformer_base.py:106-127` |

## 3. 可学习参数清单表（nuScenes 口径，bevformer_base.py）

> "侧"指参数位于投影桥的哪一端。encoder 侧（TSA/SCA/FFN）按 6 层计；SCA 内 deformable 子模块 heads=8、levels=4、points=8 来自类默认+配置（`modules/spatial_cross_attention.py:204-213`；`bevformer_base.py:97-98`）。

| 参数（组） | 所属模块 | 形状（nuScenes 口径） | 侧 | 代码出处 `path:line` |
|---|---|---|---|---|
| `bev_embedding`（BEV query 表，本模型的名片参数） | BEVFormerHead | `nn.Embedding(200×200, 256)`=**(40000,256)**，10.24M | 3D 侧（桥的主动方） | `dense_heads/bevformer_head.py:104-105`（200×200 来自 `bevformer_base.py:64-65`） |
| 位置编码 `row_embed`+`col_embed`（mmdet `LearnedPositionalEncoding`，外部库类，由 head 配置构建） | BEVFormerHead | 各 `Embedding(200,128)`，输出拼成 256 维 bev_pos | 3D 侧（仅注入 TSA，见 §2 步骤 7） | `bevformer_base.py:135-140`；消费点 `dense_heads/bevformer_head.py:139-141`、`modules/encoder.py:365-366` |
| `query_embedding` | BEVFormerHead | `nn.Embedding(900, 512)`（拆成 query+query_pos） | 3D 侧（decoder，桥之外） | `dense_heads/bevformer_head.py:106-107`（900 来自 `bevformer_base.py:66`） |
| `cams_embeds` | PerceptionTransformer | (6, 256)（每相机一个可学习向量，加到所有特征层） | 桥（2D 特征侧注入） | `modules/transformer.py:74-75,170-171` |
| `level_embeds` | PerceptionTransformer | (4, 256)（每尺度一个） | 桥（2D 特征侧注入） | `modules/transformer.py:72-73,172-173` |
| `can_bus_mlp`（can_bus→位置偏置注入；时序项） | PerceptionTransformer | Linear(18,128)+ReLU+Linear(128,256)+ReLU+LN(256) | 3D 侧 query 注入 | `modules/transformer.py:77-84,159-162` |
| SCA `sampling_offsets`（**可学习采样偏移，deformable 核心**）×6 层 | MSDeformableAttention3D | `Linear(256, 8×4×8×2=512)`；权重零初始化、bias 为放射状网格初始化 | 桥（拉取路径上的学习量） | `modules/spatial_cross_attention.py:245-246,255-267` |
| SCA `attention_weights` ×6 层 | MSDeformableAttention3D | `Linear(256, 8×4×8=256)`，softmax 于 level×point=32 维 | 桥 | `modules/spatial_cross_attention.py:247-248,268,340-343` |
| SCA `value_proj` ×6 层 | MSDeformableAttention3D | `Linear(256, 256)` | 桥（2D 特征侧） | `modules/spatial_cross_attention.py:249,334` |
| SCA 外壳 `output_proj` ×6 层（子模块内 output_proj 为 None） | SpatialCrossAttention | `Linear(256, 256)` | 桥（聚合后） | `modules/spatial_cross_attention.py:67,173,221` |
| TSA 四个 Linear（时序项，一句带过）×6 层 | TemporalSelfAttention | `sampling_offsets Linear(512,128)`、`attention_weights Linear(512,64)`、`value_proj/output_proj Linear(256,256)` | 3D 侧（时序） | `modules/temporal_self_attention.py:99-104`（points=4、levels=1：`temporal_self_attention.py:59`、`bevformer_base.py:90`） |
| encoder 每层 FFN+LN ×6 层 | BEVFormerLayer | FFN 256→512→256（2 个 Linear）+3×LN(256) | 3D 侧 | `bevformer_base.py:102-105`；`modules/custom_base_transformer_layer.py:151-163` |
| decoder `reference_points` 及 6 层 decoder（MultiheadAttention+CustomMSDeformableAttention+FFN）、cls/reg 分支 ×6 | DetectionTransformerDecoder / BEVFormerHead | `Linear(256,3)`；cls 256→…→11、reg 256→…→10 | 3D 侧（桥之外下游） | `modules/transformer.py:76,267-268`；`bevformer_base.py:106-127`；`dense_heads/bevformer_head.py:71-101` |
| `img_backbone`+`img_neck`（ResNet-101-DCN+FPN，FCOS3D 预训练） | BEVFormer | 见 §2 步骤 1–2 | 2D 侧（拉取之前） | `bevformer_base.py:43-61,249` |

要点：**投影桥上的学习量=SCA 的 offsets/weights/两个投影矩阵**（每层 328,960 参数，6 层约 1.97M，逐项见上）+ 两侧注入嵌入（cams/level embeds）；而"在哪采"的骨架——pillar 网格、z 锚点、lidar2img 投影、hit 判定——全部零参数（见 §4）。论文 Tab.5 计 BEVFormer-S 68.7M 参数（p.10，论文值、未在代码侧复算）。

## 4. 固定几何清单表

| 项 | 具体值 | 出处 |
|---|---|---|
| BEV 网格 bev_h×bev_w | 200×200，格边长 s=102.4/200=**0.512 m**（real_w=real_h=102.4） | `bevformer_base.py:35-36`；`dense_heads/bevformer_head.py:61-62`；grid_length 传参 `bevformer_head.py:149-150` |
| 感知范围 pc_range | [-51.2, -51.2, -5.0, 51.2, 51.2, 3.0]（X/Y ±51.2 m；Z −5~3 m） | `bevformer_base.py:11` |
| pillar 高度 Z | 8.0 m（pc_range[5]−pc_range[2]） | `modules/encoder.py:189` |
| 每 pillar 参考点数 num_points_in_pillar | 4（=论文 N_ref） | `bevformer_base.py:82`；`modules/encoder.py:36,42` |
| z 锚点位置 | `linspace(0.5,7.5,4)/8`→归一化 {0.0625, 0.3542, 0.6458, 0.9375}→度量 **{−4.5, −2.167, 0.167, 2.5} m**（区间内 4 个等距点，不含端点） | `modules/encoder.py:62-63,102-107` |
| xy 参考点位置 | `linspace(0.5, N-0.5, N)/N` 格中心→度量 (i+0.5)·0.512−51.2，i=0..199（即 ±50.944 m 内） | `modules/encoder.py:64-67,102-105` |
| lidar2img 来源 | `lidar2img = K(4×4) @ (sensor2lidar 求逆)ᵀ`，纯标定量，无增广改写、无学习修正 | `datasets/nuscenes_dataset.py:129-138`；随 meta 下发 `datasets/pipelines/transform_3d.py:246-254` |
| 投影与 hit 判定 | 齐次化→4×4 矩阵乘→z>eps=1e-5 且 0<u<1、0<v<1（uv 已除以 img_shape）；nan_to_num 防错 | `modules/encoder.py:109-110,122-131,124,133-138` |
| hit 粒度 | query×相机级：4 个 z 锚点任一命中即该相机参与该 query（rebatch 用 any 语义；count=命中相机数 clamp≥1，实现论文 Eq.2 的 1/\|V_hit\|） | `modules/spatial_cross_attention.py:137-141,169-172` |
| offset 归一化基准 | 各特征层 (W,H)（`offset_normalizer`）——学习偏移以特征格为单位、作用在 [0,1] uv 坐标上 | `modules/spatial_cross_attention.py:357-363` |
| 图像输入分辨率 | 900×1600，pad 到 32 整除→928×1600（size_divisor=32） | `bevformer_base.py:185,174,198` |
| 多尺度特征层（推算） | /8:116×200、/16:58×100、/32:29×50、/64:15×25，各 256ch（out_indices=(1,2,3)+FPN num_outs=4） | `bevformer_base.py:47,54-61` |
| 相机数 | 6（cams_embeds 6；SCA num_cams 默认 6；论文 Fig.2(a) "×6"） | `modules/transformer.py:74`；`modules/spatial_cross_attention.py:47` |
| 时序对齐量（非学习、时序项一句带过） | shift 由 can_bus 位移差解析算出（transformer.py:122-141）；历史 BEV 按 can_bus 航向旋转（transformer.py:143-156）；`encoder.py:196-198` 有作者自注 "keep this bug for reproducing" 的 clone 注释（仅涉及时序 shift 路径） | `modules/transformer.py:122-156`；`modules/encoder.py:196-198` |
| GridMask 增广 | use_grid_mask=True（prob 0.7，随机几何遮挡，非学习） | `bevformer_base.py:41`；`detectors/bevformer.py:52-53,82-83` |
| 训练超参（供 HTML 脚注） | AdamW lr=2e-4、backbone lr_mult=0.1、wd=0.01、cosine、24 epochs、每 GPU batch=1（6 视图） | `bevformer_base.py:228-245,198`（论文 A.1 p.16 一致） |

## 5. 2D-3D 关系小结

- **推/拉方向：拉（pull）**。BEVFormer 的主动方是 3D 侧的可学习 BEV query：先把每格 query 抬成 pillar、取 4 个固定 3D 参考点，经 lidar2img 投到 2D，再从命中相机的多尺度特征上"拉取"邻域特征（deformable 采样）。与 LSS（2D 侧推、scatter 落格）方向相反；投影方向本身是 3D→2D 定锚、特征流是 2D→3D 聚合。
- **深度：隐式、无深度分布**。不预测任何 per-pixel 深度（对比 LSS 的 41 维 α）；深度不确定性由 4 个固定 z 锚点离散枚举，由 attention 权重隐式加权，无深度监督、无深度输出头。学习量只在采样偏移与权重上修正锚点邻域。
- **几何与学习的分工**：固定几何决定"候选落点"——BEV 网格 200×200@0.512 m、z 锚点 {−4.5,−2.167,0.167,2.5} m、lidar2img 投影、hit 相机掩码、四层尺度归一化；学习决定"query 内容"（bev_embedding 40000×256）与"采多远、权重多少"（SCA sampling_offsets/attention_weights）。即"几何给锚点、学习给偏移"。
- **query 有无及形态：有，且是网格形嵌入表**。BEV query 是一张 (40000,256) 的可学习嵌入表，逐格与 BEV 平面一一对应（论文 §3.2 "grid-shaped learnable parameters"），另有 (900,512) 的 object query 供 decoder 用（桥之外）。这与 LSS 的"无 query、固定视锥点"构成四模型对比中的两极。

## 6. 论文 vs 代码差异点 + 待画图要点

### 6.1 差异与勘正记录

| # | 项 | 论文说法 | 代码事实 | 处理 |
|---|---|---|---|---|
| 1 | 机制图编号 | 任务线索"BEVFormer 图 2" | 论文 **Fig.2 即总架构图**，含 (a) encoder 层组成、(b) SCA、(c) TSA 三幅子图（p.4）——与 LSS 不同，此处图号无误 | 重绘对象直接取 Fig.2(a)+(b) |
| 2 | 多尺度层数 | §4.2："FPN with sizes of 1/16, 1/32, 1/64"（3 层） | FPN `num_outs=4` 且 SCA `num_levels=4`，实际消费 **/8、/16、/32、/64 四层**（`bevformer_base.py:34,60,98`） | 以代码为准：四层；论文少写了 /8 层。画图标 4 层 |
| 3 | 每参考点采样数 | §4.2："For each reference point on 2D view features, we use **four** sampling points … for each head" | `num_points=8`（每头每层），经 reshape 按 4 个 z 锚点拆分→**每锚点每头每层 2 个**采样点（`bevformer_base.py:97`；`modules/spatial_cross_attention.py:360-369`）；每锚点每头跨 4 层共 8 个 | 以代码为准：8 点/头/层，摊到 4 锚点各 2；论文"4 点"与发布代码数值对不上，疑为早期版本口径 |
| 4 | z 锚点取值 | §4.2："height anchors are sampled uniformly from −5 meters to 3 meters" | linspace 给开区间内 4 个等距点 {−4.5,−2.167,0.167,2.5} m，**不含 −5/3 端点**（`modules/encoder.py:62-63`） | 论文表述含糊而非矛盾；画图按代码值标 4 个点 |
| 5 | 投影矩阵形式 | Eq.4：`T_i ∈ R^{3×4}`，`z·[u,v,1]ᵀ = T_i·[x′,y′,z′,1]ᵀ` | 用 4×4 齐次矩阵乘后除以 z（`modules/encoder.py:109-128`），数学等价 | 非差异；画图用 3×4 齐次形式即可 |
| 6 | Eq.2 归一化 | `1/\|V_hit\| ΣΣ DeformAttn` | slots 汇总后除以命中相机数（clamp≥1）（`modules/spatial_cross_attention.py:169-172`），V_hit=该 query 4 锚点任一落入视锥的相机集合 | 一致；hit 判定为 any-of-4-anchors 语义，可在图注说明 |
| 7 | 位置编码消费点 | §3.2：位置编码加到 BEV query 上（未区分两个 attention） | `bev_pos` 仅作为 query_pos 注入 **TSA**；SCA 调用时 query_pos=None，不加（`modules/encoder.py:365-366` vs `382-397`） | 代码更细：画图把位置编码标注在时序支路入口 |
| 8 | can_bus 注入 | 论文正文未提 can_bus MLP（§3.4 只说 ego-motion 对齐） | `can_bus_mlp`（18→128→256）把 CAN 信号加进每帧 bev_queries（`modules/transformer.py:77-84,159-162`） | 代码侧补充机制，时序项一句带过，不进投影主图 |
| 9 | 时序 shift 的自注 bug | — | `encoder.py:196-198` 注释"this code should be … we keep this bug for reproducing our results"（clone 与 += 的写法），仅影响 TSA 参考点 shift | 记录备查；不影响 SCA 投影路径 |
| 10 | 参数量 | Tab.5：BEVFormer-S **68.7M**（p.10） | 仓库无复算脚本；结构一致但未逐参数复算 | 底稿引用 68.7M 时标"论文值" |
| 11 | 900 query/300 框/24ep/lr2e-4/backbone×0.1 | §A.3 p.16、§A.1 p.16、§4.2 p.7 | `bevformer_base.py:66,132,245,228-235` 逐项吻合 | 一致（列入以示已核对） |
| 12 | 6 层 encoder、200×200、±51.2 m、s=0.512、N_ref=4、C=256 | §3.1 p.4、§4.2 p.7 | `bevformer_base.py:80,35-36,11,82,31` + `bevformer_head.py:61-62` 逐项吻合 | 一致 |

其余核对一致项：Eq.3 格→世界坐标（ego 居中，代码取格中心，`modules/encoder.py:64-67,102-105` ↔ §3.3 p.5）、hit-view 机制（§3.3/A.4 ↔ `modules/spatial_cross_attention.py:137-141`）、GridMask 为代码侧训练细节（论文未提，`bevformer_base.py:41`）。

### 6.2 待画图要点（Fig.2(a)+(b) 重绘规范）

- **Fig.2(a) 总架构重绘**：
  - 保留构图元素：6 视图输入（"×6"）→ Backbone → 多相机特征 F_t；encoder 层内序 = TSA（接历史 BEV B_{t−1}）→ SCA（标出 V_hit）→ FFN，配 Add&Norm ×3；网格形 BEV Queries Q；6 层堆叠 → 当前 BEV B_t → Det & Seg 头。
  - 叠加编码（本笔记新增）：① 可学习元素实色——BEV query 表 (40000,256)、bev_pos(200×200)、cams/level embeds、SCA offsets/weights、FFN；② 固定几何元素虚线/灰——lidar2img、hit 判定、pillar 网格与 z 锚点；③ 注记 200×200@0.512 m、6 层、4 尺度 /8–/64。
  - 一句话构图意图：让读者看到"query 是学的、锚点是标的"。
- **Fig.2(b) SCA 机制重绘**：
  - 保留构图元素：BEV 格 (x,y)→pillar (x′,y′,z′_j)（4 个高度点）→投影射线到各相机像平面→命中视图 V_hit 与未命中视图→命中点周围采样点（带偏移）→加权求和。
  - 叠加编码：pillar 网格与 z 锚点 {−4.5,−2.167,0.167,2.5} m 用固定几何样式；采样点偏移箭头与权重用可学习样式（2 点/锚点/头/层×4 层）；标注 1/|V_hit| 归一化与 4 尺度特征图（/8、/16、/32、/64——按代码，不按论文的三层说法）。
  - 一句话构图意图：呈现"固定投影定锚 + 可学习偏移采样"的拉取式桥——与 LSS 分册的"固定视锥推举"图并排即成第一组对照。

# 07 · PETR / PETRv2 事实底稿

> 本文件为 BEV 投影对比笔记的 PETR 分册，数字与结论的唯一来源是下方标注的 `path:line` 与论文原文（PETR＝`papers/petr_2203.05625.pdf`，arXiv:2203.05625，ECCV 2022 版 PDF 18 页；PETRv2＝`papers/petrv2_2206.01256.pdf`，arXiv:2206.01256，ICCV 2023 版 PDF 10 页）。路径约定：**v1 侧一律相对 `bev_projection/code/PETR-v1/` 书写（前缀 `PETR-v1/`）、v2 侧一律相对 `bev_projection/code/PETR/` 书写（前缀 `PETR/`）**——两快照来自同一仓的两个提交（见 §1 分支勘正）。范围限定 nuScenes 检测口径：v1＝`PETR-v1/projects/configs/petr/petr_r50dcn_gridmask_p4.py`（R50-DCN＋P4、512×1408、900 query，README 训练示例所用配置）为主，VoVNet-99 1600×640 变体（论文主结果口径）仅记差异量；v2＝`PETR/projects/configs/petrv2/petrv2_vovnet_gridmask_p4_1600x640_trainval_cbgs.py`（V2-99、640×1600、1500 query）＋`petrv2_BEVseg.py`（BEV 分割）。同仓的 `denoise/`（query denoise 变体）、`mspetr3d.py`（多尺度实验代码）、lane 检测（OpenLane，已移 TopoMLP 仓）与 Waymo 口径一律不引用，仅在需要勘正时点名。StreamPETR/CMT 等后续工作不属本分册。

## 1. 元信息

| 项 | 值 |
|---|---|
| 仓库 | https://github.com/megvii-research/PETR（README 自述 "official implementation of PETR and PETRv2"，`README.md`（main）：标题同时含 ECCV2022 PETR 与 ICCV2023 PETRv2；built upon detr3d、mmdet3d 0.17.1） |
| **分支勘正（重要）** | **官方仓不存在 `petr` 分支**（GitHub API `repos/megvii-research/PETR/branches` 实测仅 `main` 与 `flash` 两分支、无 tag；main 分支 README 全文也无"v1 在某分支"原句，仅提 flash 分支）。v1 代码取 **main 历史 early commit**：PETRv2 代码于 commit `812d423e5d4fa213b6e2d8f2f9e4c3d5aa7f59a3`（"petrv2"，2022-06-16，对应 News "2022.06.16 The code of 3D object detection in PETRv2 is released"）合入 main，其父链上最后一个纯 v1 提交为 `3617936cd5f6320eb046738acf8f9847b3c8101f`（"LICENSE"，2022-06-10）——本底稿 v1 快照即取该 commit（`git fetch --depth 1 origin 3617936…` 后 checkout） |
| commit hash（v1） | `3617936cd5f6320eb046738acf8f9847b3c8101f`（2022-06-10，已用 `git rev-parse HEAD` 于 `code/PETR-v1/` 复核，与 manifest 一致） |
| commit hash（v2＝main） | `f7525f93467a33707ef401c587a52d5e7b34de74`（2023-10-11 "3D lane detection"，main 分支 HEAD，已复核，与 manifest 一致） |
| 克隆日期 | 2026-10-29（两快照均 `--depth 1`，见 `00_manifest.md`） |
| 论文 | v1：Liu, Wang, Zhang, Sun, *Position Embedding Transformation for Multi-View 3D Object Detection*, ECCV 2022，arXiv:2203.05625。机制图 **Fig.1（DETR/DETR3D/PETR 对比）p.2**、总架构 **Fig.2 p.5**、**3D Position Encoder 细节 Fig.3 p.6**、PE 相似度 Fig.4 p.7、query 消融 Tab.5 p.12、实现细节 §4.2 p.8。v2：Liu et al., *PETRv2: A Unified Framework for 3D Perception from Multi-Camera Images*, ICCV 2023，arXiv:2206.01256。范式图 **Fig.1 p.3**、坐标系变换 Fig.2 p.3、三种 query Fig.3 p.4、**FPE 结构 Fig.4 p.5**、鲁棒性 Fig.5 p.5、实现细节 §4.2 p.5-6 |
| nuScenes 配置口径（v1） | `PETR-v1/projects/configs/petr/petr_r50dcn_gridmask_p4.py`：`point_cloud_range=[-51.2,-51.2,-5.0,51.2,51.2,3.0]`（`:11`）、ResNet-50＋DCNv2(stage3/4)＋out_indices=(2,3)（`:29-42`）、CPFPN in [1024,2048]→256、num_outs=2（`:43-47`）、`num_query=900`（`:52`）、`LID=True`（`:53`）、`with_position=True`＋`with_multiview=True`（`:54-55`）、`position_range=[-61.2,-61.2,-10.0,61.2,61.2,10.0]`（`:56`）、decoder `num_layers=6`（`:63`）、self+cross 均 8 头 256 维（`:66-77`）、FFN 2048（`:78`）、NMSFreeCoder `max_num=300`、post_center_range ±61.2/±10（`:84-91`）、SinePositionalEncoding3D num_feats=128（`:92-93`）、FocalLoss(2.0)＋L1(0.25)＋GIoU(0.0)（`:94-101`）、输入 `final_dim=(512,1408)`＋`resize_lim=(0.8,1.0)`＋H=900/W=1600＋rand_flip（`:155-163`）、GlobalRotScaleTrans ±0.3925 rad/0.95–1.05（`:170-176`）、AdamW lr=2e-4（backbone ×0.1）/wd=0.01（`:219-226`）、fp16 loss_scale=512（`:228`）、CosineAnnealing＋warmup 500、24 epochs（`:231-239`）、每 GPU 1 样本（`:202`）。VoVNet-99 变体 `petr_vovnet_gridmask_p4_1600x640.py`：V2-99-eSE stage4/5（768/1024 ch，`:30-35,38-40`）、`final_dim=(640,1600)`＋`resize_lim=(0.94,1.25)`（`:149-152`）、FCOS3D 初始化（`load_from`） |
| nuScenes 配置口径（v2） | `PETR/projects/configs/petrv2/petrv2_vovnet_gridmask_p4_1600x640_trainval_cbgs.py`：`num_query=1500`（`:45`）、`with_fpe=True`/`with_time=True`/`with_multi=True`（`:49-51`）、`position_range` 同 v1（`:52`）、decoder 6 层（`:59`）、V2-99＋CPFPN(768,1024)（`:30-40`）、`final_dim=(640,1600)`（`:152`）、**pipeline 加载 1 个历史 sweep → 每样本 12 张图**（`:160-161`）、CBGSDataset＋trainval 30f pkl（`:208-212`）、24 epochs（`:245`）。800×320 基线 `petrv2_vovnet_gridmask_p4_800x320.py`：`num_query=900`（`:45`）、320×800（`:152`）。BEV 分割 `petrv2_BEVseg.py`：`num_query=900`（`:48`）、`num_lane=625`（`:49`）、`blocks=[256,256,128]`（`:50`）、`with_se=True`/`with_time=True`（`:54-55`）、50 epochs（`:314`）、地图 GT 200×200×3 |
| 关键派生量（nuScenes 口径） | 输入 v1 主口径 **512×1408**（900×1600 经 resize_lim∈[0.8,1.0]＋裁剪，`transform_3d.py:442-462`）；P4 特征 **/16**（论文 §4.2 p.8 "P4 feature with 1/16 input resolution"＝C5 上采样融合 C4，`cp_fpn.py:167-184`）→ **每相机 32×88＝2816 token**（512/16×1408/16）、6 相机共 **16,896 个 key token**；VoVNet 640×1600 口径 40×100＝4000 token/相机（6 相机 24,000；v2 含历史帧 12 图 48,000）。**视锥深度 bins D=64、LID 线性递增**（论文 §4.2 p.8 "sample 64 points along the depth axis following the LID in CaDDN"；代码 `petr_head.py:106,289-293`）：bin=(61.2−1)/(64×65)≈0.01447、档位 d_i=1+bin·i(i+1)，即 **1 m→61.2 m、相邻档距从 ≈0.029 m 线性增至 ≈1.85 m**。**3D PE 原料＝每像素 3×64=192 通道**（`position_dim=3*depth_num`，`petr_head.py:174,323`；论文写作 (D×4) 见 §6 勘正 2）→ **position_encoder＝2 层 1×1 Conv MLP：192→1024→256（中间 ReLU）**（`petr_head.py:259-263`；论文 Tab.5(a) "1×1 ReLU 1×1" p.12 一致）。**PE 融合＝相加**（论文 §3.3 p.6＋Fig.3 "added with the 2D image features"；v2 论文 §3.4 p.4 "The 3D PE is added with 2D features to obtain the key value. The projected 2D features are used as the value component"；代码 `petr_transformer.py:341-344` key＝key+key_pos、value 不加 PE——concat 仅消融 Tab.5(b)）。decoder **6 层**。query＝**可学习 3D 锚点 `nn.Embedding(900,3)`**，uniform(0,1) 初始化（`petr_head.py:265,276`） |
| **深度监督形式（重点核对项）** | **无任何深度监督**。v1 损失只有 FocalLoss(cls)＋L1Loss(bbox)（GIoU 权重 0.0 恒零；`petr_r50dcn_gridmask_p4.py:94-101`；`petr_head.py:687-711` 逐层 loss_dict 只有 loss_cls/loss_bbox）；全仓 grep `depth` 于 v1 模型目录仅命中 `depth_num/depth_step/depth_start` 超参与 detector 形参 `img_depth`（穿透未用，`PETR-v1/.../detectors/petr3d.py:143`）。视锥 64 bins 只是 **3D PE 的原料**——不做深度数值回归、无 softmax 深度分布、不参与任何 loss。PETR 是全篇唯一"视锥 bins 存在但既不推特征也不监督深度"的条目 |
| **"无"清单（grep 实证，v1 快照）** | ① `grep -rn "bev_embedding\|bev_embed" PETR-v1/projects/` → **0 命中**（无 BEV query 参数表）；② 无深度 loss（见上行）；③ `grep -rni "voxel" PETR-v1/projects/mmdet3d_plugin/models/` 仅命中 MVXTwoStageDetector 基类未构建的 `pts_voxel_*` 构造形参（`petr3d.py:31-32`，配置不设置）与**未被任何 petr 配置引用的遗留头** `dgcnn3d_head.py`——PETR 执行路径（Petr3D→PETRHead→PETRTransformer）**无体素格**；④ deformable：`Deformable3DDetrTransformerDecoder`（`utils/detr.py:34`）与 `MultiScaleDeformableAttention` import（`utils/detr3d_transformer.py:17`）为 detr3d 遗留死代码，petr 全部 4 个 v1 配置（type='Petr3D'/'PETRHead'/'PETRTransformer'）**零引用**——cross attention 为普通全局 MultiheadAttention（`petr_transformer.py:271-272`） |

## 2. 机制流水线（每步标注【固定几何】/【可学习】）

输入：`(B,6,3,512,1408)`（B=1/GPU，`petr_r50dcn_gridmask_p4.py:202`；原始 900×1600 经 `ResizeCropFlipImage` resize_lim∈[0.8,1.0] 随机缩放＋底部偏置裁剪到 512×1408＋随机水平翻转，`transform_3d.py:363-462`；图像级增广 ida_mat 直接左乘进**内参**（`:398`），lidar2img 随之重建（`:401`）——2D 增广不产生"待撤销"的 3D 侧矩阵）。3D 增广 `GlobalRotScaleTransImage`（rot ±0.3925、scale 0.95–1.05，GT 框同步旋转缩放）把增广的逆矩阵**右乘**进 lidar2img（`transform_3d.py:491-511,517-546`）——增广在进入模型前已折入投影矩阵，投影桥上无任何撤销步骤（与 UVTR 的 uni_rot_aug 体素侧逆撤销、Qwen-Drive 的无增广均不同路）。GridMask 在模型内做（prob 0.7，`petr3d.py:50-51,71-72`）。光度归一化 caffe 均值（`config:13-14,177`）。

| # | 步骤 | 形状流 | 属性 | 出处（v1 前缀 `PETR-v1/`） |
|---|---|---|---|---|
| 0 | 标定准备：`lidar2img`＝viewpad(内参 K)@lidar2cam_rt.T（每相机 4×4）；ida 增广折入内参、3D rot/scale 右乘入 lidar2img | 6×(4,4) | 【固定几何】（逐样本标定量＋增广解析量） | `datasets/nuscenes_dataset.py:46-68`；`pipelines/transform_3d.py:398-401,517-546` |
| 1 | GridMask→ResNet-50（stage3/4 DCNv2、caffe style、BN 冻结）取 C3/C4 | (B·6,3,512,1408)→1024ch@32×88(/16)、2048ch@16×44(/32) | 【可学习·2D 侧】 | `config:29-42`；前向 `detectors/petr3d.py:71-79` |
| 2 | CPFPN：两 lateral 1×1（1024→256、2048→256）＋top-down 上采样相加；仅 level0 过 3×3 conv → P4 256ch@/16；level1（/32）输出**未被 head 消费**（forward 只取 `mlvl_feats[0]`） | →256ch@32×88（P4＝论文"C5 上采样融合 C4"） | 【可学习·2D 侧】 | `necks/cp_fpn.py:111-134,156-208`；消费 `dense_heads/petr_head.py:374,383` |
| 3 | `input_proj` 1×1 Conv 256→256 | (B·6,256,32,88)→(B,6,256,32,88) | 【可学习·2D 侧】 | `petr_head.py:216-220,383-384` |
| 4 | **3D 坐标生成（机制核心①）**：视锥 meshgrid (W,H,D=64)——LID 深度 d_i=1+0.01447·i(i+1)（1→61.2 m）；像素齐次化 (u·d, v·d, d,1)；逐相机 `inverse(lidar2img)` 矩阵乘反投影到 lidar 系；按 position_range ±61.2/±10 归一化到 [0,1]；超界档占比 >50% 的 token 记入 mask | (6,32,88,64,3)→coords3d | 【固定几何】（linspace/LID 网格＋标定矩阵求逆，零参数零学习） | `petr_head.py:282-322`（LID `:289-293`、meshgrid `:300`、齐次 `:301-302`、img2lidar `:304-315`、归一 `:316-318`、mask `:320-322`） |
| 5 | **3D PE 生成（机制核心②，桥上唯一可学习量）**：coords3d 过 `inverse_sigmoid` 后按 (D×3) 排布成每像素 **192 通道** → `position_encoder`＝Conv1×1(192→1024)+ReLU+Conv1×1(1024→256) | (B·6,192,32,88)→(B,6,256,32,88) | 【可学习·桥上】（MLP 学"坐标→签名"映射，**不做几何对应、不采样**） | `petr_head.py:174,323-325,258-263` |
| 6 | 多相机 2D PE：`SinePositionalEncoding3D`（num_feats=128、normalize）对 (N,H,W) 三维 cumsum 正弦编码 → 384 通道 → `adapt_pos3d`＝Conv1×1(384→1024)+ReLU+Conv1×1(1024→256)；**与 3D PE 相加** | (B,6,384,32,88)→(B,6,256,32,88)；pos_embed＝3D PE＋MV-2D PE | 【可学习·2D 侧】（相机区分先验；论文 Tab.3 行 5 "3D PE＋2D PE＋MV" p.11） | `petr_head.py:391-395,245-250`；`utils/positional_encoding.py:58-100` |
| 7 | query 生成：`reference_points`＝`nn.Embedding(900,3)`（uniform(0,1) 初始化的可学习 3D 锚点，归一坐标系）；`pos2posemb3d`（128 频率正弦→384 维）→ `query_embedding`＝Linear(384,256)+ReLU+Linear(256,256) 出 query_pos；decoder 的 query 内容恒为零（target=zeros）——query 输入＝query_pos 本身 | (900,3)→(900,384)→(900,256) | 【可学习·3D 侧】（锚点＋编码 MLP；论文 §3.4 "learnable anchor points…small MLP with two linear layers" p.7） | `petr_head.py:265-270,276,415-417,29-41` |
| 8 | **全局 cross attention（机制核心③）**：6 相机全部 P4 token **拼接成单一 key 序列**（n·h·w 顺序 flatten，无逐相机分组）＝16,896 keys；6 层 decoder，每层 self MultiheadAttention(8 头)＋cross PETRMultiheadAttention(8 头)＋FFN 256→2048→256；**key＝特征＋pos_embed、value＝纯特征**（PE 只进 K 不进 V）；key_padding_mask 屏蔽 pad 区与超 RoI 区 token | (6·2816,B,256) keys；(900,B,256) queries ×6 层 | 【可学习·3D 侧＋桥上】（attention 权重即"对应关系"——无投影、无采样、无 deformable） | `utils/petr_transformer.py:70-110`（flatten `:90-95`）、`:341-344`（key+key_pos）、`:271-272`（nn.MultiheadAttention）；`config:59-83` |
| 9 | 输出头：cls＝2×(Linear256＋LN＋ReLU)＋Linear(256→10)；reg＝2×(Linear256＋ReLU)＋Linear(256→10)（**6 层共享同一套**——ModuleList 为同名实例别名，见 §6 勘正 5）；box 的 xy、z 回归为**相对 3D 锚点的偏移**（sigmoid 后加 reference 再 denorm 到 pc_range）；NMSFreeCoder 取 300 框（±61.2 m） | (B,900,256)→(6,B,900,10) 检测输出 | 【可学习·3D 侧下游】 | `petr_head.py:222-243,423-443`；`config:84-91`；coder `core/bbox/coders/nms_free_coder.py` |

监督仅在步骤 9 之后：FocalLoss(weight 2.0)＋L1Loss(weight 0.25) 逐 decoder 层（`petr_head.py:629-711`）。视锥 bins（步骤 4）与 3D PE（步骤 5）不进任何损失——3D PE 的 MLP 由检测损失端到端回传，几何网格本身恒定。

### 2.1 PETRv2 增补流水线（v2 前缀 `PETR/`，其余步骤与 v1 同构）

| # | 步骤 | 形状流 | 属性 | 出处 |
|---|---|---|---|---|
| T1 | **时序输入**：pipeline `LoadMultiViewImageFromMultiSweepsFiles(sweeps_num=1)` 追加 1 个历史 sweep（sweep_range [3,27]，即 0.25–2.25 s 前）→ **每样本 12 张图**（当前 6＋历史 6），各带自己的 lidar2img 与时间戳；**对齐在数据层完成**——sweep 的 lidar2img 直接 append（模型/管线内无显式 T 矩阵乘法），README 原句锚定："The processed info files contain 30 previous frames, whose transformation matrix is aligned with the current frame" | (B,12,3,640,1600)→P4 每图 40×100＝4000 token、12 图共 **48,000 keys** | 【固定几何·数据层】（论文 Eq.(2)(3) 的 pose transformation 在 pkl 生成时烘焙进 sweep 外参） | `configs/petrv2/petrv2_vovnet_gridmask_p4_1600x640_trainval_cbgs.py:160-161`；`datasets/pipelines/loading.py:28-138`（append lidar2img `:131-134`、sensors 6 相机 `:46`、sweep_range `:43`）；README（main）BEVseg 段 |
| T2 | **feature-guided position encoder（FPE）**：`SELayer(256)`＝conv_reduce 1×1(256→256)＋ReLU＋conv_expand 1×1(256→256)＋Sigmoid 得门控 g(F)，3D PE ← PE·g(F)（图像特征门控调制纯几何 PE） | (B·12,256,40,100)×2 →PE 同形 | 【可学习·桥上】（论文 Eq.(5) PE＝ξ(F)·ψ(P) p.4、Fig.4 p.5——"data-dependent"） | `dense_heads/petrv2_head.py:45-57,328-329,454-455` |
| T3 | 速度时间归一（with_time）：mean_time_stamp＝历史帧与当前帧时间戳差的均值；reg 输出的 vx,vy 除以该值（query 同时看到两帧 key，位移感知按帧距折算） | 标量/B | 【固定几何·数据层】＋可学习输出 | `petrv2_head.py:489-495,510-511` |
| T4 | 解耦回归（with_multi）：`RegLayer`＝共享 2×(Linear256＋ReLU＋Dropout)＋5 个任务头 (xy2, z1, size3, rot2, velo2) 各 Linear256→ReLU→Linear(256→dim)；cls/reg **×6 层独立**（deepcopy，非 v1 的共享别名） | (B,1500,256)→10 维框 | 【可学习·3D 侧下游】 | `petrv2_head.py:59-90,287-300`；论文 §4.2 "extra disentangled layers" p.6 |
| T5 | **BEV 分割 query（BEVseg 配置）**：`num_lane=625`＝**25×25 固定 BEV 网格锚点**（(i+0.5)/25 归一格心，普通张量非 Embedding）；`query_embedding_lane`＝Linear(256,256)+ReLU+Linear(256,256) 编码 pos2posemb2d(锚点)；分割 query 与检测共用**同一批全局 key**（transformer_lane 输入同 x/masks/pos_embed）；decoder 6 层输出 reshape (25,25) → 三个 `Decoder`（blocks [256,256,128]，每块上采样 ×2：25→50→100→200）出 **200×200×3** 地图（drivable/lane-divider/vehicle），FocalLoss 逐类监督 | (625,256)→(6,B,625,256)→(B,3,200,200) | 分割锚点【固定几何】、query MLP/Decoder【可学习·3D 侧】 | `dense_heads/petr_head_seg.py:373-377,379-383,540-543,562-572`、Decoder `:144-171`；`configs/petrv2/petrv2_BEVseg.py:48-50,54-55`；地图 GT `pipelines/loading.py:148-166`（3,200×200） |

v2 主口径 decoder 结构与 v1 逐项相同（PETRTransformer 代码仅加 query-denoise 变体类，主路径 forward 与 v1 逐字节一致——两快照 diff 实证）。query denoise（`denoise/` 配置族＋`petrv2_dnhead.py`）为收敛加速变体，主检测配置未启用，本底稿不展开。

## 3. 可学习参数清单表（nuScenes 口径）

> "侧"按 PETR 的特殊桥（3D PE 软桥＝步骤 4–6）划分：桥上唯一可学习量＝position_encoder（及 v2 的 FPE 门控）。这是全篇笔记**唯一"桥上有可学习量且不做任何采样/投影对应"**的条目（对比：BEVFormer 桥上 offsets/weights 做投影采样、UVTR/LSS 桥零参数）。v1 按 512×1408、900 query 计。

| 参数（组） | 所属模块 | 形状（nuScenes v1 口径） | 侧 | 代码出处 `path:line` |
|---|---|---|---|---|
| `img_backbone`（ResNet-50＋DCNv2 stage3/4，BN 冻结 norm_eval） | Petr3D | 消费 /16、/32 两 stage（1024/2048 ch） | 2D 侧 | `PETR-v1/projects/configs/petr/petr_r50dcn_gridmask_p4.py:29-42` |
| `img_neck`（CPFPN） | Petr3D | 2×lateral 1×1（1024→256、2048→256）＋level0 3×3 conv；出 /16、/32 各 256ch（/32 层未被 head 消费） | 2D 侧 | `PETR-v1/.../necks/cp_fpn.py:111-134`；未消费 `petr_head.py:374` |
| `input_proj`（1×1 Conv2d） | PETRHead | Conv2d 256→256 | 2D 侧（桥前接口；论文 Fig.3 "dimension reduction"） | `PETR-v1/.../dense_heads/petr_head.py:216-220,383` |
| **`position_encoder`（3D PE 生成 MLP——本模型的机制名片，桥上唯一可学习量）** | PETRHead | Conv1×1(**192**→1024)＋ReLU＋Conv1×1(1024→**256**)（192＝3 坐标×64 深度档） | **桥上**（不做几何对应、不采样——只学"3D 坐标→位置签名"） | `PETR-v1/.../petr_head.py:258-263,325`；论文 Tab.5(a) p.12 |
| `adapt_pos3d`（多相机 2D PE 适配） | PETRHead | Conv1×1(384→1024)＋ReLU＋Conv1×1(1024→256)（384＝SinePositionalEncoding3D 3×128） | 2D 侧（相机区分先验，可选组件） | `PETR-v1/.../petr_head.py:245-250,393-395` |
| `reference_points`（**3D 锚点表，可学习**） | PETRHead | `nn.Embedding(900, 3)`（uniform(0,1) 初始化；v2 主口径 1500×3） | 3D 侧（query 锚点；框回归的偏移基准） | `PETR-v1/.../petr_head.py:265,276,415-417` |
| `query_embedding`（锚点→query 编码） | PETRHead | Linear(384,256)＋ReLU＋Linear(256,256)（384＝pos2posemb3d 3×128） | 3D 侧 | `PETR-v1/.../petr_head.py:266-270,416` |
| decoder 每层 self MHA＋cross MHA＋FFN ×6 层 | PETRTransformer | MHA 256ch/8 头（cross＝普通全局 nn.MultiheadAttention，无 sampling_offsets/attention_weights）；FFN 256→2048→256；LN ×3 | 3D 侧（attention＝隐式对应） | `PETR-v1/.../utils/petr_transformer.py:271-272`；`config:59-83` |
| cls/reg 分支（**v1：6 层共享同一实例**） | PETRHead | cls 2×(Linear256＋LN＋ReLU)＋Linear(256→10)；reg 2×(Linear256＋ReLU)＋Linear(256→10) | 3D 侧（桥外下游） | `PETR-v1/.../petr_head.py:222-243` |
| v2 增补：`fpe`（SELayer） | PETRv2Head | conv_reduce 1×1(256→256)＋ReLU＋conv_expand 1×1(256→256)＋Sigmoid（门控 3D PE） | **桥上** | `PETR/.../dense_heads/petrv2_head.py:45-57,328-329` |
| v2 增补：cls/reg ×6 独立＋RegLayer | PETRv2Head | deepcopy ×6；RegLayer 共享 2×Linear256＋5 任务头 (2,1,3,2,2) | 3D 侧（桥外下游） | `PETR/.../petrv2_head.py:287-300,59-90` |
| v2 增补：`query_embedding_lane`＋三个 `Decoder`（BEVseg） | PETRHead_seg | Linear(256,256)＋ReLU＋Linear(256,256)；Decoder blocks [256,256,128]（每块 DecoderBlock 上采样 ×2）→to_logits 3×3＋1×1 conv | 3D 侧（分割 query 编码＋上采样头） | `PETR/.../dense_heads/petr_head_seg.py:379-383,337-347,144-171` |
| **无项（专列）** | — | **无**深度头、**无** bev_embedding、**无**体素格参数、**无** deformable 偏移/权重表（grep 实证见 §1） | — | §1 "无"清单 |

要点：投影桥（步骤 4 的视锥网格＋反投影）本身零参数——与 LSS/UVTR 同为"几何零参数"；差异在 PETR 在桥上挂了**一个可学习 MLP**（position_encoder），但它不建立对应关系（无采样无投影输出），只把几何坐标译成"签名"。论文未给总参数量，底稿不引。

## 4. 固定几何清单表

| 项 | 具体值 | 出处 |
|---|---|---|
| 视锥 meshgrid | (W,H,D)＝(88,32,**64**)（P4 token 网格×64 深度档；论文 §3.2 p.5 meshgrid of size (W_F,H_F,D)）；VoV 640×1600 口径 (100,40,64) | `PETR-v1/.../petr_head.py:285-300`；论文 p.5,p.8 |
| 深度 bins（LID） | **D=64 档、1 m→61.2 m**：bin=(61.2−1)/(64×65)≈0.01447，d_i=1+bin·i(i+1)；相邻档距 2·bin·(i+1)：首 ≈0.029 m、末 ≈1.85 m（线性递增） | `PETR-v1/.../petr_head.py:106,289-293`；论文 §4.2 p.8 |
| 3D 坐标归一范围（position_range/RoI） | X/Y **±61.2 m**、Z **−10~10 m**（论文 Tab.4 p.11 最优 RoI 行一致；注意≠pc_range ±51.2/−5~3——前者管 PE 归一、后者管框回归与 NMSFreeCoder） | `PETR-v1/configs/petr/petr_r50dcn_gridmask_p4.py:56`；论文 p.8,p.11 |
| 反投影矩阵 | img2lidar＝`np.linalg.inv(lidar2img)` 逐相机求逆；lidar2img＝viewpad(内参)@lidar2cam_rt.T | `PETR-v1/.../petr_head.py:304-311`；`nuscenes_dataset.py:46-68` |
| 超界 token 掩码 | 归一坐标出 [0,1] 的深度档占比 >D/2 的 token 并入 key_padding_mask（pad 掩码同或） | `PETR-v1/.../petr_head.py:320-322,377-387` |
| 输入图像分辨率 | v1 主口径 **512×1408**（resize_lim 0.8–1.0、H=900/W=1600 原生、rand_flip、pad 32 整除）；VoV/论文主口径 **640×1600**（resize_lim 0.94–1.25）；v2 800×320 基线 **320×800** | `PETR-v1/.../petr_r50dcn_gridmask_p4.py:155-163,178`；`petr_vovnet_gridmask_p4_1600x640.py:149-152`；`PETR/.../petrv2_vovnet_gridmask_p4_800x320.py:151-152` |
| token 分辨率 | **P4＝/16**（C3 直接＋C4 top-down 融合；论文 "P4 feature with 1/16 input resolution"）；512×1408→32×88/相机；640×1600→40×100/相机 | `PETR-v1/.../cp_fpn.py:167-184`；论文 §4.2 p.8 |
| 相机数 | 6（CAM_FRONT/FRONT_RIGHT/FRONT_LEFT/BACK/BACK_LEFT/BACK_RIGHT；v2 含历史帧时"相机槽"×2＝12 图） | `PETR/.../pipelines/loading.py:46`；`PETR-v1/.../nuscenes_dataset.py:52` |
| 3D 增广 | GlobalRotScaleTrans ±0.3925 rad、scale 0.95–1.05：逆矩阵右乘进 lidar2img（世界系旋转，相机跟随——桥上无撤销步骤）；图像 ida 增广折入内参 | `PETR-v1/.../transform_3d.py:469-546,398-401` |
| GridMask | use_grid_mask=True（prob 0.7） | `PETR-v1/.../petr3d.py:50-51,71-72`；`config:28` |
| 训练超参（供 HTML 脚注） | AdamW lr=2e-4、backbone ×0.1、wd=0.01、CosineAnnealing（warmup 500）、24 epochs、fp16 loss_scale=512、每 GPU 1 样本（论文 §4.2 p.8：8×V100 总 batch 8、λcls=2.0、多尺度训练短边 [640,900]）；v2＝8×A100、AdamW 2e-4、24 epochs（BEVseg 50） | `PETR-v1/.../petr_r50dcn_gridmask_p4.py:202,219-239`；论文 p.8；`PETR/.../petrv2_..._trainval_cbgs.py:228,245`、`petrv2_BEVseg.py:314` |
| 推理保留框 | NMSFreeCoder max_num=300、post_center_range X/Y ±61.2 m、Z ±10 m | `PETR-v1/.../petr_r50dcn_gridmask_p4.py:84-91` |
| v2 时序采样 | 训练随机取 [3T,27T]（0.25–2.25 s）前一帧、推理取 15T（≈1.25 s）；T≈0.083 s | 论文 v2 §4.2 p.5-6；`PETR/.../loading.py:43,105`、`config:160-161` |
| v2 BEV 分割网格 | 625 query＝**25×25 固定格心** ((i+0.5)/25)；地图 200×200 m、3 类（drivable/lane/vehicle）；Decoder 每块上采样 ×2（25→200） | `PETR/.../petr_head_seg.py:373-377,564-570`；`petrv2_BEVseg.py:49`；`loading.py:148-166`；论文 v2 §4.2 p.6（patch 25×25、625 seg queries） |

## 5. 2D-3D 关系小结

- **推/拉方向：都不算（新谱系＝学习直连）**。无特征流"过桥"：不预测深度分布（推家族定义性特征缺席）、不做 3D→2D 投影采样（拉家族定义性特征缺席）。2D↔3D 对应关系**隐式成立于 Q·K 内积**——固定几何只把每个像素的 64 个候选 3D 坐标译成位置签名（3D PE），object query 用自己的可学习 3D 锚点签名与全部相机的全部像素签名做全局 attention，"谁对应谁"由学出的相似度决定。投影桥整体消失，换成"PE 软桥"。
- **深度：不预测数值、不监督**。视锥 64 LID bins 仍然存在，但只是 3D PE 的原料（每像素 192 维坐标列），既无 softmax 深度分布（对比 LSS/UVTR/BEVDepth），也无深度回归或深度 loss（§1 grep 实证）。深度不确定性由全局 attention 权重隐式分配——一个 query 的 3D 锚点签名与"远 bin 坐标签名"和"近 bin 坐标签名"的相似度差即是隐式的深度选择。PETR 是深度谱系"族④：无假设·隐式全局分配"的定义性样本。
- **几何与学习的分工：边界移位**。固定几何退到"只发坐标"（内参 lidar2img＋LID 视锥网格＋RoI 归一＋超界掩码——全部零参数、零采样输出）；可学习量第一次出现在**桥上**（position_encoder MLP，v2 再加 FPE 门控）——但桥上学习量不做几何对应，与 BEVFormer"桥上学习量做投影采样"形成精确对照。两端（2D backbone/FPN、3D 锚点/decoder/头）如常可学习。
- **query 有无及形态：900 个稀疏、自由 3D 坐标、无网格绑定**（代码口径；论文口径见 §6 勘正 1）。锚点是 `nn.Embedding(900,3)` 的**连续可学习 3D 坐标**——不落任何 BEV 格/体素格（论文 Tab.5(c)：Fix-BEV 39×39 与 Fix-3D 16×16×6 均差于 Learned-3D，p.12-13）；decoder 全局 attention 无逐 query 投影。与 BEVFormer 的 bev_embedding（40000 BEV 格 query）、UVTR 的体素格（被填充物）均成对照。

## 6. PETR vs PETRv2 差异表＋论文/代码勘正＋待画图要点

### 6.1 PETR vs PETRv2 差异表（v1 快照 vs main 快照，同仓同源）

| 维度 | PETR（v1） | PETRv2（main） | v2 代码位置 |
|---|---|---|---|
| 输入帧数 | 1 帧 6 相机 | 当前帧＋1 历史 sweep＝**12 图**（数据层对齐：sweep lidar2img 已在当前帧 lidar 系，README "aligned with the current frame" 原句；模型内无显式 pose 矩阵乘法——论文 Eq.(2)(3) 的对齐发生在 pkl 生成层） | `PETR/.../loading.py:28-138`；`petrv2_..._trainval_cbgs.py:160-161` |
| 时序 PE 对齐（3D→2D 回归一次） | 无时序 | **历史帧像素的 3D PE 与当前帧同坐标系**：同一 3D 锚点（当前帧坐标）在两帧的 key 中获得相近签名 → attention 跨帧一致；query 不做历史传播（区别于 StreamPETR 的 query 队列）；with_time 仅把速度回归除以帧距 | `petrv2_head.py:489-495,510-511`；论文 v2 §3.2 p.3、Fig.1/2 |
| 3D PE 生成 | 纯几何：MLP(192→1024→256)（ψ(P)） | **feature-guided**：PE←PE·sigmoid(ξ(F))，ξ＝两层 1×1 conv（SELayer 门控，论文 Eq.(5)） | `petrv2_head.py:45-57,454-455`；论文 v2 §3.4 p.4、Fig.4 p.5 |
| query 数 | 900（全部 v1 配置） | 1500（1600×640 trainval cbgs 主配置、dn trainval 60ep）/ 900（800×320 基线、BEVseg、denoise 多数） | `petrv2_..._trainval_cbgs.py:45`；`petrv2_..._800x320.py:45` |
| BEV 分割 query | 无 | **625＝25×25 固定 BEV 格心**锚点＋query MLP，与检测共用同一批全局 key；Decoder 上采样 25→200 出 200×200×3 地图（v2 向 BEV 方向的回流） | `petr_head_seg.py:373-377,540-570`；`petrv2_BEVseg.py:48-50` |
| 回归头 | 2×Linear＋Linear(→10)，**6 层共享同一实例**（ModuleList 别名） | with_multi 解耦：RegLayer 5 任务头；cls/reg **×6 deepcopy 独立** | `petr_head.py:222-243` vs `petrv2_head.py:287-300,59-90` |
| cross attention | 全局 MultiheadAttention（无 deformable） | 相同（PETRTransformer 主路径逐字节一致；另有未启用的 query-denoise 变体类） | 两快照 diff 实证；`petrv2_dnhead.py` 不引用 |

### 6.2 论文 vs 代码差异与勘正记录

| # | 项 | 论文说法 | 代码事实 | 处理 |
|---|---|---|---|---|
| 1 | **num_query（最重要核对项）** | v1 论文 Tab.5(d) p.12：600/900/1200/1500 递增最优，正文 p.13 "**we simply use 1500 anchor points** to make a trade-off" | **v1 全部 4 个配置 num_query=900**（含 README 模型卡 40.4 mAP 的 1600×640 配置）；v2 论文 p.6 "1500 det queries on nuScenes test" ↔ v2 代码 1600×640 主配置 1500、800×320/BEVseg 900 | **以代码为准**：v1 引 900、v2 主配置引 1500（并注明论文 v1 正文 1500 与代码不一致）；HTML 数字白名单按代码口径（§7） |
| 2 | 3D PE 原料维数 | §3.2 p.6 P^3d ∈ R^{(D×4)×H_F×W_F}（含齐次维） | 代码 `position_dim=3*64=192`（`petr_head.py:174,323`，齐次维被丢弃、`[..., :3]`） | 以代码为准：192＝3×64；论文 D×4 为笔误/含齐次记法 |
| 3 | PE 融合方式 | §3.3 p.6 "added with the 2D image features"（Tab.5(b) Add 0.359 vs Concat 0.358，均可用） | 代码实现 Add：key＝特征＋PE、value＝纯特征（`petr_transformer.py:341-344`） | 一致（Add）；Concat 仅为消融项，画图标"相加进 K" |
| 4 | query 内容 | §3.4 p.7 锚点→"small MLP with two linear layers"→初始 query | 代码 query 内容恒零（target=zeros），输入即 query_pos；MLP＝Linear(384,256)+ReLU+Linear(256,256)（`petr_head.py:266-270,416-419`） | 一致；底稿措辞"query_pos 即 query 输入" |
| 5 | 输出头共享 | —（论文未提） | v1 cls/reg 的 6 层 ModuleList 为**同一实例的 6 个引用**（非 deepcopy，`petr_head.py:240-243`）——6 层共享一套头参数；v2 改为 deepcopy 独立 | 以代码为准；v1/v2 差异表已列 |
| 6 | 输入分辨率 | §4.2 p.8 "multi-scale training…shorter side within [640,900], longer ≤1600" | 代码为固定 final_dim（512,1408）＋resize_lim (0.8,1.0)（或 (640,1600)＋(0.94,1.25)）——随机缩放＋裁剪近似多尺度 | 以代码为准；底稿记 final_dim＋resize_lim |
| 7 | /32 特征层 | §4.2 只提 P4 | CPFPN 输出 /16＋/32 两层，head 只消费 level 0（/16），/32 层为死输出 | 记录备查（画图只画 P4 单层） |
| 8 | v2 时序机制 | §3.2 p.3-4：显式 pose transformation T_l(t)_l(t−1)（Eq.2-3）对齐历史 3D 坐标 | 代码无该矩阵乘法：对齐烘焙在 info pkl（README "transformation matrix is aligned with the current frame" 原句）＋sweep lidar2img 直接 append；模型侧仅 with_time 速度归一 | 以代码为准描述实现层；论文公式作为机制解释引用 |
| 9 | FPE 结构 | §3.4 p.4 Eq.(5)：ξ＝1×1 conv→MLP→Sigmoid 得权重、乘 ψ(P) | SELayer：conv_reduce 1×1＋ReLU＋conv_expand 1×1＋Sigmoid，PE·gate（`petrv2_head.py:45-57`） | 一致（"两层 1×1 conv"即论文的"small MLP"） |
| 10 | BEV 分割 | §3.3 p.4＋§4.2 p.6：patch 25×25、**625 seg queries**、200×200 地图 | num_lane=625、25×25 格心、(25→50→100→200) 上采样、3 通道（`petr_head_seg.py`）逐项吻合 | 一致（列入以示已核对） |
| 11 | 性能数字 | v1 论文 Tab.1 p.9（val, CBGS）：Res-50 1408×512 0.403/0.339、Res-101† 1600×900 0.442/0.370；Tab.2 p.10（test）：V2-99∗ 0.504/0.441。v2 论文 Tab.1 p.6（val）：Res-50 800×320 0.456/0.350、1600×640 0.494/0.398 | README 模型卡（val、无 CBGS、8×2080ti）：PETR-r50-p4 31.70/36.7、PETR-vov-p4-1600x640 40.40/45.5、PETRv2-vov-p4-800x320 41.0/50.3、BEVseg Drive 85.6/Lane 49.0/Vehicle 46.3 IoU | HTML 引性能时标"论文 Tab.x 值"或"README 模型卡值"，不混用 |
| 12 | 其余一致项 | 64 LID bins、RoI ±61.2/±10、λcls=2.0、24 epochs、Focal+L1、300 框、锚点 Learned-3D 优于 Fix | config/head 逐项核对一致（§4 表） | 一致 |

### 6.3 待画图要点（A 图＝论文 Fig.1(c)+Fig.2+Fig.3 重绘规范＋C 图＋B 面板）

- **A 图（三段横排，重绘对象＝v1 论文 Fig.2（总架构，p.5）＋Fig.3（3D Position Encoder，p.6），构图参照 Fig.1(c)（p.2））**：
  - 左：图像 token（灰，P4 /16 网格）＋该像素的 64 档视锥射线点（蓝色虚线斜纹，**固定几何**；LID 档距近密远疏可示意）→ 每像素 192 维坐标列 → MLP（橙，192→1024→256）→ 3D PE；
  - 中：key token（灰底橙描边小标＝"特征＋3D PE 相加"，注明 PE 只进 K 不进 V；6 相机 token 拼成一列长条）；
  - 右：object query（橙，900，各自连续 3D 锚点不锁栅格）——一条粗 attention 线横扫全部相机的全部 key（示意 Q·K 相似度）。
  - **全图无蓝色虚线投影线（指 3D→2D 采样投影）、无采样簇、无深度分布柱**——与前六节 A 图的最大视觉差异，图注必须点明："投影桥消失，3D PE 软桥接管"。
  - 叠加编码：可学习元素实色（backbone/FPN、PE 的 MLP、query 锚点、decoder）；固定几何虚线/灰（视锥 LID 网格、lidar2img 求逆、RoI 归一）；注记 D=64@LID（1–61.2 m）、192→1024→256、P4 /16、900 query、PE 加进 K。
- **C 图（机制流程线）**：① 标定 lidar2img（固定）→ ② ResNet-50＋CPFPN P4 /16＋input_proj（可学习·2D）→ ③ 视锥 LID 64 bins meshgrid＋img2lidar 反投影＋RoI 归一（固定几何）→ ④ position_encoder MLP 192→1024→256 出 3D PE（**可学习·桥上**——全篇唯一桥上可学习且不做对应的量，重点标出）→ ⑤ ＋多相机 2D PE 相加→key 拼接 6 相机全 token（固定拼接）→ ⑥ 900 query（可学习 3D 锚点）×6 层全局 cross attention（可学习·3D，attention＝隐式对应）→ ⑦ cls/reg 相对锚点回归出 3D 框（可学习·3D 下游）。形状流注记 (B,6,3,512,1408)→(B,6,256,32,88)→16,896 keys×(900,256)。
- **B 面板（petr3d）数字白名单见 §7**。

## 7. B 面板（petr3d）数字白名单

- **相机布置**：6 相机（CAM_FRONT/FRONT_RIGHT/FRONT_LEFT/BACK/BACK_LEFT/BACK_RIGHT，`PETR/.../loading.py:46`）线框；熄灭态灰。
- **深度 bins**：**64 档 LID、1 m→61.2 m**（档距 0.029 m→1.85 m 线性递增；射线上点近密远疏——与 LSS/UVTR 面板的均匀 bins 形成视觉对照）。每相机中心射线上的 bins 点云＝固定几何（灰）；**点不落任何格**（无 BEV 格/体素格/pillar）。
- **query 数**：**900**（v1 代码口径；面板文案若涉 v2 再引 1500——两者均出自本底稿 §1/§6.1）。3D query 自由移动、不锁栅格（与 BEVFormer 面板"只能拖 BEV 格"操作层面对照）。
- **attention 线示意规则**：3D query 到各相机的 soft attention 线按示意相似度点亮——**无投影线、无采样簇**；线只连 query↔相机（key 侧全体 token 的抽象），不得画成"投影到某像素再采样"。
- 其余允许数字：192（=3×64 PE 原料通道）、MLP 192→1024→256、256 通道、P4 /16（512×1408→32×88/相机 或 640×1600→40×100）、6 层 decoder、300 框、pc_range ±51.2 m/−5~3 m（仅当文案涉框范围）、RoI ±61.2 m/−10~10 m。
- 两态语义约束：「3D PE」态＝bins 变橙、点只生成位置签名附着在像素 key 上（不做对应）；「全局 attention」态＝query 自由移动＋soft 线点亮。三态＝熄灭→3D PE→全局 attention（单按钮循环）。
- **禁入数字**（他节口径）：41、105、0.512 m、200×200（LSS/BEVFormer 节）、64 m@1 m/档、128×128×5、0.8 m（UVTR 节）、118、56×32×118、16,200,200（Qwen-Drive 节）——面板任何文案不得出现。注意本节"64"指**档数**（LID 非均匀），不得与 UVTR 的"64 档均匀 1 m"混写。

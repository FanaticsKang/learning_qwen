# 05 · UVTR（相机版）事实底稿

> 本文件为 BEV 投影对比笔记的 UVTR 分册，数字与结论的唯一来源是下方标注的 `path:line`（相对于 `bev_projection/code/UVTR/`）与论文原文（arXiv:2206.00630 v2，NeurIPS 2022 版 PDF，17 页）。后续 HTML 各节只引用本底稿，不重查代码。范围限定**相机单模态口径**：基座配置 `uvtr_camera_base.py` ＋实例化 `camera/uvtr_c_r50_h5.py`（单帧 6 相机、R50、H5）；同仓的 H11/R101 变体、`camera_sweep/`（多帧）、`knowledge_distill/`（蒸馏）、`lidar_based/`、`multi_modality/`（LiDAR 与多模态分支）一律不引用，仅在 §6.1 需要勘正时点名。多帧 sweep 机制（时间偏置拼接、`sweep_cat_with_time`）为笔记非目标，一句带过。

## 1. 元信息

| 项 | 值 |
|---|---|
| 仓库 | https://github.com/JIA-Lab-research/UVTR（一作 Yanwei Li；README 自述 "implementation for the NeurIPS 2022 paper … based on mmDetection3D"，`README.md:16`；arXiv 链接 `README.md:3,10`；mmdetection3d v0.17.3 `README.md:20`；已与 arXiv 2206.00630 作者列表逐名核对一致） |
| commit hash | `8a24ec2f1f2fe9f6b65bed201dae112f83e4aaa7`（2022-10-19 "Update create_data.py"，已用 `git rev-parse HEAD` 复核，与 manifest 一致） |
| 克隆日期 | 2026-10-28（`--depth 1` 浅克隆，见 `00_manifest.md`） |
| 论文 | Li et al., *Unifying Voxel-based Representation with Transformer for 3D Object Detection*, NeurIPS 2022；arXiv:2206.00630（`papers/uvtr_2206.00630.pdf`，v2，2022-10-13）。投影机制 §3.1 p.4（Eq.1–Eq.2）、总框架 Fig.2 p.3、**视图变换机制图 Fig.3 p.4**、实现细节 §4.1 p.6、消融 Tab.1–4 p.7、主表 Tab.7 p.8、附录 A（统一增广/训练设置/naive grid sampling 说明）pp.14–15 |
| nuScenes 配置口径 | `projects/configs/uvtr/camera_based/uvtr_camera_base.py`（基座）＋`camera/uvtr_c_r50_h5.py`（纯继承，无覆盖项）。核心量：`point_cloud_range=[-51.2,-51.2,-5.0,51.2,51.2,3.0]`（`uvtr_camera_base.py:11`）、`voxel_size=[0.2,0.2,8]`＋`bev_stride=4`（`uvtr_camera_base.py:12,15`）、`sample_num=5`→`voxel_shape=[128,128,5]`（`uvtr_camera_base.py:16-19`）、`cam_sweep_num=1`（`uvtr_camera_base.py:13`）、`depth_dim=64`（`uvtr_camera_base.py:62`）、view_cfg `num_cams=6`、`num_convs=3`、`num_points=sample_num`、`kernel_size=(3,3,3)`、`num_feature_levels=4`、`embed_dims=256`（`uvtr_camera_base.py:67-77`）、`num_query=900`（`uvtr_camera_base.py:80`）、decoder `num_layers=6`（`uvtr_camera_base.py:91`）、UniCrossAtten `num_points=1`（`uvtr_camera_base.py:103`）、NMSFreeCoder `max_num=300`（`uvtr_camera_base.py:125`） |
| 关键派生量（nuScenes 口径） | 输入 **900×1600**（数据集原生分辨率，pipeline 无 Resize，`uvtr_camera_base.py:197-211`），pad 到 32 整除→**928×1600**（`uvtr_camera_base.py:208`）；FPN 四层特征 256ch @ /8、/16、/32、/64（推算 116×200、58×100、29×50、15×25，`uvtr_camera_base.py:44,51-58`）；深度档 **D=64**、每档 1 m、范围 0–64 m（论文 §3.1 p.4 "D is set to 64 to represent the perception limit 64m"；代码深度归一 `z/64` 后映射 [-1,1]，`uni3d_viewtrans.py:316-323`，>64 m 体素被掩码，`uni3d_viewtrans.py:325-330`）；**无 LSS 式显式视锥**——对应物＝逐尺度深度分布图（每相机 4 份：64×116×200、64×58×100、64×29×50、64×15×25，`uvtr.py:204-209`）＋体素采样格 **128×128×5**（每相机 81920 个格点）；体素格名义边长 102.4/128=**0.8 m**（XY），实际采样点间距 102.4/127≈**0.806 m**（linspace 含端点，见 §4），Z 采样点 {-5,-3,-1,1,3} m（5 层含端点、间距 2 m）；体素体 V_I 形状 (B,1,256,5,128,128)（`uni3d_viewtrans.py:223-226,165-167`） |
| **深度监督形式（重点核对项）** | **无任何深度监督**。论文 §3.1 p.4 原文："It is noted that D_I is predicted without supervision"；代码侧双确认——①检测头损失只有 FocalLoss(cls)＋L1Loss(bbox)（`uvtr_head.py:335-401,459-491`；`uvtr_camera_base.py:133-141`），`loss()` 虽接收 `gt_depth_prob`/`gt_depth_coord` 形参（`uvtr_head.py:407-408`，来自 `uvtr.py:238-240` 的穿透传参）但函数体从未使用；②数据管线不生成深度 GT（`CollectUnified3D` 仅收集 `['gt_bboxes_3d','gt_labels_3d','img']`，`uvtr_camera_base.py:210`；train_pipeline 全文无深度图产出，`uvtr_camera_base.py:197-211`）。LiDAR 深度仅在训练增广里用于贴样本排序（`transform_3d.py:708-728`，UnifiedDataBaseSampler 图像裁块按 LiDAR 投影深度粘贴），蒸馏变体的 partial-L2 是体素特征级损失而非深度损失——均不构成深度监督。**设计文档曾假设"原版 UVTR 有深度监督"，据此勘正：原版与 Qwen-Drive 同为无监督纯下游梯度** |

## 2. 机制流水线（每步标注【固定几何】/【可学习】）

输入：`(B,6,3,928,1600)`（B=1/GPU，`uvtr_camera_base.py:222`；原图 900×1600 无缩放，`PadMultiViewImage` size_divisor=32，`uvtr_camera_base.py:208`）。几何增广与 BEVFormer 节不同：pipeline 含 `UnifiedRotScaleTrans` 3D 全局旋转 ±0.3925 rad＋缩放 0.95–1.05（`uvtr_camera_base.py:202-204`），其合成矩阵 `uni_rot_aug` 随 meta 下传、在投影前于体素侧逆掉（`transform_3d.py:325-346,461-469`；`uni3d_viewtrans.py:275-276`）；无图像级几何增广（`img_rot_aug`/`img_trans_aug` 恒为 eye(2)/零向量，`transform_3d.py:306-311` 不在 camera pipeline）。光度增广 PhotoMetricDistortion（`uvtr_camera_base.py:199`）；GridMask 在模型内做（`uvtr.py:47,160-161`；`uvtr_camera_base.py:39`）。

| # | 步骤 | 形状流 | 属性 | 出处 |
|---|---|---|---|---|
| 0 | 标定准备：`lidar2img`（每相机 4×4）＝内参 K 与 `sensor2lidar` 外参求逆组装（`lidar2img_rt = viewpad @ lidar2cam_rt.T`）；同时下传 `uni_rot_aug`（3D 增广合成矩阵，旋转@缩放） | 6×(4,4)；(3,3) | 【固定几何】（逐样本标定量＋增广解析量） | `datasets/nuscenes_dataset.py:236-253`；`pipelines/transform_3d.py:461-469` |
| 1 | 图像主干 ResNet-50（stage3/4 带 DCNv2，FCOS3D 预训练初始化）：取 C2–C5 | (B·6,3,928,1600)→256ch@232×400(/4)、512ch@116×200(/8)、1024ch@58×100(/16)、2048ch@29×50(/32) | 【可学习·2D 侧】 | `uvtr_camera_base.py:40-50,261`；前向 `detectors/uvtr.py:160-164` |
| 2 | FPN（start_level=1、`add_extra_convs='on_output'` 补 /64 层、num_outs=4）→逐层 `input_proj`（1×1 Conv 256→256） | →256ch@116×200、58×100、29×50、15×25（/8、/16、/32、/64） | 【可学习·2D 侧】 | `uvtr_camera_base.py:51-58`；`detectors/uvtr.py:48-52,167-173` |
| 3 | 深度网络 `depth_net`：单个 1×1 Conv 256→64，对四个尺度特征依次调用（**跨尺度共享权重**），每份做 `softmax(dim=1)` →论文 Eq.(1) D_I=Softmax(Conv(F_I))；**无监督**（见 §1） | →(B·6,64,116×200)、(B·6,64,58×100)、(B·6,64,29×50)、(B·6,64,15×25) | 【可学习·2D 侧】（深度分布的唯一来源） | `detectors/uvtr.py:53-55,194-212`；`uvtr_camera_base.py:59-63` |
| 4 | 体素格点生成：`linspace(0,1,N)`（**含两端点**）×三轴 meshgrid →(128,128,5,3) 普通张量（非 Parameter/buffer）；反归一到 pc_range 度量坐标；左乘 `inverse(uni_rot_aug)` 撤 3D 增广 | (B,128,128,5,3)，每相机复用同一格点 | 【固定几何】（无参数 linspace 网格） | `models/utils/uni3d_viewtrans.py:199-202,270-276` |
| 5 | 投影（`feature_sampling` 前半）：齐次化→`lidar2img` 矩阵乘→掩码 z>eps=1e-5→uv 除以 z（透视除）→uv 除以 img_shape（原图 900×1600）再映射 [-1,1]→深度 z/64 映射 [-1,1]→hit 掩码（\|u\|<1 且 \|v\|<1 且 \|z_norm\|<1，即像内且 0<z<64 m） | (B,6,81920) 的 uv、depth 与掩码 | 【固定几何·参数自由】（纯标定投影，无学习深度修正） | `uni3d_viewtrans.py:279-303,313-330` |
| 6 | 采样：`F.grid_sample`（PyTorch 双线性，**naive 网格采样**，论文 p.15 自述可换 CUDA 算子）——四尺度特征各按 (u,v) 采样；深度分布各按 (u,v,z_norm) 三维采样 | 特征→(B,256,81920,6,1,4)；深度→(B,1,81920,6,1,4) | 【桥·参数自由】 | `uni3d_viewtrans.py:333-341,344-359`；论文附录 p.15 |
| 7 | 体素池化（加权求和入格）：`depth×mask`、`output=特征×深度概率`，对 6 相机×4 尺度求和→reshape (B,256,128,128,5)→permute →V_I (B,256,5,128,128)（Z,Y,X）；`keep_sweep_dim=True` 再抬一层 sweep 维 | →(B,1,256,5,128,128) | 【桥·参数自由】（加权求和，零参数——即论文 Eq.(2) V_I(x,y,z)=D_I(u,v,d)×F_I(u,v)） | `uni3d_viewtrans.py:215-226,165-167` |
| 8 | 体素编码器：3× [Conv3d 256→256 kernel(3,3,3) stride1 pad1 ＋ BN3d ＋ ReLU]（`conv_trans_1..3`，xavier 初始化）——在 XY 与 Z 三方向混合邻域 | (B,1,256,5,128,128) 不变 | 【可学习·3D 侧】 | `uni3d_viewtrans.py:53-81,108-111,157-163`；`uvtr_camera_base.py:68,71` |
| 9 | Transformer 解码器（Uni3DDETR，桥外下游）：`query_embedding`(900,512) 拆 query＋query_pos；`reference_points=Linear(256→3)(query_pos).sigmoid()` 生成 900 个初始参考点；6 层 [self MultiheadAttention(8头)→UniCrossAtten→FFN 256→512→256]；UniCrossAtten＝参考点映射 [-1,1]³ 后 `grid_sample` 直接采体素体、`attention_weights` Linear(256→1)+sigmoid 加权、`output_proj` 256→256、`position_encoder` MLP(3→256→256) 把参考点编码注入；`with_box_refine=True` 每层回归分支更新参考点 | (900,B,256)×6 层 | 【可学习·3D 侧下游】（消费方式＝query 从体素体拉取，检测在体素空间完成） | `models/utils/uni3d_detr.py:40,66-72,134-143,190-200,206-209,270-320`；`uvtr_camera_base.py:86-120`；`models/dense_heads/uvtr_head.py:117-119,158-180` |
| 10 | 输出头：cls 分支 2×(Linear256＋LN＋ReLU)＋Linear(256→10)，reg 分支 2×(Linear256＋ReLU)＋Linear(256→10)（code_size=10，×6 层各一套）；NMSFreeCoder 取 300 框（post_center_range ±61.2 m、Z ±10 m） | (B,900,256)→检测输出 | 【可学习·3D 侧下游】（桥之外） | `uvtr_head.py:83-115`；`uvtr_camera_base.py:81,121-127,141` |

监督仅在步骤 10 之后：`loss_cls`=FocalLoss(weight 2.0)＋`loss_bbox`=L1Loss(weight 0.25)，逐 decoder 层计算（`uvtr_camera_base.py:133-141`；`uvtr_head.py:459-491`）。深度分布 D_I（步骤 3）**不在任何损失中**，只经乘法（步骤 7）由检测损失端到端回传——与 LSS 的"显式分布、隐式监督"同构（见 §5）。

## 3. 可学习参数清单表（nuScenes 口径，uvtr_camera_base.py＋uvtr_c_r50_h5.py）

> "侧"指参数位于投影桥（步骤 5–7）的哪一端。桥本身（Uni3DDepthProj 及 feature_sampling）**零参数**。decoder/输出头按 6 层计（`with_box_refine=True` → cls/reg 分支 ×6 套）。

| 参数（组） | 所属模块 | 形状（nuScenes 口径） | 侧 | 代码出处 `path:line` |
|---|---|---|---|---|
| `img_backbone`（ResNet-50＋DCNv2(stage3/4)，FCOS3D 预训练） | UVTR | 消费 /4–/32 四 stage（256/512/1024/2048 ch） | 2D 侧（投影之前） | `uvtr_camera_base.py:40-50,261` |
| `img_neck`（FPN，4 输出 256ch） | UVTR | in_channels [256,512,1024,2048]→out 256，start_level=1，num_outs=4 | 2D 侧 | `uvtr_camera_base.py:51-58` |
| `input_proj`（1×1 Conv2d） | UVTR | Conv2d 256→256，逐尺度应用 | 2D 侧（桥前接口） | `detectors/uvtr.py:48-52,169-173` |
| `depth_net`（**深度分布唯一来源，本模型的机制名片**；单个 1×1 Conv 跨 4 尺度共享） | UVTR | Conv2d 256→**64**（=D），softmax 于通道维；输出每相机 4 份 D×fH×fW | 2D 侧（无监督） | `detectors/uvtr.py:53-55,204-209`；`uvtr_camera_base.py:59-63` |
| 体素编码器 `conv_trans_1..3` | Uni3DViewTrans | 各＝Conv3d(256,256,(3,3,3),pad1)＋BN3d(256)＋ReLU，共 3 组 | 3D 侧（池化之后） | `uni3d_viewtrans.py:70-81`；`uvtr_camera_base.py:68,71` |
| `query_embedding` | UVTRHead | `nn.Embedding(900, 512)`（拆成 query＋query_pos） | 3D 侧（decoder，桥之外） | `uvtr_head.py:117-119`（900 来自 `uvtr_camera_base.py:80`） |
| `reference_points` | Uni3DDETR | `Linear(256, 3)`（由 query_pos 生成初始参考点，xavier 初始化） | 3D 侧（decoder） | `uni3d_detr.py:40,50,69-72` |
| UniCrossAtten `attention_weights` ×6 层 | UniCrossAtten | `Linear(256, num_points=1)`，sigmoid 激活，权重零初始化 | 3D 侧（decoder 从体素体拉取时的标量门控） | `uni3d_detr.py:190,206-209,270-271`；`uvtr_camera_base.py:103` |
| UniCrossAtten `output_proj` ×6 层 | UniCrossAtten | `Linear(256, 256)` | 3D 侧 | `uni3d_detr.py:192,209,316` |
| UniCrossAtten `position_encoder` ×6 层 | UniCrossAtten | Linear(3,256)+LN+ReLU+Linear(256,256)+LN+ReLU（参考点编码注入） | 3D 侧 | `uni3d_detr.py:193-200,318` |
| decoder 每层 self-attention（MultiheadAttention 8 头）＋FFN ＋ 3×LN ×6 层 | UniTransformerDecoder | MHA 256ch/8 头；FFN 256→512→256（2 个 Linear） | 3D 侧 | `uvtr_camera_base.py:89-119`；层序 `uvtr_head.py` 转发 `uni3d_detr.py:128-133` |
| cls/reg 分支 ×6 套 | UVTRHead | cls：2×(Linear256+LN+ReLU)+Linear(256→**10**)；reg：2×(Linear256+ReLU)+Linear(256→**10**)（code_size=10；10 类，FocalLoss use_sigmoid=True → 通道数不加 1） | 3D 侧（桥之外下游） | `uvtr_head.py:45-48,83-98,108-115`；`uvtr_camera_base.py:81,133-141` |
| `SinePositionalEncoding`（num_feats=128） | UVTRHead（DETRHead 基类构建） | 构建但插件 forward 无任何引用——**死配置**（记录备查） | —（未消费） | `uvtr_camera_base.py:128-132`；插件内零引用（全仓 grep 仅此一处） |

要点：**投影桥（步骤 4–7）零参数**——体素格点、lidar2img 投影、hit 掩码、深度加权求和全部参数自由；可学习量全部在 2D 侧（backbone/FPN/input_proj/depth_net，即"什么特征、多深的概率"）与 3D 侧（体素编码器、decoder/输出头）。这与 LSS 的"学习在两端、桥零参数"完全同构，而与 BEVFormer 的"桥上有 offsets/weights 学习量"相对。论文未给总参数量（Tab.13 p.15 只有延迟），底稿不引参数量总数。

## 4. 固定几何清单表

| 项 | 具体值 | 出处 |
|---|---|---|
| 感知范围 pc_range | [-51.2, -51.2, -5.0, 51.2, 51.2, 3.0]（X/Y ±51.2 m；Z −5~3 m） | `uvtr_camera_base.py:11` |
| 体素格 voxel_shape | [128,128,5]＝((51.2+51.2)/0.2)//4 ×同 ×sample_num——名义 XY 格边长 102.4/128=**0.8 m**、Z 5 层 | `uvtr_camera_base.py:12,15-19` |
| 体素采样格点（实际） | `linspace(0,1,N)` **含两端点**：XY 共 128 点、间距 102.4/127≈**0.806 m**、首末点恰在 ±51.2 m 边界；Z 5 点 {-5,-3,-1,1,3} m、间距 8/(5-1)=**2 m**（H11 变体则为间距 0.8 m）——普通张量，非 nn.Parameter、非 buffer | `uni3d_viewtrans.py:199-202,270-272` |
| 深度分箱 | **D=64 档、每档 1 m、范围 0–64 m**（论文 "perception limit 64m" p.4）；归一化 z/64→[-1,1]，>64 m 体素被 hit 掩码剔除 | `uvtr_camera_base.py:62`；`uni3d_viewtrans.py:316-323,325-330`；论文 p.4 |
| lidar2img 来源 | `viewpad(内参 K 补 4×4) @ lidar2cam_rt.T`，其中 lidar2cam 由 `sensor2lidar` 求逆组装，纯标定量 | `datasets/nuscenes_dataset.py:236-253` |
| 投影与 hit 判定 | 齐次化→4×4 矩阵乘→z>eps=1e-5→uv 除以 z→uv 除以 img_shape（原图 900×1600）映射 [-1,1]→掩码 \|u\|<1、\|v\|<1、\|z/64−0.5\|×2<1（即像内且 0<z<64 m）；无 nan 防护 | `uni3d_viewtrans.py:279-303,313-330` |
| uv 归一化基准 | **原图 img_shape 900×1600**（`img_shape[0]`＝首相机形状），而特征图是 pad 后 928×1600——底部约 28 px（/8 约 3.5 格）存在归一化基准与采样面的系统偏移（代码现状，论文未提） | `uni3d_viewtrans.py:313-314`；`uvtr_camera_base.py:208` |
| 3D 增广及其撤销 | `UnifiedRotScaleTrans`（rot ±0.3925 rad、scale 0.95–1.05）合成 `uni_rot_aug`；投影前 `reference_voxel @ inverse(uni_rot_aug)` 在体素侧撤销；图像级 `img_rot_aug/img_trans_aug` 恒等（camera pipeline 不含图像几何增广） | `uvtr_camera_base.py:202-204`；`transform_3d.py:325-346,461-469`；`uni3d_viewtrans.py:275-276,306-311` |
| 体素池化实现 | 特征×深度概率→对 6 相机×4 尺度 **求和**（`F.grid_sample` 双线性采样；论文 p.15 自述为 naive grid sampling、可换 CUDA 算子）——非 LSS 的 cumsum scatter、非自定义 kernel | `uni3d_viewtrans.py:215-226,333-359`；论文 p.15 |
| 输入图像分辨率 | 900×1600（原生，无 Resize），pad 32 整除→928×1600 | `uvtr_camera_base.py:197-211,208` |
| 多尺度特征层（推算） | /8:116×200、/16:58×100、/32:29×50、/64:15×25，各 256ch（out_indices=(0,1,2,3)＋FPN start_level=1、num_outs=4） | `uvtr_camera_base.py:44,51-58` |
| 相机数 | 6（view_cfg `num_cams=6`；论文 §4.1 "six cameras" p.6） | `uvtr_camera_base.py:67` |
| fp16 | view_cfg `fp16_enabled=True`：lidar2img/格点/深度图在桥内 cast 半精度 | `uvtr_camera_base.py:14,77`；`uni3d_viewtrans.py:291-294` |
| GridMask 增广 | use_grid_mask=True（prob 0.7，随机几何遮挡，非学习） | `uvtr_camera_base.py:39`；`detectors/uvtr.py:47` |
| 训练超参（供 HTML 脚注） | AdamW lr=2e-4、backbone lr_mult=0.1、wd=0.01、CosineAnnealing（warmup 500 iter）、24 epochs、每 GPU batch=1（6 视图单帧）、FCOS3D 初始化、fp16 loss_scale=32 | `uvtr_camera_base.py:222,239-261,263` |
| 推理保留框 | NMSFreeCoder max_num=300，post_center_range X/Y ±61.2 m、Z ±10 m（论文 p.15 口径一致） | `uvtr_camera_base.py:121-127`；论文 p.15 |

## 5. 2D-3D 关系小结

- **推/拉方向：推家族的 3D 落点（加权语义），gather 式实现**。UVTR 与 LSS 共享同一合成语义——"显式离散深度分布 × 图像特征"决定 3D 内容（论文 Eq.(2) 即 LSS Eq.(1) 的逐采样点形式）；但实现路径相反：LSS 在 2D 侧枚举视锥点再 scatter 落格，UVTR 在 3D 侧枚举固定体素格点、经 lidar2img 投到 2D（3D→2D 定址）、`grid_sample` 拉取特征与深度概率后加权求和（2D→3D 聚合）。聚合权重是显式 64 档深度概率而非 attention 偏移——这是它与 BEVFormer 式"拉"的本质区别；落点是 **3D 体素格 5 层、保留高度不塌缩**（论文 §3.1 "preserve the 3D voxel space without collapsing in Z axis" p.4），这是它与 LSS"拍扁 2D"的本质区别。
- **深度：显式分布、无监督**。每相机每尺度各预测一份 64 档 softmax 深度分布（0–64 m、1 m/档），**无任何深度标签与深度损失**（论文 p.4 明示＋代码双确认，见 §1）；深度仅作为乘性权重由检测损失端到端回传——"显式分布、隐式（下游）监督"与 LSS 完全同构；BEVDepth 式 LiDAR 深度监督在 UVTR 中不存在（对后续 Qwen-Drive 对比节：两侧同为无监督，差异在档数/范围/尺度数而非"有无监督"）。
- **几何与学习的分工**：桥零参数——固定几何决定"落点骨架"（linspace 体素格 128×128×5、lidar2img 投影、hit 掩码、64 m 深度截断、加权求和）；学习决定"特征与深度概率"（2D 侧）与"落格后如何混合邻域、如何被 query 消费"（3D 侧 Conv3d 编码器＋decoder）。即"学习给内容与概率、几何给格点与定址"——与 LSS 同构、与 BEVFormer（桥上有 offsets/weights）相对。
- **query 有无及形态：桥上无 query，桥外有 900 object query**。体素格点是 linspace 固定网格，不存在 BEVFormer 式 bev_embedding 参数表（体素内容是"被填充物"而非"主动查询者"）；900 个 `nn.Embedding(900,512)` object query 在桥外 decoder 中以可学习参考点从体素体拉取特征。全篇对比中：UVTR＝固定 3D 格＋显式深度加权（推家族 3D 落点），与 LSS（固定 2D 视锥推＋pillar 塌缩）成第一组对照，与 VoxFormer（稀疏 voxel query 门控）成一语对照——UVTR 稠密采样满格、VoxFormer 稀疏挑选。

## 6. 论文 vs 代码差异点 + 待画图要点

### 6.1 差异与勘正记录

| # | 项 | 论文说法 | 代码事实 | 处理 |
|---|---|---|---|---|
| 1 | 机制图编号 | 设计文档留待"clone 论文后定"Fig 号 | **Fig.3 "Details in the view transform"（p.4）**即视图变换机制图；Fig.2（p.3）为多模态总框架、Fig.1（p.2）为动机 toy 图、Fig.4（p.4）为知识蒸馏细节 | A 图重绘对象取 Fig.3（裁掉 V_P/蒸馏支路，只画相机支） |
| 2 | **深度监督（最重要勘正）** | 设计文档假设"原版 UVTR 有深度监督（形式待核对）" | 论文 p.4 原文即 "It is noted that D_I is predicted without supervision"；代码无深度损失（形参穿透未用 `uvtr_head.py:407-408`←`uvtr.py:238-240`）、管线不产深度 GT（`uvtr_camera_base.py:197-211`） | **勘正：原版 UVTR 深度无监督**；HTML 对比表"深度监督"轴两侧均写"无"，差异只在档数/范围（64 档 0–64 m vs Qwen-Drive 118 档 0.5 m 步长）与分布产生方式 |
| 3 | 采样点位置 | §3.1 p.4："generated at the **center of each bin** from the voxel space" | `linspace(0,1,N)` 含两端点：Z 为 {-5,-3,-1,1,3} m（bin 中心应为 {-4.2,-2.6,-1,0.6,2.2} m）、XY 间距 102.4/127≈0.806 m 而非 0.8 m（`uni3d_viewtrans.py:199-202`） | 以代码为准：含端点等距格点；画图标 {-5,-3,-1,1,3}，格间距标名义 0.8 m＋脚注实际 0.806 m |
| 4 | FPN 尺度用法 | §3.1 p.4："FPN … aggregated that is **summed** to formulate the feature F_I" | 4 尺度各自保留：逐尺度采样特征、逐尺度各一份深度分布，深度加权后才对相机×尺度求和（`uvtr.py:204-209`；`uni3d_viewtrans.py:333-341,221-222`） | 以代码为准：4 尺度并行、加权和融合；论文"summed"表述与实现不符，画图标 4 尺度 |
| 5 | 学习率 | §4.1 p.6：camera-based "initial learning rate **1e-4** for 24 epochs" | `lr=2e-4`、24 epochs（`uvtr_camera_base.py:241,255`） | 以代码为准：2e-4；论文疑为旧口径 |
| 6 | 体素格与深度档 | §4.1 p.6："900 object queries…voxel spaces share the same shape **128×128×Z**…C=256…M=**6** for camera-based" | `num_query=900`、voxel_shape=[128,128,5]、embed_dims=256、num_layers=6 逐项吻合（`uvtr_camera_base.py:80,17-19,74,91`） | 一致（列入以示已核对） |
| 7 | D=64 与 64 m 感知上限 | §3.1 p.4："D is set to 64 to represent the perception limit 64m" | depth_dim=64（`uvtr_camera_base.py:62`）；hit 掩码 \|z/64−0.5\|×2<1 → 0<z<64 m 截断（`uni3d_viewtrans.py:316-330`） | 一致；档距 1 m 为归一化的隐含结论（z/64 且逐档），画图标 1 m/档 |
| 8 | 体素编码器 | §3.1 p.4："only **three basic convolutional blocks**"＋Fig.3 中 Conv-BN-ReLU | num_convs=3、kernel(3,3,3)＋BN3d＋ReLU（`uvtr_camera_base.py:68,71`；`uni3d_viewtrans.py:70-81`） | 一致 |
| 9 | 检测查询方式 | §3.3 p.5 Eq.(4)：CrossAttn=DeformAttn(q,p,V_U) | UniCrossAtten 实现为"参考点 grid_sample＋单标量 sigmoid 门控＋position_encoder 注入"（`uni3d_detr.py:270-320`），`num_points=1`（`uvtr_camera_base.py:103`）——非多头多点 deformable | 以代码为准描述；论文"DeformAttn"为借喻表述，画图按代码画（一点采样＋门控） |
| 10 | batch | 附录 A p.14：camera-based "8 devices with … 8 input data"（＝1/GPU） | `samples_per_gpu=1`（`uvtr_camera_base.py:222`） | 一致 |
| 11 | GridMask | 论文未提 | use_grid_mask=True、prob 0.7（`uvtr_camera_base.py:39`；`detectors/uvtr.py:47`） | 代码侧训练细节，脚注级别 |
| 12 | fp16 桥内半精度 | 论文未提 | view_cfg fp16_enabled=True，lidar2img/格点/深度图桥内 cast half（`uvtr_camera_base.py:77`；`uni3d_viewtrans.py:291-294`）——Qwen-Drive 对比轴 5 素材：Qwen-Drive 视锥保 fp32，原版反而半精度 | 记录备查（对比表工程细节轴原版侧素材） |
| 13 | 位置编码配置 | — | SinePositionalEncoding 构建（`uvtr_camera_base.py:128-132`）但插件 forward 零引用（死配置） | 记录备查，不进图 |
| 14 | uv 归一化基准 | — | 用原图 img_shape 900×1600 归一 uv，采样却在 pad 后 928×1600 特征上（`uni3d_viewtrans.py:313-314` vs `uvtr_camera_base.py:208`）——底部约 28 px 系统偏移 | 记录备查，不影响机制叙事 |
| 15 | 性能数字 | Tab.7 p.8：UVTR-C R50 41.9 NDS/33.3 mAP；Tab.1 p.7：高度 1/5/11 → 相机 NDS 31.4/34.5/35.6 | README.md:62–64 给 R50-H5 40.1/31.3、R50-H11 41.8/33.3、R101 44.1/36.1（模型卡指标，训练口径略异于论文） | HTML 引性能时标"论文 Tab.x 值"或"README 模型卡值"，二者不混用 |

其余核对一致项：Eq.(1) 单卷积＋softmax（§3.1 p.4 ↔ `uvtr.py:53-55,204-209`）、Eq.(2) 概率×特征（§3.1 p.4 ↔ `uni3d_viewtrans.py:218-222`）、pc_range ±51.2/−5~3（§4.1 p.6 ↔ `uvtr_camera_base.py:11`）、300 框推理（p.15 ↔ `uvtr_camera_base.py:125`）、naive grid sampling 自述（p.15 ↔ `uni3d_viewtrans.py:338,351,357`）、统一增广"相机版仅全局旋转缩放"（附录 A p.14 ↔ `uvtr_camera_base.py:202-204`）。

### 6.2 待画图要点（Fig.3 重绘规范＋C 图清单＋B 面板数字白名单）

- **A 图＝论文 Fig.3（p.4）"Details in the view transform" 重绘（只画相机支）**：
  - 保留构图元素：图像特征 F_I →（单卷积＋softmax）深度分布 D_I（64 档柱状）；体素格点 (x,y,z) →标定矩阵 P→ (u,v,d)；"sample with prob"（按概率采样）箭头把 F_I 与 D_I 在 (u,v,d) 处会合成体素特征；V_I 体素特征空间；VoxelEncoder（Conv-BN-ReLU ×3）。原图中 V_P 支路与 ℒ_KT 知识蒸馏部分裁掉（超出相机口径）。
  - 叠加编码（本笔记新增）：① 可学习元素实色——F_I、D_I（深度分布柱）、Conv3d 编码器；② 固定几何元素虚线/灰——linspace 体素格 128×128×5、lidar2img 投影、hit 掩码（64 m 截断）、加权求和符号 Σ；③ 注记 D=64@1 m（0–64 m）、128×128×5、C=256、Σ over 6 相机×4 尺度、无监督标签。
  - 一句话构图意图：让读者看到"深度概率在 2D 侧学出来、3D 落点由固定格点收下并保高度"——与 LSS 分册的"视锥推举拍扁"图并排即成推家族两种落点的对照。
- **C 图（机制流程图）步骤清单**（逐步标【固定几何】/【可学习】，沿用全局三色带）：① 标定 lidar2img＋uni_rot_aug（固定）→ ② ResNet-50-DCN backbone（可学习·2D）→ ③ FPN 4 尺度 256ch＋input_proj（可学习·2D）→ ④ depth_net 1×1→64＋softmax、每尺度一份、无监督（可学习·2D）→ ⑤ 体素格点 linspace 128×128×5 反归一＋逆增广（固定）→ ⑥ lidar2img 投影＋hit 掩码（0<z<64 m、像内）（固定）→ ⑦ grid_sample 特征与深度概率（桥·参数自由）→ ⑧ 特征×概率对 6 相机×4 尺度求和入格 →V_I (B,256,5,128,128)（桥·参数自由）→ ⑨ 体素编码 3×Conv3d(3,3,3)（可学习·3D）→ ⑩ 900 query×6 层 decoder（UniCrossAtten 参考点 grid_sample＋sigmoid 门控；可学习·3D·桥外）→ cls/reg 出 3D 框（可学习·3D·桥外）。形状流注记 (B,6,3,928,1600)→(B·6,256,116×200,…)→(B,256,5,128,128)→(B,900,256)。
- **B 面板数字白名单（uvtr3d 面板允许出现的全部机制数字，一律出自本底稿）**：
  - 深度：**64 档、步长 1 m、范围 0–64 m**（>64 m 点不落格）；
  - 体素：**128×128×5**（H5 口径）、XY 名义格 **0.8 m**（实际采样间距 102.4/127≈0.806 m，面板标称 0.8 m 即可）、Z 采样点 **{-5,-3,-1,1,3} m**（5 层、间距 2 m、含端点）；
  - 范围：X/Y **±51.2 m**、Z **−5~3 m**；
  - 特征与采样：**256 通道**、**6 相机**、**4 尺度**（/8 116×200…/64 15×25）、加权求和；
  - 输入：**900×1600**（pad 后 928×1600）；
  - 下游（若面板文案提及）：**900 query、6 层 decoder、3×Conv3d 3³**；
  - 两态语义约束：「UVTR 码放」＝候选点按 z 落 5 层体素格（保高度）；「LSS 拍扁」＝同一批点全落地面 2D 格、pillar 求和亮度——**拍扁态的地面网格仍用 UVTR 自己的 128×128@0.8 m XY 网格**，不得引入 LSS 节的 41 档/0.512 m 等数字（那是第一节口径）；
  - 禁入数字：41、105、0.512 m、200×200（LSS/BEVFormer 节）、118、56×32×118、16,200,200（Qwen-Drive 节）——面板任何文案不得出现。

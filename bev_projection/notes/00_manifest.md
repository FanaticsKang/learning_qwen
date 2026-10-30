# 00 · 素材 manifest（BEV 投影对比笔记）

前五仓（lift-splat-shoot、BEVFormer、TPVFormer、VoxFormer、UVTR）均为 `--depth 1` 浅克隆（默认分支）；克隆日期前四仓 2026-10-26、UVTR 2026-10-28；Qwen-Drive-1.0 为完整克隆（2026-10-28，11 commits，见行内注）；PETR 一仓两快照（2026-10-29，main＝PETRv2＋PETRv2 合入前的历史 commit＝PETR v1，**仓内无 `petr` 分支**——GitHub API 实测仅 `main`/`flash`，README 亦无 v1 分支说明，见行内注）。`code/` 与 `papers/` 不入 git（未跟踪），本 manifest 的 commit hash 为后续底稿与 HTML 页脚的唯一引用来源。

| 仓库 | URL | commit hash | 克隆日期 | nuScenes 配置定位线索 |
|---|---|---|---|---|
| lift-splat-shoot | https://github.com/nv-tlabs/lift-splat-shoot | `2903467c91ee9c12f0917a12c22ab1f04e607ae0` | 2026-10-26 | 无 config 文件。`main.py` 为 fire 入口（不含参数定义），参数默认值在 `src/train.py::train()` 函数签名：`nepochs=10000`，`xbound=[-50.0, 50.0, 0.5]`，`ybound=[-50.0, 50.0, 0.5]`，`zbound=[-10.0, 10.0, 20.0]`，`dbound=[4.0, 45.0, 1.0]`，`lr=1e-3` 等；投影/坐标工具在 `src/tools.py`（`gen_dx_bx`、`cumsum_trick`、`get_rot`、`ego_to_cam` 等） |
| BEVFormer | https://github.com/fundamentalvision/BEVFormer | `66b65f3a1f58caf0507cb2a971b9c0e7f842376c` | 2026-10-26 | `projects/configs/bevformer/bevformer_base.py`（`bevformer_small.py`、`bevformer_tiny.py` 同目录，fp16 变体在 `projects/configs/bevformer_fp16/`，v2 在 `projects/configs/bevformerv2/`）；nuScenes 数据集定义在 `projects/configs/datasets/custom_nus-3d.py` |
| TPVFormer | https://github.com/wzzheng/TPVFormer | `459bc060901c9c4920f802252f04b290a449e4a1` | 2026-10-26 | 配置目录为 `config/`（本仓无 `projects/configs/`）：`config/tpv04_occupancy.py` 与 `config/tpv_lidarseg*.py`（nuScenes-LiDARSeg/Occ3D）；数据集基座 `config/_base_/dataset.py`（`data_path="data/nuscenes/"`，label mapping 用 `config/label_mapping/nuscenes.yaml`） |
| VoxFormer | https://github.com/NVlabs/VoxFormer | `63da92415447c2e97654bad977254c907f5ce703` | 2026-10-26 | 无 nuScenes 支持（README 未勾选、论文 0 提及）——底稿与笔记按 SemanticKITTI 口径：`projects/configs/voxformer/voxformer-S.py`（SemanticKITTI 单目 SSC 基准）；同目录 `voxformer-T.py`、`*_deform3D.py` 亦为 SemanticKITTI 配置；`qpn.py` 为 stage-1 QPN 训练配置（SemanticKITTI，其 nuScenes class_names 为死配置） |
| UVTR | https://github.com/JIA-Lab-research/UVTR | `8a24ec2f1f2fe9f6b65bed201dae112f83e4aaa7` | 2026-10-28 | 相机口径：`projects/configs/uvtr/camera_based/uvtr_camera_base.py`（基座）＋`camera/uvtr_c_r50_h5.py`（实例化，纯继承）；变体 `camera/uvtr_c_r50_h11.py`、`uvtr_c_r101_h11.py`、`camera_sweep/*`、`knowledge_distill/*` 同目录不引用；LiDAR/多模态在 `lidar_based/`、`multi_modality/`。核心代码：`models/utils/uni3d_viewtrans.py`（视锥/体素采样）、`models/detectors/uvtr.py`（depth_net）、`models/dense_heads/uvtr_head.py`（损失，无深度监督项）、`models/utils/uni3d_detr.py`（decoder）。**深度无监督**（论文 p.4 明示＋代码无深度 loss） |
| Qwen-Drive-1.0 | https://github.com/QwenLM/Qwen-Drive-1.0 | `28091c1532e869bc7aee91fc0aef6b3e6fd0b2e0` | 2026-10-28 | 无 config 文件体系，超参冻结于 `src/qwen_drive_perception/configuration_perception.py`（det_pc_range/frustum_range/frustum_size/118 档深度等，inference-only 无训练代码与损失）；核心代码：`src/qwen_drive_perception/{view_transform.py,heads.py,modeling_perception.py,perception_transformer.py,bev_encoder.py,attention.py,ops/}`、文档 `docs/perception.md`。训练双口径：nuScenes 6 相机＋OpenScene/nuPlan 8 相机、固定 896×512（`docs/perception.md:110-112`）。仓在项目根 `Qwen-Drive-1.0/`（完整克隆、非浅克隆；不在 `bev_projection/code/`），底稿路径相对其仓根书写（见 `06_qwendrive_uvtr.md`） |
| PETR | https://github.com/megvii-research/PETR | `3617936cd5f6320eb046738acf8f9847b3c8101f` | 2026-10-29 | **v1 快照＝`code/PETR-v1/`（浅克隆后 fetch 指定 commit）**：PETRv2 代码于 `812d423e5d4fa213b6e2d8f2f9e4c3d5aa7f59a3`（2022-06-16 "petrv2"）合入 main，本行 hash 为其之前最后一个纯 v1 提交（2022-06-10 "LICENSE"）——仓内无 `petr` 分支（API 实测仅 `main`/`flash`），README 亦无 v1 分支说明句。nuScenes 检测口径：`projects/configs/petr/petr_r50dcn_gridmask_p4.py`（R50-DCN＋P4 /16、512×1408、900 query、LID 64 bins、6 层 decoder；VoVNet-99 1600×640 变体 `petr_vovnet_gridmask_p4_1600x640.py`）。核心代码：`projects/mmdet3d_plugin/models/dense_heads/petr_head.py`（3D PE 生成＋query）、`models/utils/petr_transformer.py`（全局 cross attention）、`models/utils/positional_encoding.py`（多相机 2D PE）。**无深度监督、无 bev_embedding、无体素格、无 deformable（执行路径）**（见 `07_petr.md` §1） |
| PETRv2 | https://github.com/megvii-research/PETR | `f7525f93467a33707ef401c587a52d5e7b34de74` | 2026-10-29 | **main 分支＝`code/PETR/`（`--depth 1` 浅克隆）**，2023-10-11 "3D lane detection" 为 HEAD。nuScenes 口径：检测 `projects/configs/petrv2/petrv2_vovnet_gridmask_p4_1600x640_trainval_cbgs.py`（1500 query、640×1600、12 图含 1 历史 sweep、with_fpe/with_time/with_multi）与 `petrv2_vovnet_gridmask_p4_800x320.py`（900 query 基线）；BEV 分割 `petrv2_BEVseg.py`（625＝25×25 seg query）。核心代码：`projects/mmdet3d_plugin/models/dense_heads/petrv2_head.py`（FPE＝SELayer 门控、with_time 速度归一、解耦回归）、`dense_heads/petr_head_seg.py`（BEV 分割 query）、`datasets/pipelines/loading.py`（多 sweep 加载、数据层时序对齐）。时序对齐烘焙在 info pkl（README："30 previous frames, whose transformation matrix is aligned with the current frame"） |

> 注：计划文档中 LSS 的 URL 写作 `nv-tabs/lift-splat-shoot`，该地址 404（组织不存在）；实际克隆自 `nv-tlabs/lift-splat-shoot`（NVIDIA Toronto AI Labs，LSS 作者 Jonah Philion 所在组织，与仓内 LICENSE 头部链接一致）。

## papers/（均以 `%PDF` 开头校验通过）

| 论文 | 文件名 | 来源 |
|---|---|---|
| Lift, Splat, Shoot（ECCV 2020） | `lss_2008.05711.pdf` | 下载自 arxiv.org/pdf/2008.05711 |
| BEVFormer（ECCV 2022） | `bevformer_2203.17270.pdf` | 下载自 arxiv.org/pdf/2203.17270 |
| TPVFormer（CVPR 2023） | `tpvformer_2302.07817.pdf` | 复制自 `reference/papers/` |
| VoxFormer（CVPR 2023） | `voxformer_2302.12251.pdf` | 复制自 `reference/papers/` |
| UVTR（NeurIPS 2022） | `uvtr_2206.00630.pdf` | 下载自 arxiv.org/pdf/2206.00630 |
| PETR（ECCV 2022） | `petr_2203.05625.pdf` | 下载自 arxiv.org/pdf/2203.05625 |
| PETRv2（ICCV 2023） | `petrv2_2206.01256.pdf` | 下载自 arxiv.org/pdf/2206.01256 |

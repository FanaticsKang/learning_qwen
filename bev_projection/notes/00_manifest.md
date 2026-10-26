# 00 · 素材 manifest（BEV 投影对比笔记）

四仓均为 `--depth 1` 浅克隆（默认分支），克隆日期 2026-10-26。`code/` 与 `papers/` 不入 git（未跟踪），本 manifest 的 commit hash 为后续底稿与 HTML 页脚的唯一引用来源。

| 仓库 | URL | commit hash | 克隆日期 | nuScenes 配置定位线索 |
|---|---|---|---|---|
| lift-splat-shoot | https://github.com/nv-tlabs/lift-splat-shoot | `2903467c91ee9c12f0917a12c22ab1f04e607ae0` | 2026-10-26 | 无 config 文件。`main.py` 为 fire 入口（不含参数定义），参数默认值在 `src/train.py::train()` 函数签名：`nepochs=10000`，`xbound=[-50.0, 50.0, 0.5]`，`ybound=[-50.0, 50.0, 0.5]`，`zbound=[-10.0, 10.0, 20.0]`，`dbound=[4.0, 45.0, 1.0]`，`lr=1e-3` 等；投影/坐标工具在 `src/tools.py`（`gen_dx_bx`、`cumsum_trick`、`get_rot`、`ego_to_cam` 等） |
| BEVFormer | https://github.com/fundamentalvision/BEVFormer | `66b65f3a1f58caf0507cb2a971b9c0e7f842376c` | 2026-10-26 | `projects/configs/bevformer/bevformer_base.py`（`bevformer_small.py`、`bevformer_tiny.py` 同目录，fp16 变体在 `projects/configs/bevformer_fp16/`，v2 在 `projects/configs/bevformerv2/`）；nuScenes 数据集定义在 `projects/configs/datasets/custom_nus-3d.py` |
| TPVFormer | https://github.com/wzzheng/TPVFormer | `459bc060901c9c4920f802252f04b290a449e4a1` | 2026-10-26 | 配置目录为 `config/`（本仓无 `projects/configs/`）：`config/tpv04_occupancy.py` 与 `config/tpv_lidarseg*.py`（nuScenes-LiDARSeg/Occ3D）；数据集基座 `config/_base_/dataset.py`（`data_path="data/nuscenes/"`，label mapping 用 `config/label_mapping/nuscenes.yaml`） |
| VoxFormer | https://github.com/NVlabs/VoxFormer | `63da92415447c2e97654bad977254c907f5ce703` | 2026-10-26 | 无 nuScenes 支持（README 未勾选、论文 0 提及）——底稿与笔记按 SemanticKITTI 口径：`projects/configs/voxformer/voxformer-S.py`（SemanticKITTI 单目 SSC 基准）；同目录 `voxformer-T.py`、`*_deform3D.py` 亦为 SemanticKITTI 配置；`qpn.py` 为 stage-1 QPN 训练配置（SemanticKITTI，其 nuScenes class_names 为死配置） |

> 注：计划文档中 LSS 的 URL 写作 `nv-tabs/lift-splat-shoot`，该地址 404（组织不存在）；实际克隆自 `nv-tlabs/lift-splat-shoot`（NVIDIA Toronto AI Labs，LSS 作者 Jonah Philion 所在组织，与仓内 LICENSE 头部链接一致）。

## papers/（均以 `%PDF` 开头校验通过）

| 论文 | 文件名 | 来源 |
|---|---|---|
| Lift, Splat, Shoot（ECCV 2020） | `lss_2008.05711.pdf` | 下载自 arxiv.org/pdf/2008.05711 |
| BEVFormer（ECCV 2022） | `bevformer_2203.17270.pdf` | 下载自 arxiv.org/pdf/2203.17270 |
| TPVFormer（CVPR 2023） | `tpvformer_2302.07817.pdf` | 复制自 `reference/papers/` |
| VoxFormer（CVPR 2023） | `voxformer_2302.12251.pdf` | 复制自 `reference/papers/` |

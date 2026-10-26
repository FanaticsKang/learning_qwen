# BEV 投影机制对比笔记 · 设计文档（2026-10-26）

## 意图与动机

用户在研究多相机 3D 感知中"2D 图像特征如何变成 3D 表征"的四种代表做法，要一篇**完全独立于 Qwen-Drive 分析系列**的中文可视化笔记。核心视角：**可学习参数和 2D-3D 投影的关系**——每条路线里，哪些东西是相机标定/深度离散化决定的固定几何，哪些参数可学习、长在 2D 侧还是 3D 侧、以什么形态存在（深度分布、采样权重、query 嵌入、深度提议、mask token）。机制图的画法参考四篇论文原图的表达，但笔记内容必须经官方开源代码校对，不能只看论文。

读者：用户本人（研究者/工程师，专家级）。

## 目标

1. 单文件 HTML 笔记，放新顶层目录 `bev_projection/index.html`，与 `analysis/` 系列零耦合：不引用其编号与内容，不出现 Qwen-Drive、UVTR、"本仓库"等概念。
2. 结构：四个模型逐个走（LSS → BEVFormer → TPVFormer → VoxFormer），最后汇总对比。
3. 每个模型一节，固定模板：
   ① 机制总览：参考该论文原图重绘的 SVG 示意图（投影方向、数据流，视觉上区分"可学习参数"与"固定几何"）；
   ② 可学习参数清单：参数名、张量形状、挂在哪一侧（2D 侧 / 3D 侧 / 桥上）、代码出处（文件+函数）；
   ③ 2D-3D 关系小结：谁发起对应（推/拉）、深度显式还是隐式、几何与学习的分工；
   ④ 代码校对框：开源代码与论文说法的印证或差异（关键函数、配置项）。
4. 末尾汇总：四家对比大表 + 一张谱系总览图（可学习参数的位置与形态 × 投影方向；谱系：无 query → 稠密 query → 三平面压缩 query → 稀疏深度门控 query）。
5. 笔记中所有具体数字（深度分箱数、query 形状、平面尺寸、每 query 采样点数）以代码为准，**统一取各仓库的 nuScenes 配置口径**（LSS、BEVFormer-base、TPVFormer-nuScenes、VoxFormer-nuScenes），每处标注配置文件路径与代码仓 commit hash；代码中找不到的标"论文值，代码未见"。

## 非目标

- 不复述各论文整体架构、实验结果、精度对比——只讲投影（view transform）环节。
- 不讲训练技巧、部署优化、时序融合（BEVFormer 的 temporal 只一句带过，不展开）。
- 不引入第五种方法（BEVDepth、FB-BEV 等）——只用户点名的四家。

## 反目标（做错了也算失败）

- 与 `analysis/` 系列发生内容或编号上的牵连。
- 只凭论文或记忆写数字——每个数字要么代码验证，要么明确标注来源。

## 约束

- 中文排版按 AGENTS.md：①②③ / 1/2/3 / A/B/C 列表换行、左对齐；中文语境用全角逗号"，"，英文用","。
- HTML 完全离线自包含：无 CDN 依赖，全部内联（样式、SVG、JS）。
- 示意图为静态 SVG：无悬停交互；箭头从框的底边/侧边出（禁顶边出）；连线少折弯、零交叉；虚线分组框；四节的机制面板风格统一。
- 交付前在浏览器实测（渲染、布局、图例与实际一致）。

## 方案选择（已与用户确认）

| 决策点 | 备选 | 选定 |
| --- | --- | --- |
| 结构骨架 | 轴先行总览再展开 vs 逐模型后汇总 | 逐模型、最后汇总 |
| 事实来源 | 仅论文 vs 论文+开源代码 | 下载四家官方开源代码校对 |
| 与 Qwen-Drive 系列关系 | 并入 analysis/ vs 完全独立 | 完全独立 |
| 数字口径 | 各论文主配置（2 nuScenes + 2 SemanticKITTI） vs 统一 nuScenes | 统一 nuScenes |

## 留给建造者的自由度（默认值，可推翻）

- 目录名 `bev_projection/`、文件名 `index.html`。
- 四家代码浅克隆到 `bev_projection/code/`：nv-tabs/lift-splat-shoot、fundamentalvision/BEVFormer、wzzheng/TPVFormer、NVlabs/VoxFormer（官方仓）；git 不跟踪。
- 四篇论文 PDF 集中放 `bev_projection/papers/`：LSS 与 BEVFormer 从 arXiv 下载，TPVFormer 与 VoxFormer 从 `reference/papers/` 复制（原处保留），本工作自包含。
- 四节顺序即谱系顺序：LSS（无 query，学习全在 2D 侧）→ BEVFormer（稠密 3D query 拉取）→ TPVFormer（三平面压缩 query）→ VoxFormer（深度门控的稀疏 query）。
- 默认全静态 SVG；若建造中发现某机制静态图讲不清，可加一个最小交互动画，先记待办再实现。
- 不生成 PNG 附件，只交 HTML。

## 建造阶段要点

1. 先克隆四仓 + 下载两篇 PDF，通读关键文件（每家的 view-transform 模块），整理"可学习参数 × 形状 × 出处"底稿，再动笔写 HTML。
2. 论文图对照：LSS 图 2（视锥 lift+splat）、BEVFormer 图 2（空间交叉注意力柱采样）、TPVFormer 图 2（三平面流水线）、VoxFormer 图 2（两阶段稀疏 query）。重绘时保留原图的构图语言，但配色与图例统一成本笔记体系，并叠加"可学习/固定"的视觉编码。
3. 浏览器实测后交付；提交按用户 git 习惯（英文提交信息、本地 commit）。

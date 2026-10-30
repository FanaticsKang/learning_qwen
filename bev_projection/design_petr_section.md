# 设计：bev_projection 增补 PETR/PETRv2 一节

日期：2026-10-29
状态：待批准
范围：`bev_projection/index.html` ＋ `bev_projection/papers/` ＋ `bev_projection/code/`

## 1. 意图

读者在通读六个条目后自然产生一个问题："为什么所有路线要么 2D 像素预测深度分布（推）、要么 BEV/体素 query 预测 2D 采样（拉）——为什么不用 2D 像素和 3D query 直接做 cross attention？" PETR（ECCV 2022）正是这条路线的答案：投影桥整体消失，换成 3D position embedding 的"软桥"，对应关系由全局 attention 自己学。增补后全篇从"五机制＋一工程实现"扩为"六机制＋一工程实现"，PETR 作为全新谱系（学习直连）独立成卡，并回答上述问题。

## 2. 目标

1. **第七节（新）PETR**：Position Embedding Transformation for Multi-View 3D Object Detection（ECCV 2022，Yanwei Li 等）。核心叙事＝"不推不拉：固定几何只负责给每个像素发 3D 位置签名（3D PE，MLP 可学习但不做几何对应），object query（数量待核，约 900）与全部相机的全部像素 token 做全局 cross attention，谁对应谁由 Q·K 内积学出"。数字一律以官方仓 clone 校对为准。
2. **节末 PETRv2 补充小节（不立节）**：时序对齐的 3D PE（历史 query 坐标投影到当前帧——3D→2D 投影在此回归一次）、feature-guided position encoder（PE 从纯几何变为几何×特征联合生成）、BEV 分割 query（v2 往 BEV 方向的回流）。
3. **B 交互面板 `petr3d`**（挂 F3D 公共库，IIFE 形式）：
   - 熄灭态：6 相机线框＋每相机中心射线上的固定深度 bins 点云（固定几何，灰）；
   - 三态由单个按钮循环切换（熄灭 → 3D PE → 全局 attention，按钮文案随态变化）——为既有两态切换模式的最小扩展，不引入时间轴动画；
   - 态 1（3D PE）：射线 bins 变橙——点不落进任何格，只生成位置签名附着在像素上；
   - 态 2（全局 attention）：3D query 在空间自由移动（不锁任何栅格，与 BEVFormer 面板"只能拖 BEV 格"形成操作层面对照），到各相机的 soft attention 线按示意相似度点亮。
   - 可旋转/缩放。
4. **全篇连锁更新**（见 §7 清单）。

## 3. 非目标

- 不改动既有六节的机制内容与面板（只动编号与交叉引用）。
- 不加 StreamPETR/CMT/Sparse4D 等后续工作——PETRv2 补充小节一句话点名谱系延伸即可。
- 不做 PETR 的 Waymo 口径，只用 nuScenes。
- PETRv2 不单独立节、不做独立交互面板。

## 4. 反目标

- **数字不可凭印象写**：本节所有数字（query 数量、深度 bins 数、图像 token 分辨率、MLP 结构、论文 Fig 号）在 clone 官方仓核对前不得进入 HTML；叙事中出现的数字仅存在于本设计文档的"待核"标注下。
- 不破坏全篇既有审计项：单 `<script>` 每面板一块（IIFE）、零外部资源、无 `Math.random`、中文标点规范、SVG 内徽标 inline stroke-width、面板数字白名单。
- 深度谱系扩为四家族时，不得改动既有三家族的任何表述。

## 5. 方案取舍

- **方案一（选定）**：PETR＝全新谱系独立成卡，节插在 Qwen-Drive 之后（第七节）、汇总之前；PETRv2 作为节末补充。理由：PETR 无投影桥、无 BEV 栅格，机制上与推/拉两家族的差异大于共性；且直接回答"2D↔3D 能否直连 attention"的读者问题。
- **方案二（弃）**：PETR 挂在 BEVFormer 拉流分支下，作为"拉取的极端形式"。理由：弱化其独立性——PETR 连 BEV 栅格都没有，"拉"的定义（3D→2D 投影定锚采样）在 PETR 中不成立。
- **方案三（弃）**：插在 TPVFormer 之后（按时间/谱系演进线）。理由：会打断"推→拉→门控稀疏→合流"的既有叙事弧，且改动更多节编号。

## 6. 第七节内容规格（`id="sec-petr"`）

- 谱系 tag：`谱系定位：全新谱系·学习直连——不推不拉，对应关系由 attention 学`（终稿措辞可调，"全新谱系"为用户指定口径）。
- 开场对照段：LSS/UVTR 推（像素预测深度分布、枚举落点）；BEVFormer/TPVFormer 拉（3D query 投影回图像采样）；PETR 两者都不做——投影桥消失，3D PE 软桥接管，几何只发"身份签名"不做对应。
- ① A 图（静态 SVG，三段横排）：左＝图像 token（灰）＋视锥 3D 锚点（蓝色虚线斜纹，固定几何）→ 3D PE 生成（MLP，橙）；中＝拼接后的带 3D PE 的 key（灰底橙描边小标）；右＝object query（橙，数量待核）——一条粗 attention 线横扫全部 key。**全图无蓝色虚线投影线、无采样簇**（与前几节图最大的视觉差异，需在图注中点明）。
- ① B 面板：见 §2.3 `petr3d`。
- ① C 图（流程线）：backbone/FPN → 3D PE 生成（视锥 bins＋MLP）→ key 拼接 → 全局 cross attention → decoder 出框；逐步标【固定几何】/【可学习】。
- ② 可学习参数清单（表：模块/位置/形态/数量，行号待 clone 核对）：
  - 3D PE 的 MLP——桥上、可学习、但不做几何对应（全篇唯一"桥上有可学习量且不做采样"的条目，重点标出）；
  - object query（数量待核）内容与 3D 锚点（3D 侧）；
  - backbone/FPN/input_proj（2D 侧）；
  - **无**项专列一行：深度头、bev_embedding、体素格、deformable 偏移表。
- ③ 2D-3D 关系（四个固定问题逐条对比）：
  - 推/拉方向：都不算——无特征流"过桥"，对应关系在 attention 权重里隐式成立（新谱系定义性特征）；
  - 深度：不预测数值深度——但视锥固定 bins 仍存在（3D PE 原料），深度不确定性由全局 attention 权重隐式分配；
  - 几何与学习的分工：分工边界移位——固定几何退到"只发坐标"（内参＋视锥 bins），可学习量第一次出现在桥上（PE 的 MLP）；
  - query 形态：稀疏（数量待核）、自由 3D 坐标、无任何网格绑定。
- 节末 PETRv2 补充小节：见 §2.2；数字待 clone main 分支核对（v1 取 main 历史 commit `3617936`——**仓内无 `petr` 分支**，GitHub API 实测仅 `main`/`flash`、README 亦无 v1 分支说明；v2 取 main 分支 HEAD）。

## 7. 全篇连锁更新清单

1. **章节编号**：原"七、汇总对比"→"八、"、原"八、深度从哪来"→"九、"（`<h2>` 文字与目录两处同步）；目录插入 PETR 条目；正文交叉引用（"第七节汇总"等，已确认 2 处）逐一核对改写。
2. **引言**：口径改为"六个机制条目＋一条工程实现，共七节"＋汇总＋深度谱系收束（共九节）；论文列表（项目号换行）插入 PETR 行（含 PETRv2 同节说明）。
3. **汇总大表（八节①）**：加 PETR 列——query 形态＝稀疏（数量待核）自由 3D 坐标无网格绑定；参数位置＝2D 侧＋3D 侧＋桥上（PE MLP）；推拉方向＝都不算（学习直连）；深度显隐＝不预测数值（视锥 bins 只作 PE 原料）；对应建立＝Q·K 相似度隐式。口径脚注补 PETR 仓分支口径。
4. **谱系总览图（八节② SVG）**：新增 PETR 卡片（独立于推/拉两支），布局重排（图原为六个条目排版）；拉线注脚与 scope 同步。
5. **深度谱系（九节）**：新家族条目"族④：无假设·隐式全局分配"；三家族图扩为四家族；汇总表加 PETR 行；收束段补第三点（PETR 证明"深度可以完全不预测"成立）。
6. **页脚锚定表**：加 PETR（arXiv:2203.05625）与 PETRv2（arXiv:2206.01256）两行，仓同为 `github.com/megvii-research/PETR`（v1 标历史 commit `3617936` hash、v2 标 main 分支 hash；仓内无 `petr` 分支），克隆日期 2026-10-29；footer-note "上表六仓"→"八仓"、口径说明补 PETR。
7. **素材下载**（实现第一步）：`papers/` 加两 PDF；`code/PETR/`（main 分支浅克隆，＝PETRv2）＋ `code/PETR-v1/`（main 历史 commit `3617936cd5f6` 浅取——PETRv2 合入前的纯 v1 快照；仓内无 `petr` 分支，见 §6 勘正）。

## 8. 验收

- HTML 标签闭合校验通过；单面板单 `<script>`、无外部资源审计与既有面板一致。
- 所有进入 HTML 的数字均有 `code/PETR/`（或分支）的 path:line 或论文页码锚定；锚定不到的写"论文值，代码未见"。
- 目录 9 条与 9 个 `section id` 一一对应；正文交叉引用无错号。
- 深度谱系四家族表述与既有三家族零冲突。

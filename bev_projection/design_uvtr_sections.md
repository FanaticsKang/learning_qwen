# 设计：bev_projection 增补 UVTR 与 Qwen-Drive 两节

日期：2026-10-28
状态：待批准
范围：`bev_projection/index.html` ＋ `bev_projection/notes/`

## 1. 意图

现有四模型对比笔记（LSS／BEVFormer／TPVFormer／VoxFormer）缺两块：推家族里"3D 落点"的代表 UVTR（与 LSS 同思想、不同落点），以及这份笔记读者最终关心的"工业界怎么用"——Qwen-Drive 把 UVTR 推流与 BEVFormer 拉流合流的真实实现。增补后全篇从"四条路线"扩为六个条目＋汇总，2D↔3D 变换关系仍是唯一主线。

## 2. 目标

1. **第二节（新）原版 UVTR**：NeurIPS 2022（Yanwei Li 等），nuScenes 口径。核心叙事＝"与 LSS 同一条推路线、两个落点：LSS 拍扁 2D（高度当场丢）vs UVTR 码放 3D 体素（高度保留）"。数字一律以官方仓 clone 校对为准（并入 manifest）。
2. **第六节（新）Qwen-Drive 的 UVTR 实现**：双流感知头（UVTR 推＋BEVFormer 拉）中的推流实际处理与合流方式。唯一事实来源＝本地 `Qwen-Drive-1.0`（commit 28091c1），逐项 path:line。
3. **Qwen-Drive 节内 ④ 对比小节**：原版 UVTR vs Qwen-Drive 实现五轴对比（特征源／深度监督／离散化／消费方式／工程细节），双侧 path:line。
4. **两个 B 交互面板**（挂 F3D 公共库，IIFE 形式）：
   - `uvtr3d`：同一根视锥射线的候选点，按钮两态切换——「UVTR 码放」（点落入 3D 体素格，z 层保留）vs「LSS 拍扁」（同批点全落地面 2D 格，pillar 求和、高度塌缩）。可旋转/缩放。
   - `qd3d`：熄灭＝体素网格灰＋相机环视（固定几何）；点亮＝体素亮橙→压平语义（体素列坍缩到地面 BEV 格）→query 格显橙（＝bev_embedding＋UVTR 加成）→SCA 拉取线到命中相机；拖拽 query 格联动拉取线（复用 bev3d 模式）。可旋转/缩放。两态切换，不做时间轴动画（与既有面板及测试方式一致）。
5. **全篇连锁更新**（见 §7 清单）。

## 3. 非目标

- 不改动既有四节的机制内容与面板（只动编号与交叉引用）。
- 不引入 analysis/ 系列笔记的任何链接或来源（项目分离原则不变；Qwen-Drive 事实重新对 Qwen-Drive-1.0 代码核对后引用）。
- Qwen-Drive 节不展开 VLM／语言侧／规划侧——只讲感知头的投影桥；TSA 时序一句带过（与 BEVFormer 节口径一致）。
- 不做 UVTR 的 Waymo 口径，只用 nuScenes。

## 4. 反目标

- **数字不可凭印象写**：UVTR 节所有数字（视锥尺寸、深度档数、体素网格、深度监督形式、论文 Fig 号）在 clone 官方仓核对前不得进入 HTML；Qwen-Drive 节所有数字必须有 Qwen-Drive-1.0 的 path:line。
- 不破坏全篇既有审计项：单 `<script>`、零外部资源、无 `Math.random`、中文标点规范、SVG 内徽标 inline stroke-width、面板数字白名单。

## 5. 方案取舍

- **方案一（选定）**：两节全套复刻现有模板（①A/B/C＋②参数清单＋③2D-3D 关系）＋Qwen-Drive 节加 ④ 对比表＋全篇连锁。理由：模板一致性是本笔记价值；Qwen-Drive 的参数清单本身有真内容且与原版 UVTR 清单对得上。
- **方案二（弃）**：Qwen-Drive 节精简（去 C 图与 ②）。理由：结构不一致，且 ③（体素压平加成→query→拉取，恰是 2D-3D 关系本体）没地方放。

## 6. 两节内容规格

### 6.1 第二节 UVTR（`id="sec-uvtr"`）

- 谱系 tag：`谱系定位：推（push）· 3D 体素码放，保高度`（终稿以核对后措辞为准）。
- ① A 图：重绘 UVTR 论文视图变换机制图（Fig 号 clone 论文 PDF 后定）：深度分布 → 体素池化 → 3D 体素。视觉编码沿用全局图例；in-box 徽标表达"推·3D 落点／保高度"语义。
- ① B 面板：见 §2.4 `uvtr3d`。
- ① C 图：机制流程图（深度网络 → 视锥地址/查表 → 散射入体素 → 体素编码），逐步标【固定几何】/【可学习】。
- ② 可学习参数清单：深度网络、体素编码器、（检测头若属桥外则照 VoxFormer 节先例标注"桥之外"）——以官方仓核对为准。
- ③ 2D-3D 关系：推方向；显式深度分布（有无深度监督按代码事实写）；3D 体素保高度（对比 LSS 的 2D 拍扁）；与第五节 VoxFormer 的"稀疏 voxel query"作一语对照（UVTR 稠密推满、VoxFormer 门控稀疏）。

### 6.2 第六节 Qwen-Drive 的 UVTR（`id="sec-qwendrive"`）

- 谱系 tag：`谱系定位：UVTR 推 ＋ BEVFormer 拉，几何先验加成`（终稿措辞可调）。
- ① A 图：重绘论文（arXiv:2609.00111）Fig.3 感知头：双流输入（LLM 图像 token→FPN→拉流；ViT patch→DepthNet→推流）＋加成点＋三输出（检测/占用/地图）。
- ① B 面板：见 §2.4 `qd3d`。
- ① C 图：全流程管线图，两路输入 → 各自 FPN → 深度＋查表散射 → 体素 [C,16,200,200] → 两路消费（1×1 Conv 压平 [40000,256] 加成 bev_embedding；原体素进占用支路）→ BEVFormer 式 encoder → 输出。
- ② 可学习参数清单（只列 Qwen-Drive 侧新增/改动，SCA/TSA 系列不重复列表、交叉引用第三节）：`depth_net`、`view_trans` 3×Conv3d、`uvtr_query_proj`（Conv2d 1×1，256×16→256）、`bev_embedding`（40000,256，加成式初始化）、`query_embedding`（900,512）、`adaptor`/`vit_neck`（SimpleFPN）、positional encoding、cls/reg 分支。已核对出处：`modeling_perception.py:61-85,117-122`、`heads.py:222-223,244-245`。
- ③ 2D-3D 关系（已核事实，`Qwen-Drive-1.0/src/qwen_drive_perception/`）：推拉合流；118 档 softmax 深度无监督（无 gt_depth/深度 loss）；视锥 56×32×118（896×512、0.5 m 步长至 60 m）；ego 系反投影链 inv(lidar2img)→lidar2ego；`voxel_pool_depth` 融合核查表散射（无外积物化）；sweep 求和；query 带几何先验入场；训练双口径 nuScenes 6 相机＋OpenScene/nuPlan 8 相机、固定 896×512（`docs/perception.md:110`）。
- ④ 对比表：五轴 × 两列（原版 UVTR｜Qwen-Drive），每格 path:line；原版侧数字待 clone 核对。五轴：
  1. 特征源：独立相机 backbone vs VLM 双流（ViT patch／LLM token）；
  2. 深度监督：原版（形式待核对）vs 无监督纯下游梯度；
  3. 离散化：原版 nuScenes 口径 vs 896×512、56×32×118、[16,200,200] 与检测网格共用；
  4. 消费方式：原版在体素空间检测 vs 压平加成 query＋占用支路（检测头 BEVFormer 式 900 query）；
  5. 工程细节：视锥 fp32 不随 bf16 cast、单层特征 assert、batch>1 取 mean（原版侧对应细节待核对）。

## 7. 全篇连锁更新清单

1. 章节编号与锚点：新增 sec-uvtr（二）、sec-qwendrive（六）；BEVFormer 二→三、TPVFormer 三→四、VoxFormer 四→五、汇总 五→七。
2. 目录（TOC）加两行并调序。
3. 引言（`index.html:186`）："四条路线"表述改六条目；"见第四节"→"见第五节"；补 Qwen-Drive 节口径说明（双数据集、896×512）。
4. 全文 sweep "四模型／四条路线／见第 X 节"（含 `index.html:2014`、`index.html:2936`）。
5. 汇总表加两行（UVTR、Qwen-Drive）；† 口径脚注更新；汇总节谱系 tag（`index.html:2723`）扩为含 UVTR（推·3D）与 Qwen-Drive（合流）。
6. 谱系 SVG 加两节点：UVTR（推家族·3D 落点）、Qwen-Drive（推拉合流）。
7. 页脚 manifest 表加两行（UVTR 官方仓 URL＋commit；QwenLM/Qwen-Drive-1.0 commit 28091c1）；"四仓"→"六仓"表述。
8. `notes/`：新增 `05_uvtr.md`（原版底稿，clone 后写）、`06_qwendrive_uvtr.md`（Qwen-Drive 底稿，来源已核）；`00_manifest.md` 同步两仓。
9. `bev_projection/papers/` 收 UVTR 论文 PDF（论文 copy 惯例）。

## 8. 约束

- 两个新面板以 IIFE 挂到既有 F3D 公共库；测试钩子 `window.__uvtr3dState`、`window.__qd3dState`；`.f3d-*` 既有样式类复用；每面板数字白名单。
- 面板/图表视觉编码、徽标做法、C 图标题格式与其余四节一致。
- 浏览器实测以程序化断言为主证据（截图面故障时回退，本轮已有先例）。

## 9. 留给计划的决策

- UVTR 官方仓 URL、commit、论文 Fig 号、具体配置数字（视锥/深度档/体素网格/深度监督形式）——clone 后入 `notes/05_uvtr.md`，manifest 记录。
- 两面板的具体几何、抽样密度、缩放范围（参照既有面板每场景范围先例）。
- 执行方式（Subagent-driven／Native）在写计划时定。
- 汇总表两行、谱系两节点、A 图徽标的具体文案。

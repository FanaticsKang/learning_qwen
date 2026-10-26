# BEV 投影机制对比笔记 · 实施计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 交付 `bev_projection/index.html`——LSS / BEVFormer / TPVFormer / VoxFormer 四种 2D→3D 投影机制的中文可视化对比笔记，主线是可学习参数与 2D-3D 关系，全部数字经四家官方代码（统一 nuScenes 口径）校对。

**Architecture:** 先克隆四仓+集齐论文 PDF 并用 manifest 钉住 commit hash；逐模型产出"事实底稿"（可学习参数清单、固定几何清单、代码出处 path:line）；再搭单文件 HTML 骨架（锁定统一视觉编码与版式类名），逐节填充机制 SVG 与参数表；最后汇总对比节 + 全篇浏览器实测与 grep 审计。

**Tech Stack:** 单文件 HTML + 内联 CSS/SVG（无 JS 依赖、无 CDN）；事实底稿为 Markdown；Git Bash 环境下 git 浅克隆。

**Spec:** `bev_projection/design.md`（本计划从该文档出发，执行者须同时读两份）

## Global Constraints

- 完全独立于 Qwen-Drive 分析：`index.html` 中不得出现 "Qwen" / "UVTR" / "本仓库" / "analysis/" 字样，不引用任何系列编号。
- 所有具体数字统一取各仓库 nuScenes 配置口径，标注配置文件路径 + 代码仓 commit hash；代码找不到的标"论文值，代码未见"。
- HTML 完全离线自包含：无任何外部 `src=`/`href=` 资源引用；样式、SVG 全内联。
- 中文排版：①②③ / 1/2/3 / A/B/C 列表逐项换行、左对齐；中文语境用全角逗号"，"，英文/代码语境用半角 ","。
- 示意图静态 SVG：无悬停交互；箭头只从框的底边/侧边出（禁顶边出）；连线少折弯、零交叉；虚线分组框；四节面板风格统一。
- "可学习参数"与"固定几何"的视觉编码用统一图例，贯穿全篇。
- git：只 `git add` 明确路径，绝不 `git add -A`；英文提交信息；本地 commit 不 push；`bev_projection/code/` 与 `bev_projection/papers/` 保持不跟踪。

## Review Focus

1. **数字无出处或口径漂移**——参数表里每个数字必须带 `path:line` 出处且来自 nuScenes 配置 → Task 2–5 底稿自查 + Task 12 引用计数审计。
2. **与 Qwen-Drive 系列耦合**——禁词出现在成品 → Task 6/7/8/9/10/11 各节落笔后即 grep，Task 12 终审（期望零命中）。
3. **SVG 箭头从顶边出 / 连线交叉**——违反框图规范 → Task 7–11 每节浏览器截图检查，Task 12 全篇复查。
4. **图例与视觉编码不一致**——各节"可学习/固定"颜色或线型漂移 → Task 6 锁定类名，Task 12 跨节比对。
5. **中文排版违规**（半角逗号混入中文句、列表未换行左对齐）→ Task 7–11 抽查 + Task 12 全篇过一遍。

---

### Task 1: 素材就位（代码仓 + 论文 PDF + manifest）

**Files:**
- Create: `bev_projection/code/{lift-splat-shoot,BEVFormer,TPVFormer,VoxFormer}/`（浅克隆）
- Create: `bev_projection/papers/`（4 个 PDF）
- Create: `bev_projection/notes/00_manifest.md`

**Interfaces:**
- Produces: manifest 中的四个 commit hash（后续所有底稿与 HTML 页脚引用它）；`papers/` 四个 PDF 的确切文件名。

- [ ] **Step 1: 浅克隆四仓**

```bash
git clone --depth 1 https://github.com/nv-tabs/lift-splat-shoot bev_projection/code/lift-splat-shoot
git clone --depth 1 https://github.com/fundamentalvision/BEVFormer bev_projection/code/BEVFormer
git clone --depth 1 https://github.com/wzzheng/TPVFormer bev_projection/code/TPVFormer
git clone --depth 1 https://github.com/NVlabs/VoxFormer bev_projection/code/VoxFormer
```

- [ ] **Step 2: 集齐论文 PDF 到 `bev_projection/papers/`**

复制（不移动）`reference/papers/tpvformer_2302.07817.pdf`、`reference/papers/voxformer_2302.12251.pdf`；下载 `https://arxiv.org/pdf/2008.05711` → `lss_2008.05711.pdf`、`https://arxiv.org/pdf/2203.17270` → `bevformer_2203.17270.pdf`（命名沿用现有 `论文_编号.pdf` 模式）。

- [ ] **Step 3: 写 `notes/00_manifest.md`**

表格：仓库名 | URL | commit hash（`git -C <repo> rev-parse HEAD`）| 克隆日期 | nuScenes 配置定位线索（LSS 无 config 文件，记 `main.py` 参数默认值 + `src/tools.py`；其余三仓记 `projects/configs/` 下含 nuscene 的配置路径，用 grep 找）。

- [ ] **Step 4: 验证**

`ls bev_projection/code`（4 个目录）、`ls bev_projection/papers`（4 个 PDF）；manifest 含 4 个 hash；`git status` 确认只新增 `bev_projection/`，未动任何已跟踪文件。

- [ ] **Step 5: Commit**

```bash
git add bev_projection/design.md bev_projection/notes/00_manifest.md bev_projection/plan.md
git commit -m "Add bev_projection design doc, plan, and materials manifest"
```

（code/ 与 papers/ 不跟踪，不入库。）

---

### Task 2–5: 四份事实底稿（每模型一份）

四个任务结构相同，只是对象不同：

- Task 2: `notes/01_lss.md`（lift-splat-shoot）
- Task 3: `notes/02_bevformer.md`（BEVFormer，取 v1）
- Task 4: `notes/03_tpvformer.md`（TPVFormer）
- Task 5: `notes/04_voxformer.md`（VoxFormer）

**Files:**
- Create: 上述 `notes/0X_*.md`

**Interfaces:**
- Consumes: manifest 的 commit hash；`code/` 对应仓库；`papers/` 对应 PDF。
- Produces: 底稿中的结构化清单（HTML 各节的内容唯一来源；后续任务只引用不重查）。

每份底稿固定六块（模板统一）：

1. **元信息**——仓库、commit hash、所引 nuScenes 配置文件完整路径。
2. **机制流水线**——从图像特征到 3D 表征的每一步，每步标注【固定几何】或【可学习】。
3. **可学习参数清单表**——参数名 | 所属模块 | 张量形状（nuScenes 口径）| 侧（2D 侧/3D 侧/桥）| 代码出处 `path:line`。
4. **固定几何清单表**——深度分箱/网格范围/投影来源等 | 具体值 | 出处。
5. **2D-3D 关系小结**——推/拉方向、深度显式/隐式、几何与学习的分工、query 有无及形态。
6. **论文 vs 代码差异点 + 待画图要点**——论文原图编号与构图元素、重绘时要保留什么、叠加什么（可学习/固定编码）。

- [ ] **Step 1: 读论文投影相关章节与机制图**（LSS 图 2；BEVFormer 图 2；TPVFormer 图 2；VoxFormer 图 2）
- [ ] **Step 2: 读代码 view-transform 模块**，定位：LSS `src/tools.py`(frustum/disc) + `src/models.py`(CamEncode/depthnet)；BEVFormer `projects/mmdet3d-plugin/bevformer/`(bev_embedding、SpatialCrossAttention、sampling_offsets)；TPVFormer 检索 `tpv` 模块与 ray 采样/三平面 query；VoxFormer 检索两阶段（depth→可见 voxel mask→masked cross-attn→stage2 补全）。以实际文件为准，此线索仅为起点。
- [ ] **Step 3: 按六块模板写底稿**，每个数字落 `path:line`。
- [ ] **Step 4: 自查**——清单中每行数字都有出处；形状与 nuScenes 配置一致（对照 config 变量，不能沿用 SemanticKITTI 数值）；论文值与代码值不一致处已记入差异块。
- [ ] **Step 5: Commit**（每份一提交，如 `Add LSS fact sheet for projection note`）

---

### Task 6: HTML 骨架 + 统一视觉编码

**Files:**
- Create: `bev_projection/index.html`

**Interfaces:**
- Consumes: manifest（页脚来源与 commit hash 表）。
- Produces（后续节填充任务依赖的钩子，类名在此锁定）：节容器 id `sec-lss` / `sec-bevformer` / `sec-tpvformer` / `sec-voxformer` / `sec-summary`；SVG 元素类 `.learnable`（可学习参数载体）`.fixed`（固定几何）`.flow`（数据流箭头）；结构类 `.panel`（机制图面板）`.legend`（统一图例条，页首一条全局）`.param-table`（参数清单表）`.code-check`（代码校对框）`.cite`（行内出处标记）；每节内固定四小节标题 ①机制总览 ②可学习参数清单 ③2D-3D 关系 ④代码校对。

- [ ] **Step 1: 写骨架**——头部（标题/引言/全局图例/目录）、五个空节壳、页脚（四仓 URL + commit hash 表 + 四篇论文 arXiv 编号）。
- [ ] **Step 2: 内联 CSS**——中文字体栈、正文行宽、`.param-table` 表格样式、`.learnable/.fixed` 的填充/描边色（实现者定色，两色对比明显且色盲可辨）与图例对应；列表换行左对齐。
- [ ] **Step 3: 验证**——本地 `python -m http.server` 起服务，浏览器打开：五节与锚点可达、图例两色可见；`grep -niE "qwen|uvtr|本仓库|analysis/" bev_projection/index.html` 零命中；`grep -nE '(src|href)="https?' bev_projection/index.html` 零命中。
- [ ] **Step 4: Commit**（`Add HTML skeleton with unified visual encoding`）

---

### Task 7–10: 四个模型节（逐个填充）

- Task 7: `sec-lss`（依据 `notes/01_lss.md`）
- Task 8: `sec-bevformer`（依据 `notes/02_bevformer.md`）
- Task 9: `sec-tpvformer`（依据 `notes/03_tpvformer.md`）
- Task 10: `sec-voxformer`（依据 `notes/04_voxformer.md`）

**Files:**
- Modify: `bev_projection/index.html` 对应节

**Interfaces:**
- Consumes: Task 6 的类名/结构与对应底稿；不新造类名，超出骨架样式需求时先改骨架 CSS。

每节四小节，内容全部来自底稿，禁止临场引数字：

- [ ] **Step 1: 画机制总览 SVG**——参考该论文原图构图重绘，元素挂 `.learnable`/`.fixed`/`.flow` 类；遵守箭头底/侧出、零交叉、虚线分组框。
- [ ] **Step 2: 填参数清单表**（`.param-table`，每行 `.cite` 出处）与 2D-3D 关系小结。
- [ ] **Step 3: 填代码校对框**（`.code-check`：论文 vs 代码印证/差异 + 配置路径 + commit hash）。
- [ ] **Step 4: 验证**——浏览器截图该节：SVG 无顶边出箭头、无连线交叉、图例编码与全局一致；节内每个数字能在底稿找到（人工对照）；禁词 grep 零命中；中文句抽查全角逗号。
- [ ] **Step 5: Commit**（如 `Add LSS section: mechanism SVG, parameter inventory, code check`）

---

### Task 11: 汇总节

**Files:**
- Modify: `bev_projection/index.html` 的 `sec-summary`

**Interfaces:**
- Consumes: 四份底稿的全部结论 + 前四节。

- [ ] **Step 1: 四家对比大表**——行=结构性维度（query 有无与形状 / 可学习参数位置与形态 / 投影方向（推·拉）/ 深度显式·隐式 / 2D-3D 对应的建立方式 / nuScenes 口径下 3D 表征尺寸），列=四模型；数字一律取自底稿。
- [ ] **Step 2: 谱系总览 SVG**——统一坐标轴（可学习参数所在侧 × 投影方向）上摆四家：无 query（LSS，学习全在 2D 侧）→ 稠密 query（BEVFormer）→ 三平面压缩 query（TPVFormer）→ 深度门控稀疏 query（VoxFormer）；复用 `.learnable/.fixed` 编码。
- [ ] **Step 3: 验证**——表中每格可回溯到某份底稿的具体行（抽查 8 格以上）；SVG 规范检查同前；禁词 grep 零命中。
- [ ] **Step 4: Commit**（`Add summary section: comparison table and lineage diagram`）

---

### Task 12: 全篇实测与收尾

**Files:**
- Modify（如发现问题的回修）: `bev_projection/index.html`、各底稿

- [ ] **Step 1: 浏览器全篇过**——http 服务 + 浏览器逐节截图：目录锚点、五节渲染、SVG 规范（顶边出箭头/交叉=0）、跨节图例一致性、中文排版（列表换行左对齐、全角逗号）。
- [ ] **Step 2: grep 终审**——禁词审计零命中；外部资源引用零命中；`.cite` 出现次数 ≥ 四份底稿参数行总数（抽查计数）。
- [ ] **Step 3: hash 对账**——页脚 commit hash 与 `notes/00_manifest.md` 一致。
- [ ] **Step 4: Commit**（`Finalize BEV projection comparison note`）并交付：向用户报文件路径与浏览器预览方式；不 push，等用户指示。

---

## Self-Review 记录

- **Spec 覆盖**：design.md 目标 1→T6/T12，目标 2→T6–T11，目标 3（四小节模板）→T7–T10，目标 4→T11，目标 5（nuScenes 口径+hash）→T1/T2–5/T12；非目标与反目标→各节审计步骤；约束（排版/SVG/离线/图例）→T6 与各节验证。
- **比例**：计划长度与 spec 相当，未替实现者写正文。

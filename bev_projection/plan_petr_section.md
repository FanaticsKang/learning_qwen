# PETR/PETRv2 一节增补 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 在 `bev_projection/index.html` 增补第七节 PETR/PETRv2（Qwen-Drive 之后、汇总之前），PETR 作为全新谱系（学习直连：不推不拉，3D PE＋全局 attention）独立成节，节末带 PETRv2 补充小节；全篇编号顺延（汇总七→八、深度谱系八→九）＋目录/引言/汇总大表/谱系图/深度谱系（新族④）/页脚锚定表连锁更新。

**Architecture:** 底稿先行（clone megvii-research/PETR 双分支——main＝PETRv2、`petr`＝PETRv1——数字全部 path:line 入 `notes/07_petr.md`），HTML 静态内容与 `petr3d` 三态交互面板分两个任务插入，随后汇总/谱系/页脚一个任务、深度谱系四家族一个任务，最后全篇审计回归。视觉编码、面板库（F3D）、审计与浏览器实测方式全部沿用既有七节先例。

**Tech Stack:** 单文件静态 HTML＋内嵌 SVG＋Canvas 2D 手写 3D（F3D 公共库）；bash/grep 审计；本地 http 服务（8791）＋浏览器程序化断言。

**Spec:** `bev_projection/design_petr_section.md`（本计划据其展开，执行者须同时读设计文档）。

## Global Constraints

- `bev_projection/index.html` 全篇**恰好一个** `<script`（现在与完成后 `grep -c '<script'` 均为 1）；零外部资源（无 CDN/外链 script/img/font）。
- 无 `Math.random`（`grep -c "Math.random" index.html` 恒为 0）；面板抖动/散列用确定性公式（黄金比/fract 先例）。
- 中文文本全角标点：`grep -P '[\x{4e00}-\x{9fa5}],' index.html` 恒为 0。
- **数字白名单**：第七节（含其 B 面板）出现的一切机制数字只能出自 `notes/07_petr.md`。底稿未核实的数字不得写入 HTML（设计文档 §4 反目标）。
- 不得引用 `analysis/` 系列、Qwen-Drive 分析笔记或本仓库其他工作作为来源或链接（新增文本不得贡献 `grep -c "analysis/" index.html` 新命中）。
- 面板以 IIFE 挂到既有 F3D 公共库；测试钩子命名 `window.__petr3dState`。
- git：英文提交信息直接提交 main；**不 push**（用户明确要求才 push）；`Qwen-Drive-1.0/`、`bev_projection/code/`、`bev_projection/papers/`、`.zcode/`、`.agents/` 保持未跟踪（不 `git add` 这些路径）。
- 视觉编码沿用全局图例（`.learnable` 橙实线／`.fixed` 蓝虚线斜纹／`.flow` 箭头）；SVG 内徽标用 inline `style="stroke-width:2.5"`；框内粗体标题行；C 图标题格式 `X · …：…` 与其余节一致。
- 既有七节的机制内容与面板**零改动**（只动编号与交叉引用）；深度谱系既有三族的表述零改动（只加族④）。
- 浏览器实测：页面 `http://127.0.0.1:8791/index.html`（服务已在跑；若掉线 `cd bev_projection && python -m http.server 8791`）。陈旧 tab 必须先 `reload()` 再断言。

## Review Focus

1. **编号与交叉引用漏改**：残留"第七节汇总"（应改"第八节汇总"）、"六节机制走完"、"六家"、"三族"、TOC href 与 section id 不匹配。→ T2 步骤含全文 grep 清单并逐项归零；T5 回归再扫一遍。
2. **数字无底稿出处**（凭印象写 PETR 配置是最可能的事故——尤其 query 数量与深度 bins 数）。→ T2/T3 各含"对照 notes/07 核对新节全部数字"步骤；T3 面板数字白名单在任务里逐个列出。
3. **面板钩子字段漂移**：`__petr3dState().stage` 不是 `'off'|'pe'|'attn'`。→ T3 浏览器断言逐字段检查。
4. **中途态断链**：T2 落静态内容时其文案不得引用"B 面板/图B"（B 相关文案随 T3 落地）；T5 全篇回归确认无悬空引用。
5. **深度谱系扩四族时破坏既有三族**：族①②③的任何文字与图形坐标除了族③整体左移（825→670，为族④腾位）外不得变化。→ T5 步骤含族①②像素坐标不变的 diff 检查。
6. **页脚 hash 与实测不一致**：两行 PETR/PETRv2 的 commit hash 必须与 `git -C code/PETR rev-parse HEAD` 实测一致。→ T4 断言 pin。

---

### Task 1: PETR 官方仓 clone（main＋petr 双分支）＋论文 PDF×2＋底稿 notes/07_petr.md＋manifest

**Files:**
- Create: `bev_projection/code/PETR/`（main 分支浅克隆，不提交）
- Create: `bev_projection/code/PETR-v1/`（`petr` 分支浅克隆，不提交；分支名以 README 实测为准）
- Create: `bev_projection/papers/petr_2203.05625.pdf`、`bev_projection/papers/petrv2_2206.01256.pdf`（不提交）
- Create: `bev_projection/notes/07_petr.md`
- Modify: `bev_projection/notes/00_manifest.md`

**Interfaces:**
- Produces（后续任务的唯一数字来源）：`notes/07_petr.md`，结构镜像 01–06 底稿：§1 元信息（仓库 URL、两个分支的 `git rev-parse HEAD` 全 40 位 hash、克隆日期 2026-10-29、论文与 arXiv id、nuScenes 配置口径与核心派生量：**图像分辨率与 token 分辨率、num_query、3D PE 生成结构（MLP 层数/维度）、视锥深度 bins 数、位置编码插入方式、decoder 层数**）；§2 机制流水线（每步形状流＋【固定几何】/【可学习】＋`path:line`——v1 侧引 `code/PETR-v1/` 路径、v2 侧引 `code/PETR/` 路径，底稿中路径前缀区分书写）；§3 可学习参数清单表；§4 固定几何清单表；§5 2D-3D 关系小结；§6 PETR vs PETRv2 差异表（时序 PE 对齐／feature-guided position encoder／BEV 分割 query）；§7 B 面板数字白名单（相机布置、bins 数、query 数、attention 线示意规则）。
- Produces: manifest 新两行（PETR 与 PETRv2 各一行，仓 URL 相同、分支与 hash 不同），与既有行同格式。

- [ ] **Step 1: 核实官方仓与分支名**

```bash
curl -s https://raw.githubusercontent.com/megvii-research/PETR/main/README.md | head -40
# 核实标准：README 标题同时含 ECCV2022 PETR 与 ICCV2023 PETRv2；
# 找到"v1 代码在哪个分支"的原句（预期 petr 分支；若 README 写其他分支名，以 README 为准并回写设计文档）
```

- [ ] **Step 2: clone 双分支并记录 hash**

```bash
cd bev_projection/code
git clone --depth 1 https://github.com/megvii-research/PETR PETR
git -C PETR rev-parse HEAD                 # main 分支，全 40 位记入底稿
git clone --depth 1 -b petr https://github.com/megvii-research/PETR PETR-v1
git -C PETR-v1 rev-parse HEAD              # v1 分支，全 40 位记入底稿
ls PETR/projects/configs/                  # 找 nuScenes PETR/PETRv2 配置文件名，记入底稿
```

- [ ] **Step 3: 下载论文 PDF 并校验**

```bash
cd ../papers
curl -L -o petr_2203.05625.pdf https://arxiv.org/pdf/2203.05625
curl -L -o petrv2_2206.01256.pdf https://arxiv.org/pdf/2206.01256
head -c 4 petr_2203.05625.pdf   # 期望 %PDF
head -c 4 petrv2_2206.01256.pdf # 期望 %PDF
```

- [ ] **Step 4: 写 `notes/07_petr.md`**

按 Interfaces 结构写全；每个数字必有 `path:line` 或论文页码。重点核对清单：
1. num_query（预期 900，须找到 config 里的实际值）；图像 token 分辨率（backbone stride 与 FPN 输出尺寸）；
2. 3D PE 生成：视锥 bins 数（论文记 3D position encoding 的 D 离散数）、3D 锚点→MLP 结构（层数/隐藏维）、PE 拼接还是相加（论文与代码分别确认）；
3. query 侧 3D 锚点来源（可学习 reference 还是回归输出）；
4. cross attention 形态确认：全局（无 deformable、无投影采样）、多相机 key 拼接方式；
5. **"无"清单确认**：grep 仓内无 depth loss、无 bev_embedding、无 voxel grid、无 deformable attention 模块（v1 侧）；把 grep 命令与结果记入底稿；
6. PETRv2 侧：时序 PE 对齐代码位置（历史 query 坐标→当前帧投影）、feature-guided position encoder 结构、BEV 分割 query 的数量与消费方式；
7. nuScenes 口径数字（输入分辨率、训练配置）。

- [ ] **Step 5: manifest 加两行（PETR/PETRv2）＋papers 表加两行**

- [ ] **Step 6: 审计＋提交**

```bash
grep -c "TODO\|待核" notes/07_petr.md        # 期望 0
git -C code/PETR rev-parse HEAD && git -C code/PETR-v1 rev-parse HEAD   # 与底稿一致
git add notes/07_petr.md notes/00_manifest.md
git commit -m "Add PETR/PETRv2 fact sheet and manifest rows from verified official repo"
```

### Task 2: 第七节 PETR 静态内容＋全篇编号顺延＋引言口径

**Files:**
- Modify: `bev_projection/index.html`（sec-qwendrive 后、sec-summary 前插入 `<section id="sec-petr">`；汇总节 h2 七→八、深度节 h2 八→九；TOC；引言；全文交叉引用）

**Interfaces:**
- Consumes: `notes/07_petr.md`（全部数字与 §7 待画图要点）。
- Produces: `sec-petr` 静态骨架——h2 `七、PETR：位置编码直连——没有投影桥的 2D↔3D`＋tag `谱系定位：全新谱系·学习直连——不推不拉，对应关系由 attention 自己学`；① A SVG（三段横排：图像 token＋视锥 3D 锚点（蓝色虚线斜纹，固定几何）→ 3D PE 生成（MLP，橙）｜带 3D PE 的 key（灰底橙描边小标）｜object query（橙）——一条粗 attention 线横扫全部 key；**全图无蓝色虚线投影线、无采样簇，图注点明这一视觉差异**）＋C SVG（流程线：backbone/FPN → 3D PE 生成（视锥 bins＋MLP）→ key 拼接 → 全局 cross attention → decoder 出框，逐步【固定几何】/【可学习】）＋② 参数清单表（3D PE 的 MLP＝桥上可学习但不做几何对应——全篇唯一，重点标出；object query 内容与 3D 锚点＝3D 侧；backbone/FPN＝2D 侧；**无**项专列一行：深度头、bev_embedding、体素格、deformable 偏移表）＋③ 2D-3D 关系四条（推/拉＝都不算；深度＝不预测数值（视锥 bins 只作 PE 原料）；分工＝边界移位（几何只发坐标、可学习量首现桥上）；query 形态＝稀疏自由 3D 坐标无网格绑定）＋节末 PETRv2 补充小节（h3 ＋四段：时序 PE 对齐（3D→2D 投影在此回归一次）、feature-guided position encoder、BEV 分割 query、谱系延伸一句话点名 StreamPETR/CMT 不展开）。**本任务文案不引用 B 面板**。编号顺延：汇总节 h2 `七、`→`八、`、深度节 h2 `八、`→`九、`（两处 h2 文字＋TOC）；TOC 在 Qwen-Drive 行后插 PETR 行；引言改"六篇工作"→"七篇工作"、"五个机制条目"→"六个机制条目"、列表插 PETR 行（项目号换行：`<strong>PETR</strong>——《Position Embedding Transformation for Multi-View 3D Object Detection》（ECCV 2022）＋ PETRv2（ICCV 2023）——同一官方仓双分支，同节展开`）；全文 sweep 交叉引用。

- [ ] **Step 1: 插入 sec-petr 静态内容（A/②/③/C/PETRv2 小节，暂无 B）**

数字全部取自 notes/07；徽标 in-box 粗体＋inline stroke-width；C 图标题 `C · …：…` 格式。

- [ ] **Step 2: 编号顺延＋TOC＋引言＋交叉引用 sweep**

```bash
grep -n "第七节汇总\|六节机制走完\|六家\|三族\|七、汇总\|八、深度" index.html
# 逐项处理："第七节汇总"→"第八节汇总"（深度节 SVG 底注，line ~4394）；
# "六节机制走完"→"七节机制走完"、"六家"→"七家"、"三族"→"四族"（深度节开头段——
# 注意：族①②③的既有表述不改，仅计数词与新增族④，本步骤先改计数词，族④内容 T5 落地；
# "六家的深度谱系"h2 改"九、深度从哪来：七家的深度谱系"）
grep -n "第[一二三四五六七八九]节" index.html   # 全部指称逐项核对新编号下语义正确
```

- [ ] **Step 3: 审计＋浏览器验证＋提交**

```bash
grep -c '<script' index.html   # 仍为 1
grep -c "Math.random" index.html   # 仍为 0
grep -P '[\x{4e00}-\x{9fa5}],' index.html | wc -l   # 0
```
浏览器：reload 后 `document.getElementById('sec-petr')` 存在、九节 h2 顺序为一LSS/二UVTR/三BEVFormer/四TPVFormer/五VoxFormer/六Qwen-Drive/七PETR/八汇总/九深度、TOC 九条 href 均有对应 id。提交：`git commit -m "Add PETR section (static with PETRv2 subsection) and renumber sections"`。

### Task 3: petr3d B 面板（三态：熄灭→3D PE→全局 attention）

**Files:**
- Modify: `bev_projection/index.html`（sec-petr 的 A 与 C 之间插 `f3d-panel`；唯一 `<script>` 块尾追加 IIFE）

**Interfaces:**
- Consumes: F3D 公共库（`F3D.mount(cfg)` 及 api，见 LSS/bev3d/uvtr3d 面板先例）；notes/07 §7 面板数字白名单（bins 数、相机布置、attention 线示意规则）。
- Produces: 面板 DOM（`petr3d-canvas/petr3d-toggle/petr3d-zoom/petr3d-state`，命名式同 `frustum3d-*`）＋`f3d-title`/`f3d-note`；IIFE 挂载。**单按钮三态循环**（熄灭 → 3D PE → 全局 attention，按钮文案随态切换，如「点亮 3D PE」→「点亮全局 attention」→「熄灭」）：
  - 熄灭态：6 相机线框（示意）＋每相机中心射线上固定深度 bins 点云（全灰，固定几何）；状态文案＝"常规：像素射线上的固定深度 bins——不落格、无 query"；
  - 态 1（3D PE）：射线 bins 变橙（可学习编码语义）——点不落进任何格，只生成"位置签名"附着在像素上；状态文案切换；
  - 态 2（全局 attention）：一个 3D query 球体在空间**自由拖动**（hitTest＋onCustomDrag，不锁任何栅格——与 bev3d"只能拖 BEV 格"形成操作层面对照）；query 到各相机的 soft attention 线按确定性示意相似度（query 与 bins 点的距离高斯衰减，无 Math.random）点亮，线宽/透明度∝权重；query 靠近某射线时该射线 bins 权重升高。
  - 可旋转/缩放；无时间轴动画。测试钩子：`window.__petr3dState()` → `{ stage: 'off'|'pe'|'attn', qx, qy, qz, yaw, pitch, radius }`（qx/qy/qz 为 query 3D 坐标，off/pe 态仍返回当前值）。
  - `f3d-note` 小结一句话（沿用前几节精简口径）：建议文案"小结：PETR 的 2D→3D＝给每个像素的射线 bins 发 3D 位置编码（固定 bins＋可学习 MLP），query 与全像素全局 attention 学出对应——没有投影桥，bins 不落任何格。"

- [ ] **Step 1: 写面板 DOM＋IIFE（含三态状态机、确定性相似度公式、拖拽联动）**

- [ ] **Step 2: 浏览器断言**

reload→钩子存在且 `stage==='off'`；程序化 click toggle 两次依次得 `stage==='pe'`、`stage==='attn'`，state 文本随态切换；`attn` 态做一次亮像素计数（attention 线与 query 球非零）；程序化改 `qx/qy/qz`（expose 直写后 redraw）attention 线随动（改前改后各截一次亮像素分布，重心位移方向与 qx 位移一致）；redraw<10ms。

- [ ] **Step 3: 审计＋提交**

```bash
grep -c '<script' index.html    # 仍为 1
git commit -m "Add petr3d interactive panel: 3D PE then global attention, free-moving query"
```

### Task 4: 汇总大表＋谱系总览图＋页脚锚定表

**Files:**
- Modify: `bev_projection/index.html`（八节①汇总大表、②谱系总览 SVG、页脚 manifest 表与 footer-note）

**Interfaces:**
- Consumes: 全部前序任务产物＋notes/07（数字与 hash）。
- Produces:
  - 汇总大表（`① 六家对比大表`标题改`七家对比大表`）：加 PETR 列（或行——以表实际方向为准，与既有六家同格式）——query 形态＝稀疏（数量取 notes/07）自由 3D 坐标无网格绑定；参数位置＝2D 侧＋3D 侧＋**桥上（PE 的 MLP）**；推拉方向＝都不算（学习直连）；深度显隐＝不预测数值（视锥 bins 只作 PE 原料）；对应建立＝全局 attention Q·K 相似度隐式。† 口径脚注补一句：PETR 列取 `megvii-research/PETR` 仓 nuScenes 配置（v1 侧 `petr` 分支、v2 侧 main 分支，见第七节）。
  - 谱系总览 SVG：新增 PETR 卡片（独立于推/拉两支——右侧独立一支，卡内副标题一句话：`3D PE＋全局 attention`）；因图原为六条目排版，卡片与拉线整体重排（实施时以既有 SVG 坐标系为基，等距平移既有卡片腾位，**既有卡片文字零改动**）；拉线注脚 scope 词同步（"六条目"→"七条目"之类，以图内实际文字为准）。
  - 汇总节 h2 tag 谱系串扩：`无 query → 推·3D 体素码放 → 稠密 query → 三平面压缩 query → 深度门控稀疏 query → 推拉合流` 尾接 `→ 位置编码直连`。
  - 页脚 manifest 表 +2 行（PETR：arXiv:2203.05625、仓 megvii-research/PETR、`petr` 分支 hash；PETRv2：arXiv:2206.01256、同仓、main 分支 hash；克隆日期 2026-10-29）；footer-note "上表六仓"→"上表八仓"、口径句补 PETR（两行同仓双分支说明）。

- [ ] **Step 1: 改汇总大表＋谱系 SVG＋页脚**

- [ ] **Step 2: 断言**

```bash
git -C bev_projection/code/PETR rev-parse HEAD && git -C bev_projection/code/PETR-v1 rev-parse HEAD
# 两 hash 与页脚两行逐一比对一致
grep -n "六仓\|六家对比\|六条目" index.html   # 期望仅剩语义正确的历史表述（人工逐项核），新文本零命中
```
浏览器：reload 后汇总表 PETR 列七行齐（query/参数位置/推拉/深度/对应建立/口径/cite）；谱系 SVG 出现 PETR 卡片文字（`grep` 页面 textContent 断言含"PETR"）。

- [ ] **Step 3: 提交**

```bash
git commit -m "Add PETR to summary table, lineage SVG, and footer manifest"
```

### Task 5: 深度谱系族④＋全篇审计回归

**Files:**
- Modify: `bev_projection/index.html`（九节深度谱系：开头段、`depth-map` SVG 扩四族、汇总表加 PETR 行、收束段）

**Interfaces:**
- Consumes: notes/07；T2 已把计数词改为"七家/四族"。
- Produces:
  - 开头段：既有族①②③句子**零改动**，句尾追加族④定义句：`<strong>族④ 无假设·隐式全局分配</strong>（PETR——不预测数值深度、不设固定 z 锚/pillar，视锥 bins 只作 3D PE 原料，深度不确定性弥散在全局 attention 权重里）`。
  - `depth-map` SVG 扩四族：viewBox `0 0 1000 525`→`0 0 1310 525`；族③分组整体 x 左移 155（框 x 825→670，组内全部元素同步 -155）；族④新框 x=995 width=310 height=407，内含：标题`族④ 隐式全局分配`、副题`在线 · 无数值深度 · attention 权重隐式`、示意图形（一个 query 球到多条像素射线 bins 的 attention 线，线宽示意权重——沿用族②的图形词汇但无固定锚点投影线）、PETR 卡片框（`PETR —— 无深度头`＋一行 cite 取 notes/07）；标题文字"（三族）"→"（四族）"、"六家"→"七家"；aria-label 同步；图底谱系一句话扩"／无假设·attention 隐式分配（族④）"。
  - 深度汇总表（`六家深度来源小结`标题改`七家深度来源小结`）：加 PETR 行——深度形式＝无（无数值深度输出）；来源＝视锥 bins 只作 3D PE 原料；监督＝无监督；方向＝无特征流过桥（attention 隐式）；离在线＝在线（各格文案以表实际列头为准，数字 cite notes/07）。
  - 收束段"读表抓两点"→"读表抓三点"，追加：`其三，PETR 证明"深度可以完全不预测"这条路线成立——不确定性不在射线（族①）不在像面（族②）不在掩码（族③），而在全局 attention 权重里`。

- [ ] **Step 1: 扩四族（族①②零改动、族③整体左移、族④新建）**

- [ ] **Step 2: 族①②零改动 diff 检查**

```bash
git diff index.html | grep -E "^[-+].*(族①|族②)" | grep "^-"   # 期望空输出（族①②相关行无删除）
# 族③：允许且仅允许坐标值变化的行，文字内容变化为 0：
git diff -U0 index.html | grep "^[-+]" | grep "族③"   # 逐行人工核，仅 x 坐标 -155
```

- [ ] **Step 3: 全篇审计回归＋浏览器验证＋提交**

```bash
grep -c '<script' index.html && grep -c "Math.random" index.html && grep -P -c '[\x{4e00}-\x{9fa5}],' index.html   # 1, 0, 0
grep -n "第七节汇总\|六节机制走完\|六家\|三族\|六仓\|六篇工作\|五个机制条目" index.html   # 期望零命中（或仅语义正确的历史表述，逐项人工核）
grep -n "见第[一二三四五六七八九]节" index.html   # 全部指称语义正确
```
浏览器：reload 后九节 h2 顺序与 TOC 九条一致；`depth-map` viewBox 为 1310 宽；PETR 行在深度表内；全部 8 个面板钩子（含 `__petr3dState`）可调用。HTML 标签闭合校验（python html.parser，与上轮同脚本）通过。提交：`git commit -m "Add depth taxonomy family 4 (PETR implicit) and full-document regression"`。

---

## 自审记录

- Spec 覆盖：设计文档 §2.1（第七节叙事）→T2；§2.2（PETRv2 补充）→T2；§2.3（petr3d 三态面板）→T3；§2.4 连锁清单七项→T2（编号/引言）、T4（汇总/谱系/页脚）、T5（深度谱系）、T1（素材下载）；§4 反目标→Global Constraints＋T1 Step4 核对清单＋各任务数字白名单步骤；§8 验收→T2/T3/T4/T5 的审计与浏览器断言。无缺口。
- 占位符扫描：无 TBD/TODO；面板文案给出建议句（实施者可按底稿微调数字，结构不变）。
- 类型一致性：`__petr3dState().stage` 枚举 `'off'|'pe'|'attn'` 在 T3 Interfaces 与 T3 Step2 断言一致；notes/07 的 §7 白名单在 T1/T3 引用一致。

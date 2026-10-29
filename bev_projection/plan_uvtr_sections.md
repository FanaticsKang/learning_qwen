# UVTR 与 Qwen-Drive 两节增补 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 在 `bev_projection/index.html` 增补两节——第二节原版 UVTR（LSS 之后）、第六节 Qwen-Drive 的 UVTR 实现（最后），全套复刻现有节模板并附 ④ 五轴对比，全篇编号/目录/汇总/谱系/页脚连锁更新。

**Architecture:** 底稿先行（clone UVTR 官方仓＋Qwen-Drive-1.0 已在本地，数字全部 path:line 入 notes/05、06），HTML 两节静态内容与两个 F3D 交互面板分四个任务插入，最后一个任务做汇总/谱系/页脚与全篇审计回归。视觉编码、面板库（F3D）、审计与浏览器实测方式全部沿用既有五节先例。

**Tech Stack:** 单文件静态 HTML＋内嵌 SVG＋Canvas 2D 手写 3D（F3D 公共库）；bash/grep 审计；本地 http 服务（8791）＋浏览器程序化断言。

**Spec:** `bev_projection/design_uvtr_sections.md`（本计划据其展开，执行者须同时读设计文档）。

## Global Constraints

- `bev_projection/index.html` 全篇**恰好一个** `<script`（现在与完成后 `grep -c '<script'` 均为 1）；零外部资源（无 CDN/外链 script/img/font）。
- 无 `Math.random`（`grep -c "Math.random" index.html` 恒为 0）；面板抖动/散列用确定性公式（黄金比/fract 先例）。
- 中文文本全角标点：`grep -P '[\x{4e00}-\x{9fa5}],' index.html` 恒为 0。
- **数字白名单**：第二节（含其 B 面板）出现的一切机制数字只能出自 `notes/05_uvtr.md`；第六节（含其 B 面板）只能出自 `notes/06_qwendrive_uvtr.md`。底稿未核实的数字不得写入 HTML。
- 不得引用 `analysis/` 系列、Qwen-Drive 分析笔记或本仓库其他工作作为来源或链接（`grep -c "analysis/\|本仓库" index.html` 中新增文本不得贡献命中——注意既有文本已有"本仓库"字样属旧内容，只约束新增）。
- 面板以 IIFE 挂到既有 F3D 公共库；测试钩子命名 `window.__uvtr3dState`、`window.__qd3dState`。
- git：英文提交信息直接提交 main；**不 push**（用户明确要求才 push）；`Qwen-Drive-1.0/`、`bev_projection/code/`、`bev_projection/papers/`、`.zcode/`、`.agents/` 保持未跟踪（不 `git add` 这些路径）。
- 视觉编码沿用全局图例（`.learnable` 橙实线／`.fixed` 蓝虚线斜纹／`.flow` 箭头）；SVG 内徽标用 inline `style="stroke-width:2.5"`；框内粗体标题行；C 图标题格式 `X · …：…` 与其余节一致。
- 浏览器实测：页面 `http://127.0.0.1:8791/index.html`（服务已在跑；若掉线 `cd bev_projection && python -m http.server 8791`）。陈旧 tab 必须先 `reload()` 再断言；程序化断言为主证据，截图面故障时以像素扫描回退（本轮已有先例）。

## Review Focus

1. **编号与交叉引用漏改**：任何残留"四、VoxFormer"、"见第四节"（应指 VoxFormer 时现为第五节）、TOC href 与 section id 不匹配。→ T3 步骤含全文 grep 清单并逐项归零；T7 回归再扫一遍。
2. **数字无底稿出处**（凭印象写 UVTR 配置是最可能的事故）。→ T3/T4/T5/T6 各含"对照 notes/05 或 06 核对新节全部数字"步骤；T4/T6 面板数字白名单在任务里逐个列出。
3. **面板钩子字段漂移**：`__uvtr3dState().mode` 不是 `'stack'|'flatten'`、`__qd3dState()` 缺 `qx/qz`。→ T4/T6 浏览器断言逐字段检查。
4. **中途态断链**：T3/T5 落静态内容时其文案不得引用"图B／B 面板"（B 相关文案随 T4/T6 落地）；T7 全篇回归确认无悬空引用。
5. **汇总与页脚口径错**：新增两行的数据集口径（UVTR=nuScenes；Qwen-Drive=nuScenes 6 相机＋OpenScene/nuPlan 8 相机、896×512）与 † 脚注、"四仓"→"六仓"、manifest hash 与 `git rev-parse` 实测一致。→ T7 断言逐项 pin。

---

### Task 1: UVTR 官方仓 clone＋论文 PDF＋底稿 notes/05_uvtr.md＋manifest

**Files:**
- Create: `bev_projection/code/UVTR/`（clone，不提交）
- Create: `bev_projection/papers/uvtr_<arxivid>.pdf`（不提交）
- Create: `bev_projection/notes/05_uvtr.md`
- Modify: `bev_projection/notes/00_manifest.md`

**Interfaces:**
- Produces（后续任务的唯一数字来源）：`notes/05_uvtr.md`，结构镜像 01–04 底稿：§1 元信息（仓库 URL、`git rev-parse HEAD` 全 40 位 hash、克隆日期 2026-10-28、论文与 arXiv id、nuScenes 配置口径与核心派生量：**图像分辨率、视锥尺寸、深度档数 D、体素网格与体素尺寸、深度监督形式、采样/体素编码配置**）；§2 机制流水线（每步形状流＋【固定几何】/【可学习】＋`path:line`）；§3 可学习参数清单表；§4 固定几何清单表；§5 2D-3D 关系小结；§6 论文 vs 代码差异表＋§6.2 待画图要点（A 图重绘规范、C 图步骤清单、**B 面板数字白名单**）。
- Produces: manifest 新行（仓库、hash、日期、口径），与 00 既有四行同格式。

- [ ] **Step 1: 定位并核实官方仓**

GitHub 搜索 UVTR（NeurIPS 2022，一作 Yanwei Li）；核实标准：README 自述 NeurIPS 2022、mmdet3d/mmdetection3d 系 nuScenes 多相机 3D 检测、论文 arXiv 链接作者一致。**不得凭记忆写 URL**。

- [ ] **Step 2: clone 并记录 hash**

```bash
cd bev_projection/code && git clone --depth 1 <核实后的URL> UVTR
git -C UVTR rev-parse HEAD   # 全 40 位记入底稿
```

- [ ] **Step 3: 下载论文 PDF 至 `papers/`（arXiv id 取自仓 README/paper 链接）**

- [ ] **Step 4: 写 `notes/05_uvtr.md`**

按 Interfaces 结构写全；每个数字必有 `path:line` 或论文页码；重点核对：深度监督的形式与开关（有无 LiDAR/foreground depth loss）、视锥与体素离散化、体素编码器、深度网络结构。§6.2 给出 B 面板数字白名单清单。

- [ ] **Step 5: 更新 `notes/00_manifest.md` 加一行**

- [ ] **Step 6: 审计＋提交**

```bash
grep -c "path:line 占位\|TODO" notes/05_uvtr.md   # 期望 0
git -C code/UVTR rev-parse HEAD                    # 与底稿一致
git add notes/05_uvtr.md notes/00_manifest.md
git commit -m "Add UVTR fact sheet and manifest row from verified official repo"
```

### Task 2: Qwen-Drive 底稿 notes/06_qwendrive_uvtr.md＋manifest

**Files:**
- Create: `bev_projection/notes/06_qwendrive_uvtr.md`
- Modify: `bev_projection/notes/00_manifest.md`

**Interfaces:**
- Consumes: T1 的 `notes/05_uvtr.md`（对比表原版侧事实）。
- Produces: `notes/06_qwendrive_uvtr.md`，同结构底稿，来源＝本地 `Qwen-Drive-1.0/`（先 `git -C Qwen-Drive-1.0 rev-parse HEAD` 核实并记录实际 hash，预期 28091c1 开头）。核心量（设计文档已核）：896×512、视锥 56×32×118（0.5 m 步长至 60 m）、[16,200,200] 体素、118 档 softmax 无监督深度、`uvtr_query_proj` 256×16→256 压平 [40000,256] 加成 `bev_embedding`（`heads.py:244-245`）、占用支路、SCA/TSA 同 BEVFormer 式、6 层 encoder、4 z 锚、900 query、双数据集口径（`docs/perception.md:110`）。§7 对比五轴×两侧素材（每格双侧 `path:line`；原版侧引 notes/05 的 UVTR 路径）。§8 B 面板数字白名单。

- [ ] **Step 1: 核实 Qwen-Drive-1.0 hash 并补漏核对**

对 `src/qwen_drive_perception/{modeling_perception.py,view_transform.py,heads.py,bev_encoder.py,attention.py}` 重走一遍引用，补齐底稿所需全部 `path:line`（设计文档 §6.2 已列大半）。

- [ ] **Step 2: 写底稿（含 §7 对比素材、§8 面板白名单）**

- [ ] **Step 3: manifest 加行＋审计＋提交**

```bash
grep -c "118" notes/06_qwendrive_uvtr.md    # ≥1（核心量在场）
grep -n "heads.py:244" notes/06_qwendrive_uvtr.md   # 命中
git add notes/06_qwendrive_uvtr.md notes/00_manifest.md
git commit -m "Add Qwen-Drive UVTR fact sheet and manifest row"
```

### Task 3: 第二节 UVTR 静态内容＋全篇编号顺延

**Files:**
- Modify: `bev_projection/index.html`（LSS 节后插入新 `<section id="sec-uvtr">`；改 h2 编号、TOC、引言、全文交叉引用）

**Interfaces:**
- Consumes: `notes/05_uvtr.md`（全部数字与 §6.2 待画图要点）。
- Produces: `sec-uvtr` 静态骨架——h2 `二、UVTR：…`＋谱系 tag（推·3D 体素码放保高度语义）；① A SVG（重绘论文机制图）＋C SVG（机制流程图，逐步固定/可学习着色）＋② 参数清单表＋③ 2D-3D 关系列表；**本任务文案不引用 B 面板**。编号顺延：BEVFormer 二→三、TPVFormer 三→四、VoxFormer 四→五、汇总五→六（T5 再改七）；TOC 加 UVTR 行；引言"四条路线"改六条目、"见第四节"→"见第五节"、补 Qwen-Drive 节口径句（第六节预告）；全文 sweep"四模型/四条路线/见第 X 节"。

- [ ] **Step 1: 插入 sec-uvtr 静态内容（A/②/③/C，暂无 B）**

数字全部取自 notes/05；徽标 in-box 粗体＋inline stroke-width。

- [ ] **Step 2: 编号顺延＋TOC＋引言＋交叉引用 sweep**

```bash
grep -n "第[一二三四五六七]节\|四、\|五、\|四模型\|四条路线" index.html   # 逐项核对新编号下语义正确
```

- [ ] **Step 3: 审计＋浏览器验证＋提交**

```bash
grep -c '<script' index.html && grep -c "Math.random" index.html   # 1 和 0
```
浏览器：reload 后 `document.getElementById('sec-uvtr')` 存在、六节 h2 顺序为 一LSS/二UVTR/三BEVFormer/四TPV/五Vox/六汇总、TOC 各 href 均有对应 id。提交：`git commit -m "Add UVTR section (static) and renumber sections"`。

### Task 4: uvtr3d B 面板

**Files:**
- Modify: `bev_projection/index.html`（sec-uvtr 的 A 与 C 之间插 `f3d-panel`；script 块尾追加 IIFE）

**Interfaces:**
- Consumes: F3D 公共库（`F3D.mount(cfg)` 及 api，见 LSS/bev3d 面板先例）；notes/05 §6.2 面板数字白名单。
- Produces: 面板 DOM（`uvtr3d-canvas/uvtr3d-toggle/uvtr3d-zoom/uvtr3d-state`）＋`f3d-note` 小结；IIFE 挂载；按钮两态：默认「UVTR 码放」/按下「LSS 拍扁」（按钮文案切换，非点亮式）；场景＝单根图像中心视锥射线的 D 档候选点（D 与档距取白名单）＋3D 体素网格（UVTR z 层数）＋地面 2D BEV 网格；码放态点落入体素格（保留高度），拍扁态同批点全落地面格、格子亮度表示 pillar 求和；两态画布标注与状态文案切换；可旋转/缩放。测试钩子：`window.__uvtr3dState()` → `{ mode: 'stack'|'flatten', yaw, pitch, radius }`。

- [ ] **Step 1: 写面板 DOM＋IIFE（含状态文案与图例行）**

- [ ] **Step 2: 浏览器断言**

reload→钩子存在且 `mode==='stack'`；程序化 click toggle 后 `mode==='flatten'` 且 state 文本切换；两态各做一次亮像素计数（地面格亮度/体素点）非零且分布合理；redraw<10ms。

- [ ] **Step 3: 审计＋提交**

```bash
grep -c '<script' index.html    # 仍为 1
git commit -m "Add uvtr3d interactive panel: stack vs flatten"
```

### Task 5: 第六节 Qwen-Drive 静态内容（A/C、②③④）

**Files:**
- Modify: `bev_projection/index.html`（VoxFormer 节后、汇总前插入 `<section id="sec-qwendrive">`；汇总 六→七）

**Interfaces:**
- Consumes: `notes/06_qwendrive_uvtr.md`（含 §7 对比素材）；③ 可交叉引用第三节（BEVFormer）既有内容。
- Produces: h2 `六、Qwen-Drive 的 UVTR：推拉合流…`＋tag；① A SVG（重绘论文 Fig.3 感知头双流）＋C SVG（两路输入→FPN→深度+散射→体素→两路消费→encoder→三输出 管线）；② 参数清单表（只列 Qwen-Drive 侧新增/改动项，SCA/TSA 行交叉引用第三节）；③ 2D-3D 关系；④ `原版 UVTR vs Qwen-Drive` 五轴对比表（每格双侧 path:line，原版侧引第二节/notes/05）。**文案不引用 B 面板**。TOC 加行；汇总节编号改七。

- [ ] **Step 1: 插入 sec-qwendrive 静态内容＋④ 对比表**

- [ ] **Step 2: 编号/TOC/审计＋浏览器验证＋提交**

浏览器：`sec-qwendrive` 存在、h2 顺序 一/二/三/四/五/六/七、④ 表五轴行齐。`grep -c '<script'` 仍 1。提交：`git commit -m "Add Qwen-Drive UVTR section with five-axis comparison"`。

### Task 6: qd3d B 面板

**Files:**
- Modify: `bev_projection/index.html`（sec-qwendrive 的 A 与 C 之间；script 块尾追加 IIFE）

**Interfaces:**
- Consumes: F3D 公共库；notes/06 §8 面板白名单。
- Produces: 面板 DOM（`qd3d-*` 同前命名式）＋IIFE；熄灭态＝体素网格灰（[16,200,200] 抽样示意）＋相机环视线框（示意，标注双口径 6/8 相机）；点亮态＝体素亮橙→地面 BEV query 格显橙（压平加成语义，状态文案描述）→到命中相机的 SCA 拉取线（蓝虚线）；拖拽 query 格联动拉取线（bev3d 模式）；两态切换无时间轴动画；可旋转/缩放。测试钩子：`window.__qd3dState()` → `{ depthOn, qx, qz, yaw, pitch, radius }`。

- [ ] **Step 1: 写面板 DOM＋IIFE**

- [ ] **Step 2: 浏览器断言**

钩子存在且 `depthOn===false`；click toggle 后 `depthOn===true`、亮橙（query 格）与拉取线像素并存；程序化改 `qx/qz`（拖拽路径合成或 expose 直写后 redraw）拉取线随动；redraw<10ms。

- [ ] **Step 3: 审计＋提交**

```bash
git commit -m "Add qd3d interactive panel: voxel flatten, query fusion, SCA pull"
```

### Task 7: 汇总表＋谱系图＋页脚 manifest＋全篇审计回归

**Files:**
- Modify: `bev_projection/index.html`（汇总表、谱系 SVG、汇总 h2 tag、页脚 manifest 表与 footer-note）

**Interfaces:**
- Consumes: 全部前序任务产物。
- Produces: 汇总表 +2 行（UVTR：推·3D 码放保高度；Qwen-Drive：推拉合流＋几何先验加成；口径列 UVTR=nuScenes、Qwen-Drive=nuScenes 6＋OpenScene/nuPlan 8 相机 896×512）＋† 脚注更新；谱系 SVG 加 UVTR（推家族·3D 落点）与 Qwen-Drive（合流）两节点；汇总 h2 tag 谱系串扩；页脚 manifest +2 行（hash 与 `git rev-parse` 实测一致）、"四仓"→"六仓"。

- [ ] **Step 1: 改汇总表/谱系/页脚**

- [ ] **Step 2: 全篇审计**

```bash
grep -c '<script' index.html && grep -c "Math.random" index.html && grep -P -c '[\x{4e00}-\x{9fa5}],' index.html   # 1, 0, 0
grep -n "见第四节\|四条路线\|四模型" index.html    # 仅允许语义正确的历史表述，逐项人工核
```

- [ ] **Step 3: 浏览器全量回归**

reload→六个面板钩子（`__frustum3dState/__bev3dState/__tpv3dState/__vox3dState/__uvtr3dState/__qd3dState`）全部存在且字段正确；TOC 锚点全可跳转；新两节截图或像素抽查（截图面故障则像素扫描回退）。

- [ ] **Step 4: 提交**

```bash
git commit -m "Integrate UVTR and Qwen-Drive into summary, lineage, and footer manifest"
```

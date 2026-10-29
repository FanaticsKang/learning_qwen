# 三模型交互式 3D 面板 · 实施计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 为 BEVFormer、TPVFormer、VoxFormer 三节各建一个交互式 3D 面板（新 B，原 B 顺延 C）+ 流程图核心步骤标识，复用并共享 LSS 面板的代码骨架，全文件保持恰好一个 `<script>`。

**Architecture:** 先把现有 LSS 面板脚本**等价重构**为公共库 `F3D`（投影/画家排序/拖拽/缩放/两态/联动命中/测试钩子）+ LSS 面板 IIFE，回归验证通过后，逐家挂载新面板；每节机制图按 LSS 先例拆分插入。三家面板与标识的语义规格全部来自设计文档。

**Tech Stack:** 原生 JS + Canvas 2D，零外部依赖，浏览器程序化验证。

**Spec:** `bev_projection/design_3d_panels.md`（执行者须同读；其裁定 A/B 已并入正文）

## Global Constraints

- 全文件 `<script>` 计数恒为 **1**（`grep -c '<script' bev_projection/index.html` → 1，script id 保持 `frustum3d-script`）。
- 外部资源零命中：`grep -nE '(src|href)="https?' bev_projection/index.html` → 0；禁词 `grep -niE 'qwen|uvtr|本仓库|analysis/'` → 0；CJK 邻接半角逗号 → 0。
- 面板小结与画布标注的数字只允许各自白名单（BEV：200×200=40000、z 锚 {−4.5,−2.167,0.167,2.5}、256 采样位；TPV：三平面形状、46400、640000、7.25%、等权求和零参数；Vox：128×128×16、262144、0.4 m）；示意数据必须带"示意"。
- 图例语义沿用：橙=可学习、蓝虚线=固定几何、灰白=数据、箭头=流。
- 每面板单帧图元（点+线）≤1 万，单帧重绘 <10ms。
- git：只 `git add bev_projection/index.html`（及本计划/设计文档首次提交）；英文提交信息；不 push。
- 浏览器验证走 http 服务 + evaluate 程序化断言（像素计数/状态钩子/增量核对），截图面故障时以程序化断言为准。

## Review Focus

1. **公共库重构破坏 LSS 行为**——T1 是硬门：拖拽（yaw/pitch 精确增量）、缩放（radius 随滑杆、像素幅值变化）、两态（暖色 0↔>0）、α 直方图拖拽（alphaC 变化）四项全过才允许进入 T2。
2. **script 数量意外增加**——每任务 Step 验证含 `grep -c '<script'` = 1。
3. **分区改名遗漏**——每节原 B→C 的标题、注释、aria-label 与全文引用逐一改；验证步 `grep -n 'B ·\|C ·'` 分区字母归属核对。
4. **Vox 门控呈现与底稿矛盾**——点亮态断言：亮橙（可见带）之外**其余槽位呈淡橙**（mask token 补全），非灰。
5. **性能超标**——每面板计时断言（dispatch 拖拽事件后 `performance.now()` 差 <10ms，图元计数 ≤1 万）。

---

### Task 1: 公共库 F3D 等价重构 + LSS 回归

**Files:**
- Modify: `bev_projection/index.html`（仅 script 块与样式类名，不动 DOM 结构与文案）

**Interfaces:**
- Produces（T2–4 依赖，签名锁定）：script 块内公共库 `var F3D = { mount: function (cfg) { … } }`；`cfg = { canvasId, toggleId, zoomId, stateId?, center:[x,y,z], yawInit, pitchInit, radiusInit, radiusMin, radiusMax, drawScene(api), hitTest(x, y, api)→string|null, onCustomDrag(mode, x, y, api), stateText(on)→string }`；`api = { ctx, W, H, project(p)→{x,y,z}|null, cam:{yaw,pitch,radius}, depthOn() }`；返回实例方法：`inst.redraw()`、`inst.expose(fn)`（fn 返回的对象并入 `inst.state()` 供测试）。
- 同时把 `#frustum3d` 前缀的 scoped 样式泛化为 `.f3d-*` 类（容器加类名 `f3d-panel`），LSS 面板外观不变；T2–4 的面板直接复用这些类。

- [ ] **Step 1: 重构 script**——抽出投影/相机基/画家排序/指针管线（含自定义命中模式）/缩放/按钮两态/state 文案切换/DPR resize 为 `F3D.mount`；LSS 场景（点云/相平面/直方图/着色）与 α 拖拽改为 `drawScene` + `hitTest('alpha')` + `onCustomDrag` 的 cfg 形式，`window.__frustum3dState` 由 `inst.expose` 重建（字段不变并新增暴露 radius 已有）。
- [ ] **Step 2: 浏览器回归（RED 在先：改前先跑一遍记录基线值）**——拖拽 yaw/pitch 精确增量、缩放 46/150 像素幅值、两态暖色 0↔>0、α 拖拽 alphaC 16→32、errs=0，全部与基线一致。
- [ ] **Step 3: 审计**——script=1、外链 0、禁词 0、逗号 0；样式泛化后 LSS 面板截图/像素与改前一致。
- [ ] **Step 4: Commit**——`Refactor frustum script into shared F3D library, LSS behavior preserved`

### Task 2: BEVFormer 面板（拉）+ A 区标识

**Files:**
- Modify: `bev_projection/index.html`（sec-bevformer ① 机制总览：拆 SVG、插面板；A 区 SCA 框标识）

**Interfaces:**
- Consumes: `F3D.mount`（T1）与 `.f3d-*` 类。
- Produces: 容器 `id="bev3d"`、canvas/toggle/zoom/state 依 `bev3d-*` 命名、测试钩子 `window.__bev3dState()`（含 `{yaw,pitch,radius,depthOn,qx,qz}`，qx/qz=聚焦 query 格的 BEV 平面坐标）。

- [ ] **Step 1: 拆分与插入**——机制图 SVG 拆为 A（encoder 总览，保留）与 C（SCA 柱采样，原 B 改名，含注释/aria）；中间插入 `bev3d` 面板（标题"B · 3D 拉取：BEV query 的 pillar 与投影（可拖拽/缩放）"、控制行=按钮+缩放滑杆、canvas、小结）。
- [ ] **Step 2: 场景实现**——BEV 平面网格（抽样）+2 台命中聚焦相机+其余灰影；聚焦 query 格的 pillar 4 z 锚 {−4.5,−2.167,0.167,2.5}（蓝虚线固定）+锚点到相机的 lidar2img 示意投影线；熄灭=仅固定几何；点亮=query 格与采样点簇亮橙（围绕参考点散布=可学习偏移，示意；每命中相机 256 采样位入小结）；联动=在 BEV 平面拖动 query 格（`hitTest('query')`），投影线与命中相机实时变化（投影入视野的相机点亮）。
- [ ] **Step 3: A 区标识**——SCA 步骤框：框线加粗＋框内加粗标题行"拉＝3D→2D"（行内布局适配，不压线、不浮框）。
- [ ] **Step 4: 验证**——两态像素断言（熄灭暖色 0/点亮 >0）、拖 query 后 `__bev3dState().qx/qz` 变化且投影线像素变化、拖拽旋转/缩放增量、重绘 <10ms、errs=0、四项 grep、分区字母归属核对、小结数字对照白名单。
- [ ] **Step 5: Commit**——`Add BEVFormer interactive pull panel (new B) and SCA badge in A`

### Task 3: TPVFormer 面板（压缩）+ C 区标识

**Files:**
- Modify: `bev_projection/index.html`（sec-tpvformer 同构改造）

**Interfaces:**
- Consumes: 同 T2。
- Produces: `tpv3d` 系列 id 与 `window.__tpv3dState()`（含 `{…, vx,vy,vz}` 选中体素坐标）。

- [ ] **Step 1: 拆分与插入**——A（三平面表示+射线，保留）/C（端到端流水线，原 B 改名）之间插 `tpv3d` 面板（标题"B · 3D 压缩：三平面 query 代替稠密体素（可拖拽/缩放）"）。
- [ ] **Step 2: 场景实现**——淡灰稠密体素体积（200×200×16 抽样线框/点，640000 的代价感）；熄灭=稠密体素；点亮=三正交平面亮橙（hw 200×200、zh 16×200、wz 200×16，46400=7.25%）、体素变淡、选中体素到三平面的双线性采样连线+“等权求和（零参数）"标注；联动=包围盒内拖动选中体素（`hitTest('voxel')`），三平面采样点跟随。
- [ ] **Step 3: C 区标识**——流水线三平面/ICA 相关框：框线加粗＋标题行"压缩：640000→46400"。
- [ ] **Step 4: 验证**——同 T2 模式（两态/联动 `vx,vy,vz`/拖拽缩放/性能/grep/分区/白名单）。
- [ ] **Step 5: Commit**——`Add TPVFormer compression panel (new B) and badge in C`

### Task 4: VoxFormer 面板（门控）+ A 区标识

**Files:**
- Modify: `bev_projection/index.html`（sec-voxformer 同构改造）

**Interfaces:**
- Consumes: 同 T2。
- Produces: `vox3d` 系列 id 与 `window.__vox3dState()`（含 `{…, gate}` 门控滑杆值）。

- [ ] **Step 1: 拆分与插入**——A（Stage-1 离线门控，保留）/C（Stage-2 在线，原 B 改名）之间插 `vox3d` 面板（标题"B · 3D 门控：深度决定谁有 query（可拖拽/缩放）"），控制行多一个"门控"滑杆。
- [ ] **Step 2: 场景实现**——128×128×16@0.4 m query 体网格全槽淡灰（262144）；熄灭=全槽灰；点亮=近相机可见带亮橙（示意 N_p）+可见体素少量拉取线+**其余全部槽位淡橙**（mask token 广播补全，忠实底稿 step 11）；联动=门控滑杆加宽亮带（淡橙集恒为补集）。
- [ ] **Step 3: A 区标识**——Stage-1 深度提议框：框线加粗＋标题行"门控：谁有 query"。
- [ ] **Step 4: 验证**——两态像素断言（熄灭全灰；点亮=亮橙带 + 非带区**淡橙非灰**）、gate 滑杆联动、拖拽缩放、性能、grep、分区、白名单。
- [ ] **Step 5: Commit**——`Add VoxFormer gating panel (new B) and badge in A`

### Task 5: 全篇回归与收尾

**Files:**
- Modify（如发现缺陷）: `bev_projection/index.html`

- [ ] **Step 1: 四面板共存回归**——同页四 canvas 全部初始化无错（四个 `__*State()` 钩子可读）、各自两态/联动抽验、逐节滚动截图抽检（截图面故障则以像素断言代替）。
- [ ] **Step 2: 全篇审计**——script=1、外链 0、禁词 0、逗号 0、目录锚点零失效、分区字母三节归属正确、图例与实际用色一致。
- [ ] **Step 3: 修复与提交**——最小修复复跑；`git commit -m "Finalize interactive 3D panels across four sections"`（无改动免提交）。
- [ ] **Step 4: 交付报告**——改动位置、验证摘要、与既有审计关系、提交栈。

---

## Self-Review 记录

- **Spec 覆盖**：设计目标 1→各任务 Step 1；2→T1；3/4/5→T2/3/4 Step 2；6→各任务 Step 3（BEV/Vox 在 A、TPV 在 C，裁定 B）；7→T1 公共件+各任务标配；非目标/反目标→全局约束与各验证步；建造顺序与验收→任务序与 T5。
- **类型一致**：`F3D.mount` cfg/api 字段在 T1 Produces 一次性锁定，T2–4 只引用；测试钩子命名模式统一 `__<panel>State`。
- **比例**：无代码正文，接口与断言以签名/清单形式给出。

# LSS 节 3D 视锥交互图 · 实施计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 在 `bev_projection/index.html` 的 LSS 节 ① 机制总览中插入一个可拖拽旋转的真 3D 视锥交互面板（新 B），原 B 面板改名 C——按钮熄灭时为常规视锥（等大候选点），点亮"加入深度估计"后解剖射线的 41 个点大小 ∝ α_d（示意）。

**Architecture:** 单文件内新增一个 `<div class="panel">`（canvas + 按钮 + 图注）与全文唯一的内联 `<script>`：手写透视投影（3D 点 → 相机 → 屏幕坐标）、画家算法按深度排序绘制、yaw/pitch 拖拽旋转；几何常量来自底稿（41 档、4.0–44.0 m、Δd=1.0 m），α 为标注"示意"的 softmax 单峰。

**Tech Stack:** 原生 JS + Canvas 2D，零外部依赖。

**Spec:** `bev_projection/design_3d_frustum.md`（执行者须同读）

## Global Constraints

- 允许且仅允许**一个**内联 `<script>` 块：`grep -c '<script' bev_projection/index.html` 结果必须为 1。
- 外部资源零命中：`grep -nE '(src|href)="https?' bev_projection/index.html` → 0。
- 禁词零命中：`grep -niE "qwen|uvtr|本仓库|analysis/" bev_projection/index.html` → 0。
- 中文语境全角逗号"，"；①②③ 列表换行左对齐。
- 面板中的数字只允许：41 档、4.0–44.0 m、Δd=1.0 m（出处 `src/models.py:157`，用 `.cite` 标注）。
- α 数据必须带"示意"字样（仓库无 checkpoint，不得冒充真实网络输出）。
- 复用现有视觉体系：`.panel` 容器、全局四条目图例语义（橙=可学习、蓝虚线=固定几何、灰白框=数据、箭头=流）、面板标题样式与 A 面板同款（`font-size="14" font-weight="600"` 系）。
- git：只 `git add` 明确路径；英文提交信息；本地 commit 不 push。
- 浏览器实测须走 http 服务（file:// 不可靠）；程序化交互用 evaluate 注入（合成点击进不了页面）；canvas 事件 dispatch 可用但 clientX 取整有 ±2px 噪声，测试避免依赖精确像素；截图超时就关 tab 重开带锚点 URL。

## Review Focus

1. **拖拽旋转无效或只渲染一帧**（事件绑定/重绘缺失）→ T1 Step 3 冒烟 + T2 Step 1 dispatch mouse 事件前后 canvas 像素比对。
2. **两态切换不完整**（点大小未随 α 变、标注未切换、残留上态元素）→ T2 Step 1 熄灭/点亮两态截图对比 + 状态断言。
3. **引入第二个 script 或外部资源** → T1 Step 2 与 T2 Step 2 的 grep 计数审计。
4. **B→C 改名遗漏或引用错乱** → T1 Step 1 的改名与 T2 Step 2 的 `grep -n 'B ·\|C ·'` 核对（新 B=3D 视锥、C=端到端流水线）。
5. **示意 α 冒充真实输出** → T2 Step 2 检查面板含"示意"且无"实测/真实权重"类措辞；数字只有白名单三个。

---

### Task 1: 3D 视锥交互面板实现

**Files:**
- Modify: `bev_projection/index.html`（LSS 节 ① 机制总览区域内，A 面板 SVG 之后、原 B 面板 SVG 之前；仅此文件）

**Interfaces:**
- Consumes: 现有 `.panel` 容器样式、全局图例、`.cite`；底稿 `bev_projection/notes/01_lss.md` 的数字（41/4.0–44.0/Δd=1.0，`src/models.py:157`）。
- Produces（T2 与后续维护依赖，命名锁定）：容器 `id="frustum3d"`、canvas `id="frustum3d-canvas"`、按钮 `id="frustum3d-toggle"`、唯一 script 块 `id="frustum3d-script"`；面板标题文案 "B · 3D 视锥：固定几何 + 深度加权（可拖拽旋转）"；原 B 标题改为 "C · 端到端流水线（重绘自论文 Fig.4）：可学习参数全在两端，中间投影核零"。

- [ ] **Step 1: 实现面板**

结构：`.panel` 容器（`id="frustum3d"`）内含——面板标题行（同 A 面板样式）、`<canvas id="frustum3d-canvas">`（devicePixelRatio 适配，建议显示尺寸 ~880×420）、按钮行（`加入深度估计` 切换按钮 + 状态说明短句）、图注小结（外积 α_d·c 语义一句话 + `.cite` 出处 + "示意"声明）。

3D 内容（全部自写、无库）：

- 几何：相机原点在左侧；视锥四棱线框（张角自定、视觉合理即可，不属白名单数字、不标数值）；41 条等距深度参考弧/线（4.0–44.0 m，Δd=1.0 m，端点数值标注在数处即可）；一根高亮解剖射线（朝锥体中部偏下，固定不可选）。
- 渲染：透视投影（视距/焦距自定）；画家算法按摄像机空间 z 排序图元；点大小 = 基础半径 × 透视因子。
- 初始态：解剖射线上 41 个**等大**候选点（蓝色系 `.fixed` 语义色）。
- 点亮态：41 个点半径 ∝ α_d（示意 softmax 单峰，中心档约 20 m 附近、展开若干档），颜色饱和度随 α 加深（橙系 `.learnable` 语义色）；点旁标注"示意 α 分布"。
- 交互：mousedown/mousemove（含 touchstart/touchmove）拖拽改变 yaw/pitch，松开停止；无自动旋转；按钮切换只改权重显示与相关标注，不重置相机。
- 无障碍与整洁：canvas 带 `role="img"` 与 aria-label；JS 无 console 输出；严格模式 `'use strict'`；IIFE 包裹不泄漏全局（只挂必要的两个 DOM 查询）。

同时完成：原 B 面板 SVG 标题文字改为 "C · …"（见 Interfaces）；检查 A 面板与其后文字无需改动（全文无"图B"字面引用，已核）。

- [ ] **Step 2: 静态审计**

`grep -c '<script' bev_projection/index.html` → 1；`grep -nE '(src|href)="https?' bev_projection/index.html` → 0；`grep -niE "qwen|uvtr|本仓库|analysis/" bev_projection/index.html` → 0；CJK 邻接半角逗号 → 0；`grep -n '示意' ` 面板范围内命中。

- [ ] **Step 3: 浏览器冒烟**

`python -m http.server` 起服务，browser-use 打开 `index.html#sec-lss`：面板渲染完整（线框/参考线/解剖射线/按钮可见）；点亮按钮后点大小肉眼可辨变化、标注切换；拖拽（evaluate 注入 mouse 事件序列）后视角明显改变；console 无错误。

- [ ] **Step 4: Commit**

`git add bev_projection/index.html` → `git commit -m "Add interactive 3D frustum panel to LSS section (new B, retitle pipeline to C)"`

### Task 2: 交互细测与全篇回归

**Files:**
- Modify（仅当发现缺陷）: `bev_projection/index.html`

**Interfaces:**
- Consumes: T1 的三个锁定 id 与两态语义。

- [ ] **Step 1: 交互与两态验证**

http 服务 + browser-use：① 拖拽前后 canvas 截图像素差异显著（或读取页内暴露的 yaw/pitch 断言变化）；② 熄灭/点亮两态各截一张解剖射线区域截图，对比点大小分层（点亮态应有明显大小梯度、熄灭态等大）；③ 连续切换 5 次无残留/无内存性错误（console 干净）；④ 窄一点窗口（~1100px）下面板不溢出。

- [ ] **Step 2: 全篇回归审计**

`grep -n 'B ·\|C ·' bev_projection/index.html`——LSS 节应为 B=3D 视锥、C=端到端流水线（其余节的分区字母不受影响，逐一目检命中行）；外部资源/禁词/逗号/script 计数四项重跑全过；A、C 面板及目录锚点渲染无恙（截图目检）；面板小结数字仅白名单三个且带 `.cite`。

- [ ] **Step 3: 修复与提交**

发现缺陷则最小修复并复跑 Step 1–2；`git add bev_projection/index.html`（如有改动）→ `git commit -m "Verify and polish 3D frustum panel interactions"`；无改动则免提交。

- [ ] **Step 4: 交付**

向用户报改动位置、浏览器查看方式、与既有审计的关系（唯一 script、离线自包含未破坏）。

---

## Self-Review 记录

- **Spec 覆盖**：design_3d_frustum.md 目标 1→T1S1（真 3D 三要素）+T1S3（拖拽），目标 2→T1S1 初始态，目标 3→T1S1 点亮态+小结，目标 4→T1S1 改名+T2S2 核对，目标 5→T1S1 样式复用；非目标/反目标→Global Constraints 与 T2 审计；方案选择表→已定不需任务；偏离说明（唯一 script）→约束第 1 条。
- **比例**：计划远小于将产出的代码，未替实现者写实现。

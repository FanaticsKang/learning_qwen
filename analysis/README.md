# Qwen-Drive-1.0 模型结构剖析笔记

依据:论文 `../qwen_drive.pdf`(arXiv:2609.00111)+ 官方代码仓库 `../Qwen-Drive-1.0/`(commit 28091c1)。

| 笔记 | 内容 |
| --- | --- |
| [00_overall_architecture.html](00_overall_architecture.html) | 整体框架图 · 精简版(配 [PNG](00_overall_architecture.png)):论文图 2 的极简展开 —— 观测图像 → Vision Encoder → Qwen3.5-4B(文本回复 / F_m 隐状态 / K·V 缓存三路输出)→ BEV 感知头(3D 检测 · 占据 · 地图)与规划专家(流匹配轨迹);框图只留模块名与数据流,已逐模块对照代码校验(浏览器打开) |
| [01_prompt_inputs.md](01_prompt_inputs.md) | 图2 中观测图像对应的 prompt 里到底输入了什么(文本模板逐段拆解 + ChatML 序列结构) |
| [02_image_tokens_vit.md](02_image_tokens_vit.md) | 图像尺寸(320p/720p 策略)、一个 token 对应多大像素、ViT 编码全流程(patch 化 → 维度 → 空间合并 → 进入 LLM) |
| [01_vit_process.html](01_vit_process.html) | 可视化详解:patch 16×16 与 token 32×32 的区别、ViT 编码六步全流程(含交互式 patch/合并演示图,浏览器打开) |
| [01a_rope.html](01a_rope.html) | 可视化详解:RoPE 旋转位置编码专题 —— 自注意力为何对顺序不敏感(置换不变性实时演示)、"二维平面内旋转 Q/K"交互动画、平移不变性动画与代码逐步验证、2D RoPE 与 M-RoPE 回顾(浏览器打开) |
| [01b_mrope.html](01b_mrope.html) | 可视化详解:多模态 RoPE(M-RoPE)专题 —— 一维位置的展平/跨结构碰撞(实时算分对比)、(t,h,w) 三轴分配交互演示、mrope_section [11,11,10] 三段维度与打分分解、get_rope_index 位置表代码、设计取舍与 FAQ(浏览器打开) |
| [01c_video_process.html](01c_video_process.html) | 可视化详解:如果输入是视频文件?—— expand(-1,2,-1,-1) 的"另一条路":解码→2fps 采样(交互帧采样器)→统一 resize→相邻两帧 3D tube 打包(交互演示)→同一套 ViT,video_grid_thw 的 T 与绝对时间 RoPE;10s 视频 token 账单、Qwen-Drive 为何走帧序列(资料原文摘录在 [../reference/](../reference/README.md))(浏览器打开) |
| [01d_permute.html](01d_permute.html) | 可视化详解:_patchify 的 permute(1,3,6,4,7,0,2,5,8) 张量变换逻辑 —— 地址轴 vs 内容轴、为什么"取连续 4 个"恰好是同一个 32×32 块、1536 维 patch 内部解剖与 dim(c,t,i,j) 下标公式验证(浏览器打开) |
| [03_pixel_to_bev.html](03_pixel_to_bev.html) | 可视化详解:从像素到 BEV —— "压缩后投影,怎么保证每个像素被有效利用?"patch 化是划分不是采样、感知路用合并前 56×32 特征、推(LSS/UVTR 软分配)与拉(BEVFormer camera-aware 采样)的双向覆盖、demo 帧 8 相机真实标定实测账(56.6% 假设点落格、80.2% BEV 柱被点亮、每命中体素平均 2.53 份投掷 / 23.3% 双相机交界、空洞 98.9% 在 30 m 外)、patch 地面足迹 vs BEV 格宽(交叉点 ≈23 m)、页内 JS 用真实 lidar2img 复算对账(浏览器打开) |
| [04_view_transform.html](04_view_transform.html) | 可视化详解:BEV 感知头的深度视图变换(论文 Eq. 2)——DepthNet/ASPP、深度分布如何把图像特征撒进 3D 体素、无深度监督、代码 scatter 实现(浏览器打开) |
| [05_uvtr_frustum.html](05_uvtr_frustum.html) | 可视化详解:UVTR vs LSS 两种深度视图变换的对比 —— Lift/Splat/Shoot 与视锥体素池化、外积内存账、输出空间 2D/3D 之别、本仓库为何选 UVTR(浏览器打开) |
| [06_bev_head_architecture.html](06_bev_head_architecture.html) | 可视化详解:图 3(a) BEV 感知头全架构 —— ViT→UVTR/LLM 两路的维数地图、BEV Transformer(6 层注意力)与 UVTR 的分工、检测/占用/建图三头(浏览器打开) |
| [07_image_journey.html](07_image_journey.html) | 可视化详解:一张图像的旅程 —— 当前/历史帧进 VLM、同帧 896×512 进感知路、ViT 前后尺寸、UVTR 前后(图像平面→视锥→3D 体素)、BEV Transformer 的 embedding/query/K·V,Qwen3.5-4B 为黑盒(浏览器打开) |
| [positional_encoding.html](positional_encoding.html) | 可视化详解:位置编码培训 v1.1.0 —— 从正弦波到 Qwen3.5 的几何直觉:1D → 2D → 3D → 2.5D,含交互式演示组件(浏览器打开) |
| [09_linear_attention.html](09_linear_attention.html) | 可视化详解:线性注意力机制 —— 从 O(N²) Softmax 到 O(N) 递归。标准注意力的成本账(softmax 三步逐行拆解:打分→exp→归一化→加权求和,配 NumPy 代码与逐步实算演示、β 锐度滑杆)、结合律与核函数替换(数字实算演示)、RNN 视角与三种计算排布、KV Cache vs 固定状态显存计算器(Qwen3.5-4B 真实几何)、变体谱系(Linear Transformer/Performer/RetNet/RWKV/Mamba/GLA/DeltaNet/GDN)、与标准注意力逐项对比、精确回忆命门与 Delta Rule 状态热图步进器,以及 Qwen3.5 "1×GQA+3×GDN" 混合的动机(规划专家的 8 份缓存为何恰来自 GQA 层)(浏览器打开) |

代码位置速查:

```
Qwen-Drive-1.0/
├── src/qwen_drive/scene.py                  # prompt 构建、图像预处理、DrivingScene/QwenDriveProcessor
├── src/qwen_drive/modeling_qwen_drive.py    # QwenDriveForPlanning 主模型(VLM + Planning Expert)
├── src/qwen_drive/configuration_qwen_drive.py  # 所有默认配置(图像预算、patch 尺寸等)
├── src/qwen_drive/planning_expert.py        # Planning Expert(1.1B DiT)
├── src/qwen_drive_perception/               # BEV 感知头
└── docs/model.md                            # 架构数字表(层数/头数/归一化尺度)
```

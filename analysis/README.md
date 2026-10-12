# Qwen-Drive-1.0 模型结构剖析笔记

依据:论文 `../qwen_drive.pdf`(arXiv:2609.00111)+ 官方代码仓库 `../Qwen-Drive-1.0/`(commit 28091c1)。

| 笔记 | 内容 |
| --- | --- |
| [01_prompt_inputs.md](01_prompt_inputs.md) | 图2 中观测图像对应的 prompt 里到底输入了什么(文本模板逐段拆解 + ChatML 序列结构) |
| [02_image_tokens_vit.md](02_image_tokens_vit.md) | 图像尺寸(320p/720p 策略)、一个 token 对应多大像素、ViT 编码全流程(patch 化 → 维度 → 空间合并 → 进入 LLM) |
| [03_vit_process.html](03_vit_process.html) | 可视化详解:patch 16×16 与 token 32×32 的区别、ViT 编码六步全流程(含交互式 patch/合并演示图,浏览器打开) |
| [03a_rope.html](03a_rope.html) | 可视化详解:RoPE 旋转位置编码专题 —— 自注意力为何对顺序不敏感(置换不变性实时演示)、"二维平面内旋转 Q/K"交互动画、平移不变性动画与代码逐步验证、2D RoPE 与 M-RoPE 回顾(浏览器打开) |
| [03b_mrope.html](03b_mrope.html) | 可视化详解:多模态 RoPE(M-RoPE)专题 —— 一维位置的展平/跨结构碰撞(实时算分对比)、(t,h,w) 三轴分配交互演示、mrope_section [11,11,10] 三段维度与打分分解、get_rope_index 位置表代码、设计取舍与 FAQ(浏览器打开) |
| [04_view_transform.html](04_view_transform.html) | 可视化详解:BEV 感知头的深度视图变换(论文 Eq. 2)——DepthNet/ASPP、深度分布如何把图像特征撒进 3D 体素、无深度监督、代码 scatter 实现(浏览器打开) |
| [05_uvtr_frustum.html](05_uvtr_frustum.html) | 可视化详解:UVTR vs LSS 两种深度视图变换的对比 —— Lift/Splat/Shoot 与视锥体素池化、外积内存账、输出空间 2D/3D 之别、本仓库为何选 UVTR(浏览器打开) |
| [06_bev_head_architecture.html](06_bev_head_architecture.html) | 可视化详解:图 3(a) BEV 感知头全架构 —— ViT→UVTR/LLM 两路的维数地图、BEV Transformer(6 层注意力)与 UVTR 的分工、检测/占用/建图三头(浏览器打开) |
| [07_image_journey.html](07_image_journey.html) | 可视化详解:一张图像的旅程 —— 当前/历史帧进 VLM、同帧 896×512 进感知路、ViT 前后尺寸、UVTR 前后(图像平面→视锥→3D 体素)、BEV Transformer 的 embedding/query/K·V,Qwen3.5-4B 为黑盒(浏览器打开) |

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

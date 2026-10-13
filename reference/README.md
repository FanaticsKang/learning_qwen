# 参考资料:Qwen-VL 系如何处理视频输入

回答的问题:`analysis/03_vit_process.html` 中 `pixels.unsqueeze(1).expand(-1, 2, -1, -1)`
(单帧图像复制 2 份凑齐时间维)——**如果输入是视频文件,这里会怎么处理?**

## 一句话结论

`temporal_patch_size = 2` 的设计本来是为视频准备的(ViViT 式 3D tube patch)。
视频输入时,时间维不再靠 `expand` 复制填充,而是由**真实采样的相邻两帧**填满:
每 2 帧打包成 1 个时空 patch(grid_t = nframes / 2),同时 M-RoPE 的时间位置
沿**绝对时间**推进,而不是图像路径里的"同一时刻"。

## 文件索引

| 文件 | 内容 | 回答了什么 |
| --- | --- | --- |
| [qwen_vl_utils_vision_process.py](qwen_vl_utils_vision_process.py) | Qwen 官方预处理库完整源码(Qwen2.5-VL / Qwen3-VL 通用):`fetch_video` / `smart_nframes` / 三种解码后端 | 视频文件从 mp4 到 `(T,C,H,W)` 张量的完整工程流程 |
| [01_qwen2_vl_paper_video.md](01_qwen2_vl_paper_video.md) | Qwen2-VL 论文(arXiv:2409.12191)视频章节原文摘录 | 为什么图像要复制成两帧(论文原句)、3D tube、2fps 采样、token 上限 |
| [02_qwen2_5_vl_paper_video.md](02_qwen2_5_vl_paper_video.md) | Qwen2.5-VL 技术报告(arXiv:2502.13923)视频章节摘录 | 绝对时间对齐 M-RoPE、动态 FPS、768 帧/24576 token 上限、窗口注意力 |
| [03_transformers_video_mrope.md](03_transformers_video_mrope.md) | transformers `modeling_qwen2_5_vl.py` 的 `get_rope_index` 视频分支代码 | 视频时间位置 id 怎么算(`second_per_grid_ts × tokens_per_second`)、与图像分支(全 0)的区别 |
| [04_qwen_vl_utils_video_pipeline.md](04_qwen_vl_utils_video_pipeline.md) | 对官方预处理源码的带注释导读 | 采样帧数公式、帧数为什么必须是偶数、每帧像素预算怎么摊销 |
| [05_qwen3_vl_video_features.md](05_qwen3_vl_video_features.md) | Qwen3-VL 官方模型卡视频相关特性摘录 | 同系最新演进:Interleaved-MRoPE、DeepStack、文本-时间戳对齐 |
| [06_qwen_drive_repo_video.md](06_qwen_drive_repo_video.md) | Qwen-Drive-1.0 仓库自身的视频线索(论文图 2、训练数据、data.md) | 本模型支持视频的证据,以及规划流水线为什么改走"帧序列"路径 |

## 下载来源

- qwen_vl_utils:`https://github.com/QwenLM/Qwen2.5-VL` → `qwen-vl-utils/src/qwen_vl_utils/vision_process.py`(main 分支)
- Qwen2-VL 论文 HTML:`https://arxiv.org/html/2409.12191v2`(§2.1、§2.2、表 5);**原始 PDF 已下载:[papers/qwen2_vl_2409.12191.pdf](papers/qwen2_vl_2409.12191.pdf)**
- Qwen2.5-VL 技术报告 HTML:`https://arxiv.org/html/2502.13923v1`(§2.1.1–2.1.3、§3.3.4);**原始 PDF 已下载:[papers/qwen2_5_vl_2502.13923.pdf](papers/qwen2_5_vl_2502.13923.pdf)**
- transformers v4.57.1 `src/transformers/models/qwen2_5_vl/modeling_qwen2_5_vl.py`(`get_rope_index`)
- Qwen3-VL 模型卡:`https://huggingface.co/Qwen/Qwen3-VL-8B-Instruct`
- Qwen-Drive 论文 arXiv:2609.00111(本地 `../2609.00111v1.pdf`)+ 仓库 `Qwen-Drive-1.0/docs/data.md`

## 论文 PDF(papers/)

| 文件 | 论文 | 对应笔记 |
| --- | --- | --- |
| [qwen2_vl_2409.12191.pdf](papers/qwen2_vl_2409.12191.pdf)(35.9MB,v2) | Qwen2-VL(arXiv:2409.12191) | [01_qwen2_vl_paper_video.md](01_qwen2_vl_paper_video.md)(§2.1.1、§2.1.2、§2.2、表 5) |
| [qwen2_5_vl_2502.13923.pdf](papers/qwen2_5_vl_2502.13923.pdf)(2.2MB,v1) | Qwen2.5-VL Technical Report(arXiv:2502.13923) | [02_qwen2_5_vl_paper_video.md](02_qwen2_5_vl_paper_video.md)(§2.1.1–2.1.3、§3.3.4) |
| `../../2609.00111v1.pdf`(仓库根目录) | Qwen-Drive(arXiv:2609.00111) | [06_qwen_drive_repo_video.md](06_qwen_drive_repo_video.md) |

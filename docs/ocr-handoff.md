# OCR 路线交接记录

最后更新：2026-08-01

这份文档用于在另一台电脑继续评估日本综艺硬字幕识别，不代表 OCR 已经接入正式流水线。

## 当前结论

- 后续翻译模型默认使用 GPT/Codex，不再继续 Gemini 对比测试。
- 当前仓库已经支持本地视频 + 外挂 ASS 字幕的翻译；这条路线已经实际跑通。
- 在线视频无外挂字幕时，原项目已有 `提取音频 → ElevenLabs Scribe ASR → SRT → 翻译` 流程。
- 本地视频无外挂字幕的 ASR 入口尚未实现，但可以复用在线流程的公共处理阶段。
- 硬字幕 OCR 尚未实现，也没有任何 OCR 结果可以作为验收结果。

## 之前的 OCR 尝试说明

曾经从 `Test/#1321-2016.09.11-I_Think_This_Item_Will_Suit_You_(Ryucheru).mkv` 抽取过少量 JPEG 帧，并把图片提交给 GPT-5.6 做视觉识别可行性尝试。

这次尝试不能作为 OCR 测试，原因是：

- 测试视频是日本原片，画面中没有要识别的英文硬字幕。
- 仓库里的 ASS 是外挂字幕文件，不是视频画面中的烧录字幕。
- 没有生成可用的 OCR JSON、SRT 或时间轴结果。
- 临时图片位于系统 `/tmp`，没有放入仓库，也不应作为项目产物依赖。

因此，后续必须使用真正带英文烧录字幕的视频短片验证，不能继续使用当前 `Test` 视频判断 OCR 质量。

## OCR 方案候选

### 方案 A：Video Subtitle Extractor（首选验证对象）

项目：[YaoFANGUK/video-subtitle-extractor](https://github.com/YaoFANGUK/video-subtitle-extractor)

它面向“硬字幕视频 → 外挂 SRT”，已经覆盖关键帧提取、字幕区域检测、OCR、连续帧去重和时间轴恢复。建议先把它作为独立工具运行，不要立即把其依赖合并进本仓库。

优先测试 `Fast` 或 `Auto` 模式；`Precise` 接近逐帧处理，速度可能不适合本地长视频。

### 方案 B：PaddleOCR PP-OCRv5（后续自建适配器）

官方文档：[PP-OCRv5 多语言识别](https://github.com/PaddlePaddle/PaddleOCR/blob/main/docs/version3.x/algorithm/PP-OCRv5/PP-OCRv5_multi_languages.md)

它支持英文、日文、中文等语言，并可返回文字框和置信度，适合处理底部字幕以及人物旁边、顶部、中部的特殊位置文字。但连续帧合并、字幕出现/消失时间和 SRT 生成需要由本项目自己实现。

### 方案 C：GPT 视觉逐帧识别（不作为主 OCR）

GPT 视觉可以理解复杂画面，但逐帧发送图片成本高，时间轴恢复和重复结果合并仍需要自行实现。建议只用于 OCR 结果纠错、字幕类型判断和翻译，不作为第一版扫描引擎。

### 方案 D：云端视频文字识别（暂不采用）

例如 Google Cloud Video Intelligence 可以返回文字时间段、置信度和文字框，但需要上传视频并产生按时长计费，不符合当前优先本地处理的方向。

## 当前机器能力

本机硬件：

```text
MacBook Pro
Apple M1 Pro
10 核 CPU
32 GB 内存
macOS 15.7.7
arm64
```

结论：本机可以运行 PaddleOCR，但应走 macOS arm64 CPU 路线，不能使用 NVIDIA CUDA。适合先做 1080p、底部区域、低频抽帧的 OCR；不适合默认逐帧扫描整部长视频。

PaddlePaddle 官方 macOS 安装说明：[Install on macOS via PIP](https://www.paddlepaddle.org.cn/documentation/docs/en/install/pip/macos-pip_en.html)

不要把 PaddleOCR 依赖直接安装到本项目的 `.venv`。应使用独立 OCR 环境，避免破坏当前项目的 Python 3.13 和翻译依赖。

## 下一台电脑的任务

### 1. 准备正确的测试样本

准备同一段 3～5 分钟、1080p、确实带英文烧录字幕的视频。最好包含：

- 底部连续对白字幕；
- 人物旁边或画面中部的特殊字幕；
- 字幕切换、两行字幕和不同字体/颜色；
- 可以人工核对的短片。

不要把视频中的日文花字误当成目标英文硬字幕，也不要把外挂 ASS 当成烧录字幕测试数据。

### 2. 分别建立隔离环境

至少分别测试：

```text
VSE 环境
PaddleOCR 环境
当前项目环境（只负责后续翻译）
```

不要先修改 `pyproject.toml` 或 `uv.lock` 来加入 OCR 依赖。

### 3. 记录统一指标

对同一段视频记录：

- 安装是否成功；
- 是否支持当前电脑的 CPU/架构；
- 总处理时间；
- 峰值内存和 CPU 占用；
- 字幕文本召回率；
- OCR 错字数量；
- 起止时间误差；
- 特殊位置字幕是否被识别；
- 是否能导出可用 SRT。

### 4. 选择接入方式

推荐判断顺序：

```text
先跑 VSE
→ 如果底部英文字幕质量和时间轴够用，先接 VSE 输出
→ 如果特殊位置召回不足，再用 PaddleOCR 做全画面补充
→ GPT 只做 OCR 纠错、筛选和翻译
```

正式接入后统一保留这些中间产物：

```text
video.source.ocr.json   # 文字、时间、坐标、置信度
video.source.ocr.srt    # 普通字幕时间轴
video.chs.srt           # GPT 翻译结果
video.chs.ass           # 最终外挂 ASS
```

特殊位置文字必须保留 JSON 坐标；SRT 本身不能表达字幕位置。普通底部字幕可以输出 SRT，特殊位置字幕后续再决定是否生成定位 ASS。

## 当前仓库状态

当前分支和远程：

```text
branch: feature/local-ass-input
HEAD: 346ff19 fix(project): fit archive and package dir names to platform path limits
origin: git@github.com:twodogwang/bangumi-grillmaster.git
upstream: git@github.com:elishahung/owarai-grillmaster.git
```

工作区目前有尚未提交的本地改动，包括本地 ASS 流程、GPT/翻译流程调整、Gemini CLI 兼容性改动和测试文件。另一台电脑只执行 `git clone` 时，拿不到这些未提交改动；要继续当前本地 ASS 开发，需要先在本机提交并推送，或者另外传递补丁。

本次 OCR 评估没有修改正式 OCR 代码，也没有把 OCR 依赖加入项目。

# HoYo Quest Voice — 交互式架构全景文档 (Archify)

本目录基于开源架构可视化引擎 [tt-a1i/archify](https://github.com/tt-a1i/archify) 构建，为 **HoYo Quest Voice（崩铁支线剧情 AI 实时配音）** 项目提供一套完整、类型化验证且可交互的多维架构文档体系。

所有产物均为**自包含的独立 HTML 文件**，内嵌 SVG 与交互式查看器，无需部署任何 Web 服务，**直接在任意现代浏览器中双击即可打开**。

---

## 快速导航与文档矩阵

| 文档类型 | 交互式 HTML 产物（点击或在本地双击打开） | 架构规格源文件 (JSON IR) | 核心表达与架构职责 |
|---|---|---|---|
| **1. 系统分层架构图**<br>`architecture` | [**`system-architecture.architecture.html`**](system-architecture.architecture.html) | [`system-architecture.architecture.json`](system-architecture.architecture.json) | **全景分层与边界划分**：崩铁游戏客户区 (HWND)、Win32 DPI-aware 捕获、8×32 灰度指纹门控、RapidOCR 引擎、六级说话人路由、HeadlessRuntime 调度、GPT-SoVITS 权重热切、Winsound 异步播放以及 FastAPI / Bun SPA 控制台。 |
| **2. 业务处理工作流图**<br>`workflow` | [**`dialogue-voice.workflow.html`**](dialogue-voice.workflow.html) | [`dialogue-voice.workflow.json`](dialogue-voice.workflow.json) | **端到端流程与分支防护**：采用 Archify Workflow v2 规范，涵盖游戏画面采集、打字机打字态等待 (`CHANGING`)、文字稳定后触发 (`STABLE_NEW`)、非对话场景短路 (`UNSUPPORTED`)、别名纠错匹配、音频同句去重与播放输出。 |
| **3. 核心调用时序图**<br>`sequence` | [**`request-playback.sequence.html`**](request-playback.sequence.html) | [`request-playback.sequence.json`](request-playback.sequence.json) | **主循环 Tick 级交互时序**：主轮询 Tick (15 Hz) 驱动捕获、指纹相似度计算门控、RapidOCR 区域文本推理、SpeakerRouter 六级决策、GPT-SoVITS `/tts` 权重切换与音频生成、Winsound 异步播报与事件发布。 |
| **4. 画面到音频数据流向图**<br>`dataflow` | [**`frame-audio.dataflow.html`**](frame-audio.dataflow.html) | [`frame-audio.dataflow.json`](frame-audio.dataflow.json) | **数据形态转换流水线**：HWND 显存缓冲区 → 原生 uint8 RGB 帧 → 降维 8×32 灰度特征与 ROI 裁切 → TextEvent 结构化文本与规范角色 ID → GPT-SoVITS 模型参数与 PCM/WAV 字节流 → 扬声器播放与遥测事件流。 |
| **5. 状态机与生命周期图**<br>`lifecycle` | [**`dialogue-state.lifecycle.html`**](dialogue-state.lifecycle.html) | [`dialogue-state.lifecycle.json`](dialogue-state.lifecycle.json) | **帧差与语音任务生命周期**：主轨道（`BOOTSTRAP` → `CHANGING` → `STABLE_NEW` → `SYNTHESIZING` → `PLAYING_DONE`）与等待分支（`UNCHANGED` 静止帧免计算、`CACHED_HIT` 缓存复用）、可恢复异常（`TTS_RETRY`）及终态退出（`SCENE_DROP`、`STALE_EXPIRED`）。 |

---

## 交互式功能与使用指南

生成的 HTML 架构图集成了完整的桌面级交互能力：

1. **主题自适应与日夜切换**：
   - 支持深色（Dark）和浅色（Light）模式，界面左上角或快捷键随时切换，与原项目主题风格契照。
2. **缩放与平移 (Pan & Zoom)**：
   - 鼠标滚轮缩放画布，按住鼠标左键任意拖拽平移，亦可通过界面缩放滑块一键重置（Reset）。
3. **节点检索与聚焦 (Search & Focus)**：
   - 支持实时输入关键字筛选节点，点击任意组件将高亮其上下游依赖关系（Upstream / Downstream Trace）。
4. **引导式章节导览 (Guided Views / Story Mode)**：
   - 每个图表均内置 3 个精选视角（如主生成通路、帧差优化链路、容错自愈分支），点击顶部视角标签即可一键聚焦特定业务场景。
5. **高清图表导出 (Export)**：
   - 支持一键导出当前高亮视图为 PNG（包含 1200×630 标准分享卡片 Share Card）、矢量 SVG、甚至 WebM 动画。

---

## 架构核心设计要点提炼

### 1. 为什么采用无侵入式设计？
本项目运行在 Windows 平台，不采用 DLL 注入、内存读写或游戏文件 Hook 等高风险方案，完全通过 Win32 API（`PrintWindow` / `BitBlt`）结合 DPI-aware 缩放抓取原生分辨率画面，确保账号安全性。

### 2. 帧差门控如何节省 80%+ 资源？
游戏对话通常带有“打字机”逐字浮现特效，若每帧都调用 OCR：
- 算力严重浪费（RapidOCR 推理每次消耗数百毫秒）。
- 容易在字未出全时抢跑合成出断句与破音。
因此，`FrameChangeGate` 将对白框下采样为极其轻量的 8×32 灰度指纹（约 1ms 纯 numpy 计算）：
- 文字变动中处于 `CHANGING` 状态，主循环保持轻量观察；
- 文字完全显示且指纹稳定时才判定 `STABLE_NEW`，仅此一刻触发 OCR 与 TTS；
- 文字未变动时处于 `UNCHANGED` 状态，完全跳过 OCR 与 TTS。

### 3. 六级说话人优先级路由
OCR 可能会受游戏特效干扰偶发错别字（如将“流萤”识别为“流茧”）。`SpeakerRouter` 采用六级严格优先级保障声线稳定性：
1. **主角 (Protagonist)**：选项框专属声线
2. **旁白 (Narrator)**：无角色名气泡专属声线
3. **规范名称 (Canonical)**：标准角色注册表匹配
4. **别名 (Alias)**：别称或缩写映射
5. **纠错规则 (Correction Rule)**：OCR 常见形近字纠错词典
6. **默认后备 (Default Fallback)**：未知角色兜底声线

---

## 维护与重新编译指南

若后续修改了项目架构、配置文件或新增了角色模型，可按如下步骤更新 Archify 文档：

### 1. 依赖环境
确保已安装 Node.js (>= 18)。

### 2. 验证规范（Showcase 质量级别）
在 Archify 工具目录下执行校验（以架构图为例）：
```powershell
node <archify_dir>\bin\archify.mjs validate architecture <path_to>\system-architecture.architecture.json --quality showcase --json
```
> 必须满足全部 9 项严苛检查，0 composition errors, 0 warnings。

### 3. 编译交付自包含 HTML
```powershell
node <archify_dir>\bin\archify.mjs deliver architecture <path_to>\system-architecture.architecture.json <path_to>\system-architecture.architecture.html --quality showcase --json
```
编译器将执行 SHA-256 校验和固定、SVG 渲染与交互组件打包，原子化输出最终的 `.html` 产物。

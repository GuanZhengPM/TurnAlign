# TurnAlign

> Streaming ASR and speaker diarization orchestration with pluggable backends.  
> 可插拔模型的流式 ASR 与说话人编排引擎：稳定流式文本输出，解耦正文、时间戳与说话人轨道，支持跨硬件调度。

[简体中文](#简体中文) · [English](#english) · [Hardware Validation / 硬件实测](docs/validation.md)

---

<a id="简体中文"></a>

## 简体中文

### 背景与解决的问题

流式语音识别（Streaming ASR）结合说话人分离在实际工程落地中通常面临以下几大挑战：

1. **界面文字跳动与回滚**：滑动窗口识别时，模型在获得后续上下文后会频繁重写句尾，导致前端展示反复闪烁；
2. **说话人切分延迟与串音**：声纹聚类需要一定的音频长度建立特征，简短插话、抢话与重叠语音容易被误判；
3. **异构模型难以对齐**：高精度的转录文本、字词级强制对齐时间戳与说话人声纹往往来自完全不同的开源模型，缺乏统一的数据管道将三者结合；
4. **硬件环境异构性**：NVIDIA CUDA、AMD ROCm、Apple Silicon MPS 与纯 CPU 环境的算子支持、显存占用与最佳 Batch 策略各不相同。

TurnAlign 针对上述问题提供了一套**分轨解耦与时间轴融合**的流式编排方案，底层识别模型可热替换，对外暴露统一的事件协议。

### 架构设计

```mermaid
flowchart LR
    A[音频流 / 文件 / 麦克风] --> B[VAD 语音切分]
    B --> C[滑动窗口 ASR]
    C --> D[LocalAgreement 稳定算法]
    D --> E[文本轨道 (commit / partial)]
    
    A --> T[时间戳轨道 (Paraformer 对齐)]
    A --> S[说话人轨道 (CAM++ 声纹)]
    
    E --> F[时间轴融合引擎]
    T --> F
    S --> F
    F --> O[统一输出: JSONL / SRT / WebSocket]
```

* **LocalAgreement 稳定前缀算法**：对比连续滑动窗口的识别结果，仅向客户端提交一致的公共前缀（`commit`），句尾保留为可变文本（`partial`），彻底消除文字频繁回退；
* **三轨解耦与时间轴融合**：转录正文、时间戳与说话人标签分轨生成，最终按统一时间区间（`segment_id`）融合成结构化数据；
* **插件式后端架构**：通过 Python Entry Point 注册组件，正文识别、VAD 与说话人模型均可独立升级或替换；
* **两遍处理流水线 (Two-pass Pipeline)**：支持实时快速第一遍输出结合离线二次对齐精调，在兼顾低延迟的同时保证终稿质量。

### 当前集成的模型与技术组件

| 角色 | 默认 / 内置适配器 | 说明 |
| :--- | :--- | :--- |
| **正文识别 (ASR)** | `glm-asr` (GLM-ASR-Nano-2512), `transformers-whisper`, `faster-whisper`, `funasr-streaming`, `whisper-cpp` | 支持中文通用、中英混说及热词注入 |
| **语音活动检测 (VAD)** | `fsmn-vad` (FunASR), `energy` (能量兜底) | 截断静音并进行安全切段 |
| **时间对齐 (Aligner)** | `paraformer` (字词级对齐) | 精确计算每个字词的毫秒级起止时间 |
| **说话人分离 (Diarizer)** | `campp` (CAM++ 离线聚类) | 提取声纹向量并完成发言人归属区分 |

### 快速开始

#### 1. 安装

TurnAlign 核心管道仅依赖 Python 标准库，按需安装扩展运行时：

```bash
# 基础安装
pip install .

# 启用麦克风输入、WebSocket 服务与 Transformers 支持
pip install ".[microphone,server,transformers]"

# 启用 FunASR 对齐与说话人识别流水线
pip install ".[funasr-pipeline]"
```

#### 2. 环境诊断与设备配置

自动探测当前系统的 GPU 类型（CUDA / ROCm / MPS / CPU）并输出最优执行 Profile：

```bash
turnalign doctor --device auto
```

#### 3. 命令行调用示例

```bash
# 文件转录（结合 GLM-ASR 识别、FSMN-VAD 切段、Paraformer 对齐与 CAM++ 说话人区分）
turnalign transcribe meeting.mp3 \
  --backend glm-asr \
  --vad-backend fsmn-vad \
  --aligner paraformer \
  --diarizer campp \
  --output meeting.jsonl

# 注入私有业务热词（提升特定专有名词识别率）
turnalign transcribe meeting.mp3 \
  --backend glm-asr \
  --hotwords-file /path/to/private-terms.txt \
  --output meeting.jsonl

# 麦克风实时流式监听
turnalign listen \
  --backend funasr-streaming \
  --model paraformer-zh-streaming \
  --language zh
```

### 统一事件流格式

无论是 WebSocket 推流还是离线输出，均遵循统一的事件定义：

```json
{"event": "partial", "segment_id": "seg_01", "text": "今天讨论下季度的"}
{"event": "commit",  "segment_id": "seg_01", "text": "今天讨论下季度的业务规划", "start": 1200, "end": 3500}
{"event": "replace", "segment_id": "seg_01", "speaker": "Speaker_A", "text": "今天讨论下季度的业务规划"}
```

### 跨平台硬件验证摘要

项目已在多类实际硬件上完成端到端全链路验证（详见 [docs/validation.md](docs/validation.md)）：

* **AMD ROCm Linux (RX 7650 GRE)**：129.4 分钟全量录音实测，GLM-ASR 耗时 980s，FunASR 对齐+CAM++ 耗时 369s（约 21 倍实时），字符对齐率中位数 88.6%；
* **Apple Silicon (Mac Studio M系列 MPS)**：PyTorch MPS 设备检测、FP16 张量计算与 Transformers Whisper 端到端转录验证通过；
* **CPU-only (INT8)**：基于 `faster-whisper` Medium INT8 4 线程，实测维持在 4.27 倍实时速度，CPU 占用稳定。

### License

核心编排代码使用 [MIT License](LICENSE)。各模型权重及算法库遵循其原始许可证。

---

<a id="english"></a>

## English

### Overview

`TurnAlign` is an orchestration framework for streaming automatic speech recognition (ASR) and speaker diarization. It decouples transcript decoding, word-level timestamp alignment, and speaker diarization into independent tracks, fusing them along a unified timeline while stabilizing streaming text outputs via the `LocalAgreement` algorithm.

### Problem Statement & Background

Streaming Automatic Speech Recognition (ASR) combined with speaker diarization faces several persistent engineering hurdles in production:

1. **UI Flicker & Text Rollback**: In sliding-window streaming, models frequently revise trailing tokens as additional context arrives, causing transcript jitter in client UIs.
2. **Diarization Latency & False Merges**: Voice embedding clustering requires temporal accumulation to stabilize speaker profiles; short interruptions, conversational overlaps, and interjections are easily misattributed.
3. **Alignment of Heterogeneous Models**: High-accuracy text decoding, word-level forced alignment, and speaker recognition often originate from distinct open-source models with no standardized integration protocol.
4. **Hardware Discrepancies**: Accelerators (NVIDIA CUDA, AMD ROCm, Apple Silicon MPS, and CPU-only hosts) exhibit divergent operator support, VRAM thresholds, and optimal batching behaviors.

TurnAlign solves these challenges through a **decoupled, multi-track timeline orchestration** architecture with hot-swappable recognition backends and unified event schemas.

### Architecture

```mermaid
flowchart LR
    A[Audio Stream / File / Mic] --> B[VAD Segmentation]
    B --> C[Sliding Window ASR]
    C --> D[LocalAgreement Algorithm]
    D --> E[Text Track (commit / partial)]
    
    A --> T[Timestamp Track (Paraformer Alignment)]
    A --> S[Speaker Track (CAM++ Embeddings)]
    
    E --> F[Timeline Fusion Engine]
    T --> F
    S --> F
    F --> O[Unified Output: JSONL / SRT / WebSocket]
```

* **LocalAgreement Prefix Stabilization**: Compares transcriptions across consecutive overlapping windows, committing only identical prefix tokens (`commit`) while keeping trailing tokens mutable (`partial`).
* **Decoupled Multi-Track Fusion**: Separates text generation, forced timestamp alignment, and speaker clustering into independent pipelines, merging them along a shared timeline via stable `segment_id` references.
* **Pluggable Backend Architecture**: Extensible via Python entry points, allowing ASR, VAD, alignment, and diarization modules to be swapped or upgraded independently.
* **Two-Pass Pipeline**: Combines rapid real-time first-pass streaming with optional second-pass offline refinement for publication-ready transcripts.

### Integrated Components & Models

| Role | Default / Built-in Adapters | Description |
| :--- | :--- | :--- |
| **Speech Recognition (ASR)** | `glm-asr` (GLM-ASR-Nano-2512), `transformers-whisper`, `faster-whisper`, `funasr-streaming`, `whisper-cpp` | General speech, mixed Chinese-English, and hotword injection support. |
| **Voice Activity Detection (VAD)** | `fsmn-vad` (FunASR), `energy` (Adaptive energy fallback) | Silence truncation and bounded speech segmentation. |
| **Forced Alignment** | `paraformer` (Word-level alignment) | Computes millisecond-accurate start and end timestamps per token. |
| **Speaker Diarization** | `campp` (CAM++ Offline Clustering) | Extracts speaker embeddings and resolves multi-speaker attributions. |

### Quick Start

#### 1. Installation

TurnAlign's core engine relies solely on the Python Standard Library. Install runtime extensions as needed:

```bash
# Core installation
pip install .

# Microphone input, WebSocket server, and Transformers support
pip install ".[microphone,server,transformers]"

# FunASR timestamp alignment and speaker diarization pipeline
pip install ".[funasr-pipeline]"
```

#### 2. Environment Diagnostics

Automatically probe accelerator hardware (CUDA / ROCm / MPS / CPU) and inspect optimal profiles:

```bash
turnalign doctor --device auto
```

#### 3. CLI Examples

```bash
# File transcription (combining GLM-ASR, FSMN-VAD, Paraformer alignment, and CAM++ diarization)
turnalign transcribe meeting.mp3 \
  --backend glm-asr \
  --vad-backend fsmn-vad \
  --aligner paraformer \
  --diarizer campp \
  --output meeting.jsonl

# Inject private domain hotwords (biasing proper nouns and domain terms)
turnalign transcribe meeting.mp3 \
  --backend glm-asr \
  --hotwords-file /path/to/private-terms.txt \
  --output meeting.jsonl

# Real-time microphone listening
turnalign listen \
  --backend funasr-streaming \
  --model paraformer-zh-streaming \
  --language zh
```

### Unified Event Stream Format

All outputs (CLI, JSONL, and WebSocket streams) adhere to a unified schema:

```json
{"event": "partial", "segment_id": "seg_01", "text": "Today we discuss the quarterly"}
{"event": "commit",  "segment_id": "seg_01", "text": "Today we discuss the quarterly roadmap", "start": 1200, "end": 3500}
{"event": "replace", "segment_id": "seg_01", "speaker": "Speaker_A", "text": "Today we discuss the quarterly roadmap"}
```

### Cross-Platform Hardware Verification

TurnAlign has been verified across diverse physical hardware environments (details in [docs/validation.md](docs/validation.md)):

* **AMD ROCm Linux (RX 7650 GRE)**: 129.4-minute full recording benchmarked; GLM-ASR completed in 980s; FunASR alignment + CAM++ finished in 369s (~21x real-time), with median character alignment accuracy of 88.6%.
* **Apple Silicon (Mac Studio M-series MPS)**: PyTorch MPS device detection, FP16 tensor operations, and Transformers Whisper end-to-end transcription passed.
* **CPU-only (INT8)**: Benchmarked with `faster-whisper` Medium INT8 on 4 threads, maintaining 4.27x real-time factor with stable low CPU utilization.

### License

Core orchestration code is licensed under the [MIT License](LICENSE). Model weights and external libraries retain their respective upstream licenses.

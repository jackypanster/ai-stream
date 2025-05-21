+++
title = '基于 FunAudioLLM/SenseVoiceSmall 搭建高效语音转录服务的实践之路'
date = 2025-05-21T15:43:08+08:00
draft = false
tags = ["ASR", "SenseVoice", "FastAPI", "Python", "AI", "FunASR", "uv"]
categories = ["LLM"]
author = "Jacky Pan"
description = "详细记录了使用 SenseVoiceSmall 模型和 FastAPI 框架搭建语音转文本服务的过程，包括环境配置、核心代码、遇到的主要问题及解决方案，并对比了其他主流 ASR 模型。"
+++

## 项目概述

实现一个语音转录文本（ASR）的服务，目标是能够高效地将用户上传的音频文件转换为文字。出于中文语音的考虑，选择了来自 `FunAudioLLM` 的 `SenseVoiceSmall` 模型，它以其多语种支持、高效率以及集成的语音理解能力（如情感识别、事件检测）吸引了我。本文将详细记录从环境配置、核心功能实现到踩坑解决的全过程，并分享一些关于模型选型的思考。

完整代码已开源在 GitHub 仓库：[https://github.com/jackypanster/FunAudioLLM-SenseVoiceSmall](https://github.com/jackypanster/FunAudioLLM-SenseVoiceSmall)

项目需求文档（`prd.md`）关键信息如下：

* **模型**: FunAudioLLM/SenseVoice (具体为 `SenseVoiceSmall`)
* **本地模型路径**: `/home/llm/model/iic/SenseVoiceSmall` (从 ModelScope 下载)
* **API框架**: FastAPI
* **Python环境管理**: `uv`

## 环境配置

为了保持开发环境的纯净和高效，采用了 `uv` 来管理 Python 依赖。

1. **创建虚拟环境** (如果尚未创建):
   ```bash
   uv venv .venv
   source .venv/bin/activate
   ```

2. **安装核心依赖**:
   初始的 `requirements.txt` 包含了 `fastapi`, `uvicorn`, `python-multipart` 等基础库。后续根据模型加载和处理的需求，逐步添加了 `torch`, `torchaudio`, `numpy`, `transformers`, `sentencepiece`, 以及最终解决模型加载问题的核心库 `funasr`。
   ```bash
   uv pip install -r requirements.txt
   ```

## 核心功能实现概览

### 项目结构

项目的主要结构包括：

*   `app/main.py`: FastAPI 应用入口，定义 API 路由和应用生命周期事件（如模型加载）。
*   `app/models/sensevoice_loader.py`: 负责加载 `SenseVoiceSmall` 模型，采用单例模式。
*   `app/services/asr_service.py`: 封装语音处理和模型推理的核心逻辑。
*   `app/schemas.py`: 定义 API 的请求和响应数据模型 (Pydantic models)。

### API 端点

关键的 API 端点设计为：

#### POST /asr_pure

*   **Content-Type**: `multipart/form-data`
*   **Body**: `file` (音频文件)

返回转录后的文本及处理时间。

## 踩坑与解决之路：模型加载的曲折历程

在项目推进过程中，模型加载部分是遇到问题最多的地方，也是收获最多的地方。

### 坑1：Hugging Face `AutoClass` 的 "Unrecognized model"

最初，尝试使用 Hugging Face `transformers` 库通用的 `AutoProcessor.from_pretrained()` 和 `AutoModelForSpeechSeq2Seq.from_pretrained()` 来加载本地的 `SenseVoiceSmall` 模型文件。

```python
# app/models/sensevoice_loader.py (早期尝试)
# from transformers import AutoModelForSpeechSeq2Seq, AutoProcessor
# ...
# self.processor = AutoProcessor.from_pretrained(MODEL_PATH)
# self.model = AutoModelForSpeechSeq2Seq.from_pretrained(MODEL_PATH)
```

然而，服务启动时立即报错：

```
ValueError: Unrecognized model in /home/llm/model/iic/SenseVoiceSmall. Should have a model_type key in its config.json...
```

这个错误表明 `transformers` 的自动发现机制无法识别模型类型，通常是因为模型目录下的 `config.json` 文件缺少 `model_type` 字段，或者该模型需要特定的加载类。

### 坑2：转向 `funasr` 与 `trust_remote_code` 的初步探索

查阅 `FunAudioLLM/SenseVoice` 的官方文档后发现，推荐使用 `funasr` 库的 `AutoModel` 来加载 `SenseVoice` 系列模型。于是调整了代码：

1. **添加 `funasr` 到 `requirements.txt`**。
2. **修改 `SenseVoiceLoader`**:
   ```python
   # app/models/sensevoice_loader.py (引入 funasr)
   from funasr import AutoModel
   # ...
   self.model = AutoModel(
       model=FUNASR_MODEL_NAME_OR_PATH, # 即本地路径
       trust_remote_code=True,
       device=self.device
   )
   ```
   同时，`asr_service.py` 中的推理逻辑也相应调整为调用 `funasr` 模型对象的 `.generate()` 方法。

本以为这样能解决问题，但启动时又遇到了新的日志：

```
Loading remote code failed: model, No module named 'model'
```

尽管这条日志出现，但后续的 API 调用测试居然成功了！这让我非常困惑。

### 坑3：`remote_code` 参数与 `model.py` 文件的“幻影”

深入研究 `funasr` 和 `SenseVoice` 的文档，注意到对于包含自定义代码（如 `model.py`）的模型，除了 `trust_remote_code=True`，有时还需要明确指定 `remote_code` 参数。

我检查了 Hugging Face 仓库 `FunAudioLLM/SenseVoiceSmall` ([https://huggingface.co/FunAudioLLM/SenseVoiceSmall/tree/main](https://huggingface.co/FunAudioLLM/SenseVoiceSmall/tree/main))，发现其文件列表中确实包含一个 `model.py`。因此，我尝试在 `AutoModel` 调用中加入 `remote_code="model.py"`。

```python
# app/models/sensevoice_loader.py (尝试指定 remote_code)
self.model = AutoModel(
    model=FUNASR_MODEL_NAME_OR_PATH,
    trust_remote_code=True,
    remote_code="model.py", # <--- 新增
    device=self.device
)
```

结果，`No module named 'model'` 的错误依旧。

### 解决方案：澄清 ModelScope 与 Hugging Face 的模型文件差异

本地模型 `/home/llm/model/iic/SenseVoiceSmall` 是从 **ModelScope** ([https://www.modelscope.cn/models/iic/SenseVoiceSmall/files](https://www.modelscope.cn/models/iic/SenseVoiceSmall/files)) 下载的，而非直接 clone Hugging Face 的仓库。通过 `ls -al /home/llm/model/iic/SenseVoiceSmall/` 查看本地文件，**发现确实没有 `model.py` 文件！**

这解释了为什么指定 `remote_code="model.py"` 依然报错。ModelScope 提供的模型包可能与 Hugging Face 仓库中的文件结构不完全一致，特别是对于这种依赖 `funasr` 特定加载方式的模型。

**最终的正确配置**：移除 `remote_code` 参数，但保留 `trust_remote_code=True`。

```python
# app/models/sensevoice_loader.py (最终正确配置)
self.model = AutoModel(
    model=FUNASR_MODEL_NAME_OR_PATH,
    trust_remote_code=True, # 保留，funasr 可能仍需此权限处理 ModelScope 模型
    # remote_code="model.py", # 移除，因为本地 ModelScope 版本无此文件
    device=self.device
)
```

这样修改后，服务启动时仍然会打印 `Loading remote code failed: model, No module named 'model'`，但 API 调用完全正常！

**原因分析**：`funasr` 在 `trust_remote_code=True` 时，会优先尝试加载自定义代码。如果本地模型路径（如从 ModelScope 下载的）没有 `model.py`，这个尝试会失败并打印日志。但随后，`funasr` 能够识别出这是一个有效的 ModelScope 模型路径，并转用其内部的标准加载流程成功加载模型。因此，该日志在这种情况下是良性的。

## 模型对比与选型思考

在解决问题的过程中，也探讨了 `FunAudioLLM/SenseVoiceSmall` 与其他主流 ASR 模型的对比：

* **OpenAI Whisper 系列** (如 `whisper-large-v3`):
  * **优势**: 极高的准确率，强大的多语言能力，庞大的社区。
  * **劣势**: 推理速度相对较慢（尤其大模型），不直接提供情感/事件检测。

* **Wav2Vec2 系列**:
  * **优势**: 自监督学习典范，大量特定语言微调模型。
  * **劣势**: 基础模型功能相对单一。

### **`SenseVoiceSmall` 的核心优势**

1. **高效推理**：其模型卡声称采用非自回归端到端框架，比 Whisper-Large 快15倍。这对于需要低延迟的应用至关重要。

2. **多任务集成**：内置 ASR、LID（语种识别）、SER（情感识别）、AED（事件检测）。如果应用场景需要这些附加信息，`SenseVoiceSmall` 提供了一站式解决方案。

3. **特定语言优化**：在中文、粤语等语言上表现突出。

### **结论**

没有绝对的“最好”，只有“最适合”。

* 若追求极致准确性和最广语言覆盖，且对延迟不敏感，Whisper 仍是首选。
* 若对**推理效率、集成的多任务语音理解（特别是情感/事件）或中文等特定场景有高要求**，`SenseVoiceSmall` 是一个极具竞争力的选择。

目前选择的 `SenseVoiceSmall`，尤其是在确认了其 ModelScope 版本能够顺畅运行后，对于我的项目目标来说是一个合适的起点。

## 当前状态与展望

目前，基于 `FunAudioLLM/SenseVoiceSmall` 和 FastAPI 的语音转录服务已成功搭建并能正确处理请求。

```bash
$ curl -X POST "http://<your_server_ip>:8888/asr_pure" -F "file=@test_audio.wav"
{"text":"太好了，那接下来咱们可以试试其他功能了。比如说你想测试一下语音合成的效果怎么样，或者是看看有没有什么新的语音处理功能出来啦。😔","status":"success","processing_time_ms":503.39...}
```

### **后续可优化的方向**

* **性能优化**：进一步测试并发处理能力，考虑多 worker 配置。
* **错误处理与日志**：完善更细致的错误捕获和日志记录。
* **功能扩展**：如果需要，可以利用 `SenseVoiceSmall` 的情感识别和事件检测能力。
* **VAD 集成**：对于长音频，考虑在 `funasr.AutoModel` 加载时集成 VAD (Voice Activity Detection) 功能，以实现自动分段处理，提升长音频处理的稳定性和效率。
* **异步处理与队列**：对于高并发场景，引入消息队列和异步任务处理。
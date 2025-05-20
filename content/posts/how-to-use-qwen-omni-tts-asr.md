+++
title = "如何使用Qwen2.5-Omni实现文本转语音(TTS)和语音转文本(ASR)"
date = 2025-05-20T20:24:06+08:00
draft = false
tags = ["Qwen", "TTS", "ASR"]
categories = ["LLM"]
+++

# 如何使用Qwen2.5-Omni实现文本转语音(TTS)和语音转文本(ASR)

## 项目概述

本项目基于Qwen2.5-Omni-7B模型，实现了两个核心功能：
1. 文本转语音（TTS）：将输入文本转换为自然流畅的语音
2. 语音转文本（ASR）：将语音文件转换为文本，支持标准ASR和纯ASR两种模式

项目地址：[https://github.com/jackypanster/qwen-omni](https://github.com/jackypanster/qwen-omni)

## 环境配置

推荐使用conda管理Python环境，确保依赖安装的稳定性：

```bash
# 创建并激活环境
conda create -n qwen-tts python=3.10
conda activate qwen-tts

# 安装PyTorch（GPU版本）
conda install pytorch=2.5.1 pytorch-cuda=12.1 -c pytorch -c nvidia
conda install torchvision torchaudio -c pytorch

# 安装其他依赖
conda install streamlit python-soundfile -c conda-forge
pip install git+https://github.com/huggingface/transformers@v4.51.3-Qwen2.5-Omni-preview
pip install qwen-omni-utils
```

## 核心功能实现

### 1. 文本转语音（TTS）

```python
def text_to_speech(text_input, output_audio_path="output/output.wav", speaker="Chelsie"):
    # 加载模型和处理器
    model = Qwen2_5OmniForConditionalGeneration.from_pretrained(
        model_path, 
        config=config, 
        torch_dtype="auto", 
        device_map="auto"
    )
    processor = Qwen2_5OmniProcessor.from_pretrained(model_path)
    
    # 构造对话
    conversation = [
        {"role": "system", "content": [{"type": "text", "text": "You are Qwen..."}]},
        {"role": "user", "content": [{"type": "text", "text": text_input}]}
    ]
    
    # 生成语音
    with torch.no_grad():
        text_ids, audio = model.generate(
            **inputs,
            speaker=speaker,
            do_sample=True,
            temperature=0.8,
            top_p=0.95,
            max_new_tokens=1024
        )
```

### 2. 语音转文本（ASR）

```python
def audio_to_text(audio_path: str) -> str:
    # 标准ASR模式
    conversation = [
        {"role": "system", "content": [{"type": "text", "text": "你是Qwen..."}]},
        {"role": "user", "content": [{"type": "audio", "audio": audio_path}]}
    ]
    
    # 生成文本
    with torch.no_grad():
        text_ids = model.generate(
            **inputs,
            do_sample=False,
            max_new_tokens=1024,
            return_audio=False
        )
```

## Web界面实现

使用Streamlit构建了简洁的Web界面：

```python
# 文本输入
text_input = st.text_area("请输入要合成的文本：", height=120, max_chars=200)

# 发音人选择
speaker = st.selectbox("请选择发音人：", ["Chelsie", "Ethan"], index=0)

# 生成按钮
if st.button("生成语音"):
    # 生成语音并播放
    audio_path = os.path.join(OUTPUT_DIR, f"tts_{uuid.uuid4().hex}.wav")
    text_to_speech(text_input, output_audio_path=audio_path, speaker=speaker)
    st.audio(audio_path, format="audio/wav")
```

## RESTful API实现

使用FastAPI构建了RESTful API接口：

```python
@app.post("/tts")
async def tts(request: TTSRequest):
    audio_filename = f"tts_{uuid.uuid4().hex}.wav"
    audio_path = os.path.join(OUTPUT_DIR, audio_filename)
    text_to_speech(request.text, audio_path, request.speaker)
    return {"audio_url": f"/output/{audio_filename}"}

@app.post("/asr")
async def asr(file: UploadFile = File(...)):
    # 处理上传的音频文件
    audio_path = os.path.join(OUTPUT_DIR, f"asr_{uuid.uuid4().hex}.wav")
    with open(audio_path, "wb") as buffer:
        shutil.copyfileobj(file.file, buffer)
    text = audio_to_text(audio_path)
    return {"text": text}
```

## 使用说明

1. 启动Web界面：
```bash
streamlit run app_text2audio.py
```

2. 启动API服务：
```bash
uvicorn fastapi_app:app --host 0.0.0.0 --port 8000
```

## 注意事项

1. 模型文件较大，建议提前下载并配置好模型路径
2. 使用conda安装依赖可以避免大多数环境问题
3. 音频文件会保存在output目录下
4. API接口支持文件上传和文本转写

## 后续优化方向

1. 支持更多发音人选项
2. 优化模型加载速度
3. 添加批量处理功能
4. 支持更多音频格式
5. 添加历史记录功能

## 参考资源

- [Qwen2.5-Omni-7B官方文档](https://huggingface.co/Qwen/Qwen2.5-Omni-7B)
- [Streamlit文档](https://docs.streamlit.io/)
- [FastAPI文档](https://fastapi.tiangolo.com/) 
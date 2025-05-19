+++
title = 'Qwen2.5-Omni-7B 文本转语音部署指南'
date = 2025-05-19T10:54:50+08:00
draft = false
tags = ["qwen", "omni"]
categories = ["LLM"]
+++

本脚本基于 Qwen2.5-Omni-7B 多模态模型实现文本转语音（TTS）功能，支持生成自然流畅的中文 / 英文语音，并提供两种语音类型（女性 “Chelsie”、男性 “Ethan”）。脚本可将输入文本转换为音频文件（.wav格式），适用于语音助手、内容创作、无障碍服务等场景。

安装依赖库
```bash
uv init
uv add git+https://github.com/huggingface/transformers@v4.51.3-Qwen2.5-Omni-preview
uv add accelerate
uv add qwen-omni-utils[decord]
uv add soundfile
uv add torchvision
uv sync
```

完整脚本代码（main_text2audio.py）
```python
import os
import soundfile as sf
import torch
from transformers import Qwen2_5OmniForConditionalGeneration, Qwen2_5OmniProcessor
from qwen_omni_utils import process_mm_info
from transformers import AutoConfig

def text_to_speech(
    text_input: str,
    output_audio_path: str = "output/test_audio.wav",
    speaker: str = "Chelsie",
    model_path: str = "/home/llm/model/qwen/Omni/"  # 改为本地路径或远程路径
):
    """
    文本转语音核心函数
    :param text_input: 输入文本（支持中文/英文）
    :param output_audio_path: 音频输出路径（含文件名）
    :param speaker: 语音类型（"Chelsie"女性/"Ethan"男性）
    :param model_path: 模型路径（本地/远程）
    """
    # 1. 加载模型配置（修复ROPE参数兼容性）
    config = AutoConfig.from_pretrained(model_path, local_files_only=True)
    if hasattr(config, "rope_scaling") and "mrope_section" in config.rope_scaling:
        config.rope_scaling.pop("mrope_section")
    
    # 2. 加载模型（支持GPU自动分配）
    model = Qwen2_5OmniForConditionalGeneration.from_pretrained(
        model_path,
        config=config,
        torch_dtype="auto",
        device_map="auto",
        local_files_only=(model_path != "Qwen/Qwen2.5-Omni-7B")
    )
    processor = Qwen2_5OmniProcessor.from_pretrained(model_path)
    
    # 3. 系统提示（必须包含语音生成能力声明）
    system_prompt = [
        {
            "role": "system",
            "content": [
                {"type": "text", "text": "You are Qwen, a virtual human developed by the Qwen Team, Alibaba Group, capable of perceiving auditory and visual inputs, as well as generating text and speech."}
            ]
        }
    ]
    
    # 4. 构建对话（纯文本输入）
    conversation = system_prompt + [
        {"role": "user", "content": [{"type": "text", "text": text_input}]}
    ]
    
    # 5. 处理输入数据
    text = processor.apply_chat_template(conversation, add_generation_prompt=True, tokenize=False)
    audios, images, videos = process_mm_info(conversation, use_audio_in_video=False)
    
    # 6. 生成语音
    inputs = processor(
        text=text, audio=audios, images=images, videos=videos,
        return_tensors="pt", padding=True, use_audio_in_video=False
    ).to(model.device, model.dtype)
    
    with torch.no_grad():
        text_ids, audio = model.generate(
            **inputs,
            speaker=speaker,
            do_sample=True,  # 启用采样模式以使用temperature/top_p
            temperature=0.8,  # 控制随机性（0.5-1.0较自然）
            top_p=0.95,       # 核采样参数
            max_new_tokens=1024,  # 控制语音时长（约15秒）
            use_audio_in_video=False
        )
    
    # 7. 保存结果
    os.makedirs(os.path.dirname(output_audio_path), exist_ok=True)
    sf.write(output_audio_path, audio.reshape(-1).cpu().numpy(), samplerate=24000)
    print(f"✅ 生成完成：{output_audio_path}")
    print(f"📄 生成文本：{processor.batch_decode(text_ids, skip_special_tokens=True)[0]}")

if __name__ == "__main__":
    # 示例输入（可替换为任意文本）
    input_text = "你好，这是Qwen2.5-Omni的文本转语音示例。祝你使用愉快！"
    
    # 调用函数（指定输出路径和语音类型）
    text_to_speech(
        input_text,
        output_audio_path="output/hello_qwen.wav",
        speaker="Chelsie"  # 可选"Ethan"
    )
```

运行脚本
```bash
uv run main.py
```
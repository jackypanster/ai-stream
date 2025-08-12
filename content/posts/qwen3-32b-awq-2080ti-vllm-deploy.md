+++
title = "Qwen3-32B AWQ 在 4×RTX 2080 Ti 上的极限部署实战"
date = 2025-07-16T14:00:00+08:00
draft = false
tags = ["Qwen3", "vLLM", "AWQ", "RTX 2080 Ti", "大模型部署"]
categories = ["AI部署", "大语言模型"]
+++

> 使用旧显卡也能跑 32B 大模型？本文手把手演示如何在 4×RTX 2080 Ti (共 88 GB 显存) 服务器上，通过 vLLM 0.8.5 + AWQ 量化，跑起 Qwen3-32B 并支持 200K+ tokens 超长上下文与高吞吐推理。全文记录了踩坑过程与参数权衡，希望给同样预算有限、硬件受限的工程师带来借鉴。

## 项目背景

- **主角**：Qwen3-32B-AWQ 量化模型 （≈ 18 GB）
- **目标**：在消费级 Turing 架构显卡（2080 Ti）上最大化利用显存与吞吐。
- **框架**：vLLM 0.8.5 (openai-compatible server)
- **取舍**：牺牲部分延迟 / 稳定性 → 换取 吞吐 + 上下文长度

## 硬件与系统环境

| 组件 | 规格 |
| --- | --- |
| GPU | 4 × RTX 2080 Ti, 22 GB each, Compute Capability 7.5 |
| CPU | ≥ 56 cores (vLLM 线程可吃满) |
| RAM | 512 GB |
| Storage | NVMe SSD 2 TB (模型 + KV 缓冲) |
| OS | Ubuntu 24.04 |
| Driver | NVIDIA 570.153.02 |
| CUDA | 12.8 |

### NVIDIA-SMI 基线信息

```bash
nvidia-smi
Wed Jul 16 13:27:17 2025
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 570.153.02             Driver Version: 570.153.02     CUDA Version: 12.8     |
+-----------------------------------------------------------------------------------------+
```

> 可以看到驱动与 CUDA 版本与上表一致，确认环境无偏差。

> 为什么 2080 Ti？ 二手市场价格友好，但 Flash-Attention-2 不支持，需要自己编译 flash-attn-1 或使用 XFormers。

## 快速部署步骤概览

1. 下载并解压 Qwen3-32B-AWQ 权重至 `/home/llm/model/qwen/Qwen3-32B-AWQ`。
2. 拉取官方 vLLM 镜像 `vllm/vllm-openai:v0.8.5`。
3. 按下文 run.sh 参数启动容器。

### 模型准备

```bash
mkdir -p /home/llm/model/qwen
# 省略 huggingface-cli 登录步骤
modelscope download --model Qwen/Qwen3-32B-AWQ --local_dir /home/llm/model/qwen/Qwen3-32B-AWQ --max-workers 32
```

### 启动脚本 run.sh

```bash
#!/usr/bin/env bash
docker run -d \
  --runtime=nvidia --gpus=all --name coder-32b \
  -v /home/llm/model/qwen/Qwen3-32B-AWQ:/model/Qwen3-32B-AWQ \
  -p 8888:8000 --cpuset-cpus 0-55 \
  --ulimit memlock=-1 --ulimit stack=67108864 --restart always --ipc=host \
  vllm/vllm-openai:v0.8.5 \
  --model /model/Qwen3-32B-AWQ --served-model-name coder \
  --tensor-parallel-size 4 \
  --quantization awq \
  --dtype auto \
  --chat-template /model/Qwen3-32B-AWQ/qwen3_nonthinking.jinja \
  --rope-scaling '{"rope_type":"yarn","factor":6.5,"original_max_position_embeddings":32768}' \
  --max-model-len 212992 \
  --max-num-batched-tokens 32768 \
  --gpu-memory-utilization 0.99 \
  --block-size 16 \
  --enable-prefix-caching \
  --max-num-seqs 1 \
  --enable-chunked-prefill \
  --enforce-eager \
  --enable-auto-tool-choice \
  --tool-call-parser hermes
```

## 关键运行参数拆解

| 参数 | 作用 / 调优思路 |
| --- | --- |
| `--tensor-parallel-size 4` | 4 卡切分模型参数，2080 Ti 单卡显存有限必须拆分。 |
| `--quantization awq` | 启用 AWQ 权重量化，显存≈再降 40%。 |
| `--max-model-len 212992` | 支持 212K tokens；超长上下文支持。 |
| `--max-num-batched-tokens 32768` | 单批次 tokens 上限。吞吐 / 显存 trade-off。 |
| `--gpu-memory-utilization 0.99` | 近乎吃满显存，谨慎调；留 0.01 作余量。 |
| `--block-size 16` | KV Cache 分块。块越小越灵活，管理开销稍增。 |
| `--enable-prefix-caching` | 高复用 prompt 命中率可>90%，显著提升长对话吞吐。 |
| `--enforce-eager` | 必须！避免Turing架构的CUDA graph编译错误。 |
| `--max-num-seqs 1` | 单用户模式，最大化上下文。 |

## Claude Code 集成要点

Claude Code CLI 要求模型具备：
1. **推理能力**：✅ Qwen3-32B 支持 Chain-of-Thought
2. **工具调用**：✅ 通过 `--enable-auto-tool-choice` 启用
3. **长上下文**：✅ 212K tokens 满足需求
4. **JSON 模式**：✅ 支持结构化输出

### Claude Code Router 配置

```json
{
  "providers": {
    "vllm": {
      "api_base_url": "http://localhost:8888/v1/chat/completions",
      "api_key": "empty",
      "models": ["coder"],
      "max_tokens": 8192,
      "temperature": 0.1,
      "timeout": 150,
      "transformer": {
        "openai": {
          "strip_special_tokens": true,
          "handle_system_prompt": true,
          "max_context_length": 212992
        }
      },
      "performance": {
        "concurrent_requests": 1,
        "batch_size": 32768,
        "enable_streaming": true,
        "connection_pool_size": 5
      }
    }
  },
  "router": {
    "default": "vllm,coder",
    "longContext": "vllm,coder",
    "contextWindow": {
      "vllm": 212992
    },
    "contextStrategy": {
      "maxTokens": 204800,
      "reserveTokens": 8192,
      "dynamicRouting": true
    }
  }
}
```

## 性能基准（2080Ti 魔改版）

| 指标 | 实测值 | 说明 |
|------|--------|------|
| **模型加载** | ~4.6GB/卡 | AWQ 量化效果显著 |
| **KV Cache** | ~12.9GB/卡 | 充足的上下文空间 |
| **首字延迟** | ~1.2秒 | XFormers 略慢于 FA2 |
| **生成速度** | ~80 tokens/s | 对编程够用 |
| **最大上下文** | 212,992 tokens | 约 850KB 文本 |
| **并发请求** | 1 | 单用户模式 |

## 常见问题与解决

### 1. Triton 编译错误
```
AssertionError: Input shapes should have M >= 16, N >= 16 and K >= 16
```
**解决**：添加 `--enforce-eager` 参数

### 2. FlashAttention 警告
```
Cannot use FlashAttention-2 backend for Volta and Turing GPUs
```
**正常**：自动使用 XFormers，性能影响约 10-15%

### 3. V0 引擎警告
```
Compute Capability < 8.0 is not supported by V1 Engine
```
**正常**：使用 V0 引擎，功能完整

## 优化建议

### 针对 2080Ti 的特殊优化

1. **显存管理**
   ```bash
   # 可以尝试更激进的显存利用
   --gpu-memory-utilization 0.995
   ```

2. **适当降低批处理**
   ```bash
   # Turing 架构下较小的批处理可能更稳定
   --max-num-batched-tokens 16384
   ```

3. **监控温度**
   ```bash
   # 2080Ti 发热较大，注意散热
   watch -n 1 'nvidia-smi | grep "RTX 2080" | awk "{print \$3, \$9}"'
   ```

## 总结与建议

使用旧世代显卡并不意味着放弃大模型。通过 vLLM + AWQ + Prefix Cache 等组合拳，4×2080 Ti 依旧能够支撑 Qwen3-32B 的 200K+ 超长上下文推理。

- **科研 / 测试 场景**：强烈推荐该方案，可用最低成本探索大模型推理极限。
- **生产 场景**：需谨慎评估崩溃概率与延迟，做好监控与自动降级。

### 后续方向

1. 迁移到 RTX 5000 Ada 等新卡，可解锁 Flash-Attn-2 与更高带宽。
2. 关注 vLLM 后续对 AWQ Kernel 的优化；升级 >=0.9 可能免去自己编译。
3. 尝试 TensorRT-LLM 自动并行拆分，获得额外 10~20% 性能。

---

*硬件限制激发创新，优化永无止境。*
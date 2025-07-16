---
title = "Qwen3-32B-AWQ vLLM 多卡 2080 Ti 极限部署实战"
date = 2025-07-16T13:19:48+08:00
draft = false
tags = ["vllm","qwen3","awq","gpu-deployment","2080ti"]
categories = ["Large-Language-Model","Deployment"]
---

> 使用旧显卡也能跑 32B 大模型？本文手把手演示如何在 **4×RTX 2080 Ti (共 88 GB 显存)** 服务器上，通过 vLLM 0.8.5 + AWQ 量化，跑起 **Qwen3-32B** 并支持 **32 K tokens** 超长上下文与高吞吐推理。全文记录了踩坑过程与参数权衡，希望给同样预算有限、硬件受限的工程师带来借鉴。

{{< toc >}}

## 1 项目背景

- 主角：`Qwen3-32B-AWQ` 量化模型  （≈ 18 GB）
- 目标：在消费级 **Turing** 架构显卡（2080 Ti）上最大化利用显存与吞吐。
- 框架：`vLLM 0.8.5` (openai-compatible server)
- 取舍：牺牲部分延迟 / 稳定性 → 换取 **吞吐 + 上下文长度**

## 2 硬件与系统环境

| 组件 | 规格 |
|------|------|
| GPU  | 4 × RTX 2080 Ti, 22 GB *each*, Compute Capability 7.5 |
| CPU  | ≥ 56 cores (vLLM 线程可吃满) |
| RAM  | 512 GB |
| Storage | NVMe SSD 2 TB (模型 + KV 缓冲) |
| OS | Ubuntu 24.04 |
| Driver | NVIDIA 570.153.02 |
| CUDA | 12.8 |

### 2.1 NVIDIA-SMI 基线信息

```bash
nvidia-smi
Wed Jul 16 13:27:17 2025
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 570.153.02             Driver Version: 570.153.02     CUDA Version: 12.8     |
+-----------------------------------------------------------------------------------------+
```

> 可以看到驱动与 CUDA 版本与上表一致，确认环境无偏差。

> **为什么 2080 Ti？** 二手市场价格友好，但 Flash-Attention-2 不支持，需要自己编译 flash-attn-1 或使用 XFormers。

## 3 快速部署步骤概览

1. 下载并解压 `Qwen3-32B-AWQ` 权重至 `/home/llm/model/qwen/Qwen3-32B-AWQ`。
2. （可选）编译 `flash-attn-1` 以替代原生 attention。
3. 拉取官方 vLLM 镜像 `vllm/vllm-openai:v0.8.5`。
4. 按下文 **run.sh** 参数启动容器。

下面拆解每一步的技术细节。

### 3.1 模型准备

```bash
mkdir -p /home/llm/model/qwen
# 省略 huggingface-cli 登录步骤
huggingface-cli download Qwen/Qwen3-32B-AWQ --local-dir /home/llm/model/qwen/Qwen3-32B-AWQ --local-dir-use-symlinks False
```

### 3.2 编译 Flash-Attention-1（2080 Ti 专用）

```bash
# CUDA 12.x + Python 3.12 示例
python3 -m pip install --upgrade pip
python3 -m pip install ninja packaging cmake
# 强制源码编译，确保生成 sm75 kernel
FLASH_ATTENTION_FORCE_BUILD=1 \
  python3 -m pip install flash-attn --no-build-isolation --no-binary :all:
```

> **容器用户请注意**：如果使用下文的官方 vLLM Docker 镜像，需在 _容器内部_ 或自建 Dockerfile 完成同样的 flash-attn-1 编译（或将已编译好的 wheel 复制进镜像）。宿主机安装的 Python 包不会被容器环境读取。

#### 3.2.1 无需重建大镜像的折中做法

| 做法 | 说明 | 额外体积 |
|------|------|-----------|
| **启动时临时 `--pip-install`** | vLLM ≥0.9 支持 `--pip` 参数，容器启动时即在线编译 `flash-attn` | 0（编译产物缓存于 volume） |
| **宿主机先编译 wheel** | `pip wheel flash-attn -w /tmp/wheels`，运行时挂载 `/tmp/wheels` 并 `pip install` | ~30-40 MB |
| **改用 XFormers** | 加 `--xformers`，性能略低于 flash-attn-1，但免编译 | 0 |
| **保持默认 attention** | 对吞吐要求一般的场景可接受 | 0 |

> 推荐顺序：临时 `--pip` > wheel 挂载 > XFormers > 默认 Attention。按业务对性能 & 简易度的权衡自行选择。


验证：
```bash
python3 - <<'PY'
import flash_attn, torch, platform
print('flash-attn', flash_attn.__version__, 'torch', torch.__version__, 'python', platform.python_version())
PY
```

### 3.3 启动脚本 `run.sh`

```bash
#!/usr/bin/env bash
docker run -d \
  --runtime=nvidia --gpus=all --name coder \
  -v /home/llm/model/qwen/Qwen3-32B-AWQ:/model/Qwen3-32B-AWQ \
  -p 8888:8000 --cpuset-cpus 0-55 \
  --ulimit memlock=-1 --ulimit stack=67108864 --restart always --ipc=host \
  vllm/vllm-openai:v0.8.5 \
  --model /model/Qwen3-32B-AWQ --served-model-name coder \
  --tensor-parallel-size 4 --quantization awq --dtype auto \
  --max-model-len 32768 --max-num-batched-tokens 32768 \
  --gpu-memory-utilization 0.96 \
  --block-size 16 \
  --enable-prefix-caching \
  --swap-space 64 \
  --max-num-seqs 64
```

> **容器 vs 本机**：直接裸跑亦可，核心参数完全相同。容器便于复现与快速重启。

## 4 关键运行参数拆解

| 参数 | 作用 / 调优思路 |
|------|-----------------|
| `--tensor-parallel-size 4` | 4 卡切分模型参数，2080 Ti 单卡显存有限必须拆分。 |
| `--quantization awq` | 启用 **AWQ** 权重量化，显存≈再降 40%。某些长文本场景下 FP16 仍更快，需实测。 |
| `--max-model-len 32768` | 支持 32 K tokens；大幅增加 KV Cache，需要配合 `--swap-space`。 |
| `--max-num-batched-tokens 32768` | 单批次 tokens 上限。吞吐 / 显存 trade-off。 |
| `--gpu-memory-utilization 0.96` | 近乎吃满显存，谨慎调；留 0.04 作余量。 |
| `--block-size 16` | KV Cache 分块。块越小越灵活，管理开销稍增。 |
| `--enable-prefix-caching` | 高复用 prompt 命中率可>90%，显著提升长对话吞吐。 |
| `--swap-space 64` | 允许 64 GB CPU RAM 作为 KV Cache 溢出。swap 大延迟高。 |
| `--max-num-seqs 64` | 控制并发序列数。越大吞吐高，长文本 OOM 风险也高。 |

## 5 API 调用范例

```bash
curl http://localhost:8888/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -d '{
    "model": "coder",
    "messages": [
      {"role": "system", "content": "你是一个聪明的 AI 助手。"},
      {"role": "user", "content": "请写一个 Python 冒泡排序。"}
    ],
    "max_tokens": 512,
    "temperature": 0.2
  }'
```

- `max_tokens` 建议 512 ~ 2048；极限 context 时过大易 **OOM**。
- `stream=true` 可获得流式输出；耗时更短，占用更低。

## 6 性能压榨技巧

1. **AWQ vs FP16**  
   - 某些推理阶段 AWQ kernel 尚未优化，🚀 结果 FP16 更快。实测二选一。
2. **Flash-Attn-1 / XFormers**  
   - 2080 Ti 无 **Flash-Attn-2**；编译 v1 或使用 XFormers 皆可。
3. **KV Cache & Swap**  
   - 监控 `gpu_kv_cache` 与 `swap_used` 两项；长文本易炸。
4. **多实例分卡**  
   - 把 4 卡拆成 2 × 2 卡实例，可提高 GPU 利用率 (不同业务负载)。
5. **自动降级**  
   - 在 API 层检测 OOM → 自动缩短上下文 or 调小并发，保证可用性。

## 7 常见问题速查

| 症状 | 解决方案 |
|------|-----------|
| **返回不完整/截断** | 增大 `max_tokens`；缩短输入；检查日志中 `context_window`。 |
| **CUDA OOM / 容器崩溃** | 降低 `max-model-len`、`max-num-batched-tokens`；增大 `swap-space`。 |
| **推理速度慢** | 确认 flash-attn-1 已启用；并发不要过高；尝试 FP16。 |
| **NCCL 死锁 / hang** | 加 `--disable-custom-all-reduce` 或升级 NCCL。 |

## 8 实战压测结果 (10 并发 · 32 K prompt)

| 指标 | 数值 |
|------|-------|
| Avg prompt throughput | **63 K tokens/s** |
| Avg generation throughput | **57 tokens/s** |
| 平均响应时间 | **5.63 s** |
| GPU KV Cache 占用 | 15 % |
| Prefix cache 命中率 | 94 % |
| 错误 / OOM | 0 |

> 高吞吐归功于：1) prefix caching 2) AWQ 量化 3) 近乎满显存利用。

### 结果解读

- **吞吐**：输入阶段 63K tokens/s，生成阶段 57 tokens/s，对 32B 模型非常可观。
- **资源**：GPU KV Cache 仅 15 %；系统还可上调并发 / 上下文。
- **稳定**：长时间压测无 OOM / pending；容器 restart=always 可兜底。

## 9 总结 & 建议

使用旧世代显卡并不意味着放弃大模型。通过 **vLLM + AWQ + Prefix Cache** 等组合拳，4×2080 Ti 依旧能够支撑 **Qwen3-32B** 的 32 K 超长上下文推理。

- **科研 / 测试** 场景：强烈推荐该方案，可用最低成本探索大模型推理极限。
- **生产** 场景：需谨慎评估崩溃概率与延迟，做好监控与自动降级。

⚙️ **后续方向**

1. 迁移到 **RTX 5000 Ada** 等新卡，可解锁 Flash-Attn-2 与更高带宽。
2. 关注 vLLM 后续对 AWQ Kernel 的优化；升级 >=0.9 可能免去自己编译。
3. 尝试 **TensorRT-LLM** 自动并行拆分，获得额外 10~20% 性能。


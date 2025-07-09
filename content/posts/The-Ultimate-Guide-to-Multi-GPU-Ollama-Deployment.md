+++
title = "从单卡瓶颈到四卡齐飞：一次完整的Ollama多GPU服务器性能优化实战"
date = "2025-07-09"
draft = false
tags = ["Ollama", "LLM", "性能优化", "Nginx", "Systemd", "多GPU", "AI部署"]
categories = ["技术实战"]
description = "记录如何将一台拥有4块RTX 2080 Ti的服务器，从最初的Ollama单点服务，逐步优化，最终搭建成一个高性能、高并发的负载均衡集群的全过程。"
+++

## 前言

服务器：4块NVIDIA RTX 2080 Ti，每张拥有22GB显存，总计88GB的VRAM。目标：让Ollama在这台机器上火力全开，为大语言模型提供强劲的推理服务。

然而，最初的想法——“如何用光所有显存？”——很快被证明是一个误区。真正的目标应该是：**如何最高效地利用所有GPU资源，实现最大的吞吐量和最低的延迟？**

本文将完整记录从最初的配置探索，到发现并解决性能瓶颈，再到最终搭建起一个健壮的4-GPU负载均衡服务集群的全过程。这不仅是一份操作指南，更是一次充满洞见的性能优化之旅。

## 第一章：初探配置，单卡运行的“真相”

首先，确认硬件已被系统正确识别。

```bash
$ nvidia-smi -L
GPU 0: NVIDIA GeForce RTX 2080 Ti (UUID: GPU-b5040762-a75e-f78a-87eb-d288e4725f64)
GPU 1: NVIDIA GeForce RTX 2080 Ti (UUID: GPU-651b0fe5-cdad-1851-df5d-e122f85ff10c)
GPU 2: NVIDIA GeForce RTX 2080 Ti (UUID: GPU-7674114f-1e22-374d-982f-7446da0ce35f)
GPU 3: NVIDIA GeForce RTX 2080 Ti (UUID: GPU-04eaff5d-28e2-94f3-3dd6-c0985dfdad24)
```

四张卡都在，一切正常。通过`systemd`来管理Ollama服务，并让它能“看到”所有GPU。

```bash
sudo systemctl edit ollama.service
```

在配置文件中，设置了以下关键环境变量：

```ini
[Service]
Environment="OLLAMA_HOST=0.0.0.0:9000"
Environment="CUDA_VISIBLE_DEVICES=0,1,2,3"
# 其他性能相关配置...
```

重启服务后，运行了一个约7.5GB的`gemma3n`模型。通过`nvidia-smi`观察，一个关键现象出现了：只有一块GPU的显存被占用了！

**结论一：Ollama足够智能。** 对于远小于单卡显存的模型，它会优先在单张卡内完成所有计算，以避免跨GPU通信带来的性能开销。这是最高效的做法，也打破了“必须用光所有显存”的迷思。

## 第二章：压力测试，揭开软件瓶颈的面纱

既然是单卡在工作，那它的性能极限在哪里？编写了一个Python异步压测脚本，模拟多个并发用户。

> **压测脚本 `benchmark.py` (核心逻辑)**:
> 使用`asyncio`和`aiohttp`库，创建多个并发的worker，向Ollama的`/api/generate`流式端点发送请求，并收集成功率、首字响应时间（TTFT）和吞吐量（TPS）等指标。

对当前配置（`OLLAMA_NUM_PARALLEL=4`）进行了测试。

| 并发用户数 | 平均首字响应 (TTFT) | 整体服务吞吐量 (TPS) |
| :----------- | :-------------------- | :----------------------- |
| 5            | 4.4 秒                | 89.06 tokens/秒          |
| 10           | **13.5 秒**             | **105.71 tokens/秒**     |
| 20           | **36.7 秒**             | **104.73 tokens/秒**     |

**结果触目惊心！**
1.  **性能拐点**：当并发数从10增加到20时，**总吞吐量不再增长**，稳定在约105 TPS。这是服务器达到性能上限的明确信号。
2.  **延迟雪崩**：与此同时，平均首字响应时间从13.5秒**灾难性地飙升至36.7秒**！这意味着用户体验已经差到无法接受。

**瓶颈分析**：硬件（单张2080 Ti）显然没有跑满，问题出在哪里？答案就在Ollama的配置里：`OLLAMA_NUM_PARALLEL=4`。这个参数限制了Ollama服务在同一时刻最多**并行处理4个请求**。当20个请求涌入时，有16个都在排队等待，导致了巨大的延迟。

我们找到了第一个真正的瓶颈：**软件配置限制**。

## 第三章：参数调优，释放单卡全部潜力

我们立即将瓶颈参数调整为一个更高的值。

```bash
# 在 systemd 配置文件中修改
Environment="OLLAMA_NUM_PARALLEL=16"
```

重启服务后，用同样的场景再次压测，结果令人振奋：

| 并发数 | 性能指标 | **优化前** (Parallel=4) | **优化后** (Parallel=16) | **性能提升幅度** |
| :------- | :--------------- | :------------------------ | :------------------------- | :--------------------------------- |
| **20**   | **吞吐量 (TPS)** | 104.73 tokens/秒          | **204.70 tokens/秒**       | **+ 95.5% (几乎翻倍)**             |
|          | **响应时间 (TTFT)**  | 36.7 秒                 | **4.8 秒**                 | **↓ 86.8% (速度提升7.5倍)**      |

**结论二：一次教科书式的成功优化。**
通过简单地调整一个参数，我们**将单卡的吞吐能力翻了一番，同时将高并发下的延迟降低了87%**。这证明了性能瓶颈已经成功地从软件队列转移到了更底层的硬件——即这块2080 Ti的原始计算能力。

## 第四章：终极形态，构建4-GPU服务集群

单卡性能已优化到极限，但还有三张GPU在“旁观”。最佳方案是：**为每张GPU部署一个独立的Ollama实例，并用Nginx实现负载均衡。**

### 1. 使用 `systemd` 模板单元

为了优雅地管理4个服务，我们使用`systemd`的模板功能，创建`ollama@.service`文件。

```bash
# 创建模板文件 /etc/systemd/system/ollama@.service
sudo nano /etc/systemd/system/ollama@.service
```

```ini
[Unit]
Description=Ollama Service Instance for GPU %i
After=network-online.target

[Service]
# 使用 %i 动态计算端口号，并绑定到对应GPU
ExecStart=/bin/bash -c 'OLLAMA_HOST=0.0.0.0:$(expr 9000 + %i) /usr/local/bin/ollama serve'
User=ollama
Group=ollama
Restart=always
RestartSec=3

Environment="CUDA_VISIBLE_DEVICES=%i"
Environment="OLLAMA_NUM_PARALLEL=16" # 每个实例都具备高并行处理能力
# ... 其他配置
[Install]
WantedBy=multi-user.target
```

然后用一个循环启动并启用所有实例：

```bash
# 先停用旧服务
sudo systemctl stop ollama.service
sudo systemctl disable ollama.service

# 启动模板实例
sudo systemctl daemon-reload
for i in {0..3}; do sudo systemctl enable --now ollama@$i.service; done
```

### 2. 配置 Nginx 负载均衡

让Nginx监听一个统一的入口端口`9999`，并将流量轮询分发给后端的4个Ollama实例。

```bash
# 在 /etc/nginx/conf.d/ 中创建配置文件
sudo nano /etc/nginx/conf.d/ollama-cluster.conf
```

```nginx
# 定义Ollama后端服务器集群
upstream ollama_backend {
    server localhost:9000;
    server localhost:9001;
    server localhost:9002;
    server localhost:9003;
}

server {
    # 监听统一入口端口
    listen 9999;
    server_name _;

    location / {
        proxy_pass http://ollama_backend;
        # 针对流式API的优化，关闭缓冲，增加超时
        proxy_read_timeout 3600s;
        proxy_buffering off;
        # 其他必要的代理头部设置...
    }
}
```

测试并重启Nginx后，我们的服务集群就搭建完成了。

## 第五章：集群压测，见证猛兽咆哮

万事俱备，我们对最终的负载均衡入口 `http://localhost:9999` 发起了最后的总攻（20并发）。

| 性能指标 | **单GPU优化** (c=20) | **4-GPU集群** (c=20) | **性能提升幅度** |
| :--------------- | :--------------------- | :----------------------- | :--------------------------------- |
| **吞吐量 (TPS)** | 204.70 tokens/秒       | **366.01 tokens/秒**     | **+ 78.8%**                        |
| **响应时间 (TTFT)**  | 4.8 秒                 | **0.75 秒**              | **↓ 84.5% (速度提升6.4倍)**      |

**最终结论：巨大成功！**
1.  **吞吐量**：集群的总吞吐能力相比优化后的单卡，**再次提升了近80%**。这证明了负载均衡架构的有效性。
2.  **响应延迟**：**TTFT从4.8秒骤降至0.75秒**，几乎实现了瞬时响应。这对于任何交互式应用都是决定性的体验提升。

成功地将一台拥有4块GPU的强大服务器，从一个单点服务演进为一个健壮、高性能、高并发的AI模型服务集群。它已经为承载生产级的应用请求做好了充分的准备。

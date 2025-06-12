# 大模型（LLM）

## 推理（Inference）

### 核心加速组件(Core Acceleration)

- 硬件加速（hardware）

	- NVLink

	- NVSwitch

	- DGX

	- Cerebras

	- Groq

	- Sambonova

	- Intel Gaudi

- 软件加速（software）

  > 下面这张“软件加速”清单把 推理端最常用、社区最活跃、且对吞吐-延迟指标提升显著 的技术都囊括进来了。为了便于在思维导图里落点，我按 Attention 内核 → KV 缓存管理 → 批处理/并行 → 解码加速 → 端到端推理框架五大块分组；每项都给出一句定位，方便后期补充细节或放外链。
  > 
  > 
  > 1 · 高效 Attention 内核
  > 
  > 技术作用典型实现 / 说明FlashAttention-2 | 多阶段 tiling + 寄存器/SMEM fusion，单 GPU 吞吐提升 2-3× | 官方 CUDA 实现；PyTorch 2.1 起内置 (github.com)
  > Triton Fused Attention | 复用 Triton JIT 写的 FlashAttention-v2 重构版本，易于自定义 | OpenAI Triton 教程示例 (triton-lang.org)
  > xFormers Block-Sparse / Mem-Efficient Attention | 稀疏或块稀疏布局降低 O(L²) 内存，适合长序列 | Meta xFormers 库 (github.com)
  > 
  > 2 · KV 缓存与显存管理
  > 
  > 技术作用典型实现 / 说明Paged KV Cache | 把 KV 缓存分页到虚拟地址，GPU 显存碎片率 < 3 % | vLLM、TensorRT-LLM 均内置 (docs.vllm.ai, github.com)
  > KV Cache Quantization (INT4/FP4) | 对缓存权重再量化，最高节省 50 % 显存 | vLLM GPTQ/AWQ 支持 (docs.vllm.ai)
  > Grouped-Query / Multi-Query Attention (GQA/MQA) | 把多头 K,V 合并，KV 缓存尺寸按 √H 缩小 | TensorRT-LLM 示例模型采用 GQA (github.com)
  > 
  > 3 · 批处理与并行
  > 
  > 技术作用典型实现 / 说明Continuous / In-Flight Batching | 不同请求自动拼批，GPU 利用率接近训练阶段 | vLLM、TensorRT-LLM “Inflight Batching” (docs.vllm.ai, github.com)
  > DeepSpeed-Inference Transformer Kernel | ZeRO-Offload + 自定义 FP16/INT8 核，跨 GPU 并行 | DeepSpeed 官方教程 (deepspeed.ai)
  > cuDNN 9 Transformer Ops | NVIDIA 在 cuDNN 级别整合 Transformer fused ops，框架无感提速 | NVIDIA 开发者博客 (developer.nvidia.com)
  > 
  > 4 · 解码加速算法
  > 
  > 技术作用典型实现 / 说明Speculative Decoding (树形 / 自回归) | 先用小模型猜测 n token，再批量校验，大幅降低解码轮次 | Staged-/Self-Speculative 论文 & Medium 教程 (arxiv.org, medium.com)
  > Chunked Prefill / Prefill Pipeline | 把长上下文预填拆块流水线化，消除序列长度抖动 | vLLM “chunked prefill” (docs.vllm.ai)
  > 
  > 5 · 端到端推理框架
  > 
  > 框架看点适用场景TensorRT-LLM | 最全的 kernel & quant 路线图，FP8/FP4/INT4/GQA，全 GPU 族 | 高吞吐、低延时线上推理 (github.com)
  > NVIDIA FasterTransformer | C++ 模板库，易嵌入 Triton Server，支持多模型实例化 | 服务端离线/在线混合负载 (github.com, developer.nvidia.com)
  > DeepSpeed-Inference | ZeRO-3 + 动态张量并行，超大模型单节点推理 | 内部私有化环境 (deepspeed.ai, github.com)
  > Hugging Face TGI | Rust + Python，AD-HOC 批处理 + 多后端 (vLLM/GPTQ) | 快速上云 / 混合云部署 (developer.nvidia.com)
  > vLLM | PagedAttention + 连续批 + 多量化，社区迭代极快 | 通用 GPU 推理、实验验证 (docs.vllm.ai, medium.com)
  > 
  > 如何在思维导图中落点
  > 
  > 在“推理 → 软件加速”节点下 建立五个二级分支（Attention 内核、KV 缓存、批处理、解码算法、推理框架）。
  > 
  > 将上表条目作为三级节点；可在备注里补充关键指标（如吞吐提升倍数、开源链接）。
  > 
  > 若想再细分，可按「GPU 平台」「是否需要自编译」「量化支持」做第四级标签，后续随项目实践动态更新。
  > 
  > ⚡ 提示：FlashAttention-2 + Paged KV Cache + 连续批处理 = 当前最容易落地、性价比最高的组合；若 GPU 型号是 Ampere / Hopper，优先评估 TensorRT-LLM 或 vLLM 的最新版集成。
  > 
	- Paged Attention

	- Chunked Prefill

	- Highly effective attention kernel

		- flash attention

	- Speculative Decoding

	- GQA/MQA

### 主流框架（framework）

- vLLM

- SGLang

- TensorRT-LLM

- Hugging Face TGI

### 核心功能（core function）

- 流式输出，Streaming

- Tool/Function Calling

	- MCP

- Guardrails - especially Gemini API

- Rate Limit

- Sampling parameter: temperature, top_p, top_k

- Smart Routing

- Structured Decoding

## 微调（Finetune）

### SFT

- Full paramter

- Lora/QLora

### RL

- DPO

- PPO

- GRPO

### 主流框架（Framework）

- LlamaFactory

- Unsloth

## 评估（Evaluation）

### Encoding Similarity

### Fuzzy Match

### Exact Match

### LLM As A Judge

## 应用场景（Adoption Scenario）

### Classification

### Extraction（OCR）

### Summarization

### Chatbot

### Machine Translation

### Knowledge Base

### Agent

### Code Generation

### Content Generation

## 主流模式（Major Pattern）

### Prompt Tunning

- Zero Shot

- Few Shot

- Chain-of-Thought (CoT)

- ReAct (Reason + Act)

- Auto/Mega Prompt

### RAG

- LlamaIndex

- LangChain

- Graph RAG

### AI Agent

### Agentic AI

- A2A

## 基座模型（Base Model）

### Llama family

### Mistral family

### Gemma family

### Qwen

### 豆包，混元，盘古，等等

## 原理（Principal）

### Transformer: Attention is all you need

### Context Window

### Decoder Only

### MoE

## 多模态（multimodal）

### Image to text

### Text to image

### Text to video

## 监测（monitoring）

### Weights & Bias

### LangGraph

### LangSmith


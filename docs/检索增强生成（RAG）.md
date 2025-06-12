# 检索增强生成（RAG）

## 基础流程

### 切片（Chunking）

- Character Based

- Sentence Window Based

- Markdown/Other Structured Document Based

- Semantic Chunking

### 向量化（Embedding）

- Embedding Input Maxium Length: max_position_embeddings

### 检索（Retrieval）

- Embedding DB

	- Milvus

	- Pgvector

	- Pinecone

### 增强（Augmented）

- RAG Query Template Population

### 生成（Generation）

- Common Hyperparameter Tuning

## 框架

### LlamaIndex

### Langchain

### MSFT Graph RAG

## 优化

### Reranker

### Customized Embedding

- Multilingual Support

- Asymmetry Embedding Model

- Embedding Model Finetune

### Retrieval Optimization

- Sparse Retrieval: BM25

- Dense Retrieval: Similarity Score

- Hybrid Retrieval: Sparse + Dense

- Query Expansion

- Graph Based

- Meta Data Extraction/Filtering

## 分类

### Normal RAG

### Agentic RAG

### Graph RAG

## 评估

### 常见指标

- 检索

	- Hit Rate

	- Context Relevance

- 生成

	- Groundness/Faithfulness

	  > 检查生成内容是否确实源于提供的上下文，避免 hallucination
	  > 
	- Relevance/Completeness/Correctness

	  > 衡量答案是否准确相关、完整覆盖问题、内容真实可靠
	  > 
	- Hallucination Rate

	- Accuracy / Precision / Recall / F1

### 框架

- TruLens

- Ragas

- DeepEval

## 部署

### Cloud

### On-prime


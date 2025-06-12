# 检索增强生成（RAG）

## 基础流程（Basic Flow）

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

## 框架（Framework）

### LlamaIndex

### Langchain

### MSFT Graph RAG

## 优化（Optimization）

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

## 分类（Type）

### Normal RAG

### Agentic RAG

### Graph RAG

## 评估（Evaluation）

### 常见指标（Metrics）

- 检索（Retrieval）

	- Hit Rate

	- Context Relevance

- 生成 （Generation）

	- Groundness/Faithfulness

	  > 检查生成内容是否确实源于提供的上下文，避免 hallucination
	  > 
	- Relevance/Completeness/Correctness

	  > 衡量答案是否准确相关、完整覆盖问题、内容真实可靠
	  > 
	- Hallucination Rate

	- Accuracy / Precision / Recall / F1

### 框架（Framework）

- TruLens

- Ragas

- DeepEval

## 部署（Deployment）

### Cloud

### On-prime


---
title: 'Cloudflare AI Search 快速创建：用 R2 + Workers AI 搭托管式知识库问答'
published: 2026-09-22
description: '在 Cloudflare 控制台从零创建 AI Search：选 R2 数据源、配 AI Gateway、选 embedding 与 rerank 模型，几步搭好托管式 RAG 搜索。'
tags: [Cloudflare, AI Search, RAG, R2, Workers AI]
category: AI 实践
slug: cloudflare-ai-search-create
---

# Cloudflare AI Search 快速创建：用 R2 + Workers AI 搭托管式知识库问答

**Cloudflare AI Search** 是 Cloudflare 提供的托管式 **RAG**（检索增强生成）服务：你把文档放进知识库，它自动建立索引，之后用一句自然语言提问，它就检索相关内容并返回带出处的答案。和自建 RAG 相比，最大的区别是它不用你自己搭向量数据库、不用管理 GPU 服务器，而是跑在 Cloudflare 全球网络上，几步就能搭出一个可用于生产的智能搜索。

## 一、开始前的两项准备

创建实例前，先在 Cloudflare 控制台准备好两样东西：

1. **R2 存储桶**：R2 是 Cloudflare 的对象存储。把要纳入知识库的文件放进一个 R2 桶。AI Search 支持纯文本（TXT、JSON、JSONL 等）和富文本（PDF、Word 等），富文本会自动转换成 Markdown 再索引。
2. **AI Gateway**：它是调用大模型的网关。整个流程里的文本向量化（embedding）、查询重写、答案生成，都要通过 AI Gateway 去调用 **Workers AI**（Cloudflare 内置的推理模型服务）里的模型。在控制台新建一个 Gateway，起个名字，其余保持默认即可。

> 除了 R2，AI Search 也支持把"某个网站"作为数据源，它会自动爬取整站页面；本文以 R2 为例。

## 二、创建 AI Search 的步骤

登录 Cloudflare 控制台，在左侧菜单进入 **AI Search**，点击 **Create / Get started**，按向导一路配置：

### 1. 选数据源

选择 R2 作为数据源，再选中你准备好的那个桶。**Path prefix（路径前缀）**用来限定只索引桶里的某个子目录；如果文件就放在桶根目录，这一项留空即可。

### 2. 绑定 AI Gateway

选择上一步创建的那个 Gateway，后续所有模型调用都走它。

### 3. 选 embedding 模型

**embedding（嵌入）模型**负责把文本转成向量，是语义检索的基础。Cloudflare Workers AI 内置了多种模型，直接选一个默认的中文 embedding 模型即可，无需自己处理数据转换。

同时要设置**文本切分（chunking）**：

- AI Search 默认采用**递归式切分**，优先在段落、句子的自然边界把长文档拆成小块，仍然过大才继续细分，以保证检索质量。
- **overlap（重叠）**让相邻两个小块之间保留一段重叠文本，避免关键句正好被切断，使检索命中的上下文更完整。

### 4. 选 rerank（重排序）模型

选一个模型用于**重排序**：系统先做一次粗检索，再用第二个模型对候选结果重新打分、排序，最后只把最相关的几段交给生成环节，从而提升答案质量。

### 5. 结果数量与相似度阈值

设置"最多返回多少条结果"和**相似度阈值**（低于多少相似度的段落直接丢弃）。新手保持默认即可。

### 6. 语义缓存

开启后，对语义相近的提问直接命中 Cloudflare 的缓存结果，而不是每次都重新生成——相似问题能复用上次的结果，既省钱又更快。保持默认开启即可。

### 7. 命名并授权

给实例起个名字，然后配置 **Service API Token（服务令牌）**。这个令牌的作用是授权 AI Search 访问和配置你 Cloudflare 账户下的相关资源，比如 R2、Vectorize（Cloudflare 的向量库）和 Workers AI；没有它，AI Search 既不能索引数据，也无法响应查询。可以新建一个令牌，也可以复用现有令牌。

最后点击 **Create**，实例就创建完成了。

## 三、这套组合为什么省事

把上面几步拆开看，其实就是把自建 RAG 通常要自己拼的四块东西托管化了：

| 自建 RAG 要自己做的事 | 在 AI Search 里由谁承担 |
|---|---|
| 存文件 | R2 对象存储 |
| 文本转向量 | Workers AI 的 embedding 模型 |
| 向量检索/存储 | Cloudflare 托管（含 Vectorize） |
| 调用模型做查询重写与答案生成 | AI Gateway + Workers AI |

你只需要决定"用哪个数据源、选哪个模型、切分和阈值怎么设"，底层的算力和数据库都由 Cloudflare 维护。

## 四、小结

创建一个可用的 AI Search，核心就是：**准备 R2 桶和 AI Gateway → 选 R2 数据源 → 绑定 Gateway → 选 embedding 与 rerank 模型、设好切分重叠 → 配置服务令牌 → 创建**。由于 Cloudflare 的模型列表、控制台选项和免费/计费政策会持续更新，具体可用模型、参数默认值与配额以官方当前文档为准。

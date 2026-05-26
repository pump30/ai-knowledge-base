# Tokenization (分词)

## Summary
Tokenizer 是 LLM 处理流程的第一步，把原始文本切分为 token IDs。**词表内容是训练出来的**（从语料中统计高频模式），但**算法和超参数是人设计的**。Tokenizer 训练独立于模型训练，训练完后冻结，决定了模型能"看到"什么粒度的信息。

## Key Points

### 三层区分（核心）
| 层级 | 例子 | 谁决定 |
|------|------|--------|
| 算法 | BPE / WordPiece / SentencePiece / Unigram | 人设计 |
| 超参数 | 词表大小、特殊 token | 人配置 |
| 词表内容 | 具体哪些 subword 进词表 | **训练出来的** |

### Tokenizer 训练 ≠ 模型训练
两个独立训练过程：
1. **Tokenizer 训练**（几小时-几天）：输入语料 → 输出 vocab.json + merges.txt
2. **模型训练**（几个月）：用 tokenizer 切好的 token IDs 训练模型权重

Tokenizer 训练完后**冻结**，模型训练期间不再改动。

## 主流算法对比

| 算法 | 思路 | 代表模型 |
|------|------|---------|
| **BPE** (Byte Pair Encoding) | 自底向上迭代合并高频字符对 | GPT, Llama, Claude |
| **WordPiece** | 类似 BPE，用似然而非频率合并 | BERT, Sentence Transformers |
| **SentencePiece** | 不依赖空格预切分，适合 CJK | Llama, T5, Gemma |
| **Unigram** | 自顶向下，从大词表删除 | T5, ALBERT |

**生动比喻**：BPE 像拼乐高（从单块组合）；Unigram 像雕大理石（从整块削减）。

## BPE 训练流程
```
初始: 词表 = 所有单字符
迭代: 找出现频率最高的相邻字符对 → 合并成新 token → 加入词表
重复: 直到词表达到目标大小（如 50000）

例：
  'l'+'o' → 'lo' → 'low' → 'lower' → ...
```

产物：
- `vocab.json` — token 到 ID 的映射
- `merges.txt` — 合并规则

## 为什么 BPE 是主流
- **无 OOV**：最坏退化到字节级，任何字符都能编码
- **平衡词表大小和序列长度**：常用词整体编码，罕见词拆分
- **训练快**

## 不同模型的 Tokenizer
| 模型 | 算法 | 词表大小 |
|------|------|---------|
| GPT-2/3 | BPE | 50,257 |
| GPT-4 | BPE (cl100k_base) | 100,277 |
| BERT | WordPiece | 30,522 |
| Llama 2 | SentencePiece (BPE) | 32,000 |
| Llama 3 | TikToken (BPE) | 128,000 |

Llama 3 词表从 32K → 128K，主要为更好编码非英语和代码。

## 中文/CJK 的特殊性
英文有空格天然分词，中文没有：
- 老式：先用 jieba 等工具预分词
- 现代：**SentencePiece 直接在原始字节上训练**，无需预分词

Llama 3 切中文："北京" 可能是 1 个 token，"天安门" 可能 1 个或 3 个——**完全看训练语料频率**。

## 关键洞察
1. **Tokenizer 是模型的"输入语言"**，决定能看到什么粒度
2. **训练数据决定词表偏向**——英文为主的 tokenizer 切中文会碎（每字 2-3 token）
3. **换 tokenizer = 换模型**——所有 embedding 会失效
4. **Tokenizer 是 LLM 隐形瓶颈**——切得差，模型再大也学不到精细信息

## Glitch Tokens 现象
GPT 词表里有些几乎没用过的 token（如 " SolidGoldMagikarp"），来自训练语料中被遗忘的论坛用户名。这些 token 存在但训练不充分，输入会让模型行为异常——是 tokenizer 训练"误收纳"的副作用。

## 分词的实际影响
- **Token 计费**：API 按 token 计费，中文 token 数通常是字数的 1-2 倍
- **上下文窗口限制**：以 token 计算，不是字符
- **检索质量**：分词不一致会影响 embedding 相似度（参见 [[rag-retrieval-augmented-generation]]）
- **Emoji / 数值 / 特殊字符**：分词器对这些处理差异很大

## Sources
- [[2026-01-09_retrieval-optimization-tokenization-vector-quantization]]
- [[2026-05-26_transformer-training-conversation]]

## Related
- [[transformer-architecture]]
- [[embedding-models]]
- [[next-token-prediction]]
- [[encoder-only-vs-decoder-only]]

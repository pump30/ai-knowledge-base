# RAG vs Fine-tuning

## Summary
RAG 通过外部检索提供知识，不改变模型本身；Fine-tuning 修改模型权重使其内化知识。两者是互补方案，RAG 适合频繁变化的知识，Fine-tuning 适合稳定的行为/风格调整。

## Comparison Table

| Dimension | RAG | Fine-tuning |
|-----------|-----|-------------|
| 知识更新 | 实时（更新文档即可） | 需要重新训练 |
| 成本 | 低（检索基础设施） | 高（训练 + GPU） |
| 可追溯性 | 高（可以引用来源） | 低（知识融入权重） |
| 幻觉控制 | 较好（有 Groundedness 检查） | 较难保证 |
| 适用场景 | 知识密集、频繁更新 | 行为/风格/格式调整 |
| 组合使用 | ✅ 可以 Fine-tune + RAG | ✅ 可以 RAG + Fine-tune |

## When to Use Which
- **RAG**：企业知识库、FAQ、文档问答、需要引用来源
- **Fine-tuning**：特定领域语言风格、输出格式统一、特殊任务能力
- **Both**：Fine-tune 提升检索/回答质量 + RAG 提供最新知识

## Sources
- [[building-evaluating-advanced-rag]]
- [[embedding-models-architecture-implementation]]

## Related
- [[rag-retrieval-augmented-generation]]
- [[embedding-models]]

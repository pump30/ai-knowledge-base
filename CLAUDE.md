# AI Knowledge Base - AI/LLM Engineering

> ## ⚠️ HIGHEST PRIORITY — NO EXCEPTIONS
>
> **任何用户提问，必须先读 `vki/index.md`，无例外。**
>
> 不管问题看起来多简单、多通用、多像"常识"，在回答前必须：
> 1. 先用 Read 工具读 `vki/index.md`
> 2. 如果有相关 VKI 页面，深入阅读后再回答
> 3. 回答时优先使用知识库内容，再补充外部知识
>
> 不允许的理由（这些不是借口）：
> - ❌ "这是个简单问题，我直接知道答案"
> - ❌ "知识库大概率是空的，读了也白读"
> - ❌ "为了效率，跳过这步"
> - ❌ "通用概念问题不需要查知识库"
>
> 这是这个仓库存在的根本目的——让 AI 从用户的知识网络中回答，而不是从自己的训练数据中回答。绕过这一步等于让仓库失去意义。

## Repository Purpose

This is a personal knowledge base focused on **AI/LLM Engineering**, including:
- AI Agents, Harness Engineering
- Prompt Engineering, RAG
- Model training, fine-tuning
- AI infrastructure, deployment
- LLM application development

## Structure

```
ai-knowledge-base/
├── CLAUDE.md              # This file - rules & workflows for AI
├── sources/               # Raw materials (human-added, never modified)
├── vki/                   # AI-maintained knowledge network
│   ├── index.md           # Master index of all VKI pages
│   ├── concepts/          # Concept pages (abstract ideas, methods)
│   ├── entities/          # Entity pages (tools, people, orgs, models)
│   ├── comparisons/       # Comparison pages (X vs Y)
│   └── articles/          # AI-written summary articles
└── assets/                # Images, diagrams
```

## Layer Rules

### Layer 1: Sources (Human Only)
- Only humans add files here
- Files are NEVER modified after creation
- Formats: markdown, txt, pdf links
- Naming: `YYYY-MM-DD_descriptive-title.md`

### Layer 2: VKI Network (AI Only)
- AI creates, updates, and maintains all files here
- All pages are structured Markdown
- Cross-references use `[[page-name]]` wiki-link syntax
- Every page must link back to its source(s)

### Layer 3: CLAUDE.md (Human + AI)
- Humans set high-level rules
- AI may suggest improvements

## VKI Page Conventions

### Concept Pages (`vki/concepts/`)
```markdown
# Concept Name

## Summary
One-paragraph explanation.

## Key Points
- ...

## Sources
- [[source-file-name]]

## Related
- [[other-concept]]
```

### Entity Pages (`vki/entities/`)
```markdown
# Entity Name

## Type
Tool / Person / Organization / Model

## Summary
What it is and why it matters.

## Key Facts
- ...

## Sources
- [[source-file-name]]

## Related
- [[other-entity]]
```

### Comparison Pages (`vki/comparisons/`)
```markdown
# X vs Y

## Summary
Core difference in one sentence.

## Comparison Table
| Dimension | X | Y |
|-----------|---|---|
| ... | ... | ... |

## When to Use Which
- ...

## Sources
- [[source-file-name]]
```

### Index (`vki/index.md`)
Master index listing all pages with:
- Page name
- Type (concept/entity/comparison/article)
- Date created/updated
- Brief one-line description
- Source references

## AI Workflows

### 1. Ingest (摄入)
When user says "ingest this" or adds a new file to `sources/`:
1. Read the source material completely
2. Identify key concepts, entities, and relationships
3. For each concept/entity:
   - Check if a VKI page already exists
   - If yes: update with new info, add source reference
   - If no: create new page (only if concept appears in 2+ sources, OR is a core concept)
4. Update `vki/index.md`
5. Report: what was added/updated, any cross-references found, observations

### 2. Query (查询)
When user asks a question:
1. Read `vki/index.md` to find relevant pages
2. Read relevant VKI pages in depth
3. Synthesize a comprehensive answer using knowledge network
4. If the answer is high-quality, ask user if they want to save it to `vki/articles/`

### 3. Maintain (维护)
When user says "health check" or "maintain":
1. Scan all VKI pages
2. Check for:
   - Contradictions between pages
   - Orphan pages (no links to/from other pages)
   - Missing entities (referenced but no page exists)
   - Outdated information
   - Broken cross-references
3. Fix issues and report what was changed

## Cross-Reference Rules
- A concept page should be created when a concept appears in **2+ sources**
- Always use `[[page-name]]` for internal links
- Every VKI page must have at least one `Sources` entry
- Prefer merging similar concepts over creating near-duplicates

## Language
- VKI content: Chinese preferred, English technical terms kept as-is
- Source files: keep original language

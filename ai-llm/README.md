# AI & LLM Integration Patterns

A comprehensive catalogue of 24 production-ready patterns for building reliable, cost-efficient, and safe LLM-powered systems.

## Patterns

| # | Pattern | Category | Key Technologies |
|---|---------|----------|-----------------|
| 01 | [Retrieval-Augmented Generation (RAG)](./01-rag.md) | Knowledge Grounding | OpenAI, pgvector, HNSW |
| 02 | [LLM Gateway & Provider Abstraction](./02-llm-gateway.md) | Infrastructure | LiteLLM, Redis, OpenAI, Anthropic |
| 03 | [Prompt Engineering & Management](./03-prompt-engineering.md) | Quality & Reliability | Prompt registry, Zod, few-shot |
| 04 | [AI Agents & Tool Use](./04-ai-agents-tool-use.md) | Agentic Systems | OpenAI function calling, ReAct, LangGraph |
| 05 | [Vector Databases](./05-vector-databases.md) | Storage & Retrieval | pgvector, Pinecone, Weaviate, Qdrant |
| 06 | [Embedding Pipelines](./06-embedding-pipelines.md) | Data Engineering for AI | sentence-transformers, Airflow, incremental |
| 07 | [Semantic Caching](./07-semantic-caching.md) | Performance & Cost | Redis, pgvector, cosine similarity |
| 08 | [AI Observability](./08-ai-observability.md) | Monitoring & Reliability | Prometheus, Grafana, LangSmith, Phoenix |
| 09 | [Guardrails & Content Safety](./09-guardrails-content-safety.md) | Safety & Compliance | Presidio, OpenAI Moderation, Zod |
| 10 | [Fine-tuning vs RAG Trade-offs](./10-finetuning-vs-rag.md) | Architecture Decision | OpenAI fine-tune API, MLflow |
| 11 | [Multimodal Pipelines](./11-multimodal-pipelines.md) | Advanced AI Patterns | GPT-4o vision, Whisper, PDF extraction |
| 12 | [AI Cost Optimisation](./12-ai-cost-optimisation.md) | FinOps for AI | Model routing, Batch API, prompt compression |
| 13 | [LLM Evaluation & Evals](./13-llm-evaluation.md) | Quality Engineering | RAGAS, PromptFoo, LLM-as-judge |
| 14 | [Function Calling & Structured Outputs](./14-function-calling-structured-outputs.md) | Integration Patterns | Zod, json_schema, parallel tools |
| 15 | [AI-Assisted Code Generation](./15-ai-code-generation-pipelines.md) | Developer Productivity | ts-morph, GitHub Actions, CI evals |
| 16 | [Harness Engineering](./16-harness-engineering.md) | Agent-First Development | AGENTS.md, architecture linters, self-review loop |
| 17 | [Model Context Protocol (MCP)](./17-model-context-protocol.md) | Agent Infrastructure | MCP SDK, stdio/SSE transport, tools, resources |
| 18 | [Multi-Agent Orchestration](./18-multi-agent-orchestration.md) | Agentic Systems | LangGraph, CrewAI, Mastra, supervisor/worker |
| 19 | [Reasoning Models & Inference-Time Compute](./19-reasoning-models.md) | Advanced AI Patterns | o1, o3, o4-mini, DeepSeek-R1, Claude thinking |
| 20 | [AI Memory Systems](./20-ai-memory-systems.md) | Agentic Systems | mem0, Zep, LangMem, episodic/semantic memory |
| 21 | [GraphRAG & Knowledge Graphs](./21-graphrag-knowledge-graphs.md) | Knowledge Grounding | Microsoft GraphRAG, Neo4j, entity extraction |
| 22 | [Local & Edge LLM Deployment](./22-local-edge-llm.md) | Infrastructure | Ollama, vLLM, llama.cpp, GGUF quantisation |
| 23 | [LLM Red Teaming & Adversarial Testing](./23-llm-red-teaming.md) | Safety & Compliance | PyRIT, Garak, prompt injection, jailbreak |
| 24 | [Synthetic Data Generation](./24-synthetic-data-generation.md) | Data Engineering for AI | Distilabel, Faker, knowledge distillation |

---

## Categories

### Knowledge Grounding
- **RAG** — Ground LLM responses in your proprietary data with verified citations

### Infrastructure & Resilience
- **LLM Gateway** — Provider failover, rate limiting, cost tracking, PII scrubbing

### Quality & Reliability
- **Prompt Engineering** — Versioned templates, few-shot, chain-of-thought, output parsers

### Agentic Systems
- **AI Agents** — ReAct loops, multi-agent supervisors, tool use, human-in-the-loop

### Storage & Retrieval
- **Vector Databases** — HNSW indexes, hybrid BM25+vector search, multi-tenancy

### Data Engineering for AI
- **Embedding Pipelines** — Incremental indexing, chunking strategies, local models

### Performance & Cost
- **Semantic Caching** — Similarity-based cache hit rate 40–80%, Redis + pgvector
- **AI Cost Optimisation** — Model routing, Batch API, prompt compression, token budgeting

### Monitoring & Reliability
- **AI Observability** — Token cost dashboards, quality evals, latency SLAs, alerts

### Safety & Compliance
- **Guardrails** — Input/output safety, PII detection, prompt injection prevention

### Architecture Decisions
- **Fine-tuning vs RAG** — Decision matrix, trade-offs, hybrid approach, MLflow tracking

### Agent Infrastructure
- **Model Context Protocol** — Plug-and-play tool connectivity standard; MCP servers, clients, resources
- **Multi-Agent Orchestration** — Supervisor/worker graphs, parallel fan-out, LangGraph, CrewAI, Mastra

### Advanced AI Patterns
- **Reasoning Models** — Inference-time compute (o1/o3/o4-mini, DeepSeek-R1); effort tuning, cost routing
- **AI Memory Systems** — Cross-session episodic and semantic memory; mem0, Zep, LangGraph checkpointer

### Knowledge Grounding (extended)
- **GraphRAG** — Entity/relationship extraction, graph traversal, multi-hop reasoning, Neo4j

### Self-Hosted Infrastructure
- **Local & Edge LLM** — Ollama, vLLM, llama.cpp, GGUF quantisation; on-prem / air-gapped deployment

### Safety & Compliance (extended)
- **LLM Red Teaming** — Adversarial attack simulation, PyRIT, Garak, indirect injection testing

### Data Engineering for AI (extended)
- **Synthetic Data Generation** — Fine-tune dataset generation, knowledge distillation, Distilabel, Faker

### Advanced AI Patterns
- **Multimodal** — Document extraction, audio analysis, image + text pipelines

### Quality Engineering
- **LLM Evals** — Automated quality regression testing, RAGAS metrics, CI/CD gates

### Integration Patterns
- **Structured Outputs** — Zod schemas, json_schema, parallel function calling

### Developer Productivity
- **AI Code Generation** — Test scaffolding, AI code review, GitHub Actions integration

### Agent-First Development
- **Harness Engineering** — Repo as system of record, architecture enforcement, self-review loops, entropy management

---

## Decision Guide

### Which pattern first?

```
What is your primary challenge?
├── "My LLM makes things up"           → 01-rag + 09-guardrails + 13-llm-evaluation
├── "Too expensive / too slow"          → 07-semantic-caching + 12-ai-cost-optimisation
├── "I need to call external APIs"      → 04-ai-agents + 14-function-calling
├── "Output format is unreliable"       → 03-prompt-engineering + 14-function-calling
├── "Multiple LLM providers needed"     → 02-llm-gateway
├── "I need to process documents/images"→ 11-multimodal + 01-rag
├── "Quality is degrading"              → 08-ai-observability + 13-llm-evaluation
├── "PII / compliance concerns"         → 09-guardrails + 06-embedding-pipelines
└── "I want to scale with AI agents"   → 16-harness-engineering + 04-ai-agents
```

### RAG vs Fine-tune Quick Guide

| Question | RAG | Fine-tune |
|----------|-----|-----------|
| Data changes frequently? | ✅ | ❌ |
| Need citations / traceability? | ✅ | ❌ |
| Need specific output style/format? | ❌ | ✅ |
| Have < 50 training examples? | ✅ | ❌ |
| High-volume, latency-sensitive? | ❌ | ✅ |
| Data must never leave your infra? | ✅ (local embed) | ❌ (GPU cloud) |

---

## Tool Ecosystem

### LLM Providers
| Tool | Type | Notes |
|------|------|-------|
| **OpenAI** | Cloud API | GPT-4o, Whisper, Embeddings, DALL-E |
| **Anthropic** | Cloud API | Claude 3.5 Sonnet / Haiku |
| **Azure OpenAI** | Cloud (enterprise) | Data residency, Private Link |
| **Google Vertex AI** | Cloud | Gemini 1.5 Pro, multimodal |
| **Ollama** | Local | Llama 3, Mistral on-premises |
| **LiteLLM** | Gateway/proxy | Unified interface for all providers |

### Vector Stores
| Tool | Notes |
|------|-------|
| **pgvector** | Postgres extension, HNSW, free |
| **Pinecone** | Managed, serverless tier |
| **Weaviate** | Hybrid search, multi-tenancy |
| **Qdrant** | Rust, payload filtering |
| **Chroma** | Embedded, local dev |

### AI Frameworks
| Tool | Notes |
|------|-------|
| **LangChain** | Chains, agents, RAG templates |
| **LangGraph** | Stateful multi-agent graphs |
| **Vercel AI SDK** | TypeScript streaming, RSC |
| **Instructor** | Structured outputs, retry logic |
| **Semantic Kernel** | Microsoft, .NET + Python |

### Observability & Evals
| Tool | Notes |
|------|-------|
| **LangSmith** | Tracing, evals, prompt registry |
| **Phoenix (Arize)** | Open-source, OTEL, LLM judge |
| **RAGAS** | RAG evaluation metrics |
| **PromptFoo** | YAML-driven multi-model evals |
| **Langfuse** | Open-source LangSmith alternative |

### Safety
| Tool | Notes |
|------|-------|
| **Azure Presidio** | PII detection + anonymisation |
| **OpenAI Moderation** | Toxicity, violence, self-harm |
| **Llama Guard** | Meta open-source content safety |
| **Perspective API** | Google toxicity scoring |

---

## Related Sections

- [Data Solutions](../data-solutions/README.md) — Feature stores, embeddings storage, data pipelines
- [Security](../security/README.md) — Application security, secrets management, encryption
- [DevSecOps](../devsecops/README.md) — Secure CI/CD for AI model deployments
- [Cloud Native / AWS](../cloud-native/aws/README.md) — Bedrock, SageMaker, Lambda AI workloads
- [Cloud Native / Azure](../cloud-native/azure/README.md) — Azure OpenAI, AI Search, Semantic Kernel

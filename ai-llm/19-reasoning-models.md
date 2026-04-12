# Reasoning Models & Inference-Time Compute

## Category

AI / LLM Integration — Reasoning, Chain-of-Thought, o1/o3, DeepSeek-R1, Cost Management

## Context

**Reasoning models** (OpenAI o1/o3/o4-mini, DeepSeek-R1, Gemini Thinking, Claude extended thinking) perform internal chain-of-thought reasoning **before** producing the final answer — trading latency and token cost for dramatically higher accuracy on complex tasks.

Unlike standard LLMs where you add chain-of-thought to your prompt, reasoning models generate internal "thinking tokens" that are hidden or partially visible to the caller.

### Standard LLM vs Reasoning Model

| Dimension | Standard LLM (GPT-4o) | Reasoning Model (o3) |
|-----------|----------------------|---------------------|
| Prompt strategy | Explicit CoT in prompt | Model reasons internally |
| Latency | 1–5 s | 10–120 s (deep reasoning) |
| Token cost | Output tokens only | Thinking tokens + output tokens |
| Best tasks | Text gen, summarisation, classification | Math, coding, multi-step logic, planning |
| Tool use | Good | Excellent with planning |
| Prompt sensitivity | High | Lower — more robust to phrasing |

### Reasoning Effort / Budget

Most reasoning models expose effort control:

| Setting | o3 / o4-mini | Claude | DeepSeek-R1 |
|---------|-------------|--------|------------|
| Low | `reasoning_effort: "low"` | `budget_tokens: 1024` | `<think>` short |
| Medium | `reasoning_effort: "medium"` | `budget_tokens: 8192` | — |
| High | `reasoning_effort: "high"` | `budget_tokens: 32000` | `<think>` long |

### When to Use Reasoning Models

| Task | Use Reasoning? |
|------|---------------|
| Complex multi-step coding | ✅ Yes |
| Mathematical proofs / calculations | ✅ Yes |
| Multi-constraint planning | ✅ Yes |
| Agent decision-making with trade-offs | ✅ Yes |
| Simple Q&A / summarisation | ❌ Standard LLM cheaper |
| Streaming chat responses | ❌ Latency too high |
| High-volume classification | ❌ Use fine-tuned model |

---

## Pros

- Substantially better at tasks requiring multi-step logic, planning, and self-correction.
- Lower prompt sensitivity — no need to carefully phrase chain-of-thought instructions.
- Thinking tokens allow progressive deepening: low effort for speed, high effort for accuracy.
- DeepSeek-R1 visible `<think>` blocks are debuggable and auditable.
- `reasoning_effort: "low"` on o4-mini is cost-competitive with GPT-4o on many tasks with better accuracy.

---

## Cons

- 10–100× higher latency than standard models — unsuitable for real-time streaming chat.
- Thinking tokens are billed — a single o3 request can cost $0.10–$2.00 at high effort.
- System prompt support is limited in some reasoning models — instructions behave differently.
- Parallel tool calls less predictable — model may choose sequential reasoning over parallelism.
- Streaming thinking tokens not always available; users see a long pause before output.

---

## Design Diagram

```mermaid
flowchart LR
    ROUTER["Model Router\n(check task complexity)"]
    STANDARD["Standard LLM\nGPT-4o / Claude 3.5\nLow latency, low cost"]
    REASONING["Reasoning Model\no3 / o4-mini / R1\nHigh accuracy, higher cost"]

    INPUT["User Request"] --> ROUTER
    ROUTER -->|simple: classify, summarise| STANDARD
    ROUTER -->|complex: plan, code, math| REASONING

    REASONING -->|thinking tokens\n(internal)| THINK[("Internal CoT\n(hidden or visible)")]
    THINK --> ANSWER["Final Answer"]
    STANDARD --> ANSWER
```

---

## Code Sample

### OpenAI o4-mini — reasoning effort control

```typescript
import OpenAI from 'openai';

const openai = new OpenAI();

async function reasoningQuery(problem: string, effort: 'low' | 'medium' | 'high') {
  const response = await openai.chat.completions.create({
    model: 'o4-mini',
    reasoning_effort: effort,
    messages: [
      {
        role: 'user',
        content: problem,
      },
    ],
    // NOTE: system prompt is supported but behaves as a developer message
    // Do NOT add chain-of-thought instructions — the model handles reasoning internally
  });

  const usage = response.usage;
  console.log(`Reasoning tokens: ${usage?.completion_tokens_details?.reasoning_tokens}`);
  console.log(`Output tokens: ${usage?.completion_tokens}`);

  return response.choices[0].message.content;
}

// Complex planning task — use high effort
const plan = await reasoningQuery(
  'Design a database migration strategy to split a monolithic orders table (500M rows) into per-region shards with zero downtime.',
  'high'
);

// Simple lookup — use low effort
const answer = await reasoningQuery('What is the capital of France?', 'low');
```

### Task complexity router — auto-select model

```typescript
import OpenAI from 'openai';

const openai = new OpenAI();

type TaskType = 'simple' | 'moderate' | 'complex';

function classifyTask(prompt: string): TaskType {
  const complexIndicators = [
    /\b(design|architect|plan|optimise|refactor|debug|implement|algorithm)\b/i,
    /\b(step[- ]by[- ]step|multi[- ]step|trade[- ]off|pros and cons)\b/i,
    /\b(sql|code|function|class|schema)\b/i,
  ];
  const matches = complexIndicators.filter(r => r.test(prompt)).length;
  if (matches >= 2) return 'complex';
  if (matches === 1) return 'moderate';
  return 'simple';
}

const MODEL_CONFIG = {
  simple:   { model: 'gpt-4o-mini', options: {} },
  moderate: { model: 'gpt-4o',      options: {} },
  complex:  { model: 'o4-mini',     options: { reasoning_effort: 'high' } },
} as const;

async function smartQuery(prompt: string): Promise<string> {
  const taskType = classifyTask(prompt);
  const { model, options } = MODEL_CONFIG[taskType];

  console.log(`Using model: ${model} (task type: ${taskType})`);

  const response = await openai.chat.completions.create({
    model,
    messages: [{ role: 'user', content: prompt }],
    ...options,
  });

  return response.choices[0].message.content ?? '';
}
```

### Claude extended thinking

```typescript
import Anthropic from '@anthropic-ai/sdk';

const client = new Anthropic();

async function claudeReasoning(problem: string, budgetTokens = 8000) {
  const response = await client.messages.create({
    model: 'claude-opus-4-5',
    max_tokens: budgetTokens + 4096,
    thinking: {
      type: 'enabled',
      budget_tokens: budgetTokens,
    },
    messages: [{ role: 'user', content: problem }],
  });

  // Response contains both thinking blocks and text blocks
  for (const block of response.content) {
    if (block.type === 'thinking') {
      console.log('--- Claude thinking ---');
      console.log(block.thinking);  // the internal reasoning
    } else if (block.type === 'text') {
      console.log('--- Final answer ---');
      console.log(block.text);
    }
  }
}

await claudeReasoning(
  'Identify all potential race conditions in this payment processing code and suggest fixes: ...',
  16000  // more budget for hard problems
);
```

### DeepSeek-R1 — visible thinking tokens

```typescript
import OpenAI from 'openai';

// DeepSeek uses OpenAI-compatible API
const client = new OpenAI({
  apiKey: process.env.DEEPSEEK_API_KEY,
  baseURL: 'https://api.deepseek.com',
});

async function deepseekReasoning(prompt: string) {
  const response = await client.chat.completions.create({
    model: 'deepseek-reasoner',  // DeepSeek-R1
    messages: [{ role: 'user', content: prompt }],
  });

  const message = response.choices[0].message;

  // DeepSeek-R1 exposes reasoning_content separately
  if ('reasoning_content' in message && message.reasoning_content) {
    console.log('Reasoning:', message.reasoning_content);
  }
  console.log('Answer:', message.content);

  return message.content;
}
```

### Evals — verify reasoning model justifies cost

```typescript
import { Eval } from 'braintrust';
import Anthropic from '@anthropic-ai/sdk';
import OpenAI from 'openai';

// Compare standard vs reasoning model on your actual tasks
Eval('reasoning-vs-standard', {
  data: () => complexCodingProblems,  // your benchmark dataset

  task: async (input) => {
    const [standard, reasoning] = await Promise.all([
      queryGPT4o(input.prompt),
      queryO4mini(input.prompt, 'high'),
    ]);
    return { standard, reasoning };
  },

  scores: [
    async ({ output, expected }) => ({
      name: 'standard_correct',
      score: await llmJudge(output.standard, expected) ? 1 : 0,
    }),
    async ({ output, expected }) => ({
      name: 'reasoning_correct',
      score: await llmJudge(output.reasoning, expected) ? 1 : 0,
    }),
  ],
});
```

---

## Related

- [12 — AI Cost Optimisation](./12-ai-cost-optimisation.md) — model routing to balance reasoning model cost vs accuracy
- [13 — LLM Evaluation](./13-llm-evaluation.md) — benchmarking reasoning models vs standard on your tasks
- [04 — AI Agents & Tool Use](./04-ai-agents-tool-use.md) — reasoning models excel as agent planners
- [18 — Multi-Agent Orchestration](./18-multi-agent-orchestration.md) — use reasoning model as supervisor, cheaper models as workers

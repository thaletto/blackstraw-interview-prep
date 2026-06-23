# GenAI / Agentic AI Interview Questions

**Your Self-Assessment:** Theory strong (ReAct, Plan-Exec-Val, Multi-Agent), hands-on limited  
**Focus:** Architecture testing, practical scenarios, decision-making

---

## Question 1: RAG Architecture Deep Dive

**Difficulty:** Advanced  
**Category:** Strength Validation

Explain RAG (Retrieval-Augmented Generation) architecture. How would you improve retrieval accuracy in a production system?

<details>
<summary><b>View Answer & Explanation</b></summary>

**Answer:**

**RAG Pipeline:** Ingestion → Retrieval → Augmentation → Generation

**Improving Retrieval Accuracy:**

1. **Better Chunking:** Use semantic chunking with overlap (RecursiveCharacterTextSplitter with 500/50)
2. **Hybrid Search:** Combine BM25 (keyword) + semantic search (vector) with weights
3. **Re-ranking:** Retrieve top 20, re-rank with cross-encoder (CohereRerank), return top 5
4. **Query Transformation:** Multi-query expansion, HyDE (hypothetical document embeddings)
5. **Metadata Filtering:** Add date, source, author filters for targeted retrieval

**Key Concepts:**
- Chunking strategy impacts retrieval quality
- Hybrid search outperforms pure semantic
- Re-ranking significantly improves precision
- Embedding model choice matters (domain-specific)

**Interview Tip:**
> "RAG has three critical components: chunking, retrieval, and re-ranking. I use semantic chunking with overlap, hybrid search (BM25 + semantic), and re-ranking for production."

</details>

---

## Question 2: Agent Architecture Patterns

**Difficulty:** Advanced  
**Category:** Strength Validation

Compare ReAct, Plan-Execute-Validate, and Multi-Agent Orchestration patterns. When would you use each?

<details>
<summary><b>View Answer & Explanation</b></summary>

**Answer:**

**1. ReAct:** Single agent, iterative loop (Thought → Action → Observation). Best for simple, linear tasks.

**2. Plan-Execute-Validate:** Separate planning phase, then execution, then validation. Best for complex tasks requiring upfront planning and error handling.

**3. Multi-Agent Orchestration:** Multiple specialized agents coordinate through conversation. Best for complex workflows with specialized roles.

**Decision Framework:**
- Simple linear task → ReAct
- Complex with dependencies → Plan-Execute-Validate
- Multiple specialized roles → Multi-Agent (like your AMS Platform)

**Your AMS Project:** Uses multi-agent orchestration with Observability → Planning → Human Approval → Execution → Validation agents.

**Key Concepts:**
- Complexity vs capability trade-off
- Multi-agent enables parallel work and specialization
- Human-in-the-loop for critical decisions

**Interview Tip:**
> "I choose the pattern based on task complexity. In my AMS project, I used multi-agent orchestration with explicit planning, human approval, and validation—each agent has a specific role."

</details>

---

## Question 3: Vector Database Selection

**Difficulty:** Intermediate  
**Category:** Learning

Compare Pinecone, FAISS, ChromaDB, and Weaviate. When would you use each?

<details>
<summary><b>View Answer & Explanation</b></summary>

**Answer:**

**Comparison:**
- **Pinecone:** Managed, scalable, expensive. Best for production at scale.
- **FAISS:** Library, fast, no metadata. Best for research/prototyping.
- **ChromaDB:** Embedded, easy, limited scale. Best for development.
- **Weaviate:** Flexible, GraphQL, complex. Best for complex metadata filtering.

**Decision:**
- Prototypes → ChromaDB or FAISS
- Production scale → Pinecone or Weaviate
- Need metadata filtering → ChromaDB or Weaviate

**Your Cortex Project:** Abstracts these databases with an Effect-native layer, avoiding vendor lock-in.

**Key Concepts:**
- Managed vs self-hosted trade-off
- Metadata filtering capabilities
- Scalability vs cost

**Interview Tip:**
> "I choose based on scale and requirements. In Cortex, I abstracted these with an adapter pattern so the application stays database-agnostic."

</details>

---

## Question 4: LLM Cost Optimization

**Difficulty:** Advanced  
**Category:** Learning

How do you optimize LLM costs in production?

<details>
<summary><b>View Answer & Explanation</b></summary>

**Answer:**

**Techniques:**

1. **Model Selection:** Use GPT-3.5 for simple tasks, GPT-4 for complex (20x cost difference)
2. **Caching:** Exact + semantic caching (30-50% savings)
3. **Prompt Optimization:** Concise prompts reduce token usage
4. **Batch Processing:** Combine multiple requests into one
5. **Fine-tuning:** For high-volume repetitive tasks

**Cost Example (1M requests/month):**
- GPT-4: $27,000/month
- GPT-3.5: $1,150/month (95% reduction)
- With caching: $805/month

**Key Concepts:**
- Model selection is the biggest cost factor
- Caching provides 30-50% savings
- Fine-tuning enables cheaper models

**Interview Tip:**
> "I optimize with model selection (GPT-3.5 vs GPT-4 based on complexity), semantic caching, prompt optimization, and batch processing. The biggest win is choosing the right model for the task."

</details>

---

## Question 5: Prompt Engineering Techniques

**Difficulty:** Intermediate  
**Category:** Strength Validation

Demonstrate advanced prompt engineering techniques.

<details>
<summary><b>View Answer & Explanation</b></summary>

**Answer:**

**Techniques:**

1. **Zero-Shot:** Direct prompt, no examples
2. **Few-Shot Learning:** Provide 3-5 examples to guide the model
3. **Chain-of-Thought (CoT):** Force step-by-step reasoning
4. **ReAct:** Reasoning + Action format
5. **Self-Consistency:** Generate multiple samples, take majority vote
6. **Structured Output:** Force JSON output (function calling, JSON mode)
7. **System Prompts:** Set behavior and constraints
8. **Temperature Control:** Low (0.0-0.3) for factual, high (0.8-1.0) for creative

**Key Concepts:**
- Few-shot learning provides examples
- CoT forces step-by-step reasoning
- Structured output ensures parseability

**Interview Tip:**
> "I use few-shot learning for complex tasks, chain-of-thought for reasoning, and structured output (JSON) for data extraction. Temperature is 0 for factual, 0.7 for creative."

</details>

---

## Question 6: Agent Error Handling and Reliability

**Difficulty:** Advanced  
**Category:** Learning

How do you handle errors and make agents reliable in production?

<details>
<summary><b>View Answer & Explanation</b></summary>

**Answer:**

**Common Failures:** Infinite loops, tool call failures, hallucinations, context overflow, validation errors, cost explosions

**Solutions:**

1. **Max Iterations:** Set hard limit (e.g., 10) to prevent infinite loops
2. **Retry with Exponential Backoff:** Handle transient failures
3. **Tool Call Validation:** Validate before execution (Pydantic)
4. **Fallback Strategies:** Use cheaper models or cached responses
5. **Context Window Management:** Truncate or summarize old messages
6. **Circuit Breakers:** Prevent cascading failures
7. **Observability:** Log iterations, duration, tokens, cost

**Key Concepts:**
- Assume failure and design defensively
- Max iterations prevent loops
- Circuit breakers prevent cascading failures

**Interview Tip:**
> "I implement max iterations, retry with exponential backoff, tool call validation, fallback strategies, and circuit breakers. Always add observability for debugging."

</details>

---

## Question 7: Embedding Models and Vector Search

**Difficulty:** Intermediate  
**Category:** Learning

Explain embedding models. How do you choose the right one?

<details>
<summary><b>View Answer & Explanation</b></summary>

**Answer:**

**Embedding Models:**
- **OpenAI:** text-embedding-3-small (1536 dims, $0.02/1M tokens), high quality
- **Open-Source:** Sentence Transformers (384 dims, free, local)
- **Domain-Specific:** CodeBERT, BioBERT for specialized domains

**Vector Search Methods:**
- **Flat Search:** Exact, O(n), for <10K vectors
- **HNSW:** Fast approximate, O(log n), for 10K-10M vectors
- **IVF:** Memory-efficient, for >1M vectors

**Similarity Metrics:**
- **Cosine Similarity:** Standard for normalized embeddings
- **Euclidean Distance:** When magnitude matters
- **Dot Product:** Fastest for normalized vectors

**Key Concepts:**
- Quality vs cost vs privacy trade-off
- Cosine similarity for normalized vectors
- HNSW for fast approximate search

**Interview Tip:**
> "I choose based on quality, cost, and privacy. OpenAI for quality, sentence-transformers for privacy/cost. HNSW for 10K-10M vectors, IVF for larger scale."

</details>

---

## Question 8: LLM Evaluation and Monitoring

**Difficulty:** Advanced  
**Category:** Learning

How do you evaluate and monitor LLM applications in production?

<details>
<summary><b>View Answer & Explanation</b></summary>

**Answer:**

**Evaluation Metrics:**

1. **RAG-Specific (RAGAS):**
   - Faithfulness (answer grounded in context)
   - Answer Relevancy (relevant to question)
   - Context Precision (retrieved docs relevant)

2. **LLM-as-Judge:** Use LLM to evaluate other LLM outputs (scalable)

3. **A/B Testing:** Compare prompt/model changes

**Production Monitoring:**

1. **Structured Logging:** Log all interactions with metadata
2. **Real-Time Metrics:** Latency (p95), cost, error rate
3. **Cost Monitoring:** Daily budgets, alerts
4. **User Feedback:** Explicit (ratings) and implicit (thumbs up/down)
5. **Drift Detection:** Alert when performance degrades
6. **Tracing Tools:** LangSmith, Langfuse

**Key Concepts:**
- RAGAS for RAG evaluation
- LLM-as-judge for scalable evaluation
- Monitor latency, cost, errors

**Interview Tip:**
> "I evaluate with RAGAS metrics and LLM-as-judge. In production, I monitor latency, cost, error rates, and use tracing tools like LangSmith. I set up drift detection and cost alerts."

</details>

---

## Question 9: Multi-Agent Communication Patterns

**Difficulty:** Advanced  
**Category:** Learning

How do agents communicate in a multi-agent system?

<details>
<summary><b>View Answer & Explanation</b></summary>

**Answer:**

**Patterns:**

1. **Message Passing:** Direct agent-to-agent communication. Tight coupling, easy to debug.
2. **Shared Memory (Blackboard):** Agents read/write to shared memory. Loose coupling, scales well.
3. **Orchestrator:** Central coordinator manages agents. Clear flow, good for sequential workflows.
4. **Group Chat:** AutoGen pattern, agents collaborate in conversation. Good for collaborative tasks.
5. **Event-Driven (Pub/Sub):** Async, distributed, scalable. Good for asynchronous systems.

**Your AMS Project:** Uses Orchestrator pattern with clear roles (Observability → Planning → Human Approval → Execution → Validation).

**Decision:**
- 2-3 agents with clear flow → Orchestrator
- Collaborative tasks → Group Chat (AutoGen)
- Distributed systems → Event-driven

**Key Concepts:**
- Coupling vs scalability trade-off
- Orchestrator for sequential workflows
- Group chat for collaboration

**Interview Tip:**
> "For multi-agent systems, I choose based on agent count and coordination needs. In my AMS project, I used the orchestrator pattern with clear roles—each agent has specific tools and the workflow is explicit."

</details>

---

## Question 10: Fine-tuning vs Prompting vs RAG

**Difficulty:** Intermediate  
**Category:** Learning

When would you fine-tune an LLM vs use prompting vs RAG?

<details>
<summary><b>View Answer & Explanation</b></summary>

**Answer:**

**Decision Framework:**

1. **Prompting:** General tasks, quick prototyping, reasoning
   - Use for: Classification, extraction, code generation

2. **RAG:** Factual knowledge from documents
   - Use for: Customer support, documentation Q&A, research

3. **Fine-tuning:** Custom style/format, domain expertise
   - Use for: Medical assistant, legal analysis, brand voice

4. **Hybrid (RAG + Fine-tuning):** Knowledge + style
   - Use for: Customer support chatbot with company tone

**Comparison:**
- Prompting: Fast, cheap, general
- RAG: Knowledge retrieval, no training
- Fine-tuning: Custom style, expensive
- Hybrid: Best of both worlds

**Key Concepts:**
- RAG for knowledge, fine-tuning for style
- Match approach to use case and budget
- Fine-tuning requires 100s-1000s of examples

**Interview Tip:**
> "I use a decision framework: RAG for factual knowledge, fine-tuning for custom style/format, and prompting for general tasks. For complex use cases, I combine RAG + fine-tuning."

</details>

---

## Question 11: Streaming and Real-time LLM Applications

**Difficulty:** Advanced  
**Category:** Learning

How do you stream LLM responses?

<details>
<summary><b>View Answer & Explanation</b></summary>

**Answer:**

**Basic Streaming:**
```python
stream = openai.ChatCompletion.create(
    model="gpt-4",
    messages=[...],
    stream=True
)
for chunk in stream:
    if chunk.choices[0].delta.content:
        print(chunk.choices[0].delta.content, end="", flush=True)
```

**FastAPI Streaming:**
```python
async def generate_stream(prompt: str):
    stream = await openai.ChatCompletion.acreate(..., stream=True)
    async for chunk in stream:
        if chunk.choices[0].delta.content:
            yield f"data: {json.dumps({'text': chunk.choices[0].delta.content})}\n\n"

@app.post("/chat/stream")
async def chat_stream(prompt: str):
    return StreamingResponse(
        generate_stream(prompt),
        media_type="text/event-stream",
        headers={"X-Accel-Buffering": "no"}
    )
```

**Key Concepts:**
- Streaming improves perceived latency
- Server-Sent Events for one-way streaming
- WebSockets for bidirectional
- Set `X-Accel-Buffering: no` to prevent proxy buffering

**Interview Tip:**
> "I use StreamingResponse with async generators. For one-way streaming, I use Server-Sent Events. I always set X-Accel-Buffering: no to prevent proxy buffering."

</details>

---

## Question 12: Context Window Management

**Difficulty:** Advanced  
**Category:** Learning

How do you handle context window limitations?

<details>
<summary><b>View Answer & Explanation</b></summary>

**Answer:**

**Strategies:**

1. **Truncation:** Simple but loses information
2. **Sliding Window:** Keep system message + recent messages
3. **Summarization:** Compress old messages with LLM
4. **Retrieval-Augmented Context:** Retrieve relevant context instead of including all
5. **Token-Efficient Prompting:** Concise prompts

**Implementation:**
```python
def truncate_messages(messages, max_tokens=4000):
    encoding = tiktoken.encoding_for_model("gpt-4")
    # Keep system message + recent messages within budget
    system_msg = messages[0] if messages[0]["role"] == "system" else None
    # ... sliding window logic
```

**Key Concepts:**
- Sliding window for recent context
- Summarization for long conversations
- RAG for document Q&A
- Token-efficient prompts

**Interview Tip:**
> "I handle context limits with summarization for long conversations, RAG for document Q&A, sliding window for recent context. I preserve system messages and use token-efficient prompts."

</details>

---

## Question 13: Building Agentic Workflows with AutoGen

**Difficulty:** Advanced  
**Category:** Strength Validation

Demonstrate building a multi-agent workflow with AutoGen. How does it compare to LangChain?

<details>
<summary><b>View Answer & Explanation</b></summary>

**Answer:**

**AutoGen Example:**
```python
from autogen import ConversableAgent, GroupChat, GroupChatManager

# Specialized agents
planner = ConversableAgent(
    name="Planner",
    system_message="Create detailed plans. Do not execute.",
    llm_config={"model": "gpt-4"}
)

executor = ConversableAgent(
    name="Executor",
    system_message="Execute steps using available tools.",
    llm_config={"model": "gpt-4"}
)

# Group chat
group_chat = GroupChat(
    agents=[planner, executor, validator],
    max_round=10,
    speaker_selection_method="auto"
)

manager = GroupChatManager(groupchat=group_chat)
planner.initiate_chat(manager, message="Deploy the application")
```

**AutoGen vs LangChain:**
- **AutoGen:** Multi-agent conversations, group chat, human-in-the-loop
- **LangChain:** Chains and pipelines, RAG, document processing

**When to Use:**
- Multi-agent workflows → AutoGen
- Simple chains/RAG → LangChain

**Your AMS Project:** Uses AutoGen pattern with specialized agents (Observability, Planning, Execution, Validation) and human-in-the-loop checkpoint.

**Interview Tip:**
> "I use AutoGen for multi-agent workflows with specialized roles—like my AMS project for incident response. For simpler chains and RAG, I use LangChain. AutoGen shines when you need agent-to-agent conversation and human-in-the-loop."

</details>

---

## Question 14: Prompt Injection and Security

**Difficulty:** Advanced  
**Category:** Learning

What is prompt injection? How do you prevent it?

<details>
<summary><b>View Answer & Explanation</b></summary>

**Answer:**

**Types:**
1. **Direct:** User tries to override system prompt
2. **Indirect:** Malicious content in RAG documents
3. **Jailbreaking:** Bypass safety guidelines

**Prevention:**

1. **Input Validation:** Pattern detection for injection attempts
2. **Separate Context from Instructions:** Use clear delimiters
3. **Output Validation:** Check for sensitive info leakage
4. **Function Calling:** LLM can only call allowed functions
5. **Prompt Hardening:** Absolute rules in system prompt
6. **Monitoring:** Log all interactions

**Key Concepts:**
- Defense in depth (multiple layers)
- Validate inputs AND outputs
- Use function calling for controlled actions
- Apply least privilege principle

**Interview Tip:**
> "I prevent prompt injection with input validation, separating context from instructions, output validation, function calling for controlled actions, and least privilege. I use robust system prompts with absolute rules."

</details>

---

## Question 15: GraphRAG and Knowledge Graphs

**Difficulty:** Advanced  
**Category:** Learning

What is GraphRAG? How does it differ from traditional RAG?

<details>
<summary><b>View Answer & Explanation</b></summary>

**Answer:**

**GraphRAG** combines knowledge graphs with vector search for better relationship understanding and multi-hop reasoning.

**Traditional RAG:** Vector similarity search, single-hop retrieval, finds similar documents.

**GraphRAG:** Graph traversal + vectors, multi-hop reasoning, understands relationships between entities.

**Example:** "How does Acme Corp's partnership with Tech Inc affect our market position?"
- Traditional RAG: Finds docs about Acme, Tech, market (no relationships)
- GraphRAG: Traverses Acme → PARTNERS_WITH → Tech Inc → COMPETES_WITH → Competitor X

**When to Use:**
- Traditional RAG: Simple Q&A, documentation search
- GraphRAG: Complex queries with relationships, multi-hop reasoning

**Key Concepts:**
- Combines knowledge graphs + vector search
- Enables multi-hop reasoning
- Better for relationship-heavy queries
- More complex (entity extraction, graph database)

**Interview Tip:**
> "GraphRAG combines knowledge graphs with vector search for better relationship understanding. I use it when queries require multi-hop reasoning. Traditional RAG is for simple Q&A."

</details>

---

## Question 16: LLM Application Architecture

**Difficulty:** Advanced  
**Category:** Learning

Design the architecture for a production LLM application.

<details>
<summary><b>View Answer & Explanation</b></summary>

**Answer:**

**Production Architecture:**

```
Client → API Gateway → Application Layer → LLM Provider
                          ↓                  ↓
                    Vector DB          Observability
                          ↓
                    Caching (Redis)
```

**Key Components:**

1. **API Gateway:** Auth, rate limiting, routing
2. **Application Layer:** RAG, agents, business logic
3. **Vector Database:** Pinecone, Qdrant, Weaviate
4. **LLM Provider:** OpenAI, Anthropic, open-source
5. **Caching:** Redis (exact + semantic)
6. **Observability:** Logging, metrics, tracing
7. **Security:** Input/output validation
8. **Error Handling:** Fallbacks, circuit breakers

**Key Concepts:**
- Separate concerns (gateway, app, data)
- Cache aggressively (exact + semantic)
- Monitor everything (latency, cost, errors)
- Fail gracefully (fallbacks, circuit breakers)
- Secure by default

**Interview Tip:**
> "A production LLM app needs: API gateway, application layer, vector DB, LLM provider, caching, observability, security, and error handling. The key is assuming failure and designing for resilience, cost, and security."

</details>

---

## Question 17: Open-Source LLMs vs OpenAI

**Difficulty:** Intermediate  
**Category:** Learning

Compare open-source LLMs vs OpenAI. When would you use each?

<details>
<summary><b>View Answer & Explanation</b></summary>

**Answer:**

**Comparison:**
- **OpenAI (GPT-4):** Highest quality, expensive, easy, API-based
- **Open-Source (Llama, Mistral):** Good quality, free, private, requires infrastructure

**Decision:**
- Use OpenAI: Complex reasoning, quick prototyping, no infrastructure concerns
- Use Open-Source: Privacy requirements, high volume, customization, low latency

**Open-Source Tools:**
- Hugging Face Transformers (standard)
- vLLM (24x faster inference)
- 4-bit quantization (run on consumer GPUs)
- LoRA fine-tuning (efficient)

**Key Concepts:**
- Quality vs cost vs privacy trade-off
- Open-source quality improving rapidly
- Quantization enables consumer GPU inference

**Interview Tip:**
> "I use OpenAI for complex reasoning and prototyping, open-source for privacy-sensitive apps or high-volume simple tasks. For open-source, I use vLLM for fast inference and quantization for consumer GPUs."

</details>

---

## Question 18: Structured Output and Function Calling

**Difficulty:** Intermediate  
**Category:** Learning

How do you get structured output from LLMs?

<details>
<summary><b>View Answer & Explanation</b></summary>

**Answer:**

**Methods (in order of reliability):**

1. **Function Calling (Most Reliable):**
   ```python
   response = openai.ChatCompletion.create(
       model="gpt-4",
       functions=[{
           "name": "extract_info",
           "parameters": {
               "type": "object",
               "properties": {"name": {"type": "string"}}
           }
       }],
       function_call={"name": "extract_info"}
   )
   ```

2. **JSON Mode (OpenAI):**
   ```python
   response = openai.ChatCompletion.create(
       model="gpt-4",
       response_format={"type": "json_object"},
       messages=[...]
   )
   ```

3. **Prompting with Pydantic Validation:** Manual parsing + validation

4. **Instructor Library:** Automatic Pydantic integration

**Key Concepts:**
- Function calling is most reliable (built-in validation)
- JSON mode guarantees valid JSON
- Pydantic for type safety
- Multi-function calling for agent actions

**Interview Tip:**
> "I use function calling for production structured output—it's the most reliable. For simpler cases, I use JSON mode with Pydantic validation. The Instructor library makes Pydantic integration seamless."

</details>

---

## Question 19: Agentic Workflow Patterns

**Difficulty:** Advanced  
**Category:** Learning

Demonstrate different agentic workflow patterns.

<details>
<summary><b>View Answer & Explanation</b></summary>

**Answer:**

**Patterns:**

1. **Sequential:** Steps in order, each depends on previous
2. **Parallel:** Independent tasks concurrent (asyncio.gather)
3. **Conditional:** Route based on input type
4. **Human-in-the-Loop:** Critical decision approval
5. **Iterative:** Refine until success (validation loop)
6. **Hierarchical:** Orchestrator delegates to sub-agents

**Your AMS Platform** combines multiple patterns:
- Sequential: Detection → Planning → Execution → Validation
- Conditional: Check if ticket exists
- Human-in-the-Loop: Approval gate
- Iterative: Validator retries if steps fail
- Hierarchical: Orchestrator coordinates specialized agents

**Key Concepts:**
- Sequential for linear tasks
- Parallel for independent operations
- Human-in-the-loop for critical decisions
- Iterative for refinement
- Hierarchical for complex coordination

**Interview Tip:**
> "I choose workflow patterns based on task requirements. In my AMS project, I combined multiple patterns: sequential detection, human-in-the-loop approval, iterative execution with validation."

</details>

---

## Question 20: Building Production RAG Systems

**Difficulty:** Advanced  
**Category:** Learning

Walk through building a production RAG system.

<details>
<summary><b>View Answer & Explanation</b></summary>

**Answer:**

**Complete Production RAG System:**

**1. Document Ingestion:**
- Load documents (PDF, DOCX, etc.)
- Semantic chunking (500/50 overlap)
- Generate embeddings (OpenAI/Cohere)
- Store in vector DB (Pinecone/Qdrant) with metadata

**2. Retrieval Pipeline:**
- Hybrid search (BM25 + semantic)
- Re-ranking (CohereRerank, top 5)
- Metadata filtering

**3. Generation:**
- Build prompt with context + citations
- LLM generation (GPT-4)
- Source attribution

**4. Production Features:**
- Caching (Redis, exact + semantic)
- Monitoring (latency, cost, quality)
- Evaluation (RAGAS metrics)
- Error handling (fallbacks)
- API layer (FastAPI with auth)

**Key Components:**
1. Document Ingestion (chunking, embedding)
2. Retrieval (hybrid + re-ranking)
3. Generation (prompt + LLM)
4. Caching (Redis)
5. Monitoring (metrics)
6. Evaluation (RAGAS)

**Interview Tip:**
> "A production RAG system has 6 components: ingestion, retrieval, generation, caching, monitoring, and evaluation. I use FastAPI for API, Pinecone for vectors, and OpenAI for embeddings. The key is measuring retrieval quality and iterating."

</details>

---

## Question 21: Production RAG Challenges

**Difficulty:** Advanced  
**Category:** Learning

What are the main challenges in production RAG systems?

<details>
<summary><b>View Answer & Explanation</b></summary>

**Answer:**

**Main Challenges:**

1. **Retrieval Quality:** Use hybrid search + re-ranking
2. **Hallucination:** Strict prompts, lower temperature, citation requirements
3. **Cost Management:** Caching, model selection, prompt optimization
4. **Latency:** Parallel processing, streaming, caching
5. **Document Updates:** Incremental updates, versioning
6. **Evaluation:** RAGAS metrics, LLM-as-judge
7. **Security:** PII detection, access control, encryption

**Solutions:**
- Hybrid search (BM25 + semantic) + re-ranking
- Strict prompts: "Answer ONLY based on context"
- Caching (30-50% cost reduction)
- Streaming for perceived latency
- Continuous evaluation with RAGAS

**Key Concepts:**
- Retrieval quality is the biggest challenge
- Hallucination requires strict prompting
- Cost through caching and model selection
- Evaluation is continuous, not one-time

**Interview Tip:**
> "The main RAG challenges are: retrieval quality (hybrid search + re-ranking), hallucination (strict prompts), cost (caching), and evaluation (RAGAS). I address each with specific techniques."

</details>

---

## Question 22: LLM Application Security

**Difficulty:** Advanced  
**Category:** Learning

What are the security considerations for LLM applications?

<details>
<summary><b>View Answer & Explanation</b></summary>

**Answer:**

**Security Threats:**

1. **Prompt Injection:** Direct and indirect
2. **Data Leakage:** Sensitive info in outputs
3. **Jailbreaking:** Bypass safety guidelines
4. **API Key Exposure:** Hardcoded keys
5. **Cost Attacks:** Expensive requests
6. **DoS:** Overwhelm system

**Prevention:**

1. **Input Validation:** Pattern detection, length limits
2. **Output Validation:** Check for sensitive data
3. **Rate Limiting:** Per user/IP
4. **API Key Security:** Environment variables, secret managers
5. **Content Moderation:** OpenAI Moderation API
6. **Monitoring:** Log all interactions

**Key Concepts:**
- Defense in depth (multiple layers)
- Input AND output validation
- Rate limiting and cost controls
- Secure API key management
- Compliance from design

**Interview Tip:**
> "LLM security requires defense in depth: input validation, output filtering, rate limiting, authentication, encryption, and monitoring. The key is assuming adversarial input and designing defensively."

</details>

---

## Question 23: LLM Observability Tools

**Difficulty:** Intermediate  
**Category:** Learning

Compare LLM observability tools: LangSmith, Langfuse, Helicone.

<details>
<summary><b>View Answer & Explanation</b></summary>

**Answer:**

**Comparison:**

1. **LangSmith:** Best LangChain integration, great UI, commercial
2. **Langfuse:** Open-source, self-hostable, flexible
3. **Helicone:** Cloud proxy, zero code changes, simple

**Key Features:**
- Tracing (full execution flow)
- Logging (all LLM calls)
- Evaluation (quality metrics)
- Monitoring (real-time dashboards)
- User feedback (ratings)

**Decision:**
- LangChain projects → LangSmith
- Self-hosted needs → Langfuse
- Quick setup → Helicone

**Key Concepts:**
- Observability is critical for production
- Tracing shows execution flow
- Evaluation measures quality
- Feedback enables improvement

**Interview Tip:**
> "I use LangSmith for LangChain projects, Langfuse for self-hosted needs, Helicone for quick setup. The key features: tracing, logging, evaluation, and feedback. Observability from day one is essential."

</details>

---

## Question 24: LLM Cost Optimization Deep Dive

**Difficulty:** Advanced  
**Category:** Learning

Walk through a detailed cost optimization strategy for 10M requests/month.

<details>
<summary><b>View Answer & Explanation</b></summary>

**Answer:**

**Baseline:** 10M requests, 500 input + 200 output tokens
- GPT-4: $270,000/month

**Optimization Strategy:**

1. **Model Selection:** Route by complexity
   - 60% simple → GPT-3.5
   - 30% medium → GPT-4-turbo
   - 10% complex → GPT-4
   - **Savings: 75% reduction**

2. **Caching (30% hit rate):** Additional 30% savings
   - Semantic caching for similar queries
   - Exact caching for identical queries

3. **Prompt Optimization:** Concise prompts (60% reduction in input tokens)

4. **Batch Processing:** 5% savings

5. **Fine-tuning:** For high-volume repetitive tasks

**Final Cost:** ~$31,000/month (88% reduction from baseline)

**Key Concepts:**
- Model selection is the biggest factor (20x cost difference)
- Caching provides 30-50% savings
- Prompt optimization reduces tokens
- Fine-tuning enables cheaper models

**Interview Tip:**
> "I optimize costs with model routing, semantic caching, prompt optimization, and batch processing. For high-volume tasks, I fine-tune smaller models. The biggest win is choosing the right model."

</details>

---

## Question 25: Real-World Agent Design - Your AMS Platform

**Difficulty:** Advanced  
**Category:** Strength Validation

Walk through the architecture of your Agentic AMS Platform. How would you improve it?

<details>
<summary><b>View Answer & Explanation</b></summary>

**Answer:**

**Your Current Architecture:**

1. **Observability Agent:** Reads Grafana alerts via WebSocket, checks ServiceNow for duplicate tickets
2. **RAG Pipeline:** Qdrant vector DB with SOPs from Confluence
3. **Planning Agent:** Creates remediation plan from incident + SOP
4. **Human-in-the-Loop:** Approval gate before execution
5. **Executor Agent:** Runs plan steps using tools (HttpRequest, JenkinsRollback, ReadPodStatus)
6. **Validator Agent:** Verifies each step, triggers retry on failure
7. **Resolution:** Closes ServiceNow ticket

**Strengths:**
- Clear separation of concerns
- Human approval for critical actions
- Validation loop ensures correctness
- Deduplication prevents duplicate processing

**Potential Improvements:**

1. **Better Error Handling:** Circuit breakers for external tools
2. **Cost Optimization:** Use GPT-3.5 for simple validation steps
3. **Observability:** Add tracing (LangSmith) for debugging
4. **Testing:** Add unit tests for each agent
5. **Caching:** Cache SOPs and similar incidents
6. **Metrics:** Track resolution time, success rate, false positives
7. **Fallback:** If validator fails repeatedly, escalate to human

**Key Concepts:**
- Multi-agent orchestration with specialized roles
- RAG for institutional knowledge
- Human-in-the-loop for trust
- Validation loop for correctness

**Interview Tip:**
> "My AMS Platform uses multi-agent orchestration with RAG for SOP retrieval, human approval before execution, and validation loops for correctness. I'd improve it with better error handling (circuit breakers), cost optimization (GPT-3.5 for simple steps), and observability (LangSmith tracing)."

</details>

---

## Summary Checklist

- [x] RAG architecture and optimization
- [x] Agent patterns (ReAct, Plan-Exec-Val, Multi-Agent)
- [x] Vector database selection
- [x] LLM cost optimization
- [x] Prompt engineering techniques
- [x] Agent error handling
- [x] Embedding models and vector search
- [x] LLM evaluation and monitoring
- [x] Multi-agent communication
- [x] Fine-tuning vs prompting vs RAG
- [x] Streaming and real-time LLM
- [x] Context window management
- [x] AutoGen workflows
- [x] Prompt injection and security
- [x] GraphRAG and knowledge graphs
- [x] LLM application architecture
- [x] Open-source vs OpenAI
- [x] Structured output and function calling
- [x] Agentic workflow patterns
- [x] Production RAG systems
- [x] Production RAG challenges
- [x] LLM application security
- [x] LLM observability tools
- [x] Cost optimization deep dive
- [x] Real-world agent design (AMS Platform)

**Total: 25 questions** covering RAG, agents, architecture, security, and production concerns.
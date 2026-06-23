# Mixed Scenarios Interview Questions

**Focus:** Real-world integration questions combining multiple technologies (React, Python, GenAI, Cloud)

---

## Question 1: Design a Scalable RAG Chat Application

**Difficulty:** Advanced  
**Category:** Mixed Scenarios

Design a production-grade RAG chat application from scratch. What are all the components, and how do they work together?

<details>
<summary><b>View Answer & Explanation</b></summary>

**Answer:**

**Architecture:**

```
Client (React) → CDN → Load Balancer → FastAPI Backend
                                              ↓
                                    ┌─────────┴─────────┐
                                    ↓                   ↓
                            Vector DB (Pinecone)   LLM (OpenAI)
                                    ↑                   ↑
                            Embedding API         Cache (Redis)
                                    ↑
                            Document Ingestion
                                    ↑
                            S3 (Document Storage)
```

**1. Frontend (React + Next.js):**

```typescript
// app/chat/page.tsx
'use client';

import { useState } from 'react';
import { useChat } from 'ai/react';

export default function ChatPage() {
  const { messages, input, handleInputChange, handleSubmit, isLoading } = useChat({
    api: '/api/chat',
  });

  return (
    <div className="flex flex-col h-screen">
      <div className="flex-1 overflow-y-auto p-4">
        {messages.map(m => (
          <div key={m.id} className={`mb-4 ${m.role === 'user' ? 'text-right' : ''}`}>
            <div className={`inline-block p-3 rounded-lg ${
              m.role === 'user' ? 'bg-blue-500 text-white' : 'bg-gray-200'
            }`}>
              {m.content}
            </div>
          </div>
        ))}
        {isLoading && <div>Thinking...</div>}
      </div>
      <form onSubmit={handleSubmit} className="p-4 border-t">
        <input
          value={input}
          onChange={handleInputChange}
          placeholder="Ask a question..."
          className="w-full p-2 border rounded"
        />
      </form>
    </div>
  );
}
```

**2. Backend API (FastAPI):**

```python
# main.py
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
from langchain.chat_models import ChatOpenAI
from langchain.embeddings import OpenAIEmbeddings
from langchain.vectorstores import Pinecone
from langchain.chains import RetrievalQA
import pinecone
import redis
import os

app = FastAPI()

# CORS
app.add_middleware(
    CORSMiddleware,
    allow_origins=[os.getenv("FRONTEND_URL")],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Initialize services
pinecone.init(api_key=os.getenv("PINECONE_API_KEY"))
vectorstore = Pinecone.from_existing_index(
    index_name="documents",
    embedding=OpenAIEmbeddings()
)
llm = ChatOpenAI(model="gpt-4", temperature=0)
qa_chain = RetrievalQA.from_chain_type(
    llm=llm,
    retriever=vectorstore.as_retriever(search_kwargs={"k": 5}),
    return_source_documents=True
)
cache = redis.Redis(host=os.getenv("REDIS_HOST"))

class Query(BaseModel):
    question: str
    user_id: str

class Response(BaseModel):
    answer: str
    sources: list[dict]

@app.post("/api/chat", response_model=Response)
async def chat(query: Query):
    # 1. Check cache
    cache_key = f"chat:{hash(query.question)}"
    cached = cache.get(cache_key)
    if cached:
        return Response.parse_raw(cached)
    
    # 2. RAG query
    try:
        result = qa_chain({"query": query.question})
        response = Response(
            answer=result["result"],
            sources=[
                {"content": doc.page_content, "metadata": doc.metadata}
                for doc in result["source_documents"]
            ]
        )
        
        # 3. Cache result
        cache.setex(cache_key, 3600, response.json())
        return response
    except Exception as e:
        raise HTTPException(500, str(e))

# Document ingestion
@app.post("/api/ingest")
async def ingest_documents(file_paths: list[str]):
    from langchain.document_loaders import PyPDFLoader
    from langchain.text_splitter import RecursiveCharacterTextSplitter
    
    documents = []
    for path in file_paths:
        loader = PyPDFLoader(path)
        documents.extend(loader.load())
    
    text_splitter = RecursiveCharacterTextSplitter(
        chunk_size=500,
        chunk_overlap=50
    )
    chunks = text_splitter.split_documents(documents)
    
    vectorstore.add_documents(chunks)
    return {"ingested": len(chunks)}
```

**3. Document Ingestion Pipeline:**

```python
# ingest.py
import boto3
from langchain.document_loaders import PyPDFLoader, Docx2txtLoader
from langchain.text_splitter import RecursiveCharacterTextSplitter
from langchain.embeddings import OpenAIEmbeddings
from langchain.vectorstores import Pinecone
import pinecone

s3 = boto3.client('s3')
pinecone.init(api_key=os.getenv("PINECONE_API_KEY"))
embeddings = OpenAIEmbeddings()
vectorstore = Pinecone.from_existing_index("documents", embeddings)

def process_s3_documents(bucket: str, prefix: str):
    """Process all documents in S3 bucket"""
    paginator = s3.get_paginator('list_objects_v2')
    pages = paginator.paginate(Bucket=bucket, Prefix=prefix)
    
    for page in pages:
        for obj in page.get('Contents', []):
            key = obj['Key']
            if key.endswith('.pdf'):
                loader = PyPDFLoader(f"s3://{bucket}/{key}")
            elif key.endswith('.docx'):
                loader = Docx2txtLoader(f"s3://{bucket}/{key}")
            else:
                continue
            
            documents = loader.load()
            
            # Chunk
            text_splitter = RecursiveCharacterTextSplitter(
                chunk_size=500,
                chunk_overlap=50
            )
            chunks = text_splitter.split_documents(documents)
            
            # Add metadata
            for chunk in chunks:
                chunk.metadata['source'] = key
                chunk.metadata['ingested_at'] = datetime.now().isoformat()
            
            # Embed and store
            vectorstore.add_documents(chunks)

if __name__ == "__main__":
    process_s3_documents("my-documents", "documents/")
```

**4. Deployment (Docker + Cloud Run):**

```dockerfile
# Dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

USER appuser

EXPOSE 8080
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8080"]
```

**5. Monitoring:**

```python
# Add metrics
from prometheus_fastapi_instrumentator import Instrumentator
Instrumentator().instrument(app).expose(app)

# Custom metrics
from prometheus_client import Counter, Histogram

chat_requests = Counter('chat_requests_total', 'Total chat requests')
chat_duration = Histogram('chat_duration_seconds', 'Chat duration')

@app.post("/api/chat")
async def chat(query: Query):
    chat_requests.inc()
    with chat_duration.time():
        # ... existing code
        pass
```

**Key Components:**

1. **Frontend:** React/Next.js with streaming chat
2. **Backend:** FastAPI with RAG pipeline
3. **Vector DB:** Pinecone for embeddings
4. **LLM:** OpenAI GPT-4
5. **Caching:** Redis for performance
6. **Storage:** S3 for documents
7. **Deployment:** Docker on Cloud Run
8. **Monitoring:** Prometheus + Grafana

**Common Mistakes:**
- No streaming (poor perceived performance)
- No caching (expensive LLM calls)
- No monitoring (blind to issues)
- Synchronous document processing (slow ingestion)
- No error handling (crashes on bad input)

**Interview Tip:**
> "A production RAG chat app has 5 layers: frontend (React with streaming), backend (FastAPI with RAG), vector DB (Pinecone), LLM (OpenAI), and supporting services (Redis cache, S3 storage, monitoring). I use Next.js for the frontend with the AI SDK for streaming, FastAPI for the backend with LangChain for RAG, and Docker for deployment. The key is streaming for perceived performance, caching for cost, and monitoring for reliability."

</details>

---

## Question 2: Build an AI-Powered Code Review System

**Difficulty:** Advanced  
**Category:** Mixed Scenarios

Design an AI-powered code review system that integrates with GitHub. What agents and workflows are needed?

<details>
<summary><b>View Answer & Explanation</b></summary>

**Answer:**

**Architecture:**

```
GitHub PR → Webhook → API → Code Review Agent
                                      ↓
                        ┌─────────────┼─────────────┐
                        ↓             ↓             ↓
                  Style Agent   Security Agent  Logic Agent
                        ↓             ↓             ↓
                        └─────────────┼─────────────┘
                                      ↓
                              Aggregator Agent
                                      ↓
                            GitHub PR Comment
```

**1. GitHub Webhook Handler:**

```python
# main.py
from fastapi import FastAPI, Request, HTTPException
import hmac
import hashlib
import os

app = FastAPI()

GITHUB_WEBHOOK_SECRET = os.getenv("GITHUB_WEBHOOK_SECRET")

@app.post(\"/webhook/github\")
async def github_webhook(request: Request):
    # Verify signature
    signature = request.headers.get(\"X-Hub-Signature-256\")
    if not signature:
        raise HTTPException(401, \"Missing signature\")
    
    body = await request.body()
    expected = \"sha256=\" + hmac.new(
        GITHUB_WEBHOOK_SECRET.encode(),
        body,
        hashlib.sha256
    ).hexdigest()
    
    if not hmac.compare_digest(signature, expected):
        raise HTTPException(401, \"Invalid signature\")
    
    event = request.headers.get(\"X-GitHub-Event\")
    payload = await request.json()
    
    if event == \"pull_request\":
        await handle_pr_opened(payload)
    elif event == \"pull_request_review_comment\":
        await handle_review_comment(payload)
    
    return {\"status\": \"ok\"}
```

**2. Multi-Agent Code Review System:**

```python
# agents.py
from autogen import ConversableAgent, GroupChat, GroupChatManager
import os

# Configure LLM
llm_config = {
    \"model\": \"gpt-4\",
    \"api_key\": os.getenv(\"OPENAI_API_KEY\")
}

# Style Agent
style_agent = ConversableAgent(
    name=\"StyleReviewer\",
    system_message=\"\"\"You are a code style reviewer. Check for:
    - PEP 8 compliance
    - Naming conventions
    - Code formatting
    - Import organization
    - Docstrings
    
    Provide specific line numbers and suggestions.\"\"\",
    llm_config=llm_config,
    human_input_mode=\"NEVER\"
)

# Security Agent
security_agent = ConversableAgent(
    name=\"SecurityReviewer\",
    system_message=\"\"\"You are a security code reviewer. Check for:
    - SQL injection
    - XSS vulnerabilities
    - Hardcoded secrets
    - Insecure dependencies
    - Authentication issues
    - Authorization flaws
    
    Flag CRITICAL, HIGH, MEDIUM, LOW severity issues.\"\"\",
    llm_config=llm_config,
    human_input_mode=\"NEVER\"
)

# Logic Agent
logic_agent = ConversableAgent(
    name=\"LogicReviewer\",
    system_message=\"\"\"You are a logic and best practices reviewer. Check for:
    - Bug patterns
    - Error handling
    - Edge cases
    - Performance issues
    - Design patterns
    - SOLID principles
    
    Suggest improvements with code examples.\"\"\",
    llm_config=llm_config,
    human_input_mode=\"NEVER\"
)

# Test Coverage Agent
test_agent = ConversableAgent(
    name=\"TestReviewer\",
    system_message=\"\"\"You are a test coverage reviewer. Check for:
    - Missing test cases
    - Edge case coverage
    - Test quality
    - Mock usage
    - Integration test needs
    
    Suggest specific test cases to add.\"\"\",
    llm_config=llm_config,
    human_input_mode=\"NEVER\"
)

# Aggregator Agent
aggregator_agent = ConversableAgent(
    name=\"Aggregator\",
    system_message=\"\"\"You aggregate feedback from all reviewers.
    Create a comprehensive code review with:
    - Summary
    - Critical issues (must fix)
    - High priority issues
    - Suggestions
    - Positive feedback
    
    Format as GitHub markdown comment.\"\"\",
    llm_config=llm_config,
    human_input_mode=\"NEVER\"
)

# Group Chat
group_chat = GroupChat(
    agents=[style_agent, security_agent, logic_agent, test_agent, aggregator_agent],
    messages=[],
    max_round=10,
    speaker_selection_method=\"round_robin\"
)

manager = GroupChatManager(groupchat=group_chat, llm_config=llm_config)
```

**3. Code Analysis Tool:**

```python
# tools.py
import ast
import subprocess
from typing import List, Dict

class CodeAnalyzer:
    def __init__(self, repo_path: str):
        self.repo_path = repo_path
    
    def get_changed_files(self, base_sha: str, head_sha: str) -> List[Dict]:
        \"\"\"Get list of changed files in PR\"\"\"
        result = subprocess.run(
            [\"git\", \"diff\", \"--name-status\", base_sha, head_sha],
            cwd=self.repo_path,
            capture_output=True,
            text=True
        )
        
        files = []
        for line in result.stdout.strip().split('\\n'):
            if not line:
                continue
            status, path = line.split('\\t')
            files.append({\"status\": status, \"path\": path})
        
        return files
    
    def get_diff(self, base_sha: str, head_sha: str, file_path: str) -> str:
        \"\"\"Get diff for specific file\"\"\"
        result = subprocess.run(
            [\"git\", \"diff\", base_sha, head_sha, \"--\", file_path],
            cwd=self.repo_path,
            capture_output=True,
            text=True
        )
        return result.stdout
    
    def run_linters(self, file_path: str) -> Dict:
        \"\"\"Run linters and return issues\"\"\"
        issues = {\"pylint\": [], \"flake8\": [], \"mypy\": []}
        
        # Pylint
        result = subprocess.run(
            [\"pylint\", file_path, \"--output-format=json\"],
            capture_output=True,
            text=True
        )
        if result.stdout:
            import json
            issues[\"pylint\"] = json.loads(result.stdout)
        
        # Flake8
        result = subprocess.run(
            [\"flake8\", file_path, \"--format=json\"],
            capture_output=True,
            text=True
        )
        if result.stdout:
            import json
            issues[\"flake8\"] = json.loads(result.stdout)
        
        # MyPy
        result = subprocess.run(
            [\"mypy\", file_path, \"--json-output\", \"/dev/stdout\"],
            capture_output=True,
            text=True
        )
        if result.stdout:
            import json
            issues[\"mypy\"] = json.loads(result.stdout)
        
        return issues
    
    def get_file_content(self, file_path: str, sha: str) -> str:
        \"\"\"Get file content at specific SHA\"\"\"
        result = subprocess.run(
            [\"git\", \"show\", f\"{sha}:{file_path}\"],
            cwd=self.repo_path,
            capture_output=True,
            text=True
        )
        return result.stdout
```

**4. Review Orchestrator:**

```python
# reviewer.py
from agents import style_agent, security_agent, logic_agent, test_agent, aggregator_agent
from tools import CodeAnalyzer
import json

class CodeReviewOrchestrator:
    def __init__(self):
        self.analyzer = CodeAnalyzer(os.getenv(\"REPO_PATH\"))
    
    async def review_pr(self, pr_data: dict) -> dict:
        \"\"\"Orchestrate code review for a PR\"\"\"
        # 1. Get changed files
        base_sha = pr_data[\"base\"][\"sha\"]
        head_sha = pr_data[\"head\"][\"sha\"]
        files = self.analyzer.get_changed_files(base_sha, head_sha)
        
        # 2. Analyze each file
        file_analyses = []
        for file_info in files:
            if not file_info[\"path\"].endswith((\".py\", \".js\", \".ts\")):
                continue
            
            diff = self.analyzer.get_diff(base_sha, head_sha, file_info[\"path\"])
            linter_issues = self.analyzer.run_linters(file_info[\"path\"])
            
            file_analyses.append({
                \"file\": file_info[\"path\"],
                \"diff\": diff,
                \"linter_issues\": linter_issues
            })
        
        # 3. Run agents in parallel
        reviews = await asyncio.gather(
            self.run_style_review(file_analyses),
            self.run_security_review(file_analyses),
            self.run_logic_review(file_analyses),
            self.run_test_review(file_analyses)
        )
        
        # 4. Aggregate results
        aggregated = self.aggregate_reviews(reviews)
        
        return aggregated
    
    async def run_style_review(self, analyses):
        prompt = f\"Review code style:\\n\\n{json.dumps(analyses, indent=2)}\"
        return await style_agent.a_generate_reply(
            messages=[{\"role\": \"user\", \"content\": prompt}]
        )
    
    # Similar methods for other agents...
    
    def aggregate_reviews(self, reviews):
        # Use aggregator agent
        prompt = f\"Aggregate these reviews:\\n\\n{json.dumps(reviews, indent=2)}\"
        return aggregator_agent.generate_reply(
            messages=[{\"role\": \"user\", \"content\": prompt}]
        )
```

**5. GitHub Comment Poster:**

```python
# github_client.py
import requests

class GitHubClient:
    def __init__(self, token: str):
        self.token = token
        self.base_url = \"https://api.github.com\"
    
    def post_review_comment(self, owner: str, repo: str, pr_number: int, review: dict):
        \"\"\"Post review as PR comment\"\"\"
        url = f\"{self.base_url}/repos/{owner}/{repo}/issues/{pr_number}/comments\"
        headers = {
            \"Authorization\": f\"token {self.token}\",
            \"Accept\": \"application/vnd.github.v3+json\"
        }
        
        response = requests.post(url, json={\"body\": review[\"comment\"]}, headers=headers)
        response.raise_for_status()
        return response.json()
    
    def post_inline_comment(self, owner: str, repo: str, pr_number: int, 
                           commit_sha: str, file_path: str, line: int, comment: str):
        \"\"\"Post inline comment on specific line\"\"\"
        url = f\"{self.base_url}/repos/{owner}/{repo}/pulls/{pr_number}/reviews\"
        headers = {
            \"Authorization\": f\"token {self.token}\",
            \"Accept\": \"application/vnd.github.v3+json\"
        }
        
        data = {
            \"commit_id\": commit_sha,
            \"event\": \"COMMENT\",
            \"comments\": [{
                \"path\": file_path,
                \"line\": line,
                \"body\": comment
            }]
        }
        
        response = requests.post(url, json=data, headers=headers)
        response.raise_for_status()
```

**Key Components:**

1. **GitHub Webhook:** Triggered on PR events
2. **Multi-Agent System:** Specialized reviewers (style, security, logic, tests)
3. **Code Analysis Tools:** Linters, diff analysis
4. **Aggregator Agent:** Combines feedback
5. **GitHub API:** Posts comments back to PR

**Workflow:**

1. Developer opens PR
2. GitHub sends webhook
3. System clones repo, analyzes diff
4. Specialized agents review in parallel
5. Aggregator creates comprehensive review
6. System posts comment to PR
7. Developer addresses feedback
8. Loop until approved

**Common Mistakes:**
- Not verifying webhook signature (security)
- Running agents sequentially (slow)
- No caching of results
- Posting too many comments (noise)
- Not handling large diffs (token limits)

**Interview Tip:**
> "An AI code review system uses multiple specialized agents (style, security, logic, tests) coordinated through a group chat. The system analyzes the diff, runs each agent in parallel, aggregates feedback, and posts to GitHub. The key is webhook security, parallel agent execution, and focused reviews (one agent per concern). I use AutoGen for agent coordination and run linters (pylint, mypy) to augment LLM reviews with deterministic checks."

</details>

---

## Question 3: Real-Time AI Dashboard with Streaming

**Difficulty:** Advanced  
**Category:** Mixed Scenarios

Build a real-time AI dashboard that streams insights from multiple data sources. How do you handle streaming, state management, and real-time updates?

<details>
<summary><b>View Answer & Explanation</b></summary>

**Answer:**

**Architecture:**

```
Data Sources → Stream Processor → WebSocket → React Dashboard
     ↓               ↓                ↓            ↓
  (APIs)        (FastAPI)        (Real-time)   (Live Updates)
```

**1. Backend: Stream Processor (FastAPI + WebSocket):**

```python
# main.py
from fastapi import FastAPI, WebSocket, WebSocketDisconnect
from fastapi.responses import StreamingResponse
import asyncio
import json
from typing import AsyncGenerator

app = FastAPI()

class ConnectionManager:
    def __init__(self):
        self.active_connections: list[WebSocket] = []
    
    async def connect(self, websocket: WebSocket):
        await websocket.accept()
        self.active_connections.append(websocket)
    
    def disconnect(self, websocket: WebSocket):
        self.active_connections.remove(websocket)
    
    async def broadcast(self, message: dict):
        for connection in self.active_connections:
            await connection.send_json(message)

manager = ConnectionManager()

# Simulated data source
async def data_stream() -> AsyncGenerator[dict, None]:
    \"\"\"Stream insights from multiple sources\"\"\"
    while True:
        # Simulate data from different sources
        insight = {
            \"timestamp\": datetime.now().isoformat(),
            \"source\": random.choice([\"sales\", \"analytics\", \"ml_model\"]),
            \"metric\": random.choice([\"revenue\", \"users\", \"conversion\", \"churn\"]),
            \"value\": random.uniform(0, 1000),
            \"anomaly\": random.random() < 0.1
        }
        yield insight
        await asyncio.sleep(1)

@app.websocket(\"/ws/insights\")
async def websocket_endpoint(websocket: WebSocket):
    await manager.connect(websocket)
    try:
        async for insight in data_stream():
            await websocket.send_json(insight)
    except WebSocketDisconnect:
        manager.disconnect(websocket)

# Server-Sent Events alternative
@app.get(\"/stream/insights\")
async def stream_insights():
    async def event_generator():
        async for insight in data_stream():
            yield f\"data: {json.dumps(insight)}\\n\\n\"
    
    return StreamingResponse(
        event_generator(),
        media_type=\"text/event-stream\",
        headers={
            \"Cache-Control\": \"no-cache\",
            \"X-Accel-Buffering\": \"no\"
        }
    )
```

**2. AI-Powered Insights:**

```python
# ai_insights.py
from openai import AsyncOpenAI
import asyncio

client = AsyncOpenAI()

async def generate_ai_insight(data: dict) -> str:
    \"\"\"Generate natural language insight from data\"\"\"
    prompt = f\"\"\"Analyze this metric and provide a brief insight (1-2 sentences):
    
    Metric: {data['metric']}
    Value: {data['value']}
    Anomaly: {data['anomaly']}
    
    Insight:\"\"\"
    
    response = await client.chat.completions.create(
        model=\"gpt-3.5-turbo\",
        messages=[{\"role\": \"user\", \"content\": prompt}],
        max_tokens=100,
        temperature=0.7
    )
    
    return response.choices[0].message.content

async def stream_insights_with_ai():
    \"\"\"Stream data with AI-generated insights\"\"\"
    async for data in data_stream():
        # Generate AI insight (non-blocking)
        insight_task = asyncio.create_task(generate_ai_insight(data))
        
        # Send raw data immediately
        yield {\"type\": \"data\", \"data\": data}
        
        # Send AI insight when ready
        insight = await insight_task
        yield {\"type\": \"ai_insight\", \"insight\": insight, \"for\": data}
```

**3. Frontend: React Dashboard with Real-Time Updates:**

```typescript
// app/dashboard/page.tsx
'use client';

import { useEffect, useState } from 'react';
import { LineChart, BarChart } from 'recharts';

interface Insight {
  timestamp: string;
  source: string;
  metric: string;
  value: number;
  anomaly: boolean;
  aiInsight?: string;
}

export default function Dashboard() {
  const [insights, setInsights] = useState<Insight[]>([]);
  const [aiInsights, setAiInsights] = useState<Map<string, string>>(new Map());
  const [ws, setWs] = useState<WebSocket | null>(null);
  
  useEffect(() => {
    // Connect to WebSocket
    const websocket = new WebSocket('ws://localhost:8000/ws/insights');
    
    websocket.onmessage = (event) => {
      const message = JSON.parse(event.data);
      
      if (message.type === 'data') {
        setInsights(prev => [...prev.slice(-49), message.data]);
      } else if (message.type === 'ai_insight') {
        setAiInsights(prev => new Map(prev).set(
          message.for.timestamp,
          message.insight
        ));
      }
    };
    
    setWs(websocket);
    
    return () => websocket.close();
  }, []);
  
  // Prepare chart data
  const chartData = insights.map(i => ({
    time: new Date(i.timestamp).toLocaleTimeString(),
    value: i.value,
    anomaly: i.anomaly
  }));
  
  return (
    <div className=\"p-6\">
      <h1 className=\"text-3xl font-bold mb-6\">Real-Time AI Dashboard</h1>
      
      {/* Live Metrics */}
      <div className=\"grid grid-cols-4 gap-4 mb-6\">
        <MetricCard
          title=\"Total Insights\"
          value={insights.length}
        />
        <MetricCard
          title=\"Anomalies Detected\"
          value={insights.filter(i => i.anomaly).length}
        />
        <MetricCard
          title=\"Avg Value\"
          value={insights.length > 0 
            ? (insights.reduce((sum, i) => sum + i.value, 0) / insights.length).toFixed(2)
            : 0
          }
        />
        <MetricCard
          title=\"AI Insights\"
          value={aiInsights.size}
        />
      </div>
      
      {/* Real-Time Chart */}
      <div className=\"bg-white p-4 rounded-lg shadow mb-6\">
        <h2 className=\"text-xl font-semibold mb-4\">Live Metrics</h2>
        <LineChart width={800} height={300} data={chartData}>
          <XAxis dataKey=\"time\" />
          <YAxis />
          <Tooltip />
          <Line type=\"monotone\" dataKey=\"value\" stroke=\"#3b82f6\" />
        </LineChart>
      </div>
      
      {/* Recent Insights with AI Commentary */}
      <div className=\"bg-white p-4 rounded-lg shadow\">
        <h2 className=\"text-xl font-semibold mb-4\">Recent Insights</h2>
        <div className=\"space-y-3\">
          {insights.slice(-10).reverse().map((insight, idx) => (
            <div
              key={idx}
              className={`p-3 border-l-4 ${
                insight.anomaly ? 'border-red-500 bg-red-50' : 'border-blue-500'
              }`}
            >
              <div className=\"flex justify-between\">
                <span className=\"font-semibold\">{insight.metric}</span>
                <span className=\"text-sm text-gray-500\">
                  {new Date(insight.timestamp).toLocaleTimeString()}
                </span>
              </div>
              <div className=\"text-sm\">
                Value: {insight.value.toFixed(2)} | Source: {insight.source}
              </div>
              {aiInsights.get(insight.timestamp) && (
                <div className=\"mt-2 text-sm italic text-gray-700\">
                  AI: {aiInsights.get(insight.timestamp)}
                </div>
              )}
            </div>
          ))}
        </div>
      </div>
    </div>
  );
}

function MetricCard({ title, value }: { title: string; value: number | string }) {
  return (
    <div className=\"bg-white p-4 rounded-lg shadow\">
      <div className=\"text-sm text-gray-600\">{title}</div>
      <div className=\"text-2xl font-bold\">{value}</div>
    </div>
  );
}
```

**4. State Management with Zustand:**

```typescript
// store.ts
import { create } from 'zustand';

interface InsightsStore {
  insights: Insight[];
  addInsight: (insight: Insight) => void;
  addAIInsight: (timestamp: string, insight: string) => void;
  clear: () => void;
}

export const useInsightsStore = create<InsightsStore>((set) => ({
  insights: [],
  addInsight: (insight) => set((state) => ({
    insights: [...state.insights.slice(-99), insight]
  })),
  addAIInsight: (timestamp, insight) => set((state) => {
    const updated = state.insights.map(i =>
      i.timestamp === timestamp ? { ...i, aiInsight: insight } : i
    );
    return { insights: updated };
  }),
  clear: () => set({ insights: [] })
}));
```

**5. Performance Optimization:**

```typescript
// useDebounce.ts
import { useEffect, useState } from 'react';

export function useDebounce<T>(value: T, delay: number): T {
  const [debouncedValue, setDebouncedValue] = useState(value);
  
  useEffect(() => {
    const timer = setTimeout(() => setDebouncedValue(value), delay);
    return () => clearTimeout(timer);
  }, [value, delay]);
  
  return debouncedValue;
}

// Usage: Debounce AI insight rendering
const debouncedInsights = useDebounce(insights, 500);
```

**Key Components:**

1. **Backend:** FastAPI WebSocket/SSE for streaming
2. **AI Layer:** Async OpenAI for insight generation
3. **Frontend:** React with WebSocket client
4. **State Management:** Zustand for global state
5. **Visualization:** Recharts for real-time charts
6. **Performance:** Debouncing, memoization

**Performance Considerations:**

- **Batch Updates:** Buffer insights, update UI every 500ms
- **Virtual Scrolling:** Only render visible insights
- **Throttle WebSocket:** Limit re-renders
- **Memoization:** React.memo for expensive components
- **Backpressure:** Handle slow consumers

**Common Mistakes:**
- Updating state on every message (re-render storm)
- No throttling/debouncing (poor performance)
- Blocking event loop with sync AI calls
- No error handling for WebSocket disconnections
- Not handling backpressure

**Interview Tip:**
> "For real-time AI dashboards, I use WebSocket or Server-Sent Events for streaming, FastAPI with async generators on the backend, and React with debounced state updates on the frontend. I batch UI updates (update every 500ms instead of every message) to prevent re-render storms. For AI insights, I generate them asynchronously and send as separate events so raw data appears immediately. The key is throttling, backpressure handling, and graceful reconnection."

</details>

---

## Question 4: End-to-End ML Model Deployment

**Difficulty:** Advanced  
**Category:** Mixed Scenarios

Deploy a machine learning model to production with FastAPI, Docker, and Kubernetes. What are all the steps and considerations?

<details>
<summary><b>View Answer & Explanation</b></summary>

**Answer:**

**Complete ML Deployment Pipeline:**

```
Training → Model Registry → API → Monitoring → Retraining
    ↓            ↓          ↓        ↓           ↓
  (Python)   (MLflow)  (FastAPI) (Prometheus)  (Airflow)
```

**1. Model Training with MLflow:**

```python
# train.py
import mlflow
import mlflow.sklearn
from sklearn.ensemble import RandomForestClassifier
from sklearn.metrics import accuracy_score, f1_score
from sklearn.model_selection import train_test_split
import pandas as pd

# Set tracking URI
mlflow.set_tracking_uri(\"http://mlflow-server:5000\")

# Load data
df = pd.read_csv(\"data/training_data.csv\")
X = df.drop(\"target\", axis=1)
y = df[\"target\"]

X_train, X_test, y_train, y_test = train_test_split(
    X, y, test_size=0.2, random_state=42
)

# Start run
with mlflow.start_run():
    # Train model
    params = {
        \"n_estimators\": 100,
        \"max_depth\": 10,
        \"min_samples_split\": 5
    }
    
    model = RandomForestClassifier(**params)
    model.fit(X_train, y_train)
    
    # Evaluate
    y_pred = model.predict(X_test)
    metrics = {
        \"accuracy\": accuracy_score(y_test, y_pred),
        \"f1_score\": f1_score(y_test, y_pred, average=\"weighted\")
    }
    
    # Log
    mlflow.log_params(params)
    mlflow.log_metrics(metrics)
    mlflow.sklearn.log_model(
        model,
        \"model\",
        registered_model_name=\"my-classifier\"
    )
    
    print(f\"Accuracy: {metrics['accuracy']:.4f}\")
```

**2. Model Loading and Serving:**

```python
# model.py
import mlflow
import mlflow.pyfunc
import numpy as np
from typing import List

class ModelService:
    def __init__(self, model_name: str, model_stage: str = \"Production\"):
        self.model = mlflow.pyfunc.load_model(
            model_uri=f\"models:/{model_name}/{model_stage}\"
        )
        self.model_name = model_name
        self.model_stage = model_stage
    
    def predict(self, features: np.ndarray) -> np.ndarray:
        return self.model.predict(features)
    
    def predict_proba(self, features: np.ndarray) -> np.ndarray:
        return self.model.predict_proba(features)

# Singleton
model_service = ModelService(\"my-classifier\", \"Production\")
```

**3. FastAPI Application:**

```python
# main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel, Field
from prometheus_client import Counter, Histogram
from prometheus_fastapi_instrumentator import Instrumentator
import numpy as np
import time

app = FastAPI(title=\"ML Model API\")

# Metrics
prediction_counter = Counter(
    'model_predictions_total',
    'Total predictions',
    ['model_version', 'status']
)
prediction_duration = Histogram(
    'model_prediction_duration_seconds',
    'Prediction duration'
)

# Instrument automatically
Instrumentator().instrument(app).expose(app)

# Request/Response models
class PredictionRequest(BaseModel):
    features: List[float] = Field(..., min_items=10, max_items=10)
    
    class Config:
        json_schema_extra = {
            \"example\": {
                \"features\": [0.1, 0.2, 0.3, 0.4, 0.5, 0.6, 0.7, 0.8, 0.9, 1.0]
            }
        }

class PredictionResponse(BaseModel):
    prediction: int
    probability: float
    model_version: str
    inference_time_ms: float

# Health check
@app.get(\"/health\")
async def health():
    return {\"status\": \"healthy\", \"model\": \"my-classifier\"}

# Prediction endpoint
@app.post(\"/predict\", response_model=PredictionResponse)
async def predict(request: PredictionRequest):
    start = time.time()
    try:
        # Preprocess
        features = np.array([request.features])
        
        # Predict
        prediction = model_service.predict(features)[0]
        probabilities = model_service.predict_proba(features)[0]
        confidence = float(max(probabilities))
        
        inference_time = (time.time() - start) * 1000
        
        # Track metrics
        prediction_counter.labels(
            model_version=\"v1\",
            status=\"success\"
        ).inc()
        
        return PredictionResponse(
            prediction=int(prediction),
            probability=confidence,
            model_version=\"v1\",
            inference_time_ms=inference_time
        )
    except Exception as e:
        prediction_counter.labels(
            model_version=\"v1\",
            status=\"error\"
        ).inc()
        raise HTTPException(500, str(e))

# Batch prediction
@app.post(\"/predict/batch\")
async def predict_batch(requests: List[PredictionRequest]):
    features = np.array([r.features for r in requests])
    predictions = model_service.predict(features)
    probabilities = model_service.predict_proba(features)
    
    return {
        \"predictions\": predictions.tolist(),
        \"probabilities\": probabilities.tolist()
    }
```

**4. Model Monitoring (Data Drift Detection):**

```python
# monitoring.py
from prometheus_client import Gauge
import numpy as np
from collections import deque

class DataDriftMonitor:
    def __init__(self, window_size: int = 1000):
        self.feature_stats = {
            i: {\"mean\": 0, \"std\": 1, \"values\": deque(maxlen=window_size)}
            for i in range(10)
        }
        self.drift_gauge = Gauge(
            'model_data_drift',
            'Data drift detected',
            ['feature']
        )
    
    def update(self, features: np.ndarray):
        for i, value in enumerate(features):
            values = self.feature_stats[i][\"values\"]
            values.append(value)
            
            if len(values) > 10:
                mean = np.mean(values)
                std = np.std(values)
                \n                # Check for drift (3 sigma rule)
                if abs(value - mean) > 3 * std:
                    self.drift_gauge.labels(feature=f\"f{i}\").set(1)
                else:
                    self.drift_gauge.labels(feature=f\"f{i}\").set(0)

drift_monitor = DataDriftMonitor()

@app.post(\"/predict\")
async def predict(request: PredictionRequest):
    features = np.array([request.features])[0]
    drift_monitor.update(features)
    # ... rest of prediction
```

**5. A/B Testing:**

```python
# ab_testing.py
import random
from typing import Dict

class ABTestRouter:
    def __init__(self):
        self.variants = {
            \"model_v1\": ModelService(\"my-classifier\", \"Production\"),
            \"model_v2\": ModelService(\"my-classifier\", \"Staging\")
        }
        self.traffic_split = {\"model_v1\": 0.9, \"model_v2\": 0.1}
    
    def route(self) -> str:
        rand = random.random()
        cumulative = 0
        for variant, percentage in self.traffic_split.items():
            cumulative += percentage
            if rand < cumulative:
                return variant
        return \"model_v1\"

ab_router = ABTestRouter()

@app.post(\"/predict\")
async def predict(request: PredictionRequest):
    variant = ab_router.route()
    model = ab_router.variants[variant]
    
    # Track which variant
    prediction_counter.labels(
        model_version=variant,
        status=\"success\"
    ).inc()
    
    # Predict with selected model
    features = np.array([request.features])
    prediction = model.predict(features)[0]
    # ...
```

**6. Docker Deployment:**

```dockerfile
# Dockerfile
FROM python:3.11-slim

WORKDIR /app

# Install dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application
COPY . .

# Model files (or download from S3)
COPY models/ ./models/

# Non-root user
RUN useradd -m -u 1000 appuser && chown -R appuser:appuser /app
USER appuser

EXPOSE 8000

CMD [\"uvicorn\", \"main:app\", \"--host\", \"0.0.0.0\", \"--port\", \"8000\", \"--workers\", \"4\"]
```

**7. Kubernetes Deployment:**

```yaml
# k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ml-model-api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: ml-model-api
  template:
    metadata:
      labels:
        app: ml-model-api
    spec:
      containers:
      - name: api
        image: myregistry/ml-model-api:v1
        ports:
        - containerPort: 8000
        resources:
          requests:
            memory: \"512Mi\"
            cpu: \"500m\"
          limits:
            memory: \"2Gi\"
            cpu: \"2000m\"
        env:
        - name: MLFLOW_TRACKING_URI
          value: \"http://mlflow:5000\"
        - name: MODEL_NAME
          value: \"my-classifier\"
        livenessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 30
        readinessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 10
---
apiVersion: v1
kind: Service
metadata:
  name: ml-model-api
spec:
  selector:
    app: ml-model-api
  ports:
  - port: 80
      targetPort: 8000
```

**8. Retraining Pipeline:**

```python
# retrain.py
import mlflow
from airflow import DAG
from airflow.operators.python import PythonOperator
from datetime import datetime, timedelta

def check_drift_and_retrain():
    \"\"\"Check for drift and trigger retraining\"\"\"
    # Query Prometheus for drift metrics
    drift_detected = query_drift_metrics()
    
    if drift_detected:
        # Trigger training
        train_model()
        # Register new model version
        register_model()
        # Deploy to staging
        deploy_to_staging()
        # Run tests
        if tests_pass():
            promote_to_production()

# Airflow DAG
dag = DAG(
    'ml_retraining',
    schedule_interval=timedelta(days=1),
    start_date=datetime(2026, 1, 1)
)

check_drift_task = PythonOperator(
    task_id='check_drift',
    python_callable=check_drift_and_retrain,
    dag=dag
)
```

**Key Components:**

1. **Training:** MLflow for experiment tracking
2. **Model Registry:** Versioned model storage
3. **Serving:** FastAPI with model loading
4. **Monitoring:** Prometheus + data drift detection
5. **Deployment:** Docker + Kubernetes
6. **Retraining:** Airflow for orchestration
7. **A/B Testing:** Traffic splitting for new models

**Common Mistakes:**
- No model versioning (can't rollback)
- No monitoring (don't know when model degrades)
- No A/B testing (risky deployments)
- Synchronous model loading (slow startup)
- No health checks (broken pods not detected)
- Missing data drift detection

**Interview Tip:**
> "For ML model deployment, I use MLflow for model registry and versioning, FastAPI for serving with async endpoints, Docker for containerization, and Kubernetes for orchestration. I add Prometheus monitoring for prediction latency and data drift detection. For safe deployments, I use A/B testing (90% traffic to production, 10% to new model) and automatic rollback if metrics degrade. The key is versioning, monitoring, and gradual rollout with retraining triggered by drift detection."

</details>

---

## Question 5: Multi-Tenant SaaS Architecture

**Difficulty:** Advanced  
**Category:** Mixed Scenarios

Design a multi-tenant SaaS application. How do you handle tenant isolation, data security, and scaling?

<details>
<summary><b>View Answer & Explanation</b></summary>

**Answer:**

**Architecture:**

```
                    ┌──────────────┐
                    │   Clients    │
                    └──────┬───────┘
                           │
                    ┌──────▼───────┐
                    │ API Gateway  │ (Auth + Tenant Routing)
                    └──────┬───────┘
                           │
        ┌──────────────────┼──────────────────┐
        │                  │                  │
   ┌────▼────┐        ┌────▼────┐        ┌────▼────┐
   │Tenant A │        │Tenant B │        │Tenant C │
   │ Service │        │ Service │        │ Service │
   └────┬────┘        └────┬────┘        └────┬────┘
        │                  │                  │
   ┌────▼────┐        ┌────▼────┐        ┌────▼────┐
   │Tenant A │        │Tenant B │        │Tenant C │
   │ Database│        │ Database│        │ Database│
   └─────────┘        └─────────┘        └─────────┘
```

**1. Tenant Isolation Strategies:**

```python
# Strategy 1: Database per Tenant (Strongest Isolation)

class TenantDatabaseRouter:
    def __init__(self):
        self.connections = {}
    
    def get_session(self, tenant_id: str):
        if tenant_id not in self.connections:
            # Create connection for new tenant
            db_url = f\"postgresql://user:pass@host/tenant_{tenant_id}\"
            engine = create_engine(db_url)
            self.connections[tenant_id] = sessionmaker(bind=engine)
        return self.connections[tenant_id]()

# Strategy 2: Schema per Tenant (Medium Isolation)

class TenantSchemaRouter:
    def get_session(self, tenant_id: str):
        # Use search_path to switch schemas
        engine = create_engine(\"postgresql://user:pass@host/main\")
        session = sessionmaker(bind=engine)()
        session.execute(f\"SET search_path TO tenant_{tenant_id}, public\")
        return session

# Strategy 3: Row-Level Security (Weakest but Cheapest)

# In PostgreSQL
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    tenant_id INT NOT NULL,
    name TEXT NOT NULL
);

ALTER TABLE users ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON users
    USING (tenant_id = current_setting('app.current_tenant')::INT);

# In application
session.execute(\"SET app.current_tenant = :tenant_id\", {\"tenant_id\": tenant_id})
# All queries automatically filtered by RLS
```

**2. Authentication & Tenant Resolution:**

```python
# auth.py
from fastapi import Depends, HTTPException, Header
import jwt

async def get_current_tenant(
    authorization: str = Header(None)
) -> str:
    if not authorization or not authorization.startswith(\"Bearer \"):
        raise HTTPException(401, \"Missing token\")
    
    token = authorization[7:]
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[\"HS256\"])
        tenant_id = payload.get(\"tenant_id\")
        user_id = payload.get(\"sub\")
        
        if not tenant_id or not user_id:
            raise HTTPException(401, \"Invalid token\")
        
        return tenant_id
    except jwt.PyJWTError:
        raise HTTPException(401, \"Invalid token\")
```

**3. Multi-Tenant FastAPI Application:**

```python
# main.py
from fastapi import FastAPI, Depends
from sqlalchemy.orm import Session

app = FastAPI()

# Dependency
async def get_db(tenant_id: str = Depends(get_current_tenant)) -> Session:
    db = TenantDatabaseRouter().get_session(tenant_id)
    try:
        yield db
    finally:
        db.close()

# All routes automatically tenant-isolated
@app.get(\"/api/users\")
async def get_users(
    db: Session = Depends(get_db),
    tenant_id: str = Depends(get_current_tenant)
):
    # Query is automatically scoped to tenant
    users = db.query(User).all()
    return users

@app.post(\"/api/users\")
async def create_user(
    user: UserCreate,
    db: Session = Depends(get_db),
    tenant_id: str = Depends(get_current_tenant)
):
    # Add tenant_id to all inserts
    db_user = User(**user.dict(), tenant_id=tenant_id)
    db.add(db_user)
    db.commit()
    return db_user
```

**4. Tenant Onboarding:**

```python
# onboarding.py
class TenantOnboarding:
    def create_tenant(self, tenant_data: TenantCreate) -> str:
        # 1. Create database/schema
        if isolation_strategy == \"database\":
            self.create_database(f\"tenant_{tenant_data.id}\")
        elif isolation_strategy == \"schema\":
            self.create_schema(f\"tenant_{tenant_data.id}\")
        
        # 2. Run migrations
        self.run_migrations(tenant_data.id)
        
        # 3. Create admin user
        admin_user = self.create_admin_user(tenant_data)
        
        # 4. Send welcome email
        send_welcome_email(admin_user.email, tenant_data)
        
        # 5. Provision resources
        self.provision_s3_bucket(tenant_data.id)
        self.create_tenant_config(tenant_data)
        
        return tenant_data.id
    
    def delete_tenant(self, tenant_id: str):
        # 1. Backup data
        self.backup_tenant_data(tenant_id)
        
        # 2. Delete database/schema
        self.delete_tenant_storage(tenant_id)
        
        # 3. Delete S3 bucket
        self.delete_s3_bucket(tenant_id)
        
        # 4. Cancel subscriptions
        self.cancel_subscriptions(tenant_id)
        
        # 5. Soft delete (retain for 30 days)
        self.mark_for_deletion(tenant_id, retention_days=30)
```

**5. Rate Limiting per Tenant:**

```python
# rate_limit.py
class TenantRateLimiter:
    def __init__(self):
        self.limits = {
            \"free\": {\"requests_per_minute\": 100, \"requests_per_day\": 10000},
            \"pro\": {\"requests_per_minute\": 1000, \"requests_per_day\": 100000},
            \"enterprise\": {\"requests_per_minute\": 10000, \"requests_per_day\": 1000000}
        }
    
    def check_limit(self, tenant_id: str, tier: str) -> bool:
        limits = self.limits[tier]
        
        # Check minute limit
        minute_key = f\"rate:{tenant_id}:minute:{int(time.time() / 60)}\"
        minute_count = redis.incr(minute_key)
        if minute_count == 1:
            redis.expire(minute_key, 60)
        if minute_count > limits[\"requests_per_minute\"]:
            return False
        
        # Check daily limit
        day_key = f\"rate:{tenant_id}:day:{datetime.now().date()}\"
        day_count = redis.incr(day_key)
        if day_count == 1:
            redis.expire(day_key, 86400)
        if day_count > limits[\"requests_per_day\"]:
            return False
        
        return True

@app.middleware(\"http\")
async def rate_limit_middleware(
    request: Request,
    call_next,
    tenant_id: str = Depends(get_current_tenant)
):
    tier = get_tenant_tier(tenant_id)
    if not rate_limiter.check_limit(tenant_id, tier):
        raise HTTPException(429, \"Rate limit exceeded\")
    return await call_next(request)
```

**6. Monitoring per Tenant:**

```python
# metrics.py
from prometheus_client import Counter, Histogram

tenant_requests = Counter(
    'tenant_requests_total',
    'Requests per tenant',
    ['tenant_id', 'endpoint']
)

tenant_latency = Histogram(
    'tenant_request_duration_seconds',
    'Latency per tenant',
    ['tenant_id']
)

@app.middleware(\"http\")
async def track_tenant_metrics(
    request: Request,
    call_next,
    tenant_id: str = Depends(get_current_tenant)
):
    start = time.time()
    response = await call_next(request)
    duration = time.time() - start
    
    tenant_requests.labels(
        tenant_id=tenant_id,
        endpoint=request.url.path
    ).inc()
    
    tenant_latency.labels(tenant_id=tenant_id).observe(duration)
    
    return response
```

**7. Tenant Configuration:**

```python
# config.py
class TenantConfig(BaseModel):
    tenant_id: str
    tier: str  # free, pro, enterprise
    features: dict  # Feature flags
    limits: dict  # Custom limits
    custom_domain: Optional[str] = None
    branding: dict = {}

# Cached in Redis
def get_tenant_config(tenant_id: str) -> TenantConfig:
    cache_key = f\"tenant:config:{tenant_id}\"
    cached = redis.get(cache_key)
    if cached:
        return TenantConfig.parse_raw(cached)
    
    # Fetch from database
    config = db.query(TenantConfig).filter(
        TenantConfig.tenant_id == tenant_id
    ).first()
    
    redis.setex(cache_key, 300, config.json())
    return config
```

**Key Components:**

1. **Tenant Isolation:** Database/Schema/RLS
2. **Authentication:** JWT with tenant_id
3. **Rate Limiting:** Per-tenant limits by tier
4. **Monitoring:** Per-tenant metrics
5. **Configuration:** Per-tenant settings
6. **Onboarding:** Automated tenant provisioning
7. **Scaling:** Shard by tenant_id

**Isolation Strategy Comparison:**

| Strategy | Isolation | Cost | Complexity | Use Case |
|----------|-----------|------|------------|----------|
| **Database per Tenant** | Strongest | High | High | Enterprise, compliance |
| **Schema per Tenant** | Medium | Medium | Medium | Mid-market |
| **Row-Level Security** | Weakest | Low | Low | SaaS, many tenants |

**Common Mistakes:**
- Not isolating tenants properly (data leaks)
- No per-tenant rate limiting (abuse)
- Hard to delete tenant data (GDPR compliance)
- No tenant-specific monitoring
- Shared resources causing noisy neighbors
- Not planning for tenant growth

**Interview Tip:**
> "For multi-tenant SaaS, I choose isolation based on requirements: database per tenant for enterprise (strongest isolation, highest cost), schema per tenant for mid-market (medium isolation), or row-level security for many small tenants (cheapest, most complex queries). I use JWT with tenant_id for authentication, Redis-based rate limiting per tenant tier, and Prometheus metrics labeled by tenant_id. The key is strong isolation (prevent data leaks), automated onboarding, and per-tenant monitoring."

</details>

---

## Question 6: Real-Time Collaboration Platform

**Difficulty:** Advanced  
**Category:** Mixed Scenarios

Build a real-time collaboration platform (like Google Docs). How do you handle concurrent editing, state synchronization, and real-time updates?

<details>
<summary><b>View Answer & Explanation</b></summary>

**Answer:**

**Architecture:**

```
Client A ──┐
           ├──→ WebSocket Server ──→ CRDT/OT Engine ──→ Database
Client B ──┘      (FastAPI)          (Conflict Resolution)
```

**1. Operational Transformation (OT) for Concurrent Editing:**

```python
# ot.py
from typing import List, Tuple
import copy

class Operation:
    \"\"\"Represents an edit operation\"\"\"
    def __init__(self, op_type: str, position: int, text: str = \"\"):
        self.op_type = op_type  # 'insert' or 'delete'
        self.position = position
        self.text = text  # For insert: text to insert; For delete: chars to delete
    
    def apply(self, document: str) -> str:
        if self.op_type == 'insert':
            return document[:self.position] + self.text + document[self.position:]
        elif self.op_type == 'delete':
            return document[:self.position] + document[self.position + len(self.text):]
        return document

class OTEngine:
    \"\"\"Operational Transformation for conflict resolution\"\"\"
    
    def transform(self, op1: Operation, op2: Operation) -> Tuple[Operation, Operation]:
        \"\"\"Transform op2 against op1, returning transformed versions\"\"\"
        # Both operations on same document
        
        if op1.op_type == 'insert' and op2.op_type == 'insert':
            if op1.position < op2.position:
                # op1 before op2
                return op1, Operation('insert', op2.position + len(op1.text), op2.text)
            elif op1.position > op2.position:
                return op1, Operation('insert', op2.position, op2.text)
            else:
                # Same position: tie-break by client_id
                return op1, Operation('insert', op2.position + 1, op2.text)
        
        elif op1.op_type == 'insert' and op2.op_type == 'delete':
            if op1.position <= op2.position:
                return op1, Operation('delete', op2.position + len(op1.text), op2.text)
            else:
                return op1, op2
        
        # ... more transformation cases
        
        return op1, op2
    
    def apply_operation(self, document: str, operation: Operation) -> str:
        return operation.apply(document)
```

**2. WebSocket Server with Document Sync:**

```python
# main.py
from fastapi import FastAPI, WebSocket, WebSocketDisconnect
import asyncio
import json

app = FastAPI()

class DocumentRoom:
    def __init__(self, doc_id: str):
        self.doc_id = doc_id
        self.content = \"\"
        self.version = 0
        self.clients: List[WebSocket] = []
        self.lock = asyncio.Lock()
    
    async def add_client(self, websocket: WebSocket):
        self.clients.append(websocket)
        # Send current state to new client
        await websocket.send_json({
            \"type\": \"init\",
            \"content\": self.content,
            \"version\": self.version
        })
    
    async def remove_client(self, websocket: WebSocket):
        self.clients.remove(websocket)
    
    async def apply_operation(self, operation: dict, websocket: WebSocket):
        async with self.lock:
            # Apply operation
            op = Operation(
                operation['type'],
                operation['position'],
                operation.get('text', '')
            )
            self.content = op.apply(self.content)
            self.version += 1
            
            # Broadcast to other clients
            for client in self.clients:
                if client != websocket:
                    await client.send_json({
                        \"type\": \"operation\",
                        \"operation\": operation,
                        \"version\": self.version
                    })

class RoomManager:
    def __init__(self):
        self.rooms = {}
    
    def get_room(self, doc_id: str) -> DocumentRoom:
        if doc_id not in self.rooms:
            self.rooms[doc_id] = DocumentRoom(doc_id)
        return self.rooms[doc_id]

room_manager = RoomManager()

@app.websocket(\"/ws/doc/{doc_id}\")
async def document_websocket(websocket: WebSocket, doc_id: str, user_id: str):
    await websocket.accept()
    room = room_manager.get_room(doc_id)
    await room.add_client(websocket)
    
    try:
        while True:
            data = await websocket.receive_json()
            
            if data['type'] == 'operation':
                await room.apply_operation(data, websocket)
            elif data['type'] == 'cursor':
                # Broadcast cursor position
                for client in room.clients:
                    if client != websocket:
                        await client.send_json({
                            \"type\": \"cursor\",
                            \"user_id\": user_id,
                            \"position\": data['position']
                        })
    except WebSocketDisconnect:
        await room.remove_client(websocket)
```

**3. Frontend: Real-Time Editor:**

```typescript
// DocumentEditor.tsx
'use client';

import { useEffect, useState, useRef } from 'react';

export default function DocumentEditor({ docId, userId }: Props) {
  const [content, setContent] = useState('');
  const [cursors, setCursors] = useState<Map<string, number>>(new Map());
  const ws = useRef<WebSocket | null>(null);
  const editorRef = useRef<HTMLTextAreaElement>(null);
  const pendingOps = useRef<any[]>([]);
  
  useEffect(() => {
    // Connect WebSocket
    const websocket = new WebSocket(`ws://localhost:8000/ws/doc/${docId}?user_id=${userId}`);
    ws.current = websocket;
    
    websocket.onmessage = (event) => {
      const message = JSON.parse(event.data);
      
      if (message.type === 'init') {
        setContent(message.content);
      } else if (message.type === 'operation') {
        // Apply remote operation
        applyRemoteOperation(message.operation);
      } else if (message.type === 'cursor') {
        setCursors(prev => new Map(prev).set(message.user_id, message.position));
      }
    };
    
    return () => websocket.close();
  }, [docId, userId]);
  
  const applyRemoteOperation = (operation: any) => {
    setContent(prev => {
      if (operation.type === 'insert') {
        return prev.slice(0, operation.position) + operation.text + prev.slice(operation.position);
      } else if (operation.type === 'delete') {
        return prev.slice(0, operation.position) + prev.slice(operation.position + operation.text.length);
      }
      return prev;
    });
  };
  
  const handleChange = (e: React.ChangeEvent<HTMLTextAreaElement>) => {
    const newContent = e.target.value;
    const oldContent = content;
    
    // Detect operation
    if (newContent.length > oldContent.length) {
      // Insert
      const position = findInsertPosition(oldContent, newContent);
      const text = newContent.slice(position, position + (newContent.length - oldContent.length));
      
      const operation = { type: 'insert', position, text };
      ws.current?.send(JSON.stringify(operation));
    } else if (newContent.length < oldContent.length) {
      // Delete
      const position = findDeletePosition(oldContent, newContent);
      const text = oldContent.slice(position, position + (oldContent.length - newContent.length));
      
      const operation = { type: 'delete', position, text };
      ws.current?.send(JSON.stringify(operation));
    }
    
    setContent(newContent);
  };
  
  return (
    <div className=\"relative\">
      <textarea
        ref={editorRef}
        value={content}
        onChange={handleChange}
        className=\"w-full h-screen p-4 border\"
        placeholder=\"Start typing...\"
      />
      <div className=\"absolute top-2 right-2\">
        {Array.from(cursors.entries()).map(([userId, position]) => (
          <div key={userId} className=\"text-xs\">
            User {userId}: pos {position}
          </div>
        ))}
      </div>
    </div>
  );
}
```

**4. CRDT Alternative (Yjs):**

```typescript
// Using Yjs for CRDT-based collaboration
import * as Y from 'yjs';
import { WebsocketProvider } from 'y-websocket';

const ydoc = new Y.Doc();
const provider = new WebsocketProvider(
  'ws://localhost:8000',
  'doc-id',
  ydoc
);

const ytext = ydoc.getText('content');

// Bind to textarea
ytext.observe(() => {
  textarea.value = ytext.toString();
});

// Listen for changes
textarea.addEventListener('input', () => {
  // Yjs handles the diff automatically
  ytext.delete(0, ytext.length);
  ytext.insert(0, textarea.value);
});
```

**5. Persistence:**

```python
# persistence.py
import asyncio
from sqlalchemy import create_engine

class DocumentPersistence:
    def __init__(self):
        self.engine = create_engine(os.getenv(\"DATABASE_URL\"))
        self.save_queue = asyncio.Queue()
    
    async def save_document(self, doc_id: str, content: str, version: int):
        \"\"\"Save document to database\"\"\"
        # Debounced save (every 5 seconds if changes)
        await asyncio.sleep(5)
        \n        with self.engine.connect() as conn:
            conn.execute(
                \"UPDATE documents SET content = :content, version = :version WHERE id = :id\",
                {\"content\": content, \"version\": version, \"id\": doc_id}
            )
            conn.commit()
    
    async def load_document(self, doc_id: str) -> tuple[str, int]:
        \"\"\"Load document from database\"\"\"
        with self.engine.connect() as conn:
            result = conn.execute(
                \"SELECT content, version FROM documents WHERE id = :id\",
                {\"id\": doc_id}
            ).first()
            return result.content, result.version

persistence = DocumentPersistence()

# Periodic save
async def save_loop():
    while True:
        await asyncio.sleep(30)  # Save every 30 seconds
        for doc_id, room in room_manager.rooms.items():
            if room.version > 0:
                await persistence.save_document(doc_id, room.content, room.version)
```

**Key Components:**

1. **Operational Transformation:** Conflict resolution algorithm
2. **WebSocket:** Real-time bidirectional communication
3. **Document Rooms:** In-memory document state
4. **Persistence:** Periodic database saves
5. **CRDT (Alternative):** Yjs for simpler implementation
6. **Cursor Tracking:** Real-time cursor positions

**OT vs CRDT:**

| Aspect | OT | CRDT |
|--------|-----|------|
| **Complexity** | High | Medium |
| **Performance** | Fast | Slightly slower |
| **Storage** | Less | More |
| **Use case** | Google Docs | Yjs, Figma |

**Common Mistakes:**
- No conflict resolution (lost updates)
- No versioning (can't sync state)
- Not handling disconnection (state divergence)
- No persistence (data loss)
- Sending full document (high bandwidth)
- Not throttling updates (poor performance)

**Interview Tip:**
> "For real-time collaboration, I use Operational Transformation (OT) or CRDTs (like Yjs) for conflict resolution. The WebSocket server maintains document state in memory with versioning, broadcasts operations to all connected clients, and persists to database periodically. For production, I'd use Yjs because it's simpler and battle-tested. The key is conflict resolution (OT/CRDT), versioning for consistency, and efficient operation-based sync (not full document)."

</details>

---

## Question 7: AI-Powered Search System

**Difficulty:** Advanced  
**Category:** Mixed Scenarios

Design an AI-powered search system that combines semantic search, keyword search, and re-ranking. How do you optimize for relevance and latency?

<details>
<summary><b>View Answer & Explanation</b></summary>

**Answer:**

**Architecture:**

```
Query → Query Understanding → Hybrid Search → Re-ranking → Results
              ↓                    ↓               ↓
        (Query Rewriting)    (BM25 + Vector)   (Cross-encoder)
```

**1. Query Understanding:**

```python
# query_processor.py
from openai import AsyncOpenAI

class QueryProcessor:
    def __init__(self):
        self.client = AsyncOpenAI()
    
    async def process(self, query: str) -> dict:
        \"\"\"Understand and rewrite query for better retrieval\"\"\"
        prompt = f\"\"\"Analyze this search query and provide:
        1. Intent (informational, navigational, transactional)
        2. Rewritten query (clearer, more specific)
        3. Key entities
        4. Filters to apply
        
        Query: {query}
        
        JSON format:
        {{
            \"intent\": \"...\",
            \"rewritten_query\": \"...\",
            \"entities\": [...],
            \"filters\": {{...}}
        }}\"\"\"
        
        response = await self.client.chat.completions.create(
            model=\"gpt-3.5-turbo\",
            messages=[{\"role\": \"user\", \"content\": prompt}],
            response_format={\"type\": \"json_object\"}
        )
        
        return json.loads(response.choices[0].message.content)
    
    async def expand_query(self, query: str) -> List[str]:
        \"\"\"Generate query variations for better recall\"\"\"
        prompt = f\"Generate 3 alternative phrasings of this query:\\n\\n{query}\"
        \n        response = await self.client.chat.completions.create(
            model=\"gpt-3.5-turbo\",
            messages=[{\"role\": \"user\", \"content\": prompt}]
        )
        \n        variations = response.choices[0].message.content.strip().split('\\n')
        return [query] + variations  # Original + variations
```

**2. Hybrid Search Engine:**

```python
# search_engine.py
from langchain.retrievers import BM25Retriever, EnsembleRetriever
from langchain.retrievers.document_compressors import CohereRerank
from langchain.vectorstores import Pinecone
from langchain.embeddings import OpenAIEmbeddings
import asyncio

class HybridSearchEngine:
    def __init__(self):
        self.embeddings = OpenAIEmbeddings()
        self.vectorstore = Pinecone.from_existing_index(\n            \"documents\",\n            self.embeddings\n        )
        self.bm25_retriever = None  # Lazy load
        self.reranker = CohereRerank(model=\"rerank-english-v2.0\", top_n=10)
    
    async def search(self, query: str, top_k: int = 10) -> List[dict]:
        # 1. Query understanding
        processed = await query_processor.process(query)
        rewritten = processed[\"rewritten_query\"]
        \n        # 2. Generate query variations
        variations = await query_processor.expand_query(rewritten)
        \n        # 3. Parallel search (BM25 + Semantic)
        results = await asyncio.gather(
            self._semantic_search(variations, top_k * 2),\n            self._keyword_search(variations, top_k * 2)\n        )
        \n        semantic_results, keyword_results = results
        \n        # 4. Combine with reciprocal rank fusion\n        combined = self._reciprocal_rank_fusion(\n            semantic_results,\n            keyword_results\n        )\n        \n        # 5. Re-rank with cross-encoder
        reranked = await self._rerank(query, combined, top_k)\n        \n        return reranked
    
    async def _semantic_search(self, queries: List[str], k: int) -> List[dict]:
        \"\"\"Semantic search with multiple query variations\"\"\"
        all_docs = []\n        for query in queries:
            docs = self.vectorstore.similarity_search(query, k=k)\n            all_docs.extend(docs)\n        return all_docs
    
    async def _keyword_search(self, queries: List[str], k: int) -> List[dict]:
        \"\"\"BM25 keyword search\"\"\"
        if not self.bm25_retriever:\n            # Lazy load documents\n            docs = self.vectorstore.get_all_documents()\n            self.bm25_retriever = BM25Retriever.from_documents(docs)\n            self.bm25_retriever.k = k\n        \n        all_docs = []\n        for query in queries:\n            docs = self.bm25_retriever.get_relevant_documents(query)\n            all_docs.extend(docs)\n        return all_docs
    
    def _reciprocal_rank_fusion(\n        self,\n        results1: List[dict],\n        results2: List[dict],\n        k: int = 60\n    ) -> List[dict]:\n        \"\"\"Combine results using RRF\"\"\"
        scores = {}\n        \n        for i, doc in enumerate(results1):\n            doc_id = doc.metadata.get('id', str(i))\n            scores[doc_id] = scores.get(doc_id, 0) + 1 / (i + k)\n        \n        for i, doc in enumerate(results2):\n            doc_id = doc.metadata.get('id', str(i))\n            scores[doc_id] = scores.get(doc_id, 0) + 1 / (i + k)\n        \n        # Sort by combined score\n        all_docs = {doc.metadata.get('id', str(i)): doc \n                   for i, doc in enumerate(results1 + results2)}\n        \n        sorted_ids = sorted(scores.keys(), \n                          key=lambda x: scores[x], \n                          reverse=True)\n        \n        return [all_docs[id_] for id_ in sorted_ids]
    
    async def _rerank(\n        self,\n        query: str,\n        documents: List[dict],\n        top_k: int\n    ) -> List[dict]:\n        \"\"\"Re-rank with cross-encoder\"\"\"
        reranked = self.reranker.compress_documents(documents, query)\n        return reranked[:top_k]
```

**3. FastAPI Endpoint:**

```python
# main.py
from fastapi import FastAPI
from pydantic import BaseModel
import time

app = FastAPI()
search_engine = HybridSearchEngine()

class SearchRequest(BaseModel):
    query: str
    top_k: int = 10
    filters: dict = {}

class SearchResponse(BaseModel):
    results: list
    query_understanding: dict
    latency_ms: float

@app.post(\"/search\", response_model=SearchResponse)
async def search(request: SearchRequest):
    start = time.time()
    \n    # Search
    results = await search_engine.search(request.query, request.top_k)\n    \n    # Process query for explanation
    query_info = await query_processor.process(request.query)\n    \n    latency = (time.time() - start) * 1000\n    \n    return SearchResponse(\n        results=[\n            {\n                \"content\": doc.page_content,\n                \"metadata\": doc.metadata,\n                \"score\": doc.metadata.get('score', 0)\n            }\n            for doc in results\n        ],\n        query_understanding=query_info,\n        latency_ms=latency
    )
```

**4. Caching for Performance:**

```python
# cache.py
import hashlib
import json
import redis

class SearchCache:
    def __init__(self, ttl: int = 3600):\n        self.redis = redis.Redis()\n        self.ttl = ttl\n    \n    def get(self, query: str, top_k: int) -> Optional[dict]:\n        key = self._make_key(query, top_k)\n        cached = self.redis.get(key)\n        return json.loads(cached) if cached else None\n    \n    def set(self, query: str, top_k: int, results: dict):
        key = self._make_key(query, top_k)\n        self.redis.setex(key, self.ttl, json.dumps(results))\n    \n    def _make_key(self, query: str, top_k: int) -> str:\n        content = f\"{query}:{top_k}\"\n        return f\"search:{hashlib.md5(content.encode()).hexdigest()}\"

cache = SearchCache()

@app.post(\"/search\")
async def search(request: SearchRequest):
    # Check cache
    cached = cache.get(request.query, request.top_k)\n    if cached:\n        return cached\n    \n    # Search
    results = await search_engine.search(request.query, request.top_k)\n    \n    # Cache result\n    cache.set(request.query, request.top_k, results)\n    \n    return results
```

**5. Frontend Search UI (React):**

```typescript
// SearchBox.tsx
'use client';

import { useState, useEffect } from 'react';
import { useDebounce } from '@/hooks/useDebounce';

export default function SearchBox() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState([]);
  const [loading, setLoading] = useState(false);
  const debouncedQuery = useDebounce(query, 300);
  \n  useEffect(() => {\n    if (debouncedQuery.length < 3) {\n      setResults([]);\n      return;\n    }\n    \n    const search = async () => {\n      setLoading(true);\n      const response = await fetch('/api/search', {\n        method: 'POST',\n        headers: { 'Content-Type': 'application/json' },\n        body: JSON.stringify({ query: debouncedQuery, top_k: 10 })\n      });\n      const data = await response.json();\n      setResults(data.results);\n      setLoading(false);\n    };\n    \n    search();\n  }, [debouncedQuery]);\n  \n  return (
    <div>\n      <input\n        value={query}\n        onChange={e => setQuery(e.target.value)}\n        placeholder=\"Search...\"\n        className=\"w-full p-3 border rounded-lg\"\n      />\n      {loading && <div>Searching...</div>}\n      <div className=\"mt-4 space-y-3\">\n        {results.map((result, idx) => (\n          <div key={idx} className=\"p-4 border rounded-lg hover:bg-gray-50\">\n            <h3 className=\"font-semibold\">{result.metadata.title}</h3>\n            <p className=\"text-sm text-gray-600\">{result.content}</p>\n            <div className=\"text-xs text-gray-500 mt-2\">\n              Score: {result.score.toFixed(3)} | {result.metadata.source}\n            </div>\n          </div>\n        ))}\n      </div>\n    </div>\n  );\n}
```

**Key Components:**

1. **Query Understanding:** Intent detection, rewriting, expansion
2. **Hybrid Search:** BM25 (keyword) + Semantic (vector)
3. **Reciprocal Rank Fusion:** Combine results
4. **Re-ranking:** Cross-encoder for precision
5. **Caching:** Redis for performance
6. **Debouncing:** Frontend optimization

**Performance Optimizations:**

```python
# 1. Async parallel search
results = await asyncio.gather(\n    semantic_search(query),\n    keyword_search(query)\n)

# 2. Caching frequent queries
if cached_result:
    return cached_result

# 3. Batch embedding generation
# 4. Use approximate nearest neighbors (HNSW, IVF)
# 5. Pre-compute embeddings
# 6. Connection pooling for vector DB
```

**Evaluation Metrics:**

```python
# Use NDCG, MRR, Precision@K
def evaluate_search(results, ground_truth, k=10):
    # NDCG@K
    dcg = sum(rel / np.log2(i + 2) for i, rel in enumerate(results[:k]))\n    idcg = sum(rel / np.log2(i + 2) for i, rel in enumerate(sorted(ground_truth, reverse=True)[:k]))\n    ndcg = dcg / idcg if idcg > 0 else 0\n    \n    # MRR\n    for i, result in enumerate(results):\n        if result in ground_truth:\n            return 1 / (i + 1)\n    return 0
```

**Common Mistakes:**
- Only semantic search (misses keyword matches)
- No re-ranking (low precision)
- No query understanding (poor recall)
- No caching (slow, expensive)
- Not measuring relevance (can't improve)
- Synchronous search (slow)

**Interview Tip:**
> "For AI-powered search, I combine BM25 (keyword) with semantic search (vector) using reciprocal rank fusion, then re-rank with a cross-encoder for precision. I use query understanding (rewriting, expansion) to improve recall, and Redis caching for performance. Key optimizations: parallel search, debouncing on frontend, and measuring with NDCG/MRR. The combination of keyword + semantic + re-ranking significantly outperforms any single approach."

</details>

---

## Question 8: Serverless AI Image Processing Pipeline

**Difficulty:** Advanced  
**Category:** Mixed Scenarios

Design a serverless pipeline for AI image processing (upload, analysis, thumbnail generation). How do you handle async processing, scaling, and cost?

<details>
<summary><b>View Answer & Explanation</b></summary>

**Answer:**

**Architecture:**

```
User Upload → API Gateway → Lambda (Trigger) → S3
                                                      ↓
                                          S3 Event → Lambda (Thumbnail)
                                                      ↓
                                          S3 Event → Lambda (AI Analysis)
                                                      ↓
                                          DynamoDB (Results)
                                                      ↓
                                          API Gateway → Lambda (Query)
```

**1. Upload API (FastAPI + S3 Presigned URL):**

```python
# upload_api.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
import boto3
import os

app = FastAPI()
s3 = boto3.client('s3')

class UploadRequest(BaseModel):
    filename: str
    content_type: str

@app.post(\"/upload/presigned\")
async def get_presigned_url(request: UploadRequest):
    \"\"\"Generate presigned URL for direct S3 upload\"\"\"
    try:
        # Generate unique key
        import uuid
        key = f\"uploads/{uuid.uuid4()}/{request.filename}\"
        \n        # Generate presigned URL (valid for 5 minutes)
        url = s3.generate_presigned_url(
            'put_object',
            Params={\n                'Bucket': 'my-images',\n                'Key': key,\n                'ContentType': request.content_type\n            },\n            ExpiresIn=300\n        )\n        \n        return {\n            \"upload_url\": url,\n            \"key\": key\n        }\n    except Exception as e:\n        raise HTTPException(500, str(e))
```

**2. Thumbnail Lambda:**

```python
# thumbnail_lambda.py
import boto3
from PIL import Image
import io
import os

s3 = boto3.client('s3')

def handler(event, context):
    \"\"\"Generate thumbnails when image uploaded to S3\"\"\"
    for record in event['Records']:\n        bucket = record['s3']['bucket']['name']\n        key = record['s3']['object']['key']\n        \n        # Download image\n        response = s3.get_object(Bucket=bucket, Key=key)\n        image_data = response['Body'].read()\n        \n        # Generate thumbnails\n        image = Image.open(io.BytesIO(image_data))\n        \n        sizes = [(150, 150), (300, 300), (600, 600)]\n        for width, height in sizes:\n            thumbnail = image.copy()\n            thumbnail.thumbnail((width, height))\n            \n            # Save to S3\n            thumb_buffer = io.BytesIO()\n            thumbnail.save(thumb_buffer, format='JPEG', quality=85)\n            \n            thumb_key = key.replace('uploads/', f'thumbnails/{width}x{height}/')\n            s3.put_object(\n                Bucket=bucket,\n                Key=thumb_key,\n                Body=thumb_buffer.getvalue(),\n                ContentType='image/jpeg'\n            )\n        \n        return {'statusCode': 200, 'body': 'Thumbnails generated'}
```

**3. AI Analysis Lambda:**

```python
# ai_analysis_lambda.py
import boto3
import json
from openai import OpenAI
import base64

s3 = boto3.client('s3')
client = OpenAI()

def handler(event, context):
    \"\"\"Analyze image with OpenAI Vision API\"\"\"
    for record in event['Records']:\n        bucket = record['s3']['bucket']['name']\n        key = record['s3']['object']['key']\n        \n        # Download image\n        response = s3.get_object(Bucket=bucket, Key=key)\n        image_data = response['Body'].read()\n        \n        # Encode for OpenAI\n        base64_image = base64.b64encode(image_data).decode('utf-8')\n        \n        # Analyze with GPT-4 Vision\n        response = client.chat.completions.create(\n            model=\"gpt-4-vision-preview\",\n            messages=[\n                {\n                    \"role\": \"user\",\n                    \"content\": [\n                        {\"type\": \"text\", \"text\": \"Describe this image in detail. Include objects, text, colors, and composition.\"},\n                        {\n                            \"type\": \"image_url\",\n                            \"image_url\": {\n                                \"url\": f\"data:image/jpeg;base64,{base64_image}\"\n                            }\n                        }\n                    ]\n                }\n            ],\n            max_tokens=300\n        )\n        \n        description = response.choices[0].message.content\n        \n        # Store results in DynamoDB\n        dynamodb = boto3.resource('dynamodb')\n        table = dynamodb.Table('image-metadata')\n        \n        table.put_item(Item={\n            'image_key': key,\n            'description': description,\n            'analyzed_at': str(datetime.now()),\n            'model': 'gpt-4-vision'\n        })\n        \n        return {'statusCode': 200, 'body': 'Analysis complete'}
```

**4. SAM Template:**

```yaml
# template.yaml
AWSTemplateFormatVersion: '2010-09-09'\nTransform: AWS::Serverless-2016-10-31\n\nResources:\n  UploadApi:\n    Type: AWS::Serverless::Function\n    Properties:\n      CodeUri: ./upload_api\n      Handler: app.handler\n      Runtime: python3.11\n      MemorySize: 512\n      Timeout: 30\n      Environment:\n        Variables:\n          S3_BUCKET: !Ref ImageBucket\n      Events:\n        Api:\n          Type: Api\n          Properties:\n            Path: /{proxy+}\n            Method: ANY\n\n  ThumbnailFunction:\n    Type: AWS::Serverless::Function\n    Properties:\n      CodeUri: ./thumbnail_lambda\n      Handler: handler.handler\n      Runtime: python3.11\n      MemorySize: 1024\n      Timeout: 60\n      Policies:\n        - S3ReadPolicy:\n            BucketName: !Ref ImageBucket\n        - S3WritePolicy:\n            BucketName: !Ref ImageBucket\n      Events:\n        S3Trigger:\n          Type: S3\n          Properties:\n            Bucket: !Ref ImageBucket\n            Events: s3:ObjectCreated:*\n            Filter:\n              S3Key:\n                Rules:\n                  - Name: prefix\n                    Value: uploads/\n\n  AIAnalysisFunction:\n    Type: AWS::Serverless::Function\n    Properties:\n      CodeUri: ./ai_analysis_lambda\n      Handler: handler.handler\n      Runtime: python3.11\n      MemorySize: 2048\n      Timeout: 300\n      Environment:\n        Variables:\n          OPENAI_API_KEY: !Sub '{{resolve:secretsmanager:openai-key}}'\n      Events:\n        S3Trigger:\n          Type: S3\n          Properties:\n            Bucket: !Ref ImageBucket\n            Events: s3:ObjectCreated:*\n            Filter:\n              S3Key:\n                Rules:\n                  - Name: prefix\n                    Value: uploads/\n\n  ImageBucket:\n    Type: AWS::S3::Bucket\n    Properties:\n      BucketName: my-images\n      LifecycleConfiguration:\n        Rules:\n          - Id: DeleteOldUploads\n            Status: Enabled\n            ExpirationInDays: 90\n\n  ImageMetadataTable:\n    Type: AWS::DynamoDB::Table\n    Properties:\n      TableName: image-metadata\n      AttributeDefinitions:\n        - AttributeName: image_key\n          AttributeType: S\n      KeySchema:\n        - AttributeName: image_key\n          KeyType: HASH\n      BillingMode: PAY_PER_REQUEST
```

**5. Query Results API:**

```python
# query_api.py
from fastapi import FastAPI, HTTPException
import boto3

app = FastAPI()
dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('image-metadata')

@app.get(\"/image/{key:path}\")
async def get_image_metadata(key: str):
    response = table.get_item(Key={'image_key': key})\n    item = response.get('Item')\n    \n    if not item:\n        raise HTTPException(404, \"Not found\")\n    \n    return item

@app.get(\"/images\")
async def list_images(limit: int = 20):\n    response = table.scan(Limit=limit)\n    return response['Items']
```

**6. Frontend Upload (React):**

```typescript
// ImageUpload.tsx
'use client';

import { useState } from 'react';

export default function ImageUpload() {
  const [uploading, setUploading] = useState(false);
  const [results, setResults] = useState<any>(null);\n  \n  const handleUpload = async (file: File) => {\n    setUploading(true);\n    \n    // 1. Get presigned URL
    const presignedResponse = await fetch('/upload/presigned', {\n      method: 'POST',\n      headers: { 'Content-Type': 'application/json' },\n      body: JSON.stringify({\n        filename: file.name,\n        content_type: file.type\n      })\n    });\n    const { upload_url, key } = await presignedResponse.json();\n    \n    // 2. Upload directly to S3\n    await fetch(upload_url, {\n      method: 'PUT',\n      body: file,\n      headers: { 'Content-Type': file.type }\n    });\n    \n    // 3. Poll for results (or use WebSocket)\n    let attempts = 0;\n    while (attempts < 30) {\n      const metadataResponse = await fetch(`/image/${encodeURIComponent(key)}`);\n      if (metadataResponse.ok) {\n        const metadata = await metadataResponse.json();\n        setResults(metadata);\n        break;\n      }\n      await new Promise(r => setTimeout(r, 2000));  // Wait 2 seconds\n      attempts++;\n    }\n    \n    setUploading(false);\n  };\n  \n  return (\n    <div>\n      <input\n        type=\"file\"\n        accept=\"image/*\"\n        onChange={e => e.target.files && handleUpload(e.target.files[0])}\n        disabled={uploading}\n      />\n      {uploading && <div>Processing...</div>}\n      {results && (\n        <div className=\"mt-4\">\n          <h3>Analysis:</h3>\n          <p>{results.description}</p>\n        </div>\n      )}\n    </div>\n  );\n}
```

**Key Components:**

1. **Upload API:** Generates presigned URLs (direct S3 upload)
2. **S3 Event Triggers:** Lambda functions triggered on upload
3. **Thumbnail Lambda:** Generates multiple sizes
4. **AI Analysis Lambda:** Uses OpenAI Vision
5. **DynamoDB:** Stores metadata
6. **Query API:** Retrieves results

**Benefits:**

- **Serverless:** No infrastructure management
- **Auto-scaling:** Lambda scales automatically
- **Cost-effective:** Pay per execution
- **Event-driven:** Async processing via S3 events
- **Decoupled:** Each step is independent

**Cost Optimization:**

```python
# 1. Right-size Lambda memory
# Thumbnail: 1024 MB (PIL needs memory)
# AI Analysis: 2048 MB (image processing)

# 2. S3 lifecycle policies
# - Move old uploads to Glacier after 30 days
# - Delete after 90 days

# 3. DynamoDB on-demand pricing
# Pay per request, not provisioned capacity

# 4. CloudFront for thumbnails
# Cache frequently accessed images
```

**Common Mistakes:**
- Synchronous processing (blocks user)
- No presigned URL (uploads through API = slow)
- No error handling (failed uploads)
- Not cleaning up old files (storage costs)
- Cold starts (Lambda initialization)
- No monitoring (blind to failures)

**Interview Tip:**
> "For serverless image processing, I use S3 for storage, Lambda for processing, and S3 events to trigger async workflows. The upload API generates presigned URLs for direct S3 upload (avoids API Gateway bandwidth costs). Multiple Lambda functions process the image in parallel: thumbnail generation, AI analysis with OpenAI Vision, and metadata extraction. Results go to DynamoDB. The user polls or uses WebSocket for results. This is cost-effective (pay per execution) and auto-scales."

</details>

---

## Question 9: AI-Powered Customer Support Platform

**Difficulty:** Advanced  
**Category:** Mixed Scenarios

Design an AI-powered customer support platform with ticket routing, AI assistance, and human handoff. What agents and workflows are needed?

<details>
<summary><b>View Answer & Explanation</b></summary>

**Answer:**

**Architecture:**

```
Customer → Chat Widget → API → Triage Agent
                                      ↓
                        ┌─────────────┼─────────────┐
                        ↓             ↓             ↓
                  Billing Agent  Tech Agent  General Agent
                        ↓             ↓             ↓
                        └─────────────┼─────────────┘
                                      ↓
                              Human Handoff (if needed)
                                      ↓
                              Support Agent Dashboard
```

**1. Multi-Agent Support System:**

```python
# agents.py
from autogen import ConversableAgent, GroupChat, GroupChatManager
import os

llm_config = {
    \"model\": \"gpt-4\",
    \"api_key\": os.getenv(\"OPENAI_API_KEY\")
}

# Triage Agent (routes to specialized agents)
triage_agent = ConversableAgent(
    name=\"Triage\",
    system_message=\"\"\"You are a triage agent for customer support.
    Analyze the customer's issue and route to the appropriate specialist:
    - 'billing' for payment, subscription, refund issues
    - 'technical' for bugs, errors, technical problems
    - 'account' for login, password, profile issues
    - 'general' for other inquiries
    
    Respond with: ROUTE:<agent_name> followed by the reason.\"\"\",
    llm_config=llm_config,
    human_input_mode=\"NEVER\"
)

# Billing Specialist
billing_agent = ConversableAgent(
    name=\"BillingSpecialist\",
    system_message=\"\"\"You are a billing support specialist. Help with:
    - Subscription questions
    - Payment issues
    - Refund requests
    - Invoice inquiries
    - Plan changes
    
    Use the billing tools to check accounts, process refunds, etc.
    Escalate to human if customer is unsatisfied.\"\"\",
    llm_config=llm_config,
    human_input_mode=\"NEVER\"
)

# Technical Specialist
tech_agent = ConversableAgent(
    name=\"TechSpecialist\",
    system_message=\"\"\"You are a technical support specialist. Help with:
    - Bug reports
    - Error messages
    - Feature questions
    - Integration issues
    - API problems
    
    Use the technical tools to check logs, reproduce issues, etc.
    Escalate to human if issue is complex or customer is frustrated.\"\"\",
    llm_config=llm_config,
    human_input_mode=\"NEVER\"
)

# Escalation Agent (decides when to handoff to human)
escalation_agent = ConversableAgent(
    name=\"Escalation\",
    system_message=\"\"\"You decide when to escalate to human support.
    Escalate if:
    - Customer is frustrated or angry
    - Issue is complex and requires human judgment
    - Customer explicitly asks for human
    - AI confidence is low
    - Issue involves legal/compliance
    
    Respond with: ESCALATE:<reason> or CONTINUE.\"\"\",
    llm_config=llm_config,
    human_input_mode=\"NEVER\"
)

# Group Chat
group_chat = GroupChat(
    agents=[triage_agent, billing_agent, tech_agent, escalation_agent],
    messages=[],
    max_round=10,
    speaker_selection_method=\"auto\"
)

manager = GroupChatManager(groupchat=group_chat, llm_config=llm_config)
```

**2. Tools for Agents:**

```python
# tools.py
import requests
from sqlalchemy.orm import Session

class SupportTools:
    def __init__(self, db: Session):
        self.db = db
    
    def get_account_info(self, user_id: str) -> dict:
        \"\"\"Get customer account information\"\"\"
        user = self.db.query(User).filter(User.id == user_id).first()
        return {
            \"name\": user.name,\n            \"email\": user.email,\n            \"plan\": user.subscription.plan,
            \"status\": user.subscription.status,\n            \"created_at\": user.created_at.isoformat()
        }
    
    def process_refund(self, order_id: str, amount: float, reason: str) -> dict:
        \"\"\"Process a refund\"\"\"
        # Call payment processor API
        response = requests.post(
            \"https://api.stripe.com/v1/refunds\",
            headers={\"Authorization\": f\"Bearer {STRIPE_KEY}\"},\n            json={\n                \"charge\": order_id,\n                \"amount\": int(amount * 100),\n                \"reason\": reason
            }
        )
        return response.json()
    
    def check_service_status(self) -> dict:
        \"\"\"Check if services are operational\"\"\"
        response = requests.get(\"https://status.api.com/health\")
        return response.json()
    
    def search_knowledge_base(self, query: str) -> list:
        \"\"\"Search knowledge base for relevant articles\"\"\"
        # Use RAG to search knowledge base
        docs = vectorstore.similarity_search(query, k=3)
        return [{\"title\": doc.metadata['title'], \"content\": doc.page_content} 
                for doc in docs]
    
    def create_ticket(self, user_id: str, issue: str, priority: str) -> str:
        \"\"\"Create support ticket\"\"\"
        ticket = Ticket(
            user_id=user_id,
            issue=issue,
            priority=priority,
            status=\"open\"
        )
        self.db.add(ticket)
        self.db.commit()
        return ticket.id
    
    def escalate_to_human(self, ticket_id: str, reason: str):
        \"\"\"Escalate ticket to human agent\"\"\"
        # Notify human agents via Slack/PagerDuty
        send_slack_notification(
            channel=\"#support-escalations\",
            message=f\"Ticket {ticket_id} escalated: {reason}\"
        )
        # Update ticket
        ticket = self.db.query(Ticket).filter(Ticket.id == ticket_id).first()
        ticket.escalated = True
        ticket.escalation_reason = reason
        self.db.commit()
```

**3. Chat API with Streaming:**

```python
# chat_api.py
from fastapi import FastAPI, WebSocket
from fastapi.responses import StreamingResponse
import json
import asyncio

app = FastAPI()
support_tools = SupportTools(db)

@app.websocket(\"/ws/chat/{user_id}\")
async def chat_websocket(websocket: WebSocket, user_id: str):
    await websocket.accept()
    \n    try:
        while True:
            data = await websocket.receive_json()\n            user_message = data[\"message\"]\n            \n            # Stream agent response
            async for chunk in stream_agent_response(user_id, user_message):
                await websocket.send_json(chunk)\n    except WebSocketDisconnect:\n        pass

async def stream_agent_response(user_id: str, message: str):\n    \"\"\"Stream agent response\"\"\"
    # Add tools to agents
    # ... (tool registration)\n    \n    # Run agent conversation\n    async for response in run_agents_streaming(user_id, message):\n        yield response
```

**4. Sentiment Analysis for Escalation:**

```python
# sentiment.py
from openai import AsyncOpenAI

class SentimentAnalyzer:
    def __init__(self):
        self.client = AsyncOpenAI()
    \n    async def analyze(self, message: str) -> dict:
        \"\"\"Analyze customer sentiment and frustration\"\"\"
        prompt = f\"\"\"Analyze the sentiment of this customer message:
        
        Message: {message}
        \n        Provide:
        1. Sentiment (positive, neutral, negative, angry)
        2. Frustration level (0-10)
        3. Urgency (low, medium, high)
        4. Should escalate? (yes/no)
        \n        JSON format.\"\"\"
        \n        response = await self.client.chat.completions.create(\n            model=\"gpt-3.5-turbo\",
            messages=[{\"role\": \"user\", \"content\": prompt}],\n            response_format={\"type\": \"json_object\"}
        )\n        \n        return json.loads(response.choices[0].message.content)
```

**5. Knowledge Base RAG:**

```python
# knowledge_base.py
from langchain.vectorstores import Pinecone
from langchain.embeddings import OpenAIEmbeddings

class KnowledgeBase:
    def __init__(self):
        self.embeddings = OpenAIEmbeddings()
        self.vectorstore = Pinecone.from_existing_index(\n            \"support-kb\",\n            self.embeddings\n        )
    \n    async def search(self, query: str, k: int = 3) -> list:
        \"\"\"Search knowledge base\"\"\"
        docs = self.vectorstore.similarity_search(query, k=k)\n        return [\n            {\n                \"title\": doc.metadata['title'],\n                \"content\": doc.page_content,\n                \"url\": doc.metadata.get('url')\n            }\n            for doc in docs\n        ]
```

**6. Human Handoff Dashboard (React):**

```typescript
// SupportDashboard.tsx
'use client';

import { useEffect, useState } from 'react';

interface Ticket {
  id: string;
  user: string;
  issue: string;
  priority: string;
  escalated: boolean;
  conversation: Message[];
}

export default function SupportDashboard() {
  const [tickets, setTickets] = useState<Ticket[]>([]);
  const [selectedTicket, setSelectedTicket] = useState<Ticket | null>(null);\n  \n  useEffect(() => {\n    // Connect to WebSocket for real-time ticket updates\n    const ws = new WebSocket('ws://localhost:8000/ws/support');\n    \n    ws.onmessage = (event) => {\n      const data = JSON.parse(event.data);\n      if (data.type === 'new_ticket') {\n        setTickets(prev => [data.ticket, ...prev]);\n      } else if (data.type === 'escalated') {\n        setTickets(prev => \n          prev.map(t => t.id === data.ticket_id ? {...t, escalated: true} : t)\n        );\n      }\n    };\n    \n    return () => ws.close();\n  }, []);\n  \n  return (\n    <div className=\"grid grid-cols-3 h-screen\">\n      {/* Ticket List */}\n      <div className=\"border-r overflow-y-auto\">\n        <h2 className=\"p-4 font-bold\">Tickets</h2>\n        {tickets.map(ticket => (\n          <div\n            key={ticket.id}\n            onClick={() => setSelectedTicket(ticket)}\n            className={`p-3 border-b cursor-pointer hover:bg-gray-50 ${\n              ticket.escalated ? 'border-l-4 border-red-500' : ''\n            }`}\n          >\n            <div className=\"font-semibold\">{ticket.user}</div>\n            <div className=\"text-sm text-gray-600\">{ticket.issue}</div>\n            <div className=\"text-xs text-gray-500 mt-1\">\n              {ticket.priority} | {ticket.escalated ? '🚨 Escalated' : '🤖 AI'}\n            </div>\n          </div>\n        ))}\n      </div>\n      \n      {/* Conversation */}\n      <div className=\"col-span-2 p-4\">\n        {selectedTicket ? (\n          <div>\n            <h2 className=\"text-xl font-bold mb-4\">\n              Ticket #{selectedTicket.id}\n            </h2>\n            <div className=\"space-y-3 mb-4 max-h-96 overflow-y-auto\">\n              {selectedTicket.conversation.map((msg, idx) => (\n                <div key={idx} className={`p-3 rounded ${\n                  msg.role === 'user' ? 'bg-blue-100' : 'bg-gray-100'\n                }`}>\n                  <div className=\"font-semibold text-sm\">{msg.role}</div>\n                  <div>{msg.content}</div>\n                </div>\n              ))}\n            </div>\n            \n            {selectedTicket.escalated && (\n              <button className=\"bg-green-500 text-white px-4 py-2 rounded\">\n                Take Over Conversation\n              </button>\n            )}\n          </div>\n        ) : (\n          <div className=\"text-gray-500\">Select a ticket</div>\n        )}\n      </div>\n    </div>\n  );\n}
```

**Key Components:**

1. **Triage Agent:** Routes to specialists
2. **Specialist Agents:** Billing, Technical, General
3. **Escalation Logic:** Sentiment + complexity analysis
4. **Tools:** Account lookup, refund processing, KB search
5. **Human Handoff:** Dashboard for support agents
6. **Real-time Updates:** WebSocket for live ticket monitoring

**Workflow:**

1. Customer sends message
2. Triage agent classifies issue
3. Specialist agent handles with tools
4. Escalation agent monitors for handoff
5. If escalated: human agent takes over
6. Resolution and feedback collection

**Key Concepts:**
- Multi-agent orchestration for specialized support
- Sentiment analysis for proactive escalation
- Tool integration for real actions (refunds, account changes)
- Human-in-the-loop for complex issues
- Real-time dashboard for support team
- Knowledge base RAG for accurate answers

**Common Mistakes:**
- No escalation path (AI gets stuck)
- Not using tools (can't take real actions)
- No sentiment analysis (miss frustrated customers)
- No human handoff (poor customer experience)
- Not learning from interactions (same mistakes)

**Interview Tip:**
> "For AI customer support, I use multi-agent orchestration with AutoGen: triage agent routes to specialists (billing, technical, general), each with specific tools. I add sentiment analysis to detect frustrated customers and escalate proactively. The system has a clear handoff path to human agents when needed. Tools enable real actions (refunds, account changes), and a React dashboard lets support agents monitor and take over conversations. The key is knowing when AI should handle vs. escalate to human."

</details>

---

## Question 10: Distributed Task Queue with AI Workers

**Difficulty:** Advanced  
**Category:** Mixed Scenarios

Design a distributed task queue system where AI workers process tasks (e.g., document analysis, content generation). How do you handle scaling, retries, and monitoring?

<details>
<summary><b>View Answer & Explanation</b></summary>

**Answer:**

**Architecture:**

```
Client → API → Task Queue (Redis/RabbitMQ) → AI Workers
                ↓                                ↓
            Database                       Process Task
                ↑                                ↓
                └──────── Update Status ←───────┘
```

**1. Task Queue with Celery:**

```python
# tasks.py
from celery import Celery
import openai

# Configure Celery
app = Celery(
    'tasks',
    broker='redis://localhost:6379/0',
    backend='redis://localhost:6379/1'
)

# Configure
app.conf.update(
    task_serializer='json',
    accept_content=['json'],
    result_serializer='json',
    timezone='UTC',
    enable_utc=True,
    task_track_started=True,
    task_time_limit=600,  # 10 minutes max
    task_soft_time_limit=540,  # 9 minutes warning
    worker_prefetch_multiplier=1,
    worker_max_tasks_per_child=50,  # Restart worker after 50 tasks (memory leak prevention)
)

@app.task(bind=True, max_retries=3, autoretry_for=(Exception,),\n          retry_backoff=True, retry_backoff_max=600, retry_jitter=True)
def analyze_document(self, document_id: str):
    \"\"\"Analyze document with AI\"\"\"
    try:
        # 1. Fetch document
        document = db.query(Document).filter(Document.id == document_id).first()\n        \n        # 2. Update status\n        document.status = \"processing\"\n        db.commit()\n        \n        # 3. Call AI API
        client = openai.OpenAI()\n        response = client.chat.completions.create(\n            model=\"gpt-4\",\n            messages=[\n                {\"role\": \"system\", \"content\": \"Analyze this document...\"},\n                {\"role\": \"user\", \"content\": document.content}\n            ]\n        )\n        \n        # 4. Store results\n        document.analysis = response.choices[0].message.content\n        document.status = \"completed\"\n        document.analyzed_at = datetime.now()\n        db.commit()\n        \n        return {\"status\": \"completed\", \"document_id\": document_id}\n    \n    except Exception as exc:\n        # Update status\n        document.status = \"failed\"\n        document.error = str(exc)\n        db.commit()\n        \n        # Retry with exponential backoff
        raise self.retry(exc=exc, countdown=2 ** self.request.retries)
\n@app.task(bind=True, max_retries=2)\ndef generate_content(self, prompt: str, user_id: str):\n    \"\"\"Generate content with AI\"\"\"
    try:\n        client = openai.OpenAI()\n        response = client.chat.completions.create(\n            model=\"gpt-4\",\n            messages=[{\"role\": \"user\", \"content\": prompt}]\n        )\n        \n        # Save to database\n        content = GeneratedContent(\n            user_id=user_id,\n            prompt=prompt,\n            content=response.choices[0].message.content\n        )\n        db.add(content)\n        db.commit()\n        \n        return content.id\n    \n    except Exception as exc:\n        raise self.retry(exc=exc, countdown=60)
```

**2. API to Enqueue Tasks:**

```python
# api.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from tasks import analyze_document, generate_content
from celery.result import AsyncResult

app = FastAPI()
celery_app = analyze_document.app

class TaskRequest(BaseModel):
    document_id: str

class TaskResponse(BaseModel):
    task_id: str
    status: str

@app.post(\"/tasks/analyze\", response_model=TaskResponse)\nasync def analyze(request: TaskRequest):
    \"\"\"Enqueue document analysis task\"\"\"
    task = analyze_document.delay(request.document_id)\n    return {\"task_id\": task.id, \"status\": \"queued\"}

@app.get(\"/tasks/{task_id}\")\nasync def get_task_status(task_id: str):\n    \"\"\"Get task status and result\"\"\"
    result = AsyncResult(task_id, app=celery_app)\n    \n    if result.state == \"PENDING\":\n        return {\"status\": \"pending\"}\n    elif result.state == \"PROCESSING\":\n        return {\"status\": \"processing\"}\n    elif result.state == \"SUCCESS\":\n        return {\"status\": \"completed\", \"result\": result.result}\n    elif result.state == \"FAILURE\":\n        return {\"status\": \"failed\", \"error\": str(result.info)}\n```

**3. Worker Deployment with Docker:**

```dockerfile
# Dockerfile.worker
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

CMD [\"celery\", \"-A\", \"tasks\", \"worker\", \"--loglevel=info\", \"--concurrency=4\"]
```

**4. Kubernetes Deployment:**

```yaml
# k8s/worker-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: celery-worker
spec:
  replicas: 3
  selector:\n    matchLabels:\n      app: celery-worker\n  template:\n    metadata:\n      labels:\n        app: celery-worker\n    spec:\n      containers:\n      - name: worker\n        image: myregistry/celery-worker:latest\n        resources:\n          requests:\n            memory: \"512Mi\"\n            cpu: \"500m\"\n          limits:\n            memory: \"2Gi\"\n            cpu: \"2000m\"\n        env:\n        - name: REDIS_URL\n          value: \"redis://redis:6379\"\n        - name: OPENAI_API_KEY\n          valueFrom:\n            secretKeyRef:\n              name: openai-secret\n              key: api-key\n```

**5. Auto-Scaling Based on Queue Length (KEDA):**

```yaml
# k8s/celery-scaler.yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject\nmetadata:\n  name: celery-worker-scaler\nspec:\n  scaleTargetRef:\n    name: celery-worker\n  minReplicaCount: 1\n  maxReplicaCount: 20\n  triggers:\n  - type: redis\n    metadata:\n      address: redis:6379\n      listName: \"celery\"\n      listLength: \"5\"  # Scale when queue > 5 tasks\n```

**6. Monitoring with Flower:**

```bash
# Start Flower (Celery monitoring tool)\ncelery -A tasks flower --port=5555
```

```python
# Custom metrics\nfrom prometheus_client import Counter, Histogram, Gauge

tasks_total = Counter('celery_tasks_total', 'Total tasks', ['task_name', 'status'])\ntask_duration = Histogram('celery_task_duration_seconds', 'Task duration', ['task_name'])\nactive_workers = Gauge('celery_active_workers', 'Active workers')\n\n@app.task(bind=True)\ndef analyze_document(self, document_id: str):\n    start = time.time()\n    try:\n        # ... task logic\n        tasks_total.labels(task_name='analyze_document', status='success').inc()\n        task_duration.labels(task_name='analyze_document').observe(time.time() - start)\n        return result\n    except Exception:\n        tasks_total.labels(task_name='analyze_document', status='failure').inc()\n        raise
```

**7. Dead Letter Queue for Failed Tasks:**

```python
# Dead letter handler\nfrom celery.signals import task_failure\n\n@task_failure.connect\ndef handle_task_failure(task_id, exception, traceback, sender, **kwargs):\n    \"\"\"Move permanently failed tasks to DLQ\"\"\"
    if task.request.retries >= task.max_retries:\n        # Store in DLQ for manual review\n        dlq = redis.Redis(db=2)\n        dlq.lpush('dlq', json.dumps({\n            'task_id': task_id,\n            'task_name': sender.name,\n            'exception': str(exception),\n            'traceback': traceback,\n            'timestamp': datetime.now().isoformat()\n        }))\n```

**Key Components:**

1. **Task Queue:** Celery with Redis/RabbitMQ
2. **Workers:** Distributed AI processing
3. **Retries:** Exponential backoff for transient failures
4. **Dead Letter Queue:** For permanently failed tasks
5. **Auto-Scaling:** KEDA based on queue length
6. **Monitoring:** Flower + Prometheus
7. **Resource Limits:** Prevent memory leaks

**Scaling Strategies:**

```yaml
# 1. Horizontal scaling: More worker pods
# 2. Vertical scaling: More CPU/memory per worker
# 3. Concurrency: Multiple tasks per worker
# 4. Priority queues: Different priorities for different tasks
# 5. Rate limiting: Don't overwhelm external APIs
```

**Common Mistakes:**
- No retries (transient failures break workflow)
- No dead letter queue (lost tasks)
- Memory leaks (workers don't restart)
- No rate limiting (API quota exceeded)
- No monitoring (blind to failures)
- Synchronous processing (blocks API)

**Interview Tip:**
> "For distributed AI task processing, I use Celery with Redis for the queue. Tasks have retry logic with exponential backoff for transient failures (API rate limits, network issues). I use KEDA to auto-scale workers based on queue length, and Flower + Prometheus for monitoring. A dead letter queue captures permanently failed tasks for manual review. Workers restart after N tasks to prevent memory leaks. The key is retries with backoff, auto-scaling, and observability to ensure tasks complete reliably."

</details>

---

## Summary Checklist

- [x] Scalable RAG chat application
- [x] AI-powered code review system
- [x] Real-time AI dashboard with streaming
- [x] End-to-end ML model deployment
- [x] Multi-tenant SaaS architecture
- [x] Real-time collaboration platform
- [x] AI-powered search system
- [x] Serverless AI image processing
- [x] AI-powered customer support
- [x] Distributed task queue with AI workers

**Total: 10 questions** covering real-world integration scenarios combining React, Python, GenAI, and Cloud technologies.
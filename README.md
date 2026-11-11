# Model Context Protocol (MCP) — Comprehensive Reference Card

---

## 1. Basics & Motivation

### 1.1 What is MCP?
Model Context Protocol is an **open, standardized protocol** (originally released by Anthropic, Nov 2024) that defines how AI applications (hosts) connect LLMs to external context: data sources, tools, and predefined interaction templates. It solves the **M×N integration problem**: without a standard, every AI app needs custom integration code for every data source/tool combination (M apps × N sources = M×N integrations). MCP reduces this to **M+N** — each app implements the client side once, each system implements the server side once.

### 1.2 Analogy
MCP is often described as **"USB-C for AI applications"** — a universal connector. Just as USB-C lets any device talk to any peripheral through one physical/logical standard, MCP lets any MCP-compliant host talk to any MCP-compliant server.

### 1.3 Why not just use function calling / plugins?
| Aspect | Native Function Calling | MCP |
|---|---|---|
| Portability | Tied to one model provider's schema | Provider-agnostic, standardized schema |
| Discovery | Static, hardcoded in app | Dynamic discovery at runtime (`list_tools`, etc.) |
| Reusability | Rebuilt per app | Server built once, used by any MCP host |
| Scope | Tools only | Tools + Resources + Prompts + Sampling + Roots |
| Statefulness | Usually stateless | Stateful session with capability negotiation |

### 1.4 Core Design Principles
- **Separation of concerns**: the LLM app (host) never talks to the data source directly; a server mediates.
- **Capability negotiation**: client and server declare what they support before doing real work.
- **Transport agnostic**: same JSON-RPC 2.0 messages over different transports (stdio, HTTP+SSE, Streamable HTTP).
- **Security & consent by design**: hosts are expected to gate tool calls, resource access, and sampling requests behind user approval.
- **Composability**: a host can connect to many servers simultaneously; servers can be simple (single tool) or complex (full subsystem).

---

## 2. Core Architecture

```
┌────────────────────────────────────────────────────────────┐
│                          HOST                               │
│  (Claude Desktop, Claude Code, IDE, custom agent app, etc.) │
│                                                              │
│   ┌────────────┐   ┌────────────┐   ┌────────────┐          │
│   │  MCP Client │   │  MCP Client │  │  MCP Client │  ...    │
│   │  (1:1 conn) │   │  (1:1 conn) │  │  (1:1 conn) │          │
│   └──────┬──────┘   └──────┬──────┘   └──────┬──────┘        │
└──────────┼─────────────────┼─────────────────┼───────────────┘
           │ JSON-RPC 2.0    │                 │
           ▼                 ▼                 ▼
   ┌───────────────┐ ┌───────────────┐ ┌───────────────┐
   │  MCP Server A  │ │  MCP Server B  │ │  MCP Server C  │
   │  (e.g. GitHub) │ │ (e.g. Postgres)│ │ (e.g. Slack)   │
   └───────┬───────┘ └───────┬───────┘ └───────┬───────┘
           ▼                 ▼                 ▼
     External System   External System   External System
```

### 2.1 Three Roles

| Role | Responsibility |
|---|---|
| **Host** | The user-facing AI application. Owns the LLM, orchestrates multiple client connections, enforces security/consent policy, aggregates results back into the model's context. |
| **Client** | Lives *inside* the host. Maintains a **1:1 stateful connection** to exactly one server. Handles protocol-level message exchange, capability negotiation, and routing. |
| **Server** | A (usually lightweight) program that exposes capabilities — Tools, Resources, Prompts — over MCP. Wraps a specific system: a database, an API, a filesystem, a SaaS product. |

**Key rule:** one client ↔ one server. A host that talks to five servers instantiates five clients.

### 2.2 Local vs Remote Servers
- **Local server**: runs as a subprocess on the same machine as the host, communicates via **stdio**. Typical for filesystem, local git, local databases.
- **Remote server**: runs elsewhere (cloud), communicates via **HTTP** (with SSE or Streamable HTTP for streaming). Typical for SaaS integrations (GitHub, Slack, Google Drive), often requires **OAuth 2.1** authentication.

---

## 3. Protocol Design

### 3.1 Message Format — JSON-RPC 2.0
All MCP messages are JSON-RPC 2.0 objects. Three message types:

```json
// Request (expects a response)
{"jsonrpc": "2.0", "id": 1, "method": "tools/call", "params": {"name": "get_weather", "arguments": {"city": "Berlin"}}}

// Response
{"jsonrpc": "2.0", "id": 1, "result": {"content": [{"type": "text", "text": "18°C, cloudy"}]}}

// Notification (no response expected, no "id")
{"jsonrpc": "2.0", "method": "notifications/tools/list_changed"}
```

### 3.2 Transports

| Transport | Use case | Mechanics |
|---|---|---|
| **stdio** | Local subprocess servers | Host spawns server process; messages exchanged over stdin/stdout, newline-delimited JSON. Simple, no network, no auth needed. |
| **Streamable HTTP** (current standard, replaced HTTP+SSE) | Remote servers | Single HTTP endpoint supports POST (client→server messages) and optional SSE streams (server→client, for streaming/async responses). Supports session IDs via headers. |
| **HTTP+SSE** (legacy, pre-2025-03) | Remote servers | Separate SSE endpoint for server→client, separate POST endpoint for client→server. Superseded by Streamable HTTP. |

### 3.3 Session Lifecycle

```
Client                                   Server
  │                                        │
  │──── initialize (capabilities) ───────▶│
  │                                        │
  │◀─── initialize result (capabilities) ─│
  │                                        │
  │──── notifications/initialized ───────▶│
  │                                        │
  │◀════ normal operation ════════════════▶│
  │   tools/list, tools/call,              │
  │   resources/list, resources/read,      │
  │   prompts/list, prompts/get,           │
  │   sampling/createMessage (server→client)│
  │                                        │
  │──── (shutdown / connection close) ────▶│
```

1. **Initialization / Handshake**: client sends `initialize` with its protocol version + capabilities (e.g. supports sampling, roots). Server responds with its own protocol version + capabilities (tools, resources, prompts, logging).
2. **Capability negotiation**: both sides now know exactly what the other supports — servers must not assume a client can do sampling; clients must not call methods the server didn't declare.
3. **Operation phase**: normal request/response and notification exchange.
4. **Change notifications**: servers can emit `list_changed` notifications (e.g. `tools/list_changed`) if their available tools change dynamically, prompting the client to re-fetch.
5. **Shutdown**: connection closes (transport-dependent cleanup).

---

## 4. Core Components (Primitives)

MCP defines primitives split by **who controls them**:

### 4.1 Server-side primitives

| Primitive | Controlled by | Description | Analogy |
|---|---|---|---|
| **Tools** | Model-controlled | Executable functions the LLM can invoke (with sql query, side effects) to take action or fetch data. Discovered via `tools/list`, invoked via `tools/call`. Each tool has a name, description, and JSON Schema for inputs. | POST endpoint / function call |
| **Resources** | Application-controlled | Read-only, addressable data (files, DB rows, API responses) exposed via URIs. Fetched via `resources/read`. Meant to be attached to context, not executed. | GET endpoint / file |
| **Prompts** | User-controlled | Reusable, parameterized prompt templates / workflows the user can explicitly select (e.g. slash commands) to kick off a structured interaction. Discovered via `prompts/list`, invoked via `prompts/get`. | Slash command / macro |

### 4.2 Client-side primitives (server → client requests)

| Primitive | Description |
|---|---|
| **Sampling** | Lets a *server* ask the *client's* LLM to generate a completion — enabling agentic servers without embedding their own model credentials. Requires explicit human-in-the-loop approval by design. |
| **Roots** | Lets a *client* tell a server which directories/URIs it's allowed to operate within — scoping filesystem or resource access. |
| **Elicitation** | Lets a server ask the client to prompt the user for additional information mid-workflow (structured follow-up questions). |

### 4.3 Cross-cutting features
- **Logging**: servers can send structured log messages to clients (`notifications/message`).
- **Pagination**: `list` endpoints (tools/list, resources/list) support cursor-based pagination for large capability sets.
- **Completion / autocomplete**: servers can offer argument autocompletion for prompts/resources.
- **Progress notifications**: long-running tool calls can emit progress updates tied to a `progressToken`.

---

## 5. Structures — Message & Schema Reference

### 5.1 Tool Definition
```json
{
  "name": "search_flights",
  "description": "Search for flights between two airports on a given date.",
  "inputSchema": {
    "type": "object",
    "properties": {
      "origin": {"type": "string", "description": "IATA airport code"},
      "destination": {"type": "string", "description": "IATA airport code"},
      "date": {"type": "string", "format": "date"}
    },
    "required": ["origin", "destination", "date"]
  }
}
```

### 5.2 Tool Call Result
```json
{
  "content": [
    {"type": "text", "text": "Found 3 flights..."},
    {"type": "image", "data": "<base64>", "mimeType": "image/png"}
  ],
  "isError": false
}
```

### 5.3 Resource Definition
```json
{
  "uri": "postgres://mydb/customers/1024",
  "name": "Customer #1024",
  "mimeType": "application/json"
}
```

### 5.4 Prompt Definition
```json
{
  "name": "code_review",
  "description": "Generate a structured code review",
  "arguments": [
    {"name": "language", "description": "Programming language", "required": true}
  ]
}
```

### 5.5 Capability Object (exchanged at initialize)
```json
{
  "capabilities": {
    "tools": {"listChanged": true},
    "resources": {"subscribe": true, "listChanged": true},
    "prompts": {"listChanged": true},
    "sampling": {},
    "roots": {"listChanged": true}
  }
}
```

---

## 6. Workflows

### 6.1 Typical Tool-Use Workflow (agentic loop)
```
1. Host connects client(s) to server(s), completes handshake
2. Client calls tools/list → gets tool schemas from server
3. Host injects tool schemas into LLM context (system/tools param)
4. User sends a message
5. LLM decides to call a tool → emits tool_use block
6. Host maps tool_use → client.tools/call(name, arguments)
7. Server executes (e.g. hits an API, queries a DB)
8. Server returns result → client → host
9. Host feeds tool result back into LLM context as tool_result
10. LLM produces final answer (or loops back to step 5)
```

### 6.2 Resource-Attachment Workflow
```
1. Client calls resources/list → discovers available resources
2. User (or host logic) selects a resource to attach (e.g. "@file.pdf")
3. Client calls resources/read(uri)
4. Server returns resource content
5. Host inserts content into LLM context window
6. LLM answers using the attached context
```

### 6.3 Sampling Workflow (server-initiated LLM call)
```
1. Server is mid-task and needs an LLM completion it doesn't have direct model access to
2. Server sends sampling/createMessage request to client
3. Client surfaces the request to the user for approval (human-in-the-loop)
4. On approval, client forwards the prompt to its LLM
5. Client returns the LLM's completion to the server
6. Server continues its task using that completion
```

### 6.4 Multi-Server Orchestration Workflow
```
Host maintains N client connections (1 per server)
   │
   ├─ Client → GitHub MCP Server      (code, issues, PRs)
   ├─ Client → Postgres MCP Server    (structured data)
   ├─ Client → Slack MCP Server       (notifications)
   └─ Client → Filesystem MCP Server  (local files, scoped via Roots)

Host aggregates tools/list results from all servers into one
unified tool registry presented to the LLM. LLM picks whichever
tool (from whichever server) fits the current reasoning step.
Host routes the call to the correct client transparently.
```

---

## 7. Architectural Patterns

### 7.1 Single-Server Pattern
Simplest setup: one host, one client, one server. Good for a single well-scoped integration (e.g. "just Git operations").

### 7.2 Multi-Server Aggregation
Host connects to many servers simultaneously and merges their capabilities into one logical toolset for the LLM — the pattern used by Claude Desktop, Claude Code, and most production agent hosts. Requires namespacing (e.g. `github.create_issue` vs `jira.create_issue`) to avoid tool name collisions.

### 7.3 Gateway / Proxy Server Pattern
A single "gateway" MCP server sits between the host and many backend systems, itself acting as an aggregator/router. Useful for centralizing auth, rate limiting, and access policy across an organization instead of pushing that logic into every host.

### 7.4 Server-of-Servers (Composition)
An MCP server internally acts as a *client* to other MCP servers, composing/orchestrating them, then exposing a simplified, higher-level tool surface to its own clients. Enables building layered abstractions (e.g. a "trip-planner" server that internally calls flight, hotel, and calendar servers).

### 7.5 Sidecar / Local-Remote Split
Local MCP server (stdio) handles latency-sensitive or privacy-sensitive operations (local file access) while delegating heavier or shared operations to a remote MCP server (HTTP) — common in IDE agents (Claude Code) that need both local repo access and cloud APIs.

### 7.6 Human-in-the-Loop Gateway
Architected so every tool call, resource read, and sampling request passes through an explicit host-side consent/policy layer — critical for enterprise deployments where audit and approval workflows matter.

---

## 8. Security Model

| Concern | Mechanism |
|---|---|
| **Authorization to remote servers** | OAuth 2.1 (authorization code + PKCE), dynamic client registration |
| **User consent for tool execution** | Host-level approval UI before executing a tool call (esp. destructive actions) |
| **Scoped filesystem/data access** | Roots primitive — client tells server the allowed boundary |
| **Sampling abuse prevention** | Sampling requests must be explicitly approved by the human; servers cannot silently invoke the model |
| **Prompt injection from tool/resource content** | Host responsibility: treat tool/resource output as untrusted data, not instructions |
| **Transport security** | TLS for HTTP transports; stdio inherits OS process trust boundary |

**Design tenet:** MCP explicitly does **not** grant servers implicit trust — every capability is opt-in, negotiated, and (for sensitive actions) meant to be gated by human approval in the host.

---

## 9. SDKs & Ecosystem

| Layer | Options |
|---|---|
| **Official SDKs** | Python, TypeScript, Java, Kotlin, C#, Swift, Ruby (community + Anthropic-maintained) |
| **Server frameworks** | `FastMCP` (Python, high-level decorator-based server authoring), low-level SDK for full control |
| **Testing/dev tools** | **MCP Inspector** — a standalone GUI/CLI tool to connect to any MCP server, list its tools/resources/prompts, and manually invoke them for debugging |
| **Registries** | Community and vendor-hosted directories of public MCP servers (e.g. for GitHub, Slack, Google Drive, Postgres, Stripe, Notion, filesystem, Puppeteer/browser control) |
| **Hosts that speak MCP** | Claude Desktop, Claude Code, Claude Cowork, Claude.ai (via connectors), Cursor, Windsurf, various custom agent frameworks (LangGraph, LlamaIndex integrations, etc.) |

### 9.1 Minimal Python Server (FastMCP-style)
```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("weather-server")

@mcp.tool()
def get_weather(city: str) -> str:
    """Get current weather for a city."""
    return f"Weather in {city}: 18°C, cloudy"

@mcp.resource("config://settings")
def get_settings() -> str:
    """Expose read-only server settings."""
    return '{"units": "metric"}'

if __name__ == "__main__":
    mcp.run(transport="stdio")
```

### 9.2 Minimal Client Connection (Python)
```python
from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client

server_params = StdioServerParameters(command="python", args=["weather_server.py"])

async with stdio_client(server_params) as (read, write):
    async with ClientSession(read, write) as session:
        await session.initialize()
        tools = await session.list_tools()
        result = await session.call_tool("get_weather", {"city": "Berlin"})
```

---

## 10. Applications & Use Cases

| Domain | Example MCP servers / use |
|---|---|
| **Software engineering** | Git/GitHub (issues, PRs, code search), filesystem access, CI/CD status, code execution sandboxes |
| **Data & analytics** | Postgres/MySQL/Snowflake/Databricks connectors exposing schema-aware query tools, BI dashboard resources |
| **Productivity / knowledge work** | Slack, Gmail, Google Drive, Notion, Confluence — search, read, and act on documents/messages |
| **DevOps / infra** | Kubernetes, AWS/Azure/GCP resource inspection and control, monitoring/alerting systems |
| **Browser / web automation** | Puppeteer/Playwright-backed servers for navigating and scraping pages, filling forms |
| **Enterprise agent platforms** | Internal gateway servers exposing company-specific APIs/tools/data uniformly to any MCP-compatible agent |
| **Multi-agent systems** | One agent's MCP server acting as another agent's toolset — composable agent-as-a-tool patterns |
| **IDE / coding agents** | Claude Code, Cursor, Windsurf using local filesystem + git servers plus remote issue-tracker servers in one session |

---

## 11. Quick Comparison Cheat Table

| Concept | MCP Term | Rough Equivalent Elsewhere |
|---|---|---|
| Executable action | Tool | Function/plugin, REST POST |
| Read-only data | Resource | REST GET, file, DB row |
| Reusable template | Prompt | Slash command, macro |
| Server→client LLM call | Sampling | Callback / inversion of control |
| Access scoping | Roots | Chroot, sandbox path allowlist |
| Handshake | Initialize | HTTP handshake / gRPC negotiation |
| Wire format | JSON-RPC 2.0 | gRPC, REST+JSON, GraphQL |
| Transport (local) | stdio | Unix pipe |
| Transport (remote) | Streamable HTTP | REST + SSE / WebSockets |

---

## 12. Enterprise Knowledge Base Architecture with MCP

### 12.1 Why MCP fits internal knowledge bases
Large companies typically have knowledge scattered across Confluence, SharePoint, file shares, wikis, ticketing systems, and databases — much of it confidential (HR, legal, financial, customer data). MCP is a good fit because:
- **Uniform interface**: one protocol regardless of whether the backend is a document store, a database, or a vector index.
- **Fine-grained scoping**: Roots + resource-level metadata allow per-department, per-classification, per-role access control enforced *before* content ever reaches the model.
- **Air-gap friendly**: stdio-based local servers mean a fully on-prem stack (model + retrieval + orchestration) never has to call an external API.
- **Auditability**: every `tools/call` and `resources/read` is a discrete, loggable event — important for compliance (GDPR, DSGVO, SOC 2, industry-specific regulation).

### 12.2 Reference Architecture (on-prem / air-gapped)

```
┌───────────────────────────────────────────────────────────────────┐
│                         Corporate Network (on-prem / VPC)          │
│                                                                     │
│  ┌───────────────┐         ┌────────────────────────────────────┐ │
│  │   Host / Agent │◀──────▶│         MCP Client Layer            │ │
│  │  (internal UI, │         │  (1 client per server, in-host)    │ │
│  │  Streamlit/    │         └───────────┬────────────┬──────────┘ │
│  │  custom chat)  │                     │            │            │
│  └────────┬───────┘                     │            │            │
│           │ prompt/response             ▼            ▼            │
│           │                  ┌────────────────┐ ┌────────────────┐│
│           │                  │ Doc/KB MCP     │ │ Structured-Data ││
│           │                  │ Server         │ │ MCP Server      ││
│           │                  │ (resources +   │ │ (SQL/NoSQL      ││
│           │                  │  search tool)  │ │  read-only tool)││
│           │                  └───────┬────────┘ └───────┬────────┘│
│           │                          ▼                  ▼         │
│           │                 ┌─────────────────┐ ┌───────────────┐ │
│           │                 │ Vector Store     │ │ Data Vault /  │ │
│           │                 │ (Qdrant/Chroma/  │ │ Warehouse      │ │
│           │                 │  pgvector, local)│ │ (read replica) │ │
│           │                 └────────┬─────────┘ └───────────────┘ │
│           │                          ▲                             │
│           │                 ┌────────┴─────────┐                  │
│           │                 │ Ingestion Pipeline│                  │
│           │                 │ (chunk+embed, on  │                  │
│           │                 │  a schedule/CDC)  │                  │
│           │                 └───────────────────┘                  │
│           ▼                                                       │
│  ┌────────────────────────┐                                       │
│  │  Local LLM Inference    │  (Ollama / vLLM / TGI — no external   │
│  │  (Llama 3.x, Mistral,   │   network calls; sampling stays       │
│  │   Qwen, DeepSeek, etc.) │   inside the perimeter)                │
│  └────────────────────────┘                                       │
└───────────────────────────────────────────────────────────────────┘
        No traffic crosses the corporate network boundary
```

Key property: **nothing in the request/response path (prompt, retrieved chunks, completion) leaves the network boundary.** This is the architecture used for GDPR/DSGVO-sensitive deployments (e.g. HR knowledge bases, customer contracts, source code) where sending data to a hosted model API is not acceptable.

### 12.3 Core Components

| Component | Role |
|---|---|
| **Document/KB MCP Server** | Exposes internal documents as **Resources** (by URI, e.g. `confluence://space/HR/page/123`) and a **Tool** like `search_knowledge_base(query, filters)` that performs retrieval and returns ranked chunks. |
| **Ingestion Pipeline** | Offline/batch process: pulls source documents (Confluence export, SharePoint API, file share crawl), chunks them, computes embeddings with a **local** embedding model, upserts into the vector store. Runs independently of the MCP server (cron, Airflow/Databricks job, CDC-triggered). |
| **Vector Store** | Local vector DB (Qdrant, Chroma, pgvector, Milvus, Weaviate — all self-hostable). Never a hosted/managed cloud vector service if data must stay on-prem. |
| **Structured-Data MCP Server** | Read-only tool layer over a database or warehouse (e.g. a Data Vault 2.0 Business Vault or a curated Gold layer) — returns tabular results, never raw write access. |
| **Local LLM Inference Server** | Serves the open-source model. **Ollama** for simplicity/single-node, **vLLM** or **TGI (Text Generation Inference)** for higher-throughput/multi-user serving with continuous batching. |
| **Host / Agent App** | Orchestrates the conversation, aggregates tool results, enforces the classification/consent policy, renders the UI (internal chat app, Streamlit, Slack bot, IDE plugin). |

---

## 13. Open-Source Model Stack for Confidential Data

### 13.1 Model choices (all self-hostable, no external API calls)

| Model family | Typical sizes | Notes |
|---|---|---|
| **Llama 3.x / 3.2** | 1B–70B | Strong general baseline, good tool-calling support in recent versions, wide tooling compatibility (Ollama, vLLM, llama.cpp). |
| **Mistral / Mixtral** | 7B–8x22B (MoE) | Efficient, permissive license, good multilingual (relevant for German-language corpora). |
| **Qwen 2.5 / 3** | 0.5B–72B | Strong multilingual + coding performance, competitive with larger closed models at similar scale. |
| **DeepSeek-R1 (distilled)** | 1.5B–70B | Reasoning-focused distillations, useful for multi-step retrieval-then-reason tasks. |
| **Embedding models** | bge-m3, nomic-embed-text, multilingual-e5-large | Run locally alongside the generator; multilingual variants matter for German/English mixed corpora. |

### 13.2 Serving layer options

| Serving stack | Fit |
|---|---|
| **Ollama** | Fastest to stand up; good for single-node / small-team deployments, dev/test, or a single power-user chat app. Simple REST API, built-in model management. |
| **vLLM** | Production-grade, high-throughput, continuous batching, PagedAttention — fit for many concurrent users inside a company. OpenAI-compatible API surface, easy to slot behind an MCP server. |
| **TGI (Hugging Face Text Generation Inference)** | Similar production-grade target to vLLM, strong Hugging Face ecosystem integration. |
| **llama.cpp** | Lowest resource footprint, CPU-only or small-GPU edge/laptop deployments, quantized (GGUF) models. |

### 13.3 Deployment topologies

| Topology | Description | When to use |
|---|---|---|
| **Fully air-gapped** | No internet egress at all from the inference/retrieval network segment; model weights and embedding models pre-downloaded and mirrored internally. | Highest-sensitivity data (defense, government, top-secret IP). |
| **On-prem, internet-restricted** | Inference and vector store on-prem; only patch/update traffic allowed through an internal proxy/allowlist. | Standard enterprise confidential data (HR, legal, financial). |
| **Private cloud / VPC, no public model API** | Runs in a cloud VPC (Azure/AWS/GCP) with self-hosted model containers (e.g. AKS/EKS + vLLM), but VNet/VPC has no route to public model provider APIs. | Companies already cloud-native but with contractual/regulatory data-residency requirements. |
| **Hybrid** | Confidential retrieval + local model for sensitive queries; escalate only non-sensitive, already-redacted queries to a hosted frontier model for higher quality when needed. | Balances cost/quality against strict-but-not-total confidentiality requirements. |

---

## 14. Sample MCP Server Configurations

### 14.1 Document/KB MCP Server (Python, FastMCP-style) — Resources + Search Tool

```python
from mcp.server.fastmcp import FastMCP
from qdrant_client import QdrantClient
from sentence_transformers import SentenceTransformer

mcp = FastMCP("internal-knowledge-base")

# Local embedding model — no external calls
embedder = SentenceTransformer("BAAI/bge-m3")
vector_db = QdrantClient(host="localhost", port=6333)  # self-hosted, on-prem

# Classification levels gate which collection(s) a given user/session may query
CLASSIFICATION_COLLECTIONS = {
    "public": ["kb_public"],
    "internal": ["kb_public", "kb_internal"],
    "confidential": ["kb_public", "kb_internal", "kb_confidential"],
}

@mcp.tool()
def search_knowledge_base(query: str, user_clearance: str, top_k: int = 5) -> str:
    """Search the internal knowledge base for relevant document chunks,
    scoped to the caller's clearance level."""
    collections = CLASSIFICATION_COLLECTIONS.get(user_clearance, ["kb_public"])
    query_vector = embedder.encode(query).tolist()

    results = []
    for collection in collections:
        hits = vector_db.search(collection_name=collection, query_vector=query_vector, limit=top_k)
        results.extend(hits)

    results.sort(key=lambda h: h.score, reverse=True)
    top_hits = results[:top_k]

    return "\n\n".join(
        f"[{h.payload['source']}] (score={h.score:.2f})\n{h.payload['text']}"
        for h in top_hits
    )

@mcp.resource("kb://document/{doc_id}")
def get_document(doc_id: str) -> str:
    """Fetch the full text of a specific internal document by ID."""
    # lookup against document store / object storage, permission-checked upstream
    return fetch_document_text(doc_id)  # implementation-specific

if __name__ == "__main__":
    mcp.run(transport="stdio")  # local/on-prem — no network exposure
```

### 14.2 Retrieval Tool Schema (as advertised via `tools/list`)

```json
{
  "name": "search_knowledge_base",
  "description": "Search the internal knowledge base for relevant document chunks, scoped to the caller's clearance level.",
  "inputSchema": {
    "type": "object",
    "properties": {
      "query": {"type": "string"},
      "user_clearance": {"type": "string", "enum": ["public", "internal", "confidential"]},
      "top_k": {"type": "integer", "default": 5}
    },
    "required": ["query", "user_clearance"]
  }
}
```

### 14.3 Sample `docker-compose.yml` for a Self-Hosted Stack

```yaml
version: "3.9"
services:
  ollama:
    image: ollama/ollama:latest
    volumes: ["ollama_models:/root/.ollama"]
    ports: ["11434:11434"]
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]
    networks: [kb_net]

  qdrant:
    image: qdrant/qdrant:latest
    volumes: ["qdrant_data:/qdrant/storage"]
    ports: ["6333:6333"]
    networks: [kb_net]

  kb_mcp_server:
    build: ./kb_mcp_server
    depends_on: [qdrant]
    environment:
      - QDRANT_HOST=qdrant
      - EMBEDDING_MODEL=BAAI/bge-m3
    networks: [kb_net]
    # stdio transport: this service is spawned by the host, not exposed via ports

  ingestion_pipeline:
    build: ./ingestion
    depends_on: [qdrant]
    environment:
      - SOURCE=confluence
      - QDRANT_HOST=qdrant
    networks: [kb_net]
    restart: "no"  # run as scheduled job, not always-on

networks:
  kb_net:
    internal: true   # no external egress from this network

volumes:
  ollama_models:
  qdrant_data:
```

Note the `internal: true` network flag — this is the mechanism that actually enforces "no external egress" at the infrastructure level, not just the application level.

### 14.4 Host Configuration Connecting to Internal MCP Servers

```json
{
  "mcpServers": {
    "internal-knowledge-base": {
      "command": "python",
      "args": ["/opt/mcp-servers/kb_server/main.py"],
      "env": {
        "QDRANT_HOST": "localhost",
        "CLEARANCE_SOURCE": "sso_claims"
      }
    },
    "structured-data": {
      "command": "python",
      "args": ["/opt/mcp-servers/data_server/main.py"],
      "env": {
        "DB_CONNECTION_STRING": "postgresql://readonly_user@warehouse-replica/gold"
      }
    }
  }
}
```

Both servers run **locally as subprocesses of the host** (stdio transport) — no network hop, no external exposure, ideal for a company-managed desktop/VDI deployment of an internal chat client.

---

## 15. Data Classification & Access Control Patterns

### 15.1 Resource classification tagging
Every ingested document/chunk should carry classification metadata (`public` / `internal` / `confidential` / `restricted`) at ingestion time, stored alongside the vector embedding as payload metadata — enabling **filtering at query time**, not just after retrieval.

### 15.2 Roots for per-role scoping
The client can set **Roots** dynamically per session based on the authenticated user's role (from SSO/SAML claims), restricting which resource URIs or vector collections a given session's server calls are allowed to touch — enforced at the protocol level, not just inside application logic.

### 15.3 Audit logging
Because every `tools/call` and `resources/read` is a discrete JSON-RPC message, the MCP server (or a logging proxy in front of it) can persist a full audit trail: who queried what, when, which documents were retrieved, and what the model was ultimately shown — a requirement for many compliance regimes (GDPR Art. 30 records of processing, SOC 2 audit trails).

### 15.4 Redaction / PII filtering layer
For confidential corpora, consider inserting a redaction step between retrieval and context assembly — a local NER model or a rules-based scrubber removes PII/PCI fields before the chunk is handed to the LLM, especially useful in hybrid topologies where output might later be escalated to a less-trusted downstream system.

---

## 16. Sample End-to-End Workflow: Employee Query Against Confidential Corpus

```
1. Employee authenticates to internal chat host (SSO) → host resolves clearance = "internal"
2. Host initializes MCP client → connects to internal-knowledge-base server (stdio, local)
3. Client calls tools/list → gets search_knowledge_base tool schema
4. Employee asks: "What's our current parental leave policy?"
5. Local LLM (via Ollama/vLLM) decides to call search_knowledge_base(
     query="parental leave policy", user_clearance="internal")
6. MCP server embeds query locally (bge-m3), searches Qdrant scoped to
   ["kb_public", "kb_internal"] collections only — confidential collection excluded
7. Server returns top-k chunks with source citations
8. Host logs the tool call + retrieved doc IDs to the audit store
9. Host injects retrieved chunks into LLM context as tool_result
10. Local LLM generates final answer, citing internal HR policy documents
11. Response rendered to employee — at no point did any prompt, chunk,
    or completion leave the corporate network boundary
```

---

## 17. Key Takeaways

1. MCP standardizes the **host–client–server** triangle so AI apps and external systems can interoperate without bespoke integration code (M+N instead of M×N).
2. Three server-side primitives — **Tools** (model-controlled), **Resources** (app-controlled), **Prompts** (user-controlled) — cover the full spectrum of "do something," "read something," "run a known workflow."
3. **Sampling** and **Roots** flip the direction of control, letting servers request LLM inference or respect client-defined boundaries.
4. Everything rides on **JSON-RPC 2.0** over either **stdio** (local) or **Streamable HTTP** (remote), with an explicit **capability-negotiation handshake**.
5. Production architectures typically **aggregate many servers behind one host**, sometimes via gateway/composition layers, always with human-in-the-loop consent as a first-class design constraint.
6. For **enterprise knowledge bases with confidential data**, MCP's local-first design (stdio transport, self-hosted servers) combined with **open-source models** (Llama/Mistral/Qwen via Ollama/vLLM) and **self-hosted vector stores** (Qdrant/Chroma/pgvector) enables a fully on-prem RAG stack where no prompt, retrieved content, or completion ever leaves the corporate network.
7. **Classification-aware retrieval** (metadata filtering + Roots-based scoping) and **full audit logging** of every tool/resource call are the two mechanisms that make MCP-based knowledge bases viable under GDPR/DSGVO and similar compliance regimes.

# Mastra Framework — Architecture Diagrams

Modular AI agent framework. Central `Mastra` class wires agents, workflows, memory, storage, MCP, observability. Four levels of detail below.

> Rendered with Mermaid `flowchart` (cleaner auto-layout than native C4 shapes).

---

## Level 1 — System Context

```mermaid
flowchart TB
    classDef person fill:#08427B,stroke:#052E56,color:#fff,stroke-width:1px
    classDef system fill:#1168BD,stroke:#0B4884,color:#fff,stroke-width:1px
    classDef ext fill:#999,stroke:#6B6B6B,color:#fff,stroke-width:1px

    dev(["👤 Developer<br/>builds agentic apps"]):::person
    eu(["👤 End User<br/>chats with agents"]):::person

    mastra["🧠 Mastra Application<br/>Agent orchestration runtime<br/>HTTP API + Studio UI"]:::system

    llm["LLM Providers<br/>OpenAI · Anthropic · Google · Groq · Bedrock"]:::ext
    vec["Vector Stores<br/>pgvector · Pinecone · Qdrant · Chroma · LanceDB"]:::ext
    db["Storage Backends<br/>Postgres · LibSQL · Mongo · DynamoDB · ClickHouse"]:::ext
    voice["Voice Providers<br/>OpenAI · ElevenLabs · Deepgram · Google · Azure"]:::ext
    obs["Observability<br/>Langfuse · Langsmith · Braintrust · Datadog · Sentry · OTel"]:::ext
    mcp["MCP Ecosystem<br/>external tool servers"]:::ext
    auth["Auth Providers<br/>Auth0 · Clerk · WorkOS · Supabase · Firebase · Okta"]:::ext
    deploy["Deploy Targets<br/>Mastra Cloud · Vercel · Netlify · Cloudflare"]:::ext

    dev -->|defines agents<br/>workflows· tools| mastra
    eu  -->|prompts · workflow triggers| mastra

    mastra -->|generate completions| llm
    mastra -->|semantic recall · RAG| vec
    mastra -->|threads · traces · runs| db
    mastra -->|STT / TTS| voice
    mastra -->|spans · metrics · logs| obs
    mastra -->|tool discovery · invocation| mcp
    mastra -->|authn / authz| auth
    dev    -->|ships via deployer| deploy
```

---

## Level 2 — Container

```mermaid
flowchart TB
    classDef person fill:#08427B,stroke:#052E56,color:#fff
    classDef cont fill:#438DD5,stroke:#2E6295,color:#fff
    classDef db fill:#438DD5,stroke:#2E6295,color:#fff,stroke-dasharray: 4 2
    classDef ext fill:#999,stroke:#6B6B6B,color:#fff

    dev(["👤 Developer"]):::person
    eu(["👤 End User"]):::person

    subgraph APP["Mastra Application"]
        direction TB

        subgraph EDGE["Edge / Clients"]
            direction LR
            cli["Mastra CLI<br/>create-mastra · dev · build · deploy"]:::cont
            studio["Studio (Playground UI)<br/>React + Vite"]:::cont
            sdk["Client SDK<br/>client-js · react · ai-sdk"]:::cont
        end

        subgraph API["HTTP Surface"]
            server["Mastra Server<br/>Hono (Express/Fastify/Koa adapters)<br/>REST + SSE · auth middleware"]:::cont
        end

        subgraph RUNTIME["Runtime"]
            direction LR
            core["Core Runtime<br/>@mastra/core<br/>Mastra class · DI · hooks · registries"]:::cont
            builder["Agent Builder<br/>@mastra/agent-builder"]:::cont
            mcpHost["MCP Host/Client<br/>@mastra/mcp"]:::cont
        end

        subgraph SERVICES["Domain Services"]
            direction LR
            mem["Memory Service<br/>@mastra/memory<br/>threads · working · semantic"]:::cont
            rag["RAG Pipeline<br/>@mastra/rag<br/>chunk · embed · retrieve · rerank"]:::cont
            evals["Evals / Scorers<br/>@mastra/evals"]:::cont
            obsBridge["Observability Bridge<br/>otel-bridge + exporters"]:::cont
        end

        subgraph INFRA["Infra Adapters"]
            direction LR
            stores[("Store Adapters<br/>stores/*")]:::db
            deployers["Deployers<br/>deployers/*"]:::cont
        end
    end

    llm["LLM Providers"]:::ext
    vecExt["Vector DBs"]:::ext
    dbExt["SQL/NoSQL DBs"]:::ext
    obsExt["Observability SaaS"]:::ext
    mcpExt["External MCP Servers"]:::ext

    dev    --> cli
    dev    --> sdk
    eu     --> studio
    studio --> server
    sdk    --> server
    server --> core

    core --> builder
    core --> mem
    core --> rag
    core --> evals
    core --> mcpHost
    core --> obsBridge
    core --> stores
    core --> llm

    mem      --> vecExt
    rag      --> vecExt
    stores   --> dbExt
    obsBridge --> obsExt
    mcpHost  --> mcpExt

    cli --> deployers
```

---

## Level 3 — Core Runtime Components

Internals of `@mastra/core`. Grouped by responsibility.

```mermaid
flowchart TB
    classDef hub fill:#F59E0B,stroke:#B45309,color:#fff,stroke-width:2px
    classDef comp fill:#85BBF0,stroke:#5D82A8,color:#000
    classDef abs fill:#C7D2FE,stroke:#6366F1,color:#000,stroke-dasharray: 4 2
    classDef evt fill:#FCA5A5,stroke:#B91C1C,color:#000

    subgraph CORE["@mastra/core"]
        direction TB

        Mastra["🎯 Mastra<br/>central DI container<br/>registers everything"]:::hub

        subgraph AGENTS["Agent Layer"]
            direction LR
            agent["Agent<br/>generate / stream"]:::comp
            tla["ToolLoopAgent<br/>lightweight variant"]:::comp
            loop["Loop Engine<br/>tool-call loop"]:::comp
            stream["Stream<br/>unified primitives"]:::comp
            proc["Processors<br/>guardrails · PII · transforms"]:::comp
        end

        subgraph TOOLING["Tools"]
            direction LR
            tools["ToolAction<br/>typed w/ Zod"]:::comp
            tprov["Tool Provider<br/>dynamic sources"]:::comp
            mcpMod["MCP Server/Client<br/>host + consume"]:::comp
        end

        subgraph MODEL["Model"]
            direction LR
            llmMod["LLM + Gateways<br/>model routing"]:::comp
        end

        subgraph STATE["State / Memory"]
            direction LR
            memAbs["MastraMemory<br/><<abstract>>"]:::abs
            storeAbs["MastraCompositeStore<br/><<abstract>>"]:::abs
            vecAbs["MastraVector<br/><<abstract>>"]:::abs
            cache["Server Cache"]:::comp
        end

        subgraph FLOW["Workflows"]
            direction LR
            wf["Workflow Engine<br/>step DAG · suspend/resume"]:::comp
            wfep["WorkflowEventProcessor"]:::evt
        end

        subgraph OPS["Ops"]
            direction LR
            obsMod["Observability<br/>spans · metrics · logger ctx"]:::comp
            hooks["Hooks + PubSub"]:::evt
            scorers["Scorers (evals)"]:::comp
            reqCtx["Request Context<br/>AsyncLocalStorage"]:::comp
            bg["Background Tasks"]:::comp
            datasets["DatasetsManager"]:::comp
            ws["Workspace"]:::comp
        end

        subgraph EDGE["Edge"]
            direction LR
            srvAbs["MastraServerBase<br/><<abstract>>"]:::abs
            bundler["Bundler + Deployer"]:::comp
            voiceAbs["MastraTTS / Voice<br/><<abstract>>"]:::abs
        end
    end

    Mastra --> agent
    Mastra --> wf
    Mastra --> memAbs
    Mastra --> storeAbs
    Mastra --> vecAbs
    Mastra --> mcpMod
    Mastra --> scorers
    Mastra --> voiceAbs
    Mastra --> srvAbs
    Mastra --> obsMod
    Mastra --> hooks
    Mastra --> bg
    Mastra --> datasets
    Mastra --> cache
    Mastra --> ws

    agent --> llmMod
    agent --> loop
    agent --> tools
    agent --> tprov
    agent --> memAbs
    agent --> proc
    agent --> voiceAbs
    agent --> stream
    agent --> reqCtx
    tla   --> agent

    wf    --> wfep
    wf    --> storeAbs
    wfep  --> hooks

    memAbs --> storeAbs
    memAbs --> vecAbs

    obsMod  --> hooks
    scorers --> hooks
    bundler --> srvAbs
```

---

## Level 4 — Agent Class

```mermaid
classDiagram
    direction LR

    class Mastra {
        +agents: Record
        +workflows: Record
        +memory
        +storage: MastraCompositeStore
        +vectors: Record
        +mcpServers: Record
        +scorers: Record
        +getAgent(name)
        +getWorkflow(name)
    }

    class Agent {
        +name: string
        +instructions: string | fn
        +model: LanguageModel
        +tools: Record
        +memory?: MastraMemory
        +voice?: MastraVoice
        +inputProcessors: Processor[]
        +outputProcessors: Processor[]
        +scorers: Record
        +generate(messages, opts)
        +stream(messages, opts)
        +getMemory()
        +getTools(ctx)
    }

    class ToolAction {
        +id: string
        +description: string
        +inputSchema: ZodSchema
        +outputSchema?: ZodSchema
        +execute(input, ctx)
    }

    class MastraMemory {
        <<abstract>>
        +saveMessages()
        +query(threadId, opts)
        +rememberMessages()
        +getWorkingMemory()
        +updateWorkingMemory()
    }

    class Workflow {
        +id: string
        +steps: Step[]
        +triggerSchema
        +createRun()
        +suspend()
        +resume()
    }

    Mastra "1" o-- "*" Agent
    Mastra "1" o-- "*" Workflow
    Mastra "1" o-- "1" MastraMemory
    Agent "1" o-- "*" ToolAction
    Agent ..> MastraMemory : uses
    Workflow ..> Agent : may call
```

---

## Runtime Sequences

Dynamic views for major paths through Mastra. Each shows one realistic flow end-to-end.

---

### Seq 1 — Agent Stream: Memory + Tool Call + RAG

Most common path. User asks question → agent pulls history + RAG context → LLM decides tool call → loops → streams answer back.

```mermaid
sequenceDiagram
    autonumber
    participant Client as Client<br/>(Studio / SDK)
    participant Server as Mastra Server<br/>(Hono)
    participant Mastra as Mastra<br/>(DI hub)
    participant Agent
    participant IProc as InputProcessors
    participant Mem as MastraMemory
    participant Store as Storage
    participant Vec as Vector Store
    participant RAG as RAG Pipeline
    participant LLM as LLM Provider
    participant Tool as ToolAction
    participant OProc as OutputProcessors
    participant Obs as Observability

    Client->>Server: POST /agents/:id/stream { messages, threadId }
    Server->>Mastra: getAgent(id)
    Mastra-->>Server: Agent
    Server->>Agent: stream(messages, ctx)
    Agent->>Obs: start span "agent.stream"

    Agent->>IProc: run(messages)
    IProc-->>Agent: sanitized messages

    par history recall
        Agent->>Mem: rememberMessages(threadId)
        Mem->>Store: load last N msgs
        Store-->>Mem: messages
        Mem->>Vec: semantic recall (query embedding)
        Vec-->>Mem: similar past msgs
        Mem-->>Agent: context messages
    and RAG retrieval
        Agent->>RAG: retrieve(query)
        RAG->>Vec: similarity search (docs index)
        Vec-->>RAG: top-k chunks
        RAG-->>Agent: context chunks
    end

    Agent->>LLM: generate(prompt + tools schema)
    LLM-->>Agent: tool_call(name=search, args)
    Agent->>Obs: span "tool.search"
    Agent->>Tool: execute(args, ctx)
    Tool-->>Agent: result
    Agent->>LLM: continue(tool result)
    LLM-->>Agent: final assistant text (stream chunks)

    Agent->>OProc: transform chunks (PII, format)
    OProc-->>Agent: safe chunks
    Agent-->>Server: SSE: text deltas + events
    Server-->>Client: stream response

    Agent->>Mem: saveMessages(user + assistant)
    Mem->>Store: persist
    Agent->>Obs: end span
```

---

### Seq 2 — Workflow Execution: Suspend / Resume

Workflows are event-sourced step DAGs. Any step can suspend (wait for human/external signal) and resume later — state persisted to storage.

```mermaid
sequenceDiagram
    autonumber
    participant Client
    participant Server
    participant Mastra
    participant WF as Workflow
    participant Run as WorkflowRun
    participant EP as WorkflowEventProcessor
    participant Store as Storage
    participant Hook as Hooks / PubSub
    participant Step1 as Step A (fn)
    participant Step2 as Step B (agent)
    participant Agent

    Client->>Server: POST /workflows/:id/runs { input }
    Server->>Mastra: getWorkflow(id)
    Mastra-->>Server: Workflow
    Server->>WF: createRun(input)
    WF->>Run: new run(id, input)
    Run->>Store: persist run (status=running)

    Run->>Step1: execute(input)
    Step1-->>Run: output A
    Run->>EP: emit "step.complete" A
    EP->>Store: append event
    EP->>Hook: publish

    Run->>Step2: execute(A)
    Step2->>Agent: generate(prompt from A)
    Agent-->>Step2: draft
    Step2->>Run: suspend(reason="await human approval", snapshot)
    Run->>Store: persist snapshot + status=suspended
    Run-->>Server: { status: suspended, runId }
    Server-->>Client: 202 Accepted { runId }

    Note over Client,Store: ...time passes — human reviews...

    Client->>Server: POST /workflows/runs/:id/resume { approval }
    Server->>WF: resume(runId, approval)
    WF->>Store: load snapshot
    WF->>Run: rehydrate
    Run->>Step2: continue(approval)
    Step2-->>Run: final output
    Run->>EP: emit "run.complete"
    EP->>Store: append event
    Run->>Store: status=success + result
    Server-->>Client: final result (or webhook)
```

---

### Seq 3 — RAG Ingestion Pipeline

Developer indexes documents so agents can retrieve them. One-shot (or batch) path — not per-request.

```mermaid
sequenceDiagram
    autonumber
    participant Dev as Developer / Job
    participant RAG as RAG Pipeline
    participant Chunk as Chunker
    participant Embed as Embedding Model
    participant Vec as Vector Store
    participant Store as Storage (metadata)

    Dev->>RAG: ingest(docs[], config)
    loop per document
        RAG->>Chunk: split(doc, strategy)
        Chunk-->>RAG: chunks[]
        RAG->>Embed: embed(chunks)
        Embed-->>RAG: vectors[]
        RAG->>Vec: upsert({id, vector, metadata})
        Vec-->>RAG: ack
        RAG->>Store: save doc manifest
    end
    RAG-->>Dev: { indexed, count, index }

    Note over Dev,Vec: Later: Agent / RAG.retrieve(query) → similarity search → top-k chunks
```

---

### Seq 4 — MCP Tool Discovery + Invocation

Mastra as MCP **client** — pulls tools from external MCP servers into agent's tool set at runtime.

```mermaid
sequenceDiagram
    autonumber
    participant Mastra
    participant Agent
    participant MCPHost as MCP Host/Client
    participant MCPSrv as External MCP Server<br/>(stdio / SSE / HTTP)
    participant LLM

    Note over Mastra,MCPHost: Boot — register MCP servers in Mastra config

    Mastra->>MCPHost: connect(serverUrl / command)
    MCPHost->>MCPSrv: initialize handshake
    MCPSrv-->>MCPHost: capabilities
    MCPHost->>MCPSrv: tools/list
    MCPSrv-->>MCPHost: [{name, schema}...]
    MCPHost-->>Mastra: cached tool registry

    Note over Agent,LLM: Per request

    Agent->>MCPHost: getTools()
    MCPHost-->>Agent: ToolAction wrappers
    Agent->>LLM: generate(prompt + tool schemas)
    LLM-->>Agent: tool_call(name=mcp__github__create_issue, args)
    Agent->>MCPHost: invoke(name, args)
    MCPHost->>MCPSrv: tools/call
    MCPSrv-->>MCPHost: result
    MCPHost-->>Agent: result
    Agent->>LLM: continue(result)
    LLM-->>Agent: final text
```

Inverse direction — Mastra as MCP **server** — expose agents/workflows as tools to external clients (Claude Desktop, Cursor, etc.):

```mermaid
sequenceDiagram
    autonumber
    participant ExtClient as External MCP Client<br/>(Claude Desktop / Cursor)
    participant MCPSrv as MCPServerBase<br/>(Mastra)
    participant Mastra
    participant Agent

    ExtClient->>MCPSrv: initialize
    MCPSrv-->>ExtClient: capabilities
    ExtClient->>MCPSrv: tools/list
    MCPSrv->>Mastra: list agents / workflows
    Mastra-->>MCPSrv: registry
    MCPSrv-->>ExtClient: [{name: "agent_support", schema}, ...]

    ExtClient->>MCPSrv: tools/call "agent_support" { query }
    MCPSrv->>Mastra: getAgent("support")
    MCPSrv->>Agent: generate(query)
    Agent-->>MCPSrv: response
    MCPSrv-->>ExtClient: result
```

---

### Seq 5 — Evals / Scorer Hook

Scorers run async after agent run via hooks. Don't block response. Results persisted for dashboards.

```mermaid
sequenceDiagram
    autonumber
    participant Agent
    participant Hook as Hooks (PubSub)
    participant Bg as BackgroundTaskManager
    participant Scorer as MastraScorer
    participant Judge as LLM Judge (optional)
    participant Store as Storage
    participant Obs as Observability

    Agent->>Agent: finish run
    Agent->>Hook: publish "agent.run.complete" {runId, input, output}
    Agent-->>Agent: return to caller (non-blocking)

    Hook->>Bg: enqueue scorer job
    Bg->>Scorer: score({input, output, expected?})
    alt LLM-judge scorer
        Scorer->>Judge: evaluate(rubric, output)
        Judge-->>Scorer: verdict + reasoning
    else deterministic scorer
        Scorer->>Scorer: compute metric (regex, similarity, etc.)
    end
    Scorer-->>Bg: { score, reason, metadata }
    Bg->>Store: saveScore(runId, score)
    Bg->>Obs: emit metric
```

---

### Seq 6 — Voice (STT → Agent → TTS)

Full voice round-trip. Used by real-time agents.

```mermaid
sequenceDiagram
    autonumber
    participant Client as Client (mic)
    participant Server
    participant Agent
    participant STT as Voice (STT)
    participant LLM
    participant TTS as Voice (TTS)

    Client->>Server: WS /agents/:id/voice (audio stream)
    Server->>Agent: streamVoice(audio)
    Agent->>STT: transcribe(audio chunks)
    STT-->>Agent: text
    Agent->>LLM: generate(text + context)
    LLM-->>Agent: text response
    Agent->>TTS: synthesize(text)
    TTS-->>Agent: audio stream
    Agent-->>Server: audio chunks
    Server-->>Client: WS audio back
```

---

### Seq 7 — Deploy Flow (CLI → Deployer → Target)

Build-time path. Not runtime, but worth showing for ops.

```mermaid
sequenceDiagram
    autonumber
    participant Dev as Developer
    participant CLI as Mastra CLI
    participant Bundler
    participant Deployer as Deployer<br/>(cloud / vercel / netlify / cf)
    participant Target as Deploy Target

    Dev->>CLI: mastra deploy
    CLI->>Bundler: bundle(entry, config)
    Bundler->>Bundler: tree-shake · compile · resolve adapters
    Bundler-->>CLI: artifact (zip / wrangler / etc.)
    CLI->>Deployer: push(artifact, env)
    Deployer->>Target: upload + configure
    Target-->>Deployer: deployment URL
    Deployer-->>CLI: url + logs
    CLI-->>Dev: ✅ deployed at https://...
```

---

## Key Takeaways

- `Mastra` class = DI hub. Everything registered on it.
- `Agent` = instructions + model + tools + memory + processors. Runs tool-loop.
- `Workflow` = step DAG, suspend/resume, event-sourced via storage.
- Memory split: thread messages (storage) + semantic recall (vector) + working memory (storage).
- MCP dual-role: Mastra hosts MCP servers AND consumes them.
- Observability decouples via hooks + OTel bridge → many exporters.
- Stores / voice / deployers = pluggable adapters behind abstract interfaces.

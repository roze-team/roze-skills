# AI Patterns

Use this reference for `roze-ai`, `rozectl ai generate`, provider-neutral AI runtime integration, workflow graphs, RAG, teams, streaming, checkpoints, and AI module ownership.

## Runtime Boundary

`roze-ai` is experimental and does not change stable REST, RPC, model, search, OpenAPI, SDK, stream, Docker, or Kubernetes generation behavior. It provides Roze-native contracts for chat models, tools, bounded agents, workflow graphs, prompt templates, RAG, teams, OpenAI-compatible providers, runtime events, and deterministic mock models.

Do not introduce parallel framework systems for AI code. AI providers and tools must reuse:

- `roze_context::Context` for request id, trace id, tenant, locale, deadline, cancellation, subject, and permissions
- `roze-config` for provider settings and secret references
- `roze-service` and generated `ApplicationExtensions` for lifecycle attachment
- existing governance for timeout, retry, breaker, rate limit, fallback, and metrics policy
- `roze-search`, `roze-storage`, database/cache/transaction/MQ/report crates for data and persistence

The core Agent never retries model or tool calls by itself. Provider network retries remain a governance integration concern.

## Generation Workflow

Generate an AI module inside an existing generated REST/RPC project:

```bash
cargo run -p rozectl -- ai generate assistant --out services/support --roze-source path
rozectl ai generate assistant --out services/support --with-workflow --with-rag --with-team
```

The target must already contain `Cargo.toml`, `src/application.rs`, and `src/svc/mod.rs`. The command is transactional: it edits the manifest, formats generated Rust, and promotes the staged project only after validation succeeds.

Generated ownership:

- `src/ai/mod.rs` and `src/ai/generated.rs` are framework-owned and refreshed.
- `src/ai/agent.rs`, `src/ai/tools.rs`, and `src/ai/prompts/**` are application-owned.
- Optional `src/ai/workflow.rs`, `src/ai/rag.rs`, and `src/ai/team.rs` are application-owned and added with `--with-workflow`, `--with-rag`, and `--with-team`.
- `config/ai.example.yaml` is application-owned provider configuration guidance to copy into preserved `config.yaml`.
- The marker-delimited AI module declaration appended to `src/application.rs` is generator-owned; the rest of `src/application.rs` remains application-owned.
- `Cargo.toml` is preserved except for adding `roze-ai` and, when RAG is generated, `roze-search`.

Use `--update` for normal regeneration. Use `--force` only when intentionally replacing the complete `src/ai` scaffold.

## Service Attachment And Config

Attach the runtime once in the preserved `configure_context` hook:

```rust
pub async fn configure_context(ctx: ServiceContext) -> anyhow::Result<ServiceContext> {
    ai::attach(&ctx)?;
    Ok(ctx)
}
```

Application code accesses it through `application::ai::runtime(&ctx)` and passes the inbound `&roze_context::Context` to model, tool, graph, RAG, or team calls.

AI provider settings live under `ServiceConfig.ai`; `roze-ai` does not load files or environment variables itself. `roze-config` resolves `${NAME}`, `env://NAME`, and `file://path` before `AiRuntime::from_config` builds shared provider clients. Provider names must be non-empty, `default_provider` must exist, `max_steps` must be 1..=64, URLs must be HTTP(S) without embedded credentials, timeouts must be non-zero, and debug output must not include API keys.

Without an `ai` section, generated application-owned `agent.rs` keeps deterministic mock development mode. With `ai` configured, providers are built from config.

## Tools And Agents

Tools declare required permissions in `ToolDefinition`. `ToolRegistry` checks all declared permissions against the inbound Roze context and fails closed when any permission is missing.

Agents require a positive bounded `max_steps`, check cancellation/deadline before and after model and tool calls, emit bounded runtime events, and must not log message bodies, tool arguments, tool outputs, secrets, or provider error payloads by default. HTTP 429 from providers maps to Roze rate-limited errors; timeout/connection/5xx maps to service unavailable; other provider rejections map to internal provider errors without copying response bodies into errors.

## Workflow Graphs

`GraphBuilder` compiles application-owned `WorkflowNode`/`FnNode` components into immutable `CompiledGraph` values. Compilation rejects missing nodes, duplicate edges, cycles, unreachable nodes, and graphs without an explicit path from `START` to `END`.

Every node receives the inbound context. A branch receives cloned input; a join receives a JSON array in edge declaration order. `ParallelLayers` may execute independent nodes concurrently, but commits outputs and streaming events in deterministic node order. A compiled graph can be wrapped as a permission-checked AI `Tool`.

`CompiledGraph::stream` emits deterministic run/node lifecycle events. `CompiledGraph::stream_chunks` is only for strict linear `START -> ... -> END` paths, applies backpressure, and requires a mandatory `max_chunks` budget. Do not infer broadcast, merge, zip, or branch chunk semantics; model those explicitly in application nodes.

## Checkpoints, RAG, And Teams

`WorkflowRunner` stores opaque checkpoints after completed nodes and can interrupt/resume before configured nodes. Resume validates checkpoint version, graph revision, tenant, and subject, and completed runs delete their checkpoint.

`MemoryCheckpointStore` is for development and tests. `ObjectStorageCheckpointStore` reuses `roze-storage`; configure `.json` and `application/json` in storage validation. Object storage has no compare-and-swap or lease contract, so concurrent resumes require an application-selected database lock, distributed lock, or job ownership mechanism.

RAG uses `CharacterTextSplitter`, `Embedder`, `Retriever`, `Indexer`, `RagPipeline`, and adapters over existing `roze-search`. Retrieval and indexing reuse inbound context cancellation/deadline and do not add retries, caches, or storage implementations. Ranking, filters, reranking, domain mapping, embedding persistence, and authorization policy remain application-owned.

`AgentTeam` runs bounded sequential or parallel tasks, preserves explicit task ordering, aggregates usage, and propagates the same context boundary. `delegation_tool` exposes only registered agent names under the `ai.delegate` permission. Delegated agents do not receive delegation recursively unless the application explicitly wires that behavior.

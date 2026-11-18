# ROADMAP — pedantigo-ai

This roadmap carries **both** the dependency-ordered build plan **and** the complete
feature-parity catalog targeting **100% parity with [Pydantic AI](https://github.com/pydantic/pydantic-ai)**.

Planned work that is decided but not yet implemented. Pedantigo AI is greenfield, so the plan
is organized as dependency-ordered **phases**; the detail is organized as the full **feature
catalog** (8 pillars). Every catalog feature is tagged with the phase that delivers it, so a
contributor can read top-down (what to build first) or by pillar (the full spec of a feature).

**Verification basis:** the Pydantic AI feature surface below was enumerated from
`pydantic-ai==1.25.0` (114 top-level exports across the `models/`, `providers/`, `toolsets/`,
`profiles/`, `common_tools/`, `durable_exec/`, `ui/`, `ext/` subpackages; `messages.py`,
`output.py`, `exceptions.py`, `builtin_tools.py`, `usage.py`, `settings.py`, `retries.py`,
`direct.py`). The pedantigo baseline is from the core [`pedantigo`](https://github.com/Pedantigo/pedantigo)
library. Nothing here is from model memory.

---

## Milestones

- **M1 — Tier 2 (typed extraction):** end of **Phase 4**. One prompt in, one validated struct
  out, with automatic retry. An `instructor-go` equivalent.
- **M2 — Tier 1 (full agent):** end of **Phase 7**. The model takes multiple actions (tools,
  streaming) to reach a validated result. A Pydantic AI equivalent.
- **M3 — Production-grade / full parity:** end of **Phase 9**. Usage limits, observability,
  durable execution, provider breadth, and ecosystem integrations.

## Guiding constraints

- **Core stays untouched.** Pedantigo AI is a separate module; it depends on a *released*
  `github.com/Pedantigo/pedantigo/v2` and never forces its dependencies onto core consumers.
- **Reuse, don't reinvent.** Tool schemas, output schemas, and streaming-result validation go
  through pedantigo's `validator` package, not a second schema/validation mechanism.
- **Test without the network.** `TestModel`/`FunctionModel` (Phase 2) land early so every
  later phase is table-driven and offline.

---

## The pedantigo baseline — what already exists (so we don't re-port it)

| Capability | Status |
|---|---|
| Struct-tag validation (100+ rules: email, URL, UUID, regex, numeric range, string length) | ✅ exists |
| JSON Schema generation (`SchemaJSONLLM[T]()`, no `$schema`, for LLM tool-calling + OpenAPI) | ✅ exists |
| Cached schema generation (~240× speedup) | ✅ exists |
| Streaming validation of **partial JSON** (byte/structure level) | ✅ exists |
| Discriminated unions (type-safe) | ✅ exists |
| Cross-field validation | ✅ exists |

pedantigo is the Go equivalent of *Pydantic the validation library* — the **foundation**.
Everything in Pillars 1–8 is the **runtime** that Pydantic AI stacks on top. pedantigo's schema
engine, unions, and partial-JSON validation are a genuine head start on three of the hardest
pieces (tool schemas, output schemas, partial-output validation).

## Parity gap at a glance

| Pillar | Features | Baseline help | Phase(s) |
|---|---|---|---|
| 1. Model connectivity | `Model` interface, 15 backends, 28 providers, profiles, settings, fallback, HTTP retries | none | 1, 9 |
| 2. Agent & run loop | Agent, run/run_sync/run_stream/iter, graph nodes, EndStrategy, deps injection, system prompts | none | 3 |
| 3. Structured output | 4 output modes, output validators, `ModelRetry` loop, JSON-schema transformers | schema gen + unions ✅ | 4 |
| 4. Tools | function tools + schema gen, 13 toolset combinators, deferred/approval, 7 built-in tools, common tools, MCP | schema gen ✅ | 6, 9 |
| 5. Messages & multimodal | message/part protocol, multimodal content, history serialization, thinking parts | none | 5 |
| 6. Streaming & events | run-level event stream, text/tool-call/thinking deltas, parts manager, UI adapters | partial-JSON validation ✅ | 7 |
| 7. Reliability & ops | usage/limits/cost, error taxonomy, observability (OTel), durable execution | none | 8 |
| 8. Ecosystem | A2A, LangChain/ACI ext, CLI, test/function models | none | 2, 9 |

---

# Part A — Build phases (the plan)

## Phase 0 — Scaffolding & module layout
- **Goal:** a buildable, lintable, CI-green empty module.
- **Scope:** `go.mod` (`module github.com/Pedantigo/pedantigo-ai`, Go 1.25+, `require` core);
  package layout (`model/`, `provider/`, `agent/`, `output/`, `tool/`, `toolset/`, `message/`,
  `stream/`, `usage/`, `otel/`, `direct/`, `cli/`); `Makefile`, `.golangci.yml`, pre-commit, CI +
  coverage workflows mirroring core pedantigo; `LICENSE` (MIT).
- **Acceptance:** `make lint test` passes on an empty skeleton; CI and coverage badges green.

## Phase 1 — Model connectivity (minimum viable) — *delivers Pillar 1 (core)*
- **Goal:** send a request to one real provider and get a response back.
- **Scope:** the `Model` interface; one backend (OpenAI wire format) + one provider (OpenRouter);
  `Settings`; HTTP-level retry transport kept separate from `ModelRetry`.
- **Acceptance:** a key-gated integration test completes one chat request; unit tests cover
  settings merging and retry/backoff.

## Phase 2 — Test & function models — *delivers Pillar 8.5*
- **Goal:** everything after this is testable with zero network and zero API keys.
- **Scope:** `TestModel` (schema-valid canned output) and `FunctionModel` (scriptable), both
  implementing `Model`.
- **Acceptance:** a stubbed agent driven by `FunctionModel` asserts exact model I/O in a
  table-driven test.

## Phase 3 — The Agent & run loop — *delivers Pillar 2*
- **Goal:** a reusable `Agent` that executes a run against a `Model`.
- **Scope:** `Agent[Deps, Out]`; `RunSync`/`Run`/`RunStream`; the run loop + `EndStrategy`;
  `RunContext[Deps]`; static + dynamic system prompts/instructions.
- **Acceptance:** `agent.RunSync` returns a result from a `FunctionModel`; deps and dynamic
  prompts observable in callbacks; table-driven loop tests.

## Phase 4 — Structured output + validation + `ModelRetry` — *delivers Pillar 3* → **M1**
- **Goal:** typed, validated output with automatic self-correction.
- **Scope:** the 4 output modes; schema injection via pedantigo `SchemaJSONLLM`; unmarshal +
  validate with pedantigo; output-validator hooks; the `ModelRetry` loop; per-profile JSON-schema
  transformers.
- **Acceptance:** given invalid-then-valid model output, the agent retries and returns a
  validated struct; each output mode covered; retry budget enforced.
- **Milestone:** usable `instructor-go` equivalent.

## Phase 5 — Message & multimodal protocol — *delivers Pillar 5*
- **Goal:** a fully typed conversation substrate with history.
- **Scope:** `Message`/`Part` types; multimodal content; JSON (de)serialization with per-part
  discriminators; `message_history` in/out of runs.
- **Acceptance:** a multi-turn conversation round-trips through JSON and resumes; a vision request
  carrying an `ImageURL` serializes to the provider format.

## Phase 6 — Tools (function calling + execution) — *delivers Pillar 4 (core)*
- **Goal:** the model can call registered Go functions and the framework executes them.
- **Scope:** `Tool`/`ToolDefinition` + registration (reflect → pedantigo schema); execute-and-
  feed-back loop; toolset combinators; human-in-the-loop approval.
- **Acceptance:** an agent calls a tool, receives its result, continues to final output; a gated
  tool pauses and resumes on approval.

## Phase 7 — Streaming & events — *delivers Pillar 6* → **M2**
- **Goal:** stream a whole run as typed events.
- **Scope:** run-level event stream; text/tool-call/thinking deltas; parts manager; streaming
  partial-JSON validation of the forming output via pedantigo; UI adapters (AG-UI, Vercel AI).
- **Acceptance:** `RunStream` yields ordered deltas; output struct progressively validated; an
  AG-UI/Vercel SSE stream is emitted.
- **Milestone:** functional Pydantic AI equivalent.

## Phase 8 — Reliability & operations — *delivers Pillar 7*
- **Goal:** production safety and visibility.
- **Scope:** usage accounting + cost; `UsageLimits`; the full error taxonomy; model fallback;
  OpenTelemetry instrumentation; durable execution via Temporal.
- **Acceptance:** a token cap aborts with `UsageLimitExceeded`; primary-model failure falls back;
  OTel spans appear; a Temporal-backed run resumes after a simulated crash.

## Phase 9 — Breadth & ecosystem — *delivers Pillar 1 (breadth), Pillar 4 (built-ins/MCP), Pillar 8*
- **Goal:** feature-complete parity.
- **Scope:** remaining backends/providers + profiles; built-in provider-executed tools; common
  client-side tools; MCP client; direct/imperative API; A2A; external integrations; the CLI.
- **Acceptance:** each new backend passes a shared provider conformance suite; MCP tools appear as
  a toolset; the CLI runs an agent.

---

# Part B — Feature parity catalog (the 100% detail)

Each feature: **What it is / How it works / Real-world use cases / Porting to Go / Parity status /
Delivered in.** Parity status: `MISSING` / `PARTIAL (reuse pedantigo)` / `FOUNDATION EXISTS`.

## Pillar 1 — Model connectivity

### 1.1 The `Model` abstraction — *Phase 1*
- **What it is.** A single provider-agnostic interface every LLM backend implements.
- **How it works.** Pydantic AI's `models/__init__.py` defines an abstract `Model` with
  `request()` (one-shot) and `request_stream()` (streaming), taking `ModelMessage`s + settings and
  returning a `ModelResponse`. Each concrete file adapts it to a vendor wire format.
- **Use cases.** Swap `gpt-4o` for `claude-3.5-sonnet` by changing a string; point one agent at a
  self-hosted Ollama; A/B-test providers behind one code path.
- **Porting to Go.**
  ```go
  type Model interface {
      Request(ctx context.Context, msgs []Message, s Settings) (ModelResponse, error)
      RequestStream(ctx context.Context, msgs []Message, s Settings) (ResponseStream, error)
      Name() string
      Profile() ModelProfile
  }
  ```
- **Parity status:** MISSING.

### 1.2 Model backends (15 implementations) — *Phase 1 (OpenAI), Phase 9 (rest)*
- **What it is.** Concrete `Model` implementations per vendor API shape.
- **How it works.** Backends: `openai`, `anthropic`, `gemini`, `google`, `groq`, `mistral`,
  `cohere`, `bedrock`, `huggingface`, `openrouter`, plus `outlines` (local constrained decoding),
  `test`/`function` (testing), `fallback`, `wrapper`, `instrumented`, `mcp_sampling`. Each
  translates the common protocol to/from the vendor's request/response, tool-call, and streaming
  formats.
- **Use cases.** Hide vendor quirks (Anthropic's top-level `system`; Gemini's `contents`/`parts`;
  Bedrock's SigV4 wrapping).
- **Porting to Go.** One file per backend implementing `Model`, over existing Go SDKs.
- **Parity status:** MISSING (15 backends).

### 1.3 Providers / connection profiles (28) — *Phase 1 (OpenRouter), Phase 9 (rest)*
- **What it is.** The connection + auth + base-URL + default-model config, distinct from the wire-
  format model.
- **How it works.** `providers/`: `openai`, `azure`, `anthropic`, `google_gla`, `google_vertex`,
  `bedrock`, `cohere`, `groq`, `mistral`, `huggingface`, `deepseek`, `grok`, `ollama`, `litellm`,
  `together`, `fireworks`, `cerebras`, `github`, `heroku`, `nebius`, `moonshotai`, `ovhcloud`,
  `vercel`, `openrouter`, `gateway`. Many reuse the OpenAI wire format with a different endpoint/
  auth (e.g. OpenRouter = OpenAI format + `base_url=https://openrouter.ai/api/v1` + `sk-or-…` key).
- **Use cases.** Route the same model through OpenRouter, Azure, Together, or a corporate gateway
  by selecting a provider.
- **Porting to Go.** A `Provider` struct (`BaseURL`, `APIKey`, auth strategy) + a factory returning
  a configured `Model`.
- **Parity status:** MISSING (28 providers).

### 1.4 Model profiles — *Phase 1 (default), Phase 9 (per-family)*
- **What it is.** Per-model capability/quirk descriptors so the framework adapts to what a model
  supports.
- **How it works.** `profiles/` has one file per family (`openai`, `anthropic`, `google`, `grok`,
  `groq`, `deepseek`, `mistral`, `cohere`, `amazon`, `meta`, `moonshotai`, `qwen`, `harmony`). A
  `ModelProfile`/`ModelProfileSpec` records "supports tool calling?", "JSON mode?", "which schema
  dialect?", "thinking?", plus a `JsonSchemaTransformer` that rewrites the schema to the dialect
  the model accepts.
- **Use cases.** A model without native tool calls falls back to prompted JSON; a model that
  rejects `$ref` gets an inlined schema; strict-mode schemas get required extra constraints.
- **Porting to Go.** A `ModelProfile` struct + `SchemaTransformer` func; **plugs into pedantigo's
  schema generator** (post-process `SchemaJSONLLM` output).
- **Parity status:** MISSING (hooks into pedantigo schema gen).

### 1.5 Model settings — *Phase 1*
- **What it is.** Per-request knobs: temperature, max_tokens, top_p, timeout, stop, seed, penalties.
- **How it works.** `settings.py`'s `ModelSettings` is merged from agent defaults + per-run
  overrides and passed into `Model.request`.
- **Use cases.** Deterministic extraction (`temperature=0`) vs creative writing; cap `max_tokens`
  for cost; per-call `timeout`.
- **Porting to Go.** A `Settings` struct with pointer fields (unset ≠ zero) or functional options;
  precedence run > agent > provider.
- **Parity status:** MISSING.

### 1.6 Fallback models — *Phase 8*
- **What it is.** Try a sequence of models in order; on failure, advance.
- **How it works.** `models/fallback.py` wraps N models; a retriable `ModelAPIError` advances to the
  next, aggregating into `FallbackExceptionGroup`.
- **Use cases.** Auto-failover on provider outage; cheap-first then escalate.
- **Porting to Go.** A `FallbackModel` looping over `[]Model`, collecting errors (`errors.Join`).
- **Parity status:** MISSING.

### 1.7 HTTP-level retries — *Phase 1*
- **What it is.** Transport-layer retry/backoff for transient network/rate-limit errors — **not**
  the semantic `ModelRetry` (3.3).
- **How it works.** `retries.py` provides retrying HTTP transports (exponential backoff, respect
  `Retry-After`) under the model's HTTP client.
- **Use cases.** Survive `429` or a dropped connection without failing the run.
- **Porting to Go.** An `http.RoundTripper` wrapper with backoff; strictly separate from the
  `ModelRetry` loop.
- **Parity status:** MISSING.

## Pillar 2 — The Agent & run loop

### 2.1 The `Agent` type — *Phase 3*
- **What it is.** The central object binding model + instructions + tools + output type + settings
  + typed deps into one reusable callable.
- **How it works.** `agent/__init__.py` defines `Agent` generic over deps and output type
  (`output_type` defaults to `str`); construction wires in model, prompts, tools/toolsets, output
  spec, default settings. Reusable, effectively immutable config.
- **Use cases.** Define a `support_agent` once, call it for thousands of tickets; a `planner` with
  a structured output type.
- **Porting to Go.**
  ```go
  type Agent[Deps any, Out any] struct { /* model, tools, prompts, settings, outputSpec */ }
  func New[Deps, Out any](m Model, opts ...Option[Deps, Out]) *Agent[Deps, Out]
  ```
  Generics (`Agent[Deps, Out]`) drive the Go 1.25+ requirement.
- **Parity status:** MISSING.

### 2.2 Run methods: `run`, `run_sync`, `run_stream`, `iter` — *Phase 3*
- **What it is.** Four ways to execute an agent: async, blocking, streaming, node-by-node.
- **How it works.** `agent/abstract.py` + `run.py`: `run` (async → `AgentRunResult`), `run_sync`
  (blocking), `run_stream` (live stream), `iter`/`AgentRun` (step through graph nodes).
  `capture_run_messages` captures history even on failure.
- **Use cases.** `run_sync` in a CLI; `run` in a server handler; `run_stream` for chat; `iter` for
  debugging/custom control.
- **Porting to Go.** No async/sync split — `RunSync` blocks, `RunStream` returns a channel/iterator,
  `Iter` yields nodes via `iter.Seq`.
- **Parity status:** MISSING.

### 2.3 The run graph: nodes + `EndStrategy` — *Phase 3*
- **What it is.** The agentic loop as an explicit state graph.
- **How it works.** `_agent_graph.py` (on `pydantic-graph`) defines `UserPromptNode` →
  `ModelRequestNode` → `CallToolsNode` → (loop) → end; `EndStrategy` controls stop-at-first-output
  vs exhaust-tool-calls.
- **Use cases.** Trace where a run is; custom stop conditions; checkpointing for durable execution.
- **Porting to Go.** A small graph engine or an explicit state machine with a `Node` interface + an
  `EndStrategy` enum (the node abstraction later enables durable execution and `iter`).
- **Parity status:** MISSING.

### 2.4 Dependency injection (`deps` + `RunContext`) — *Phase 3*
- **What it is.** Typed per-run dependencies threaded into tools and dynamic prompts.
- **How it works.** `_run_context.py`'s `RunContext[AgentDepsT]` carries `deps`, usage, retry count,
  current message; tools/dynamic prompts receive it. `Agent` is generic over the deps type.
- **Use cases.** A `get_orders` tool needing the authed user ID + a DB pool, injected per request; a
  prompt embedding the current user's name.
- **Porting to Go.**
  ```go
  type RunContext[Deps any] struct { Deps Deps; Usage *Usage; Retry int; Messages []Message }
  ```
- **Parity status:** MISSING.

### 2.5 System prompts & instructions (static + dynamic) — *Phase 3*
- **What it is.** Fixed and per-run-computed steering text; **instructions** (not persisted) vs
  **system prompts** (persisted into history).
- **How it works.** `_system_prompt.py` supports static strings and decorated functions receiving
  `RunContext` and returning a string at run time, concatenated in order.
- **Use cases.** Static role text; dynamic "user tz is X, today is Y"; the instructions/system-prompt
  split matters when replaying a conversation.
- **Porting to Go.** A `SystemPrompt` option accepting `string` or
  `func(RunContext[Deps]) (string, error)`; keep an `Instructions` vs `SystemPrompt` distinction.
- **Parity status:** MISSING.

## Pillar 3 — Structured output

### 3.1 Output modes (4 strategies) — *Phase 4*
- **What it is.** Four strategies for getting typed output, because models differ.
- **How it works.** `output.py`: `ToolOutput` (force the result through a tool call), `NativeOutput`
  (provider `response_format` JSON-schema mode), `PromptedOutput` (schema in the prompt, parse
  text), `TextOutput` (free text, optional post-process). `StructuredDict`/`OutputObjectDefinition`
  support schema-described dicts.
- **Use cases.** `NativeOutput` strict mode for guaranteed-valid JSON; `ToolOutput` on good-function-
  calling models; `PromptedOutput` as a weak-model fallback; `TextOutput` for a summarizer.
- **Porting to Go.** An `OutputMode` interface with implementations; each decides how the pedantigo
  schema is injected and how the reply is extracted. Reliability: Native(strict) > Tool > Prompted.
- **Parity status:** PARTIAL — schema gen exists; the mode strategies + injection/extraction MISSING.

### 3.2 Typed output + validation — *Phase 4*
- **What it is.** Coerce the reply into a typed Go value and validate it.
- **How it works.** The output is JSON-unmarshaled into the target and validated; output validator
  functions run custom rules and can trigger a retry via `ModelRetry`.
- **Use cases.** "The plan must have 2–5 steps and unique ids" — a validator enforces it and re-asks.
- **Porting to Go.** **pedantigo's struct-tag validator *is* this step.** Add an
  `OutputValidator[Out] func(RunContext, Out) error` hook.
- **Parity status:** PARTIAL — validation engine exists (pedantigo); validator-hook + retry wiring
  MISSING.

### 3.3 The `ModelRetry` loop (self-correction) — *Phase 4*
- **What it is.** On validation failure, feed the error **back to the model** to fix the output, up
  to a budget.
- **How it works.** `exceptions.ModelRetry` is raised by output/tool validators; the loop appends a
  `RetryPromptPart` describing the failure and re-requests. `ToolRetryError`/`IncompleteToolCall` are
  related; retries counted in `RunContext`.
- **Use cases.** The model returns `{"age": -3}`; the validator rejects it; the framework tells the
  model "age must be ≥ 0" and it self-corrects — no human. The dividing line between foundation and
  Tier-2.
- **Porting to Go.** A sentinel `ModelRetry{Message string}`; the loop detects it, emits a
  `RetryPromptPart`, re-calls until `maxRetries`.
- **Parity status:** MISSING (pedantigo validates but never re-asks).

### 3.4 JSON-schema transformers — *Phase 4*
- **What it is.** Rewrite a generated schema into the dialect a model accepts.
- **How it works.** `JsonSchemaTransformer`/`InlineDefsJsonSchemaTransformer` + per-profile
  transformers inline `$defs`, drop unsupported keywords, enforce `additionalProperties:false` for
  strict mode.
- **Use cases.** OpenAI strict mode forbids extra properties; Gemini rejects `$ref`. One generated
  schema works everywhere.
- **Porting to Go.** A `SchemaTransformer func(Schema) Schema` applied to pedantigo's output per
  profile.
- **Parity status:** MISSING (operates on pedantigo's schema output).

## Pillar 4 — Tools

### 4.1 Function tools + automatic schema generation — *Phase 6*
- **What it is.** Register a Go function as a tool the model can call; generate its schema, advertise
  it, execute it, feed the result back.
- **How it works.** `tools.py` + `_function_schema.py` + `_griffe.py`: `Tool`/`ToolDefinition` hold
  name/description/parameters; `@agent.tool` (gets `RunContext`) / `@agent.tool_plain` register
  functions; parameter schemas are reflected and **docstrings parsed (griffe) for descriptions**.
- **Use cases.** `get_weather(city)`, `search_db(query)`, `send_email(to, body)` — called mid-run.
- **Porting to Go.**
  ```go
  func (a *Agent[Deps, Out]) Tool(name, desc string, fn any) // fn reflected via pedantigo
  ```
  Go lacks docstrings, so descriptions come from struct tags, not griffe.
- **Parity status:** PARTIAL — schema gen exists; registration/execution/feedback MISSING.

### 4.2 Toolsets (13 combinators) — *Phase 6*
- **What it is.** Composable *collections* of tools with transformations.
- **How it works.** `toolsets/`: `AbstractToolset`, `FunctionToolset`, `CombinedToolset`,
  `FilteredToolset`, `PrefixedToolset`/`RenamedToolset`, `PreparedToolset` (dynamic per-run tool
  list), `WrapperToolset`, `ExternalToolset`/`DeferredToolset`, `ApprovalRequiredToolset`;
  `ToolsetFunc`/`ToolsetTool` building blocks.
- **Use cases.** Mount an MCP server's tools under a `github_` prefix; expose only read-only tools to
  untrusted users; compute the tool list from the user's plan; gate destructive tools.
- **Porting to Go.** A `Toolset` interface with the same combinators, each wrapping another.
- **Parity status:** MISSING (13 types).

### 4.3 Deferred tools & human-in-the-loop approval — *Phase 6*
- **What it is.** Tool calls that can't complete synchronously — need approval or run out-of-process.
- **How it works.** `exceptions.ApprovalRequired`/`CallDeferred`; `DeferredToolRequests`,
  `DeferredToolResults`, `ToolApproved`, `ToolDenied`, `DeferredToolCalls`. A tool raises
  `ApprovalRequired`; the run pauses and surfaces the request; a human approves/denies; the run
  resumes.
- **Use cases.** "Approve a $5,000 refund?"; require a human click for a destructive migration; a
  long external job returning later.
- **Porting to Go.** Sentinel errors + a `DeferredRequests` payload from a paused run + a resume API
  taking `ToolApproved`/`ToolDenied`. Pairs with durable execution (7.5).
- **Parity status:** MISSING.

### 4.4 Built-in (provider-executed) tools — *Phase 9*
- **What it is.** Tools the **provider** runs server-side; you just enable them.
- **How it works.** `builtin_tools.py`: `WebSearchTool` (+ `WebSearchUserLocation`), `WebFetchTool`,
  `UrlContextTool`, `CodeExecutionTool`, `ImageGenerationTool`, `MemoryTool`, `MCPServerTool`
  (extend `AbstractBuiltinTool`). Enabling one tells the provider to perform the action and return
  it inline (`BuiltinToolCallPart`/`BuiltinToolReturnPart`).
- **Use cases.** Live web search without wiring a search API; run model-authored code in the
  provider sandbox; inline image generation.
- **Porting to Go.** Marker structs per tool setting provider request flags; backend-specific, gated
  by profile.
- **Parity status:** MISSING (7 tools).

### 4.5 Common (client-side) tool integrations — *Phase 9*
- **What it is.** Ready-made tools for popular services you run client-side.
- **How it works.** `common_tools/`: `duckduckgo`, `tavily` search tools as prebuilt `Tool`s.
- **Use cases.** Add web search in one line.
- **Porting to Go.** Prebuilt `Tool` constructors (`tools.DuckDuckGo()`, `tools.Tavily(key)`).
- **Parity status:** MISSING.

### 4.6 MCP (Model Context Protocol) client — *Phase 9*
- **What it is.** Connect to external MCP servers and expose their tools to your agent.
- **How it works.** `mcp.py` + `_mcp.py` + `toolsets/fastmcp.py`: an MCP client (stdio or HTTP/SSE)
  lists a server's tools and surfaces them as a `Toolset`. `MCPServerTool` (4.4) is the provider-side
  variant.
- **Use cases.** Plug in the GitHub/filesystem MCP servers and the agent gains those tools with no
  bespoke integration.
- **Porting to Go.** An MCP client (Go MCP SDKs exist) adapting a server's tool list into the
  `Toolset` interface.
- **Parity status:** MISSING.

## Pillar 5 — Messages & multimodal content

### 5.1 The message / part protocol — *Phase 5*
- **What it is.** A fully typed conversation: requests, responses, and the heterogeneous parts
  inside.
- **How it works.** `messages.py`: a `ModelMessage` is a `ModelRequest` (`SystemPromptPart`,
  `UserPromptPart`, `ToolReturnPart`, `RetryPromptPart`) or `ModelResponse` (`TextPart`,
  `ThinkingPart`, `ToolCallPart`, `FilePart`, built-in tool parts) — each a distinct typed struct.
- **Use cases.** Persist and replay a conversation; inspect which tool was called with which args;
  multi-turn chat with history passed back in.
- **Porting to Go.** A `Message` interface (`ModelRequest`/`ModelResponse`) + a `Part` interface with
  one struct per part type; type switches replace isinstance dispatch.
- **Parity status:** MISSING.

### 5.2 Multimodal content — *Phase 5*
- **What it is.** Pass images, audio, video, documents, and binary data to capable models.
- **How it works.** `messages.py`: `FileUrl` (abstract) + `ImageUrl`/`AudioUrl`/`VideoUrl`/
  `DocumentUrl`; `BinaryContent`/`BinaryImage`; media-type/format enums; `CachePoint` for prompt-
  cache boundaries; `UserContent` is the union a user message can hold.
- **Use cases.** "Describe this screenshot"; transcribe audio; answer about an uploaded PDF; vision
  OCR; prompt-cache a large static document to cut cost.
- **Porting to Go.** `FileURL` interface + URL structs + `BinaryContent{MediaType, Data}` + a
  `UserContent` union.
- **Parity status:** MISSING.

### 5.3 Message history serialization — *Phase 5*
- **What it is.** Serialize/deserialize whole conversations across requests and processes.
- **How it works.** `ModelMessagesTypeAdapter` gives validated JSON (de)serialization; runs accept a
  `message_history` argument.
- **Use cases.** Store chat in Postgres/Redis and rehydrate; export a transcript; audit what was sent.
- **Porting to Go.** `json.Marshal`/`Unmarshal` over `Message` types with per-part discriminators; a
  `MessageHistory` type passed into each run.
- **Parity status:** MISSING.

### 5.4 Thinking / reasoning parts — *Phase 5*
- **What it is.** First-class reasoning-model "thinking" output, separate from the answer.
- **How it works.** `_thinking_part.py` + `ThinkingPart` + `ThinkingPartDelta`: chain-of-thought is a
  distinct part type (streamed via its own delta — 6.2).
- **Use cases.** A collapsible "Thinking…" panel; log reasoning for debugging while hiding it; strip
  thinking before persisting.
- **Porting to Go.** A `ThinkingPart` struct + `ThinkingPartDelta` event + a profile flag.
- **Parity status:** MISSING.

## Pillar 6 — Streaming & run-level events

### 6.1 Run-level event stream — *Phase 7*
- **What it is.** Stream a whole run as typed events (text, tool-call, tool-result, final output) so
  the caller can react mid-run.
- **How it works.** Exports: `AgentStreamEvent`, `PartStartEvent`, `PartDeltaEvent`, `PartEndEvent`,
  `FinalResultEvent`, `FunctionToolCallEvent`, `FunctionToolResultEvent`, `HandleResponseEvent`,
  `ModelResponseStreamEvent`, `AgentRunResultEvent`. `_parts_manager.py` reassembles deltas into
  parts; `run_stream`/`result.py` expose it.
- **Use cases.** The chat "typing" effect; a live "agent is doing X" log; rendering a structured
  result as it fills in (pedantigo's partial-JSON validation plugs in here); early cancellation.
- **Porting to Go.** A `ResponseStream` yielding an `Event` sum type over a channel/`iter.Seq`; a
  `PartsManager` accumulating deltas.
- **Parity status:** PARTIAL — partial-JSON validation exists; the semantic event stream MISSING.

### 6.2 Deltas: text, tool-call, thinking — *Phase 7*
- **What it is.** Typed increments of in-progress content, each tagged by role.
- **How it works.** `TextPartDelta` (next prose chunk), `ToolCallPartDelta` (next chunk of a pending
  tool call's name/args JSON), `ThinkingPartDelta` (next reasoning chunk); `ModelResponsePartDelta`
  is the union; the parts manager stitches them into parts.
- **Use cases.** *Text:* stream words to a UI or TTS. *Tool-call:* show "🔍 calling search_flights…"
  the instant the name arrives, and run a guardrail on the streaming args to **abort a dangerous
  call before it fires**. *Thinking:* live reasoning panel.
- **Porting to Go.** `TextDelta`/`ToolCallDelta`/`ThinkingDelta` structs; accumulate tool-call arg
  deltas, then validate the completed JSON with pedantigo.
- **Parity status:** MISSING.

### 6.3 UI streaming adapters (AG-UI, Vercel AI SDK) — *Phase 7*
- **What it is.** Adapters translating the run event stream into frontend wire protocols.
- **How it works.** `ag_ui.py` + `ui/ag_ui` implement **AG-UI**; `ui/vercel_ai` implements the
  **Vercel AI SDK** protocol; `ui/_event_stream.py` + `ui/_adapter.py` are the shared machinery.
- **Use cases.** Drop an agent behind a Vercel AI SDK `useChat()` React frontend with no glue;
  interoperate with any AG-UI UI.
- **Porting to Go.** HTTP handlers marshaling the Go event stream into AG-UI / Vercel AI SSE frames.
- **Parity status:** MISSING.

## Pillar 7 — Reliability, limits & operations

### 7.1 Usage accounting & cost — *Phase 8*
- **What it is.** Count tokens/requests per run and across runs; compute cost.
- **How it works.** `usage.py`: `RequestUsage` (one call), `RunUsage` (aggregated) on
  `AgentRunResult`; coupled with `genai-prices` for dollar cost.
- **Use cases.** Per-tenant billing; dashboards; "this run cost $0.03."
- **Porting to Go.** `Usage{InputTokens, OutputTokens, Requests}` accumulated through `RunContext`; a
  pricing table.
- **Parity status:** MISSING.

### 7.2 Usage limits — *Phase 8*
- **What it is.** Hard caps that abort a run before it overspends.
- **How it works.** `usage.UsageLimits` (max requests/tokens) checked each step; exceeding raises
  `UsageLimitExceeded`; also bounds runaway tool-call loops.
- **Use cases.** Prevent infinite tool loops; enforce a token budget; protect against injection-driven
  cost blowups.
- **Porting to Go.** A `UsageLimits` struct checked in the loop; return `UsageLimitExceeded`.
- **Parity status:** MISSING.

### 7.3 Error taxonomy — *Phase 8*
- **What it is.** A structured hierarchy of error types for distinct handling.
- **How it works.** `exceptions.py`: `ModelRetry`, `CallDeferred`, `ApprovalRequired`, `UserError`,
  `AgentRunError`, `UsageLimitExceeded`, `UnexpectedModelBehavior`, `ModelAPIError`, `ModelHTTPError`,
  `FallbackExceptionGroup`, `ToolRetryError`, `IncompleteToolCall`.
- **Use cases.** Retry on `ModelHTTPError` but not `UserError`; surface `UsageLimitExceeded` to
  billing; treat `UnexpectedModelBehavior` as a bug signal.
- **Porting to Go.** Error types/sentinels with `errors.Is`/`As`; some are control-flow signals
  (`ModelRetry`, `ApprovalRequired`), the rest failures.
- **Parity status:** MISSING.

### 7.4 Observability / instrumentation (OpenTelemetry) — *Phase 8*
- **What it is.** Emit OTel spans/events for runs, model calls, and tool calls.
- **How it works.** `_instrumentation.py` + `models/instrumented.py` + `_otel_messages.py` +
  `InstrumentationSettings`: an `InstrumentedModel` wrapper traces every request; integrates with
  Logfire and any OTel backend.
- **Use cases.** See a waterfall of a slow run; debug a loop; monitor latency/cost in production.
- **Porting to Go.** An OTel-instrumented `Model` decorator (`go.opentelemetry.io/otel`); an
  `InstrumentationSettings` struct.
- **Parity status:** MISSING.

### 7.5 Durable execution — *Phase 8*
- **What it is.** Run agents as **durable, resumable workflows** surviving crashes and long pauses.
- **How it works.** `durable_exec/` has `temporal`, `dbos`, `prefect` backends; each wraps the agent
  so each node/step is a durable activity with checkpointing, so a run resumes where it left off.
- **Use cases.** A multi-day approval workflow; a long research agent surviving a deploy; exactly-once
  tool execution via Temporal.
- **Porting to Go.** Integrate Temporal's Go SDK (first-class in Go); map run graph nodes (2.3) to
  Temporal activities — where the node abstraction pays off.
- **Parity status:** MISSING (Temporal is the natural Go fit).

## Pillar 8 — Ecosystem & integration

### 8.1 Direct / imperative API — *Phase 9*
- **What it is.** A single model request with only input/output schemas, no agent abstraction.
- **How it works.** `direct.py`: imperative requests with minimal abstraction — messages + output
  schema in, `ModelResponse` out, skipping the agent/graph loop.
- **Use cases.** A one-shot classification/extraction with no tools/retries/history; building custom
  orchestration on top.
- **Porting to Go.** A package-level `direct.Request(ctx, model, msgs, outSchema)`.
- **Parity status:** MISSING (thin layer over Pillar 1).

### 8.2 Agent-to-Agent (A2A) protocol — *Phase 9*
- **What it is.** Expose an agent over A2A so other agents call it as a service, and call other A2A
  agents.
- **How it works.** `_a2a.py` + FastA2A expose an agent as an A2A server endpoint.
- **Use cases.** A fleet of specialized agents delegating across process boundaries; publish your
  agent for third-party orchestration.
- **Porting to Go.** An HTTP server/client implementing the A2A spec, wrapping `Agent.Run`.
- **Parity status:** MISSING.

### 8.3 External framework integrations — *Phase 9*
- **What it is.** Bridge tools from other ecosystems.
- **How it works.** `ext/langchain.py` adapts LangChain tools; `ext/aci.py` integrates ACI.dev tools
  — each converts a foreign tool into a `Tool`/`Toolset`.
- **Use cases.** Reuse an existing LangChain tool library without rewriting it.
- **Porting to Go.** Adapters converting third-party Go tool definitions into the `Toolset` interface.
- **Parity status:** MISSING.

### 8.4 CLI (`clai`) — *Phase 9*
- **What it is.** A command-line interface to chat with / run an agent.
- **How it works.** `_cli.py` + `__main__.py` provide an interactive REPL over any configured model.
- **Use cases.** Quick manual testing; a scriptable `clai "summarize this"` in a pipeline.
- **Porting to Go.** A `cobra`/`urfave-cli` command wrapping `RunSync`/`RunStream` with a TTY REPL.
- **Parity status:** MISSING.

### 8.5 Test & function models — *Phase 2 (built early)*
- **What it is.** Fake models for deterministic, network-free, zero-cost unit testing.
- **How it works.** `models/test.py` (`TestModel`) auto-generates schema-valid responses;
  `models/function.py` (`FunctionModel`) runs a supplied function as the model, scripting exact
  responses/tool-calls.
- **Use cases.** Table-driven tests of tool dispatch and output validation in CI, in milliseconds,
  no API keys; assert "given this message, the agent calls `refund` with these args."
- **Porting to Go.** `TestModel` (schema-driven canned output) + `FunctionModel`
  (`func(msgs) ModelResponse`) implementing `Model` — built **early** (Phase 2) to test everything
  else.
- **Parity status:** MISSING (build first).

---

## Packaging notes

- Module path: `github.com/Pedantigo/pedantigo-ai` initially; a vanity path (`pedantigo.dev/ai`) may
  be adopted once to make future relocation invisible to consumers.
- Provider dependencies: start with all backends as subpackages (Go has no pip-style extras). If
  `go.sum` bloat from unused provider SDKs becomes a real complaint, split heavy providers into their
  own nested modules.
- Local multi-repo development against an unreleased core uses a `go.work` file (core + pedantigo-ai),
  removed before tagging a release.

## Appendix — verification provenance

- Pydantic AI surface enumerated from `pydantic-ai==1.25.0`: 114 `__all__` exports; `models/`
  (18 files), `providers/` (28), `toolsets/` (13 exports), `profiles/` (13), `common_tools/`,
  `durable_exec/{temporal,dbos,prefect}`, `ui/{ag_ui,vercel_ai}`, `ext/{langchain,aci}`;
  `builtin_tools.py` (7 tools), `output.py` (4 modes), `exceptions.py` (12 classes), `messages.py`
  (multimodal/part classes), `usage.py`, `settings.py`, `retries.py`, `direct.py`.
- Pedantigo baseline from the core [`pedantigo`](https://github.com/Pedantigo/pedantigo) library.

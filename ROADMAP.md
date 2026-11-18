# ROADMAP — pedantigo-ai

Planned work that is decided but not yet implemented. Pedantigo AI is greenfield, so this
roadmap is organized as **dependency-ordered phases**: each phase states its goal, the scope
it covers, the key types/interfaces it introduces, what it depends on, and the acceptance
criteria that mark it done — so any contributor can pick up a phase without re-deriving the
design.

The feature surface targets 100% parity with [Pydantic AI](https://github.com/pydantic/pydantic-ai)
(verified against `pydantic-ai==1.25.0`). Everything here reuses
[`pedantigo`](https://github.com/Pedantigo/pedantigo) for schema generation and validation and
adds only the LLM-facing runtime.

## Milestones

- **M1 — Tier 2 (typed extraction):** reached at the end of Phase 4. One prompt in, one
  validated struct out, with automatic retry. An `instructor-go` equivalent.
- **M2 — Tier 1 (full agent):** reached at the end of Phase 7. The model takes multiple
  actions (tools, streaming) to reach a validated result. A Pydantic AI equivalent.
- **M3 — Production-grade:** reached at the end of Phase 9. Usage limits, observability,
  durable execution, provider breadth, and ecosystem integrations.

## Guiding constraints

- **Core stays untouched.** Pedantigo AI is a separate module; it depends on a *released*
  `github.com/Pedantigo/pedantigo/v2` and never forces its dependencies onto core consumers.
- **Reuse, don't reinvent.** Tool schemas, output schemas, and streaming-result validation
  go through pedantigo's `validator` package, not a second schema/validation mechanism.
- **Test without the network.** `TestModel`/`FunctionModel` (Phase 2) land early so every
  later phase is table-driven and offline.

---

## Phase 0 — Scaffolding & module layout

- **Goal:** a buildable, lintable, CI-green empty module.
- **Scope:** `go.mod` (`module github.com/Pedantigo/pedantigo-ai`, Go 1.25+, `require`
  pedantigo core); package layout (`model/`, `provider/`, `agent/`, `output/`, `tool/`,
  `toolset/`, `message/`, `stream/`, `usage/`, `otel/`, `direct/`, `cli/`); `Makefile`,
  `.golangci.yml`, pre-commit, CI + coverage workflows mirroring core pedantigo; `LICENSE` (MIT).
- **Dependencies:** none.
- **Acceptance:** `make lint test` passes on an empty skeleton; CI and coverage badges green.

## Phase 1 — Model connectivity (minimum viable)

- **Goal:** the framework can send a request to one real provider and get a response back.
- **Scope:** the `Model` interface; one backend (OpenAI wire format) + one provider
  (OpenRouter config: `base_url=https://openrouter.ai/api/v1`); `Settings`
  (temperature, max_tokens, top_p, timeout, stop); HTTP-level retry transport (backoff,
  respect `Retry-After`) kept strictly separate from the `ModelRetry` loop (Phase 4).
- **Key types:**
  ```go
  type Model interface {
      Request(ctx context.Context, msgs []Message, s Settings) (ModelResponse, error)
      RequestStream(ctx context.Context, msgs []Message, s Settings) (ResponseStream, error)
      Name() string
      Profile() ModelProfile
  }
  ```
- **Dependencies:** Phase 0; a Go OpenAI client for transport.
- **Acceptance:** an integration test (gated behind an API key) completes one chat request;
  unit tests cover settings merging and retry/backoff.

## Phase 2 — Test & function models

- **Goal:** everything after this is testable with zero network and zero API keys.
- **Scope:** `TestModel` (auto-generates schema-valid canned output) and `FunctionModel`
  (`func([]Message) ModelResponse`, scriptable responses/tool-calls), both implementing `Model`.
- **Dependencies:** Phase 1 (the `Model` interface and message types).
- **Acceptance:** an `Agent` (stubbed) run driven entirely by `FunctionModel` asserts exact
  model inputs and outputs in a table-driven test.

## Phase 3 — The Agent & run loop

- **Goal:** a reusable `Agent` that executes a run against a `Model`.
- **Scope:** `Agent[Deps, Out]`; `RunSync`/`Run`/`RunStream` entry points; the run loop
  (user prompt → model request → (tool calls) → end) with an `EndStrategy`; `RunContext[Deps]`
  dependency injection; static + dynamic system prompts/instructions (instructions not
  persisted into history).
- **Key types:**
  ```go
  type Agent[Deps any, Out any] struct { /* model, prompts, tools, settings, outputSpec */ }
  func New[Deps, Out any](m Model, opts ...Option[Deps, Out]) *Agent[Deps, Out]
  type RunContext[Deps any] struct { Deps Deps; Usage *Usage; Retry int; Messages []Message }
  ```
- **Dependencies:** Phases 1–2.
- **Acceptance:** `agent.RunSync` returns a result from a `FunctionModel`; deps and dynamic
  system prompts are observable inside a tool/prompt callback; table-driven tests for the loop.

## Phase 4 — Structured output + validation + `ModelRetry`  → **M1 (Tier 2)**

- **Goal:** typed, validated output with automatic self-correction.
- **Scope:** the four output modes (`ToolOutput`, `NativeOutput`, `PromptedOutput`,
  `TextOutput`); schema injection via pedantigo's `SchemaJSONLLM`; unmarshal + **validate with
  pedantigo**; output-validator hooks; the `ModelRetry` loop (on failure, append a retry
  prompt describing the error and re-request up to a budget); per-profile JSON-schema
  transformers (inline `$defs`, strict-mode tweaks).
- **Key types:**
  ```go
  type ModelRetry struct{ Message string } // sentinel; the loop re-prompts on it
  type OutputValidator[Out any] func(RunContext[any], Out) error
  ```
- **Dependencies:** Phase 3; pedantigo `validator` (schema + validation).
- **Acceptance:** given a `FunctionModel` that returns invalid-then-valid output, the agent
  retries and returns a validated struct; each output mode covered; retry budget enforced.
- **Milestone:** Pedantigo AI is now a usable `instructor-go` equivalent.

## Phase 5 — Message & multimodal protocol

- **Goal:** a fully typed conversation substrate with history.
- **Scope:** `Message` (`ModelRequest`/`ModelResponse`) and `Part` types (`SystemPromptPart`,
  `UserPromptPart`, `TextPart`, `ThinkingPart`, `ToolCallPart`, `ToolReturnPart`,
  `RetryPromptPart`); multimodal content (`ImageURL`, `AudioURL`, `VideoURL`, `DocumentURL`,
  `BinaryContent`); JSON (de)serialization with per-part discriminators; `message_history`
  passed into and returned from runs.
- **Dependencies:** Phase 3 (used by the loop) — may land incrementally alongside Phases 3–4.
- **Acceptance:** a multi-turn conversation round-trips through JSON and resumes correctly; a
  vision request carrying an `ImageURL` serializes to the provider format.

## Phase 6 — Tools (function calling + execution)

- **Goal:** the model can call registered Go functions and the framework executes them.
- **Scope:** `Tool`/`ToolDefinition` + registration (reflect params → schema via pedantigo);
  the execute-and-feed-back loop inside the run graph; toolset combinators (`Combined`,
  `Filtered`, `Prefixed`, `Renamed`, `Prepared`, `Wrapper`); human-in-the-loop approval
  (`ApprovalRequired`/`CallDeferred`, deferred results, resume API).
- **Key types:**
  ```go
  func (a *Agent[Deps, Out]) Tool(name, desc string, fn any) // fn reflected for its schema
  type Toolset interface { Tools(RunContext[any]) []ToolDefinition }
  ```
- **Dependencies:** Phases 3–5.
- **Acceptance:** an agent calls a tool, receives its result, and continues to a final output
  (driven by `FunctionModel`); a gated tool pauses the run and resumes on approval.

## Phase 7 — Streaming & events  → **M2 (Tier 1)**

- **Goal:** stream a whole run as typed events.
- **Scope:** the run-level event stream (`PartStart`/`PartDelta`/`PartEnd`, `FinalResult`,
  `ToolCall`/`ToolResult` events); text / tool-call / thinking deltas; a parts manager that
  reassembles deltas into parts; validating the forming output struct via pedantigo's
  streaming partial-JSON validation; UI adapters (AG-UI, Vercel AI SDK) as HTTP handlers.
- **Dependencies:** Phases 3–6.
- **Acceptance:** `RunStream` yields ordered text and tool-call deltas; the output struct is
  progressively validated; an AG-UI/Vercel SSE frame stream is emitted.
- **Milestone:** Pedantigo AI is now a functional Pydantic AI equivalent.

## Phase 8 — Reliability & operations

- **Goal:** production safety and visibility.
- **Scope:** usage accounting (`RequestUsage`/`RunUsage`) + cost; `UsageLimits`
  (max requests/tokens) that abort runaway loops; the error taxonomy (`UserError`,
  `AgentRunError`, `ModelAPIError`, `ModelHTTPError`, `UnexpectedModelBehavior`,
  `UsageLimitExceeded`, `FallbackExceptionGroup`, `IncompleteToolCall`) with
  `errors.Is`/`As` support; model fallback; OpenTelemetry instrumentation
  (an `InstrumentedModel` wrapper); durable execution via Temporal (Go-native) mapping run
  nodes to activities.
- **Dependencies:** Phases 3–7.
- **Acceptance:** a token cap aborts with `UsageLimitExceeded`; a primary-model failure falls
  back to a secondary; OTel spans appear for run/model/tool; a Temporal-backed run resumes
  after a simulated crash.

## Phase 9 — Breadth & ecosystem

- **Goal:** feature-complete parity.
- **Scope:** remaining model backends and provider configs (Anthropic, Gemini/Google, Groq,
  Mistral, Cohere, Bedrock, HuggingFace, Ollama, LiteLLM, Together, Fireworks, DeepSeek,
  Grok, Azure, Vertex, …) + per-model profiles; built-in provider-executed tools (web search,
  web fetch, code execution, image generation, memory); common client-side tools (DuckDuckGo,
  Tavily); MCP client (expose MCP servers as toolsets); the direct/imperative API;
  agent-to-agent (A2A) server/client; external integrations (LangChain-style tool adapters);
  the `clai`-style CLI.
- **Dependencies:** Phases 1–8 (each backend implements the stable `Model` interface).
- **Acceptance:** each new backend passes a shared provider conformance test suite; MCP tools
  appear as a toolset; the CLI runs an agent from the terminal.

---

## Packaging notes

- Module path: `github.com/Pedantigo/pedantigo-ai` initially; a vanity path
  (`pedantigo.dev/ai`) may be adopted once to make future relocation invisible to consumers.
- Provider dependencies: start with all backends as subpackages of this module (Go has no
  pip-style extras). If `go.sum` bloat from unused provider SDKs becomes a real complaint,
  split heavy providers into their own nested modules.
- Local multi-repo development against an unreleased core uses a `go.work` file (core +
  pedantigo-ai), removed before tagging a release.

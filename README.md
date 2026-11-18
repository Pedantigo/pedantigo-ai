# Pedantigo AI

![Status](https://img.shields.io/badge/status-design%20phase-orange)
![Go](https://img.shields.io/badge/go-1.25%2B-00ADD8)
![License](https://img.shields.io/badge/license-MIT-blue)

Type-safe LLM agent framework for Go, built on [Pedantigo](https://github.com/Pedantigo/pedantigo).

> **Status: design phase.** This repository currently holds the design and roadmap.
> The public API below is the *target* and is subject to change until the first tagged
> release. See [ROADMAP.md](ROADMAP.md) for the dependency-ordered build plan.

## What it is

Pedantigo is the Go equivalent of *Pydantic* — struct validation and JSON Schema
generation. **Pedantigo AI is the Go equivalent of *Pydantic AI*** — the agent framework
that sits on top of that foundation: it calls models, holds a typed conversation, runs
tools, and coerces model output into validated Go structs.

Where `pedantigo` *validates data you already have*, `pedantigo-ai` *gets validated data
out of a model* and lets the model *take actions* to produce it.

## Relationship to Pedantigo

| Layer | Package | Role |
|-------|---------|------|
| Foundation | [`pedantigo`](https://github.com/Pedantigo/pedantigo) | Struct validation + JSON Schema generation (incl. `SchemaJSONLLM`) + partial-JSON streaming validation |
| Runtime | **`pedantigo-ai`** (this repo) | Model providers, the `Agent`, tools, typed output + retry, the message protocol, run-level streaming, usage limits, observability |

Pedantigo AI reuses pedantigo's schema generator and validator for the parts most
frameworks get wrong — tool schemas, output schemas, and validating a streaming result —
and adds the LLM-facing runtime around them.

## Installation (planned)

```bash
go get github.com/Pedantigo/pedantigo-ai
```

Requires Go 1.25+ (generics on `Agent[Deps, Out]`).

## Vision — Quick Example

> Planned API, subject to change.

```go
type Step struct {
    ID   int    `json:"id"   validate:"min=1"   jsonschema:"description=Execution order, starting at 1"`
    Task string `json:"task" validate:"required" jsonschema:"description=A single clear step"`
}
type Plan struct {
    Steps []Step `json:"steps" validate:"min=2,max=5"`
}

agent := pai.New[NoDeps, Plan](
    openai.Model("gpt-4o"),
    pai.WithInstructions("You are a task planner. Break the goal into 2-5 ordered steps."),
)

res, err := agent.RunSync(ctx, "Compare PostgreSQL and MongoDB for a transactional app.", NoDeps{})
// res.Output is a *validated* Plan; pedantigo validated it and, on failure,
// re-prompted the model with the error (ModelRetry) until it was valid.
for _, s := range res.Output.Steps {
    fmt.Println(s.ID, s.Task)
}
```

## Scope (planned feature pillars)

See [ROADMAP.md](ROADMAP.md) for the full breakdown and build order.

| Pillar | Covers | Status |
|--------|--------|--------|
| **1. Model connectivity** | `Model` interface, provider backends, settings, fallback, HTTP retries | planned |
| **2. Agent & run loop** | `Agent`, run/run_sync/run_stream, run graph, deps injection, system prompts | planned |
| **3. Structured output** | output modes, validation (reuses pedantigo), `ModelRetry` self-correction | planned |
| **4. Tools** | function tools + schema gen, toolsets, human-in-the-loop approval, MCP | planned |
| **5. Messages & multimodal** | typed message/part protocol, image/audio/doc content, history | planned |
| **6. Streaming & events** | run-level event stream, text/tool-call/thinking deltas, UI adapters | planned |
| **7. Reliability & ops** | usage/limits/cost, error taxonomy, OpenTelemetry, durable execution | planned |
| **8. Ecosystem** | direct API, agent-to-agent, framework integrations, CLI, test models | planned |

## Maturity tiers

The roadmap crosses two meaningful thresholds:

- **Tier 2 — typed extraction** (after Pillar 3): one prompt in, one validated struct out,
  with automatic retry. An `instructor-go` equivalent.
- **Tier 1 — full agent** (after Pillars 4 + 6): the model takes multiple actions (tools,
  handoffs, streaming) to reach a validated result. A true Pydantic AI equivalent.

## Documentation

Full documentation will live at **[pedantigo.dev](https://pedantigo.dev)** alongside the core
library docs.

## Credits

Built by the [Pedantigo](https://github.com/Pedantigo) project, alongside
[smritea.ai](https://smritea.ai) — the persistent AI-memory platform whose day-to-day needs
these open-source tools grew out of. Design inspired by
[Pydantic AI](https://github.com/pydantic/pydantic-ai).

## License

MIT

# Understanding TrueForge — Architecture Deep Dive & Build-Your-Own Blueprint

> A complete walkthrough of what this codebase is, how every layer fits together, why it is
> shaped the way it is, and how you would build something similar from scratch.
>
> Read top-to-bottom the first time. After that, use it as a map.

---

## Table of contents

1. [What TrueForge actually is](#1-what-trueforge-actually-is)
2. [The mental model in one page](#2-the-mental-model-in-one-page)
3. [Repository map](#3-repository-map)
4. [The five layers](#4-the-five-layers)
5. [Layer 1 — The harness core (`trueforge-core/src/core`)](#5-layer-1--the-harness-core-trueforge-coresrccore)
6. [Layer 2 — Durability: sessions & turns (`trueforge-core/src/agent-session`)](#6-layer-2--durability-sessions--turns-trueforge-coresrcagent-session)
7. [Layer 3 — The server (`packages/trueforge`)](#7-layer-3--the-server-packagestrueforge)
8. [Layer 4 — The generated SDK (`packages/trueforge-sdk`)](#8-layer-4--the-generated-sdk-packagestrueforge-sdk)
9. [Layer 5 — The UI (`trueforge-ui` + `frontend`)](#9-layer-5--the-ui-trueforge-ui--frontend)
10. [The data model](#10-the-data-model)
11. [End-to-end request walkthrough](#11-end-to-end-request-walkthrough)
12. [Context engineering — the differentiating features](#12-context-engineering--the-differentiating-features)
13. [Sandbox architecture](#13-sandbox-architecture)
14. [Deployment topologies](#14-deployment-topologies)
15. [Engineering conventions and why they exist](#15-engineering-conventions-and-why-they-exist)
16. [Build your own: a staged blueprint](#16-build-your-own-a-staged-blueprint)
17. [Design decisions worth stealing (and ones to skip)](#17-design-decisions-worth-stealing-and-ones-to-skip)
18. [Glossary](#18-glossary)

---

## 1. What TrueForge actually is

TrueForge is an **agent harness**: the runtime layer that sits between an LLM and the outside
world and turns "a model that emits text" into "an agent that gets work done."

An LLM by itself can only produce tokens. To make it useful you need a program around it that:

- Feeds it a system prompt, conversation history, and a list of tools it may call
- Parses tool calls out of its response, executes them, and feeds results back
- Loops until the model stops asking for tools
- Streams every intermediate step to a user interface
- Persists everything so a browser refresh, a server restart, or a crash doesn't lose the run
- Stops for human approval before dangerous actions
- Keeps the context window from overflowing on long tasks
- Provides an isolated place to run code without handing the model your secrets

That program is the harness. TrueForge implements it and exposes it three ways:

| Surface | Package | Who uses it |
| --- | --- | --- |
| Chat UI | `packages/frontend` (bundled into the server) | End users in a browser |
| HTTP API + SSE | `packages/trueforge` | Anything over the network |
| TypeScript SDK | `packages/trueforge-sdk` (generated) | Your backend code |
| Embeddable React UI | `packages/trueforge-ui` | Your own React app |
| Library | `packages/trueforge-core` | Embedding the loop in your own process |

**The single most important architectural decision:** most agent frameworks run the agent
*inside* a sandbox VM. TrueForge keeps the agent loop **on the server** and treats the sandbox
as *just another tool*, provisioned lazily only when the agent actually needs to execute code.
Consequences of that choice ripple through the whole design:

- API keys, OAuth tokens and MCP credentials never enter the sandbox
- One server process runs many concurrent agents (no VM per conversation)
- Turns that don't need code execution cost nothing extra and start instantly
- The sandbox needs a *bridge* back to the harness so sandboxed code can still call MCP tools
  (this is "Code Mode", see §12)

---

## 2. The mental model in one page

```
                     ┌───────────────────────────────────────────────┐
   Browser / your    │                TrueForge server               │
   code / SDK        │  (Hono HTTP + SSE, one process, many agents)  │
        │            │                                               │
        │  POST /sessions/{id}/turns  (stream: true)                 │
        └───────────>│                                               │
                     │   SessionHandle.createTurn()                  │
                     │            │                                  │
                     │            ▼                                  │
                     │   TurnResourceResolver  ── resolves ──┐       │
                     │   (spec → live objects)               │       │
                     │            │                          ▼       │
                     │            ▼                    model client  │
                     │   AgentThreadOrchestrator      MCP toolsets   │
                     │     ├── AgentThread (root)     sandbox handle │
                     │     ├── AgentThread (sub #1)   skills         │
                     │     └── AgentThread (sub #2)                  │
                     │            │                                  │
                     │            │ yields events                    │
                     │            ▼                                  │
                     │   TurnHandle.stream()  ── persist ──> DB      │
                     │            │           (persist BEFORE yield) │
                     │            ▼                                  │
                     │   EventSubscription (in-mem | Redis)          │
                     │            │                                  │
        <────────────│────────────┘  SSE, resumable by sequence #    │
                     └───────────────────────────────────────────────┘
                            │              │              │
                            ▼              ▼              ▼
                      SQLite/Postgres   MCP servers    Sandbox (Daytona/local)
```

Five nouns you must internalize:

| Noun | Meaning | Lives in |
| --- | --- | --- |
| **AgentSpec** | The *static configuration* of an agent: model, instructions, MCP servers, skills, limits, feature flags. Pure JSON, Zod-validated. | `agent-session/schemas/agentSpec.ts` |
| **Session** | A conversation. Bound to an agent (by reference to a saved agent, or with an inline spec). Owns a title, metadata, and an ordered list of turns. | `agent-session/models/SessionRecord.ts` |
| **Turn** | One user request and everything the agent did in response. The unit of execution, cancellation, streaming, and billing. | `agent-session/models/TurnRecord.ts` |
| **Thread** | One agent's message history inside a turn. The root agent has one thread; each subagent gets its own. Threads persist across turns. | `core/runtime/AgentThread.ts` |
| **Event** | Everything the agent does, as an append-only log: `model.message`, `tool.response`, `thread.done`, `turn.done`, … | `core/events/schema.ts` |

The relationship: **Session → many Turns → each Turn spans many Threads → each Thread emits Events.**

---

## 3. Repository map

pnpm workspace, Node ≥ 22.14, TypeScript throughout.

```
trueforge/
├── packages/
│   ├── trueforge-core/     ← the harness library (agent loop + session persistence)
│   ├── trueforge/          ← the server (HTTP API, DB, auth, catalogs, CLI)
│   ├── trueforge-sdk/      ← GENERATED TypeScript API client (Fern) — never edit
│   ├── trueforge-ui/       ← embeddable React chat UI SDK
│   └── frontend/           ← the bundled chat app (private, ships inside the server)
├── docs/                   ← Mintlify docs site + generated openapi.json
├── charts/                 ← Helm chart
├── benchmark/              ← cost/accuracy comparisons vs other harnesses
├── scripts/                ← build/codegen/clean helpers
├── tests/                  ← cross-repo shell tests
├── AGENTS.md               ← the repo's coding law (CLAUDE.md just @-includes it)
├── docker-compose.yml      ← smoke-test stack (Postgres + Redis + server)
├── docker-compose.dev.yml  ← dev infra only (Postgres + Redis)
└── package.json            ← every workspace task is a script here
```

### Dependency direction (strictly one-way)

```
frontend ──> trueforge-ui ──> trueforge-sdk ──┐
                                              ├──> (HTTP) ──> trueforge (server) ──> trueforge-core
                                              ┘
```

`trueforge-core` depends on nothing in the repo. The server depends on core. The UI depends on
the generated SDK, never on the server or core. This is what makes each package independently
publishable.

### The `trueforge-dev` export condition (a neat trick)

Every published package's `exports` map has a `trueforge-dev` branch pointing at `src/` instead
of `dist/`:

```jsonc
".": {
  "trueforge-dev": { "types": "./src/index.ts", "default": "./src/index.ts" },
  "types": "./dist/index.d.ts",
  "require": "./dist/index.js",
  "default": "./dist/index.mjs"
}
```

Dev scripts run with `NODE_OPTIONS=--conditions=trueforge-dev`, so the server resolves
`@truefoundry/trueforge-core` straight from TypeScript source — **no build step needed between
editing core and restarting the server.** Consumers outside the repo never see that condition
and get `dist/`. Steal this pattern; it removes the single most annoying part of monorepo dev.

---

## 4. The five layers

| # | Layer | Package/dir | Responsibility | Knows about |
| --- | --- | --- | --- | --- |
| 1 | **Harness core** | `trueforge-core/src/core` | The agent loop. Pure, in-memory, no DB, no HTTP. | LLMs, MCP, sandbox |
| 2 | **Session/turn durability** | `trueforge-core/src/agent-session` | Persist-and-replay around the loop. Storage-agnostic. | Layer 1 + a store interface |
| 3 | **Server** | `packages/trueforge` | HTTP, SSE, auth, DB, catalogs, config, scheduling, multi-replica | Layers 1–2 |
| 4 | **SDK** | `packages/trueforge-sdk` | Typed client generated from OpenAPI | The wire only |
| 5 | **UI** | `trueforge-ui` + `frontend` | React chat surface | Layer 4 only |

The discipline that makes this work: **each layer is usable standalone.** You can `npm i
@truefoundry/trueforge-core` and run the agent loop in your own process with an in-memory store
and no HTTP server at all.

---

## 5. Layer 1 — The harness core (`trueforge-core/src/core`)

```
core/
├── runtime/
│   ├── AgentThread.ts             ← THE agent loop (1442 lines; the heart)
│   ├── AgentThread.types.ts       ← event/state type unions
│   ├── AgentThreadOrchestrator.ts ← runs root + subagent threads concurrently
│   ├── AgentDefinition.ts         ← resolved runtime config (vs AgentSpec = wire config)
│   ├── DeferredTool.ts            ← lazy tool discovery meta-tools
│   ├── OpenToolCallCloser.ts      ← repairs dangling tool calls after a crash
│   ├── UserInputMessage.ts        ← normalizes user input (text/files)
│   ├── contextUsage.ts / contextUtils.ts / metrics.ts
├── capabilities/
│   ├── AgentCapability.ts         ← the plugin interface
│   ├── AgentContextProcessor.ts   ← the 4 hook points
│   ├── ToolResponseProcessor.ts
│   └── builtins/                  ← AskUserQuestion, ContextCompaction, CurrentDateTime,
│                                     DynamicSubAgents, LargeToolResponse, OpenUI
├── llm/
│   ├── ILLM.ts                    ← the only model contract (OpenAI chat shape, model removed)
│   ├── VercelAILLM.ts             ← adapter over Vercel AI SDK → many providers
│   └── LLMTypes.ts / usage.ts / toOpenAIChatMessage.ts / responseFormat.ts
├── mcp/
│   ├── IMCPServer.ts              ← IToolSet / ToolSource contracts
│   ├── RemoteMCP.ts               ← real MCP over HTTP (+ OAuth)
│   ├── LocalToolMCP.ts            ← in-process tools that look like an MCP server
│   ├── ClientSideTool.ts          ← tools executed by the *client*, not the server
│   ├── ToolSet.ts / ToolSelectorPolicy.ts / toolSelectors.ts  ← enable/disable/approval policy
│   └── executeToolCalls.ts / convertMCPServers.ts
├── sandbox/
│   ├── Sandbox.ts                 ← the sandbox-as-a-tool implementation
│   ├── provider/                  ← Daytona, TrueFoundry, Provider interface
│   ├── codeMode/                  ← NATS/UDS bridge letting sandbox code call MCP tools
│   ├── skills/                    ← git-backed SKILL.md mounting
│   └── scripts/                   ← Python injected into the sandbox (mcp_client.py, git_downloader.py)
├── events/schema.ts               ← the wire event union (Zod)
├── tracing/                       ← OpenTelemetry spans (Noop by default)
└── InstructionBuilder.ts          ← composes the system prompt from tagged sections
```

### 5.1 AgentThread — the loop, as an explicit state machine

This is the most important file in the repo. Rather than an ad-hoc `while (true)`, the loop is a
**three-state machine** whose current state is *derived from the message history*, not stored:

```ts
type AgentThreadState = 'llm-call-required' | 'tool-response-required' | 'user-input-required';

const VALID_TRANSITIONS = {
  'llm-call-required':      ['tool-response-required', 'user-input-required'],
  'tool-response-required': ['llm-call-required'],
  'user-input-required':    ['tool-response-required', 'llm-call-required'],
};
```

`deriveAgentThreadState(context)` walks the context tail-first:

- No open tool calls (every `tool_call` id has a matching `tool` message) → **llm-call-required**
- Open calls exist, and one needs approval or is client-side → **user-input-required**
- Otherwise → **tool-response-required**

**Why derive instead of store?** Because state is then a pure function of the persisted context.
A server can crash mid-turn, a different replica can pick the session up, and the correct next
step is recomputed from the database with zero extra bookkeeping. This is the single cleverest
idea in the codebase and the one most worth copying.

The loop itself (`execute()`, ~line 1302):

```ts
for (;;) {
  drain pending sandbox.created events
  switch (deriveState()) {
    case 'llm-call-required':
      if (signal.aborted) return;
      if (iterations >= iterationLimit) { yield errorEvent(...); return; }
      iterations++;
      outcome = yield* stepLLMCall(tools, toolMapping, signal);
      break;
    case 'tool-response-required':
      if (signal.aborted) return;
      outcome = yield* stepToolResponse(toolMapping);
      break;
    case 'user-input-required':
      // deliberately NOT cancellable — the required-action event must be emitted
      outcome = yield* stepUserInputRequired(currentModelMessageEventId);
      break;
  }
  if (outcome === 'exit') return;
}
```

Notice:

- **The whole method is an async generator.** It `yield`s events; it never writes to a database
  and never touches HTTP. The consumer decides what persistence and transport mean.
- **Errors never throw out of `execute()`.** The top-level `try/catch` converts any failure into
  a yielded error event, so a provider hiccup degrades into a visible "Agent Steps" error instead
  of a 500.
- **Cancellation checks sit only before side-effecting steps** (LLM call, tool execution), never
  before `user-input-required` — otherwise an approval prompt could be swallowed after the
  assistant message was already committed.

`stepLLMCall` does: run pre-LLM context processors → build the request → open the streaming
call → emit one `model.message` "opening" event → forward every delta as `model.message.delta`
→ assemble the final enriched assistant message → append to context → decide whether the thread
is done (no tool calls) or should continue.

`stepToolResponse` executes the open tool calls (in parallel where allowed), routes each result
through `ToolResponseProcessor`s, and appends `tool` messages.

### 5.2 The durability contract (from `core/runtime/AGENTS.md`)

```ts
yield event;      // consumer persists; may throw
mutateMemory();   // runs only after persistence succeeds
```

Every state mutation happens **after** the event describing it has been successfully yielded (and
therefore persisted by the consumer). If persistence throws, memory is never mutated and the
in-memory objects are discarded wholesale. There is no half-committed state, ever. This is why
`AgentThread` is a generator and not a callback-driven class.

### 5.3 Capabilities — the plugin system

Rather than a giant switch of feature flags, every optional behavior is an `AgentCapability`:

```ts
export interface AgentCapability {
  readonly systemToolSets?:            readonly IToolSet[];
  readonly preSendProcessors?:         readonly PreSendContextProcessor[];
  readonly preLLMProcessors?:          readonly PreLLMAgentContextProcessor[];
  readonly preLLMEphemeralProcessors?: readonly PreLLMEphemeralAgentContextProcessor[];
  readonly postToolCallProcessors?:    readonly PostToolCallAgentContextProcessor[];
  readonly toolResponseProcessors?:    readonly ToolResponseProcessor[];
  readonly instructionBuilders?:       readonly ((b: InstructionBuilder) => void)[];
  readonly state?: { key: string; load(state: JsonValue): void };
}
```

Six extension points, each firing at a precise moment in the loop:

| Hook | Fires | Used by |
| --- | --- | --- |
| `preSend` | before a user message enters the thread | OpenToolCallCloser-style repairs |
| `preLLM` | before each model call; **may rewrite context durably** | ContextCompaction |
| `preLLMEphemeral` | before each model call; **not persisted** | cache-control markers, transient hints |
| `postToolCall` | after tool execution | bookkeeping |
| `toolResponse` | on each raw tool result, before it enters context | LargeToolResponse offloading |
| `instructionBuilders` | at system-prompt assembly | every capability that needs to teach the model |
| `systemToolSets` | tool registration | AskUserQuestion, DynamicSubAgents, OpenUI, sandbox |

Plus `state`: a namespaced cross-turn key/value store. Keys must be globally unique per thread
(`tfy.` is reserved for builtins); writes go through `CAPABILITY_STATE` events so they are
persisted with the same discipline as context. Unclaimed keys are dropped with a warning on
hydration.

**The six builtins:**

| Capability | What it does |
| --- | --- |
| `ContextCompaction` | When estimated input tokens exceed a threshold (default: 80% of the model's context length, or 50 000 if unknown), summarizes older history into a structured in-context summary and rewrites the context |
| `LargeToolResponse` | When a tool result exceeds ~6 000 tokens (or the batch exceeds ~10 000), writes the full payload to a sandbox file and replaces it in context with a ~100-char preview plus the file path |
| `DynamicSubAgents` | Exposes `create_sub_agent`, letting the model spawn child threads with fresh contexts |
| `AskUserQuestion` | A *client-side* tool: the harness pauses, the UI renders a question with up to 5 options plus free text, and the answer comes back as a tool response |
| `OpenUI` | Teaches the model a component DSL so it can stream charts/tables/cards into chat ("Generative UI") |
| `CurrentDateTime` | A trivial `get_current_datetime` tool (models have no clock) |

Capabilities are assembled from the AgentSpec by `agent-session/builtinsFromSpec.ts`, so the wire
config (`config.context_management.compaction.enabled`, etc.) maps mechanically onto plugins.

### 5.4 Tools — everything is an MCP server

The unifying abstraction is `IToolSet`:

```ts
interface IToolSet {
  readonly name: string;
  readonly id: string;
  readonly preload: boolean;             // true = all schemas eagerly in context
  readonly hasPreloadedTools: boolean;
  listTools(): Promise<ListToolsResponse>;
  callTool(params, approvalDecision?): Promise<CallToolResponse>;
  toolCallInfo(params, resolveUnderlyingTool?): Promise<InternalToolCallInfo>;
  getAllowedToolNamesForSandbox?(): string[] | undefined;
}
```

Four implementations, and the loop can't tell them apart:

1. **`RemoteMCP`** — a real MCP server over HTTP, with header auth or full OAuth (including
   Dynamic Client Registration).
2. **`LocalToolMCP`** — in-process TypeScript functions dressed as an MCP server
   (`defineTool({ name, description, schema, handler })`). All builtins use this.
3. **`ClientSideTool`** — the *client* executes it. `callTool` returns
   `{ clientSideToolRequired: {...} }`, the loop transitions to `user-input-required`, the UI
   renders something, and the answer arrives as a `user.tool_response` message. This is how
   `ask_user_question` and interactive Generative UI work.
4. **`Sandbox`** — exposes `exec` and file tools; provisions a VM on first call.

`callTool`'s return type is a discriminated union covering every outcome the loop must handle:

```ts
type CallToolResponse =
  | CallToolResolvedResponse      // { result, wasInitialized, sandboxCreated?, events? }
  | AuthRequiredResponse          // { authRequired: { servers } }  → pause, ask user to connect
  | CallToolCreateSubAgentResponse// { createSubAgent: AgentInfo }  → orchestrator spawns a thread
  | ApprovalRequiredResponse      // { approvalRequired: { tool_info } } → pause for a human
  | ClientSideToolRequiredResponse// { clientSideToolRequired: { tool_info } } → pause for the UI
```

Every pause in the agent's life is *a tool call that returned a non-result*. That's an unusually
tidy way to model human-in-the-loop.

**Tool selector policy** (`ToolSelectorPolicy.ts`) layers per-agent policy over any tool source
using tag selectors:

- `enable_tools` / `disable_tools` / `preload_tools`: `@all`, `@read-only`, or literal names
- `require_approval_for_tools`: `@all`, `@write`, `@destructive`, or literal names
  (default: `["@write", "@destructive"]`)

The tags come from MCP tool annotations, so "pause before anything destructive" works without
naming individual tools.

### 5.5 The model layer

```ts
export type LLMCreateParams = Omit<ChatCompletionCreateParams, 'model'>;

export interface ILLM {
  create(body: LLMCreateParamsStreaming): AsyncGenerator<ExtendedChatCompletionChunk, RawAssistantMessageWithUsage>;
  createNonStream(body: LLMCreateParams): Promise<RawAssistantMessageWithUsage>;
}
```

Two deliberate choices:

- **The OpenAI chat-completion shape is the internal lingua franca.** Every provider is adapted
  *into* it rather than the loop learning many shapes.
- **`model` is removed from the params.** Model identity belongs to the bound client instance,
  not to each call. This stops callers from inventing a model string just to satisfy a required
  field, and it means swapping models is swapping a client.

`VercelAILLM` implements `ILLM` over the Vercel AI SDK, giving OpenAI, Anthropic, Google Gemini,
Fireworks, Moonshot, Alibaba, and any OpenAI-compatible endpoint. It also normalizes reasoning
blocks, cache read/write token accounting, and cost.

### 5.6 The orchestrator

`AgentThreadOrchestrator` owns a `Map<threadId, AgentThread>` and runs the *leaf* threads
concurrently (up to `MAX_PARALLEL_SUB_AGENTS = 5`), merging their event generators with
`mergeAsyncGenerators`. When a thread finishes it produces a `send_to_parent` tool message that
the orchestrator delivers to the parent thread, resolving the parent's `create_sub_agent` call.
The root thread is the one with no `parent`.

Subagent isolation is the point: the child's tool calls, search results and reasoning never enter
the parent's context — only its final answer does.

---

## 6. Layer 2 — Durability: sessions & turns (`trueforge-core/src/agent-session`)

Layer 1 is pure and in-memory. Layer 2 wraps it with persistence, *without* choosing a database.

```
agent-session/
├── Sessions.ts                ← create / get / getOrCreateByExternalId
├── SessionHandle.ts           ← createTurn(): builds threads from the last snapshot
├── TurnHandle.ts              ← stream(): execute-once, persist-before-yield
├── ITurnResourceResolver.ts   ← spec → live objects (the host's wiring seam)
├── TurnResourceResolver.ts    ← batteries-included default implementation
├── builtinsFromSpec.ts        ← AgentSpec flags → AgentCapability instances
├── models/  SessionRecord.ts, TurnRecord.ts
├── schemas/ agentSpec.ts, session.ts, turn.ts, events.ts, subject.ts, pagination.ts
└── store/
    ├── ISessionStore.ts       ← THE persistence contract
    ├── InMemorySessionStore.ts
    └── PageToken.ts, SessionStoreErrors.ts, …
```

### 6.1 `ITurnResourceResolver` — the seam that makes core reusable

The AgentSpec says `model: { name: "openai/gpt-5.5" }` and `mcp_servers: [{ name: "linear" }]` —
**names only, no URLs, no keys.** Something has to turn those names into live objects. That
something is the resolver, and it is an interface the host implements:

```ts
interface ITurnResourceResolver<TTurnCustom> {
  readonly logger: Logger;
  createTracing(): AgentTracing;
  resolveAgentSpec(input: { agent_id }): Promise<AgentSpec>;
  resolveSandbox(input: { spec, existing?, previousTurn?, signal, tracing }): Promise<Sandbox | undefined>;
  resolveAgentDefinition(input: { spec, agent_info?, previousTurn?, signal, tracing }): Promise<ResolvedAgentDefinition>;
  close(): Promise<void>;   // idempotent, best-effort, called even on failed runs
}
```

This is the boundary between "the harness" and "your product." The shipped
`TurnResourceResolver` is a construct-and-go default that takes three callbacks (`llm(model)`,
`mcp(name)`, `agent(id)`) plus an optional sandbox factory. The server subclasses it to add
tenant-scoped stores, secret decryption, OAuth tokens and OpenTelemetry.

**Why credentials live here and nowhere else:** the wire spec is safe to store, log, copy between
environments and show in a UI, because it never contains a secret. Secrets are injected at turn
time by the resolver. Copy this separation.

### 6.2 `SessionHandle.createTurn()` — rebuild, then run

Each turn:

1. Resolve `previous_turn_id` (`'auto'` → last turn, `'none'` → fresh branch, or an explicit id —
   this is how you get conversation branching for free).
2. Load the previous turn's `TurnSnapshot`, which holds every thread's context, capability state,
   parent link, and completion marker.
3. Rebuild each `AgentThread` from its snapshot; register any brand-new threads.
4. Ask the resolver for the model client, tool sets, sandbox, capabilities.
5. Construct the `AgentThreadOrchestrator`.
6. `send()` the new user input, **draining the resulting append events into memory**.
7. Persist the whole thing in one `createTurn` store call — the turn row, new threads, all context
   appends, all capability states, and the session title (first write wins).
8. Return a `TurnHandle`.

Step 6's comment is worth quoting because it explains the atomicity argument:

> Draining without persisting is safe: `send()` mutates only in-memory threads, and if
> `createTurn` fails the whole in-memory state (orchestrator, threads, resolver) is discarded —
> so either this turn commits atomically with its appends, or nothing was persisted at all.

Only the last 20 ancestors are kept as a materialized chain (`MAX_TURN_ANCESTORS`), bounding the
work of walking history.

### 6.3 `TurnHandle.stream()` — execute-once, persist-before-yield

```ts
for await (const event of orchestrator.execute({ signal })) {
  if (isDelta(event)) { yield event; continue; }   // deltas pass through, never persisted
  await store.appendEvent(...);                    // persist first
  yield persistedEvent;                            // then hand to the transport
}
```

Rules encoded here:

- **`stream()` may be called once.** A second call throws. Reconnecting clients don't re-execute;
  they *subscribe* to the already-running stream (§7.4).
- **Deltas are ephemeral.** `model.message.delta` events carry no sequence number and are never
  written; the assembled `model.message` is the durable record. Replaying a turn from the DB
  yields a "folded" stream that matches what a live viewer saw, minus the typing animation.
- **Terminal state is always written**, including on abort. `CancellationReason` distinguishes
  `ClientCancelled`, `CancelledForNextTurn`, `ServerExecutionTimeout`, and `Abandoned`
  (server shutting down).
- **`resolver.close()` runs in `finally`, after the terminal write** — so a sandbox VM is released
  even when the run failed.

### 6.4 `ISessionStore` — one contract, three implementations

Postgres, SQLite, and in-memory all implement the same interface, and — per
`store/AGENTS.md` — shared behavior is verified by a single `storeContractSuite.ts` that each
backend merely binds a store to. Changing a method means changing the interface, all three
implementations, and the suite in the same commit.

Key operations: `createSession`, `getSession`, `getSessionByExternalId`, `updateSession`,
`deleteSession`, `listSessions` (with token pagination, metadata filters, agent filters, time
ranges), `createTurn`, `getTurn`, `listTurns`, `appendEvent`, `listEvents`, `freezeAndGetTurn`
(atomically cancel a running turn and write its `turn.done`).

Pagination is opaque **page tokens**, not offsets (`OffsetPageToken.ts`,
`SessionListPageToken.ts`, `SessionEventPageToken.ts`) — stable under concurrent inserts.

---

## 7. Layer 3 — The server (`packages/trueforge`)

Hono + `@hono/zod-openapi` + Kysely. ~86 environment variables, all funnelled through one
config module.

```
src/
├── main.ts             ← boot: config → migrate → wire stores → listen (627 lines)
├── app.ts              ← the Hono app: routers, OpenAPI doc, Swagger UI, error handler
├── cli.ts              ← npx entry (--help, --port), then loads main
├── config.ts           ← THE ONLY place process.env is read (836 lines)
├── controller.ts / controller-main.ts / controller/  ← periodic control loops (schedules)
├── routes/             ← route DEFINITIONS (createRoute: method, path, schemas)
├── apis/               ← route HANDLERS (business logic)
├── schemas/            ← Zod wire schemas (snake_case, OpenAPI-registered)
├── db/
│   ├── postgres/  + migrations/   ← Kysely, jsonb, GIN indexes
│   ├── sqlite/    + migrations/   ← better-sqlite3
│   └── *Store.ts                  ← per-resource store interfaces
├── auth/               ← OIDC, standalone, TrueFoundry authenticators + authorizer
├── catalog/            ← YAML preset loaders (models, MCP, skills, sandbox)
├── runtime/            ← activeTurns, event-subscription, peeringIds, redis, cron
├── sandbox/local/      ← a local (non-cloud) sandbox provider
├── mcp/auth/           ← MCP OAuth token store + DCR
├── truefoundry/        ← optional enterprise control-plane integration
└── frontend.ts         ← serves the built UI for non-API routes
```

### 7.1 routes/ vs apis/ — a split worth copying

`routes/*.ts` contains only `createRoute({ method, path, request, responses, tags })` — pure
OpenAPI metadata. `apis/*.ts` contains the handlers. Benefits:

- The OpenAPI document is generated from the same objects that dispatch requests, so docs can't
  drift from behavior.
- The SDK is generated from that document (Fern), so the client can't drift either.
- You can read the entire API surface by skimming ~15 small files.

The full surface, under `/api/v1`:

| Group | Paths |
| --- | --- |
| Sessions | `POST /sessions`, `GET/PATCH/DELETE /sessions/{id}`, `GET /sessions`, `POST /sessions/get-or-create-by-external-id`, `POST /sessions/{id}/cancel`, `GET /sessions/{id}/events` |
| Turns | `POST /sessions/{id}/turns` (the big one), `GET .../turns`, `GET .../turns/{turn_id}`, `GET .../turns/{turn_id}/events`, `GET .../turns/{turn_id}/subscribe`, `GET .../turns/{turn_id}/download-sandbox-file` |
| Agents | `GET/POST /agents`, `GET/PATCH/DELETE /agents/{id}`, `GET /agents/{id}/code-snippets` |
| Settings | `/settings/model-providers`, `/settings/mcp-servers` (+ `/{name}/tools`, `/{name}/authorize`), `/settings/skills`, `/settings/sandbox-providers` |
| Catalog | `/catalog/model-providers`, `/catalog/mcp-servers`, `/catalog/skills`, `/catalog/sandbox-providers` |
| Schedules | `/schedules`, `/schedules/{id}`, `/schedules/runs`, `/schedules/{id}/runs` |
| Auth | `/auth/login`, `/auth/callback`, `/auth/logout`, `/auth/me` |
| Metrics | `/metrics/charts`, `/metrics/charts-data`, `/metrics/meters` |
| Meta | `/api/v1/docs` (Swagger UI), `/api/v1/openapi.json`, `/healthz` |

### 7.2 Configuration — one door

`AGENTS.md` forbids `process.env` anywhere in server code; everything goes through `config.ts`,
which validates at *import* time so a bad env var fails the process before it can listen. `main.ts`
imports it inside a `try/catch` and exits 1 with a readable message.

The most consequential var is `STANDALONE`:

| | `STANDALONE=true` | `STANDALONE=false` |
| --- | --- | --- |
| Database | SQLite (`SQLITE_PATH`) | Postgres (`POSTGRES_*` / `DATABASE_URL`) |
| Redis | none | required (`REDIS_URL`) |
| Event streams | in-process | Redis-backed, resumable across replicas |
| Controller | in the server process | a separate single-replica process |
| Store modules | dynamically imported | dynamically imported |

Store modules are loaded with dynamic `import()` so only the active engine's driver is ever
pulled in — `better-sqlite3` isn't loaded in a Postgres deployment and vice versa.

### 7.3 Catalogs vs settings — presets vs configuration

Two distinct concepts people constantly conflate:

- **Catalog** (`packages/trueforge/catalog/*.yaml`, read-only, inlined at build time by
  `build:gen`): *presets you could configure* — "Anthropic exists, here are its models and their
  context lengths"; "Linear's MCP server is at this URL and uses DCR"; "here are 40 permissively
  licensed skills." **Never executed against.** Purely discovery data for the UI.
- **Settings** (DB tables, `PUT /api/v1/settings/*`): *what you actually configured*, including
  API keys.

The UI flow is: `GET /catalog/model-providers` → user picks one → UI copies the preset into a
`PUT /settings/model-providers` body → user adds their key. Catalogs are overridable via
`MODEL_CATALOG_PATH`, `MCP_CATALOG_PATH`, `SKILL_CATALOG_PATH`, `SANDBOX_CATALOG_PATH`.

### 7.4 Streaming, resumption and multi-replica peering

This is the most intricate part of the server, and it solves a real problem: *a browser refresh
in the middle of a 3-minute agent run must not lose or duplicate anything, even if the reconnect
lands on a different replica.*

Four cooperating pieces:

**1. `EventSubscriptionRegistry`** (`runtime/event-subscription/`) — hands out a per-turn
resumable stream, backed by Redis when configured and by an in-process store otherwise. Both
backends implement identical semantics:

```ts
interface EventSubscription<T> {
  put(event, { streamTTLSeconds }): Promise<number>;   // assigns a dense 1-indexed sequence
  assertSubscribable(): Promise<void>;                  // 412 if gone/expiring soon
  poll(afterSequenceNumber?, { signal }): AsyncGenerator<SequencedEvent<T>>;
}
```

Dense sequence numbers mean a client that saw event 42 reconnects with `?after=42` and gets 43
onward — no gaps, no duplicates. Streams carry a TTL (`TURN_STREAM_TTL_SECONDS`,
`TURN_STREAM_POST_COMPLETION_TTL_SECONDS`) so completed turns eventually evict.

**2. `ActiveTurnRegistry`** (`runtime/activeTurns.ts`) — an in-memory map of turns executing *in
this process*, keyed `sessionId:turnId`, each with an `AbortController`. `track()` registers the
run and returns a wrapping generator that deregisters on completion. On shutdown it aborts
everything with `Abandoned` and waits for the terminal writes.

**3. Executor peering** (`runtime/peeringIds.ts`) — every turn id is minted as
`${ulid}.${EXECUTOR_ID}`. The owning replica is therefore *derivable from the id itself*, with no
lookup table:

```ts
mintPeeredTurnId(executorId) => `01j8x…v.replica-7`
executorFromTurnId(turnId)   => 'replica-7'
```

**4. Request-reply over Redis** (`trueforge-core/src/request-reply/`) — a tiny RPC layer:
`RequestReplyRouter` (a dispatch table this replica serves), `RequestReplyExecutor` (subscribes
to its own channel), and `redisRequest()` (publish + await a reply key with a timeout). Used for
cross-replica cancellation: a `POST /sessions/{id}/cancel` that lands on replica A for a turn
owned by replica B decodes B from the turn id and forwards the cancel over Redis; B aborts its
local `AbortController`.

The write path is a **dual write**: `drainTurnEvents` writes each event to the DB (via
`TurnHandle.stream()`) *and* to the `EventSubscription`, then out to the SSE response. The
non-streaming variant (`startTurnInProcess`) waits for the first dual-written event before
returning, so an immediate `subscribe` can't 412 on a stream that doesn't exist yet.

`SERVER_EXECUTION_TIMEOUT_SECONDS` arms a wall-clock timer per turn; `GRACEFUL_TIMEOUT_SECONDS`
bounds shutdown.

### 7.5 Auth

Pluggable `Authenticator` + `Authorizer`:

- **`standaloneAuthenticator`** — no login; stamps a default local user. The default for
  `npx`-style local use, and the reason local mode must stay on localhost.
- **`oidcAuthenticator`** — real OIDC (`openid-client`) with HttpOnly cookies or
  `Authorization: Bearer`. Configurable claims for user reference, display name, and role;
  `OIDC_ADMIN_ROLE_VALUE` splits admin from user; `OIDC_ALLOWED_EMAILS` is an allowlist.
- **`trueFoundryAuthenticator`** — enterprise control-plane mode, where model providers, MCP
  servers, agents and sandboxes come from an external service per-request rather than the local DB.

The TrueFoundry mode is a nice demonstration of the store-interface design: `main.ts` swaps
`IModelProviderStore`, `IMcpServerStore`, `IAgentStore`, `ISandboxProviderStore` for
token-bound HTTP-backed implementations *per request*, and nothing else in the server changes.

### 7.6 The controller

A separate concern from serving requests: periodic **control loops** that must run in exactly one
process per database. In standalone mode that's the server; with `STANDALONE=false` it's a
dedicated single-replica process (`controller-main.ts`, `pnpm start:controller`).

```ts
interface ControlLoop {
  readonly name: string;
  readonly intervalMs: number;
  tick(signal: AbortSignal): Promise<void>;
}
```

Each loop has its own interval, its own re-entrancy guard, and its own error boundary — a slow or
failing loop can't stall another, and a throwing `tick` is logged and retried next interval. Loops
must be idempotent-ish: "a pass must leave nothing half-applied that a later pass cannot recover
from." The current loop is `scheduleDispatch` (cron-scheduled agent runs).

---

## 8. Layer 4 — The generated SDK (`packages/trueforge-sdk`)

**Never hand-edited.** Generated by Fern from the OpenAPI document that `app.ts` produces:

```
routes/*.ts (createRoute)  →  buildOpenApiDocument()  →  .github/fern/openapi/openapi.json
                                                      →  docs/openapi.json  (identical copies)
                                                      →  Fern  →  packages/trueforge-sdk/src
```

`pnpm sdk:generate` (Docker required) regenerates it; CI does it after merge, and fork PRs are
expected to change source only. Ships dual CJS/ESM with separate `.d.ts` per format.

The lesson: **generate the client, don't write it.** Three artifacts (docs, SDK, server) stay in
lockstep because two of them are derived.

---

## 9. Layer 5 — The UI (`trueforge-ui` + `frontend`)

### `packages/trueforge-ui` — the embeddable SDK

Built on `assistant-ui` (`@assistant-ui/react`) plus a TrueFoundry runtime fork.

```
src/
├── atoms/        ← primitives, agent-chat, agent-details, draft, schedules, adapters
├── containers/   ← SettingsBuilder, McpOauthContainer
├── layouts/  routing/  hooks/  icons/  theme/{presets}/  constants/  utils/
├── server/       ← ServerContext, TrueForgeServerConfig, port TYPES (re-exported only)
└── plugins/trueforge-agent-server-adapter/
    ├── client.ts             ← createTrueForgeClient({ baseUrl, fetch })
    ├── chatServer.ts         ← implements the AgentChatServer port
    ├── builderServer.ts      ← the agent-builder port
    ├── agentSessionsServer.ts, agentMetricsServer.ts, schedules/, catalogs/
    └── toUiTurnState.ts      ← maps harness events → UI state
```

The key idea is **ports and adapters**. `trueforge-ui` is written against abstract "server ports"
(`AgentChatServer`, `AgentBuilderServer`, catalog ports, session/turn DTOs, stream events) whose
canonical definitions live in `@truefoundry/assistant-ui-runtime`. `trueforge-ui` re-exports them
as pass-through aliases and **must not** define parallel copies (an explicit `AGENTS.md` rule).
The TrueForge server is then one *plugin* implementing those ports. You could point the same UI at
a different backend by writing another adapter.

Theming is token-based with presets, plus `SlotOverrides` for replacing whole components.

### `packages/frontend` — the bundled app

A thin Vite + React 19 + react-router app (~15 files) that composes the SDK:

```tsx
<ThemeProvider theme={appTheme}>
  <TrueForgeUI server={createTrueForgeClient({ baseUrl: API_BASE_URL, fetch: authAwareFetch })} … />
</ThemeProvider>
```

It owns only host concerns: auth probing (`/auth/me`), the welcome/get-started screens, brand
tokens, logout, and public-path handling. Everything else comes from the SDK. Monaco is bundled
for editing agent specs.

**How it's served**, which differs between dev and prod:

- **Dev**: two processes. Vite on `:3000` serves the UI with HMR and proxies `/api/*` to Hono on
  `:8790` (with SSE buffering explicitly disabled in the proxy config).
- **Prod / Docker**: one process. `frontend.ts` serves the built bundle from `dist/_frontend`
  (or `packages/frontend/dist`, or `FRONTEND_DIR`) for every non-API route; `/api/*` and
  `/healthz` are the API. One origin, no CORS.

---

## 10. The data model

### 10.1 AgentSpec — the central document

```jsonc
{
  "model": {
    "name": "openai/gpt-5.5",              // FQN: provider/model
    "params": { "temperature": 0.7, "max_tokens": 4096, "reasoning_effort": "medium" }
  },
  "instructions": "You are a support agent for …",
  "messages": [{ "type": "user.message", "content": "seed message injected every session" }],
  "mcp_servers": [{
    "name": "linear",                       // name of a CONFIGURED server; no url/keys here
    "enable_tools": ["@all"],
    "disable_tools": [],
    "preload_tools": [],
    "require_approval_for_tools": ["@write", "@destructive"],
    "preload": false                        // false = deferred discovery (the default)
  }],
  "skills": [{ "name": "pdf-report" }],     // name-only refs; mount details from the skill store
  "response_format": { /* optional structured output */ },
  "config": {
    "iteration_limit": 100,                 // 1–1024
    "sandbox": { "enabled": false, "file_downloads": true },
    "dynamic_sub_agents": { "enabled": true },
    "context_management": {
      "compaction": { "enabled": true, "trigger": { "type": "input_tokens", "value": 120000 } },
      "large_tool_response": { "enabled": true }
    },
    "generative_ui": { "enabled": true },
    "ask_user_questions": { "enabled": true }
  }
}
```

Three properties make this design work:

1. **No secrets.** Names only. Safe to log, diff, version, and display.
2. **Every field has a default** (`RuntimeConfigSchema.parse({})` materializes the whole tree), so
   the minimal valid spec is `{ "model": { "name": "openai/gpt-5.5" } }`.
3. **Zod is the single source of truth** — the TypeScript type is `z.infer<typeof AgentSpecSchema>`,
   and the OpenAPI schema comes from the same object via `.openapi('AgentSpec')`. The repo
   forbids hand-written interfaces mirroring a Zod schema.

A session binds to an agent as a discriminated union: `{ type: 'ref', agent_id }` (live lookup
each turn — edits to the saved agent affect running sessions) or `{ type: 'inline', spec }`
(frozen with the session).

### 10.2 Events

Harness events (`core/events/schema.ts`):

| Event | Meaning |
| --- | --- |
| `user.message` | user input entered the thread |
| `model.message` | an assistant message (the durable record) |
| `model.message.delta` | streaming fragment — **never persisted** |
| `tool.response` | a tool returned |
| `tool.approval_required` | paused: a human must approve |
| `tool.response_required` | paused: the client must execute a tool |
| `user.tool_approval` / `user.tool_response` | the human's/client's answer |
| `mcp.initialize` | an MCP server connected |
| `mcp.auth_required` | paused: an MCP server needs OAuth |
| `sandbox.created` | a VM was provisioned |
| `thread.created` / `thread.done` | subagent lifecycle |
| `agent.context.overwrite` | compaction rewrote history |

Session-level events (`agent-session/schemas/events.ts`): `turn.created`, `turn.done`.

Internal-only orchestration events (never on the wire) are namespaced `internal.*`:
`internal.agent.create_subagent`, `internal.agent.context.append`, `internal.agent.done`,
`internal.mcp.auth_required`, `internal.capability.state`.

### 10.3 Database tables

Identical logical schema in Postgres and SQLite (15 tables), with parallel migration directories:

| Table | Holds |
| --- | --- |
| `session` | conversation, agent binding, title, metadata, source, external_id, creator |
| `turn` | one request/response cycle: input, state, metrics, previous_turn_id, ancestors |
| `turn_thread` | per-thread checkpoint within a turn (parent, agent_info, completion) |
| `session_event` | the append-only persisted event log |
| `thread_context_log` | context messages appended per thread per turn |
| `thread_capability_state` | cross-turn capability KV |
| `model_provider`, `mcp_server`, `skill`, `sandbox_provider` | configured settings (with secrets) |
| `agent` | the saved-agent registry |
| `schedule`, `schedule_run` | cron-scheduled runs |
| `oauth_token`, `oauth_pending_authorization` | MCP OAuth (user-scoped) |

Naming rule from `AGENTS.md`: **every wire field and DB identifier is `snake_case`** — path params,
query params, JSON fields, table names, column names, and jsonb document keys.

---

## 11. End-to-end request walkthrough

*"User types 'summarize my open Linear issues' and hits enter."*

```
 1. Browser  POST /api/v1/sessions/{sid}/turns   { input: [{type:'user.message', content:'…'}], stream: true }
 2. Hono     auth middleware → resolveRequestContext(c) → { tenant_id, subject, user_credential }
 3. apis/turns.ts  beginTurnExecution():
      • build a TurnResourceResolver bound to this request's stores + credentials
      • mintPeeredTurnId(EXECUTOR_ID)  → "01j8x….replica-7"
      • derive the session title from the first message (first turn only)
 4. SessionHandle.createTurn():
      • load previous turn snapshot; rebuild AgentThreads from it
      • resolver.resolveAgentSpec() (ref sessions) → AgentSpec
      • builtinsFromSpec(spec) → capabilities
      • resolver.resolveAgentDefinition() → model client + IToolSets
      • resolver.resolveSandbox() → undefined (sandbox disabled for this agent)
      • orchestrator.send([user message]) — drained to memory
      • store.createTurn(...) — ONE atomic write
 5. activeTurns.track({ sessionId, turnId, abortController, stream: turn.stream() })
    eventSubscriptions.get(turnStreamId(tenant, sid, turnId))
    a SERVER_EXECUTION_TIMEOUT_SECONDS timer is armed
 6. TurnHandle.stream() → orchestrator.execute() → AgentThread.execute():
      state = 'llm-call-required'
      • preLLM processors run  (compaction checks the token estimate — under threshold, no-op)
      • InstructionBuilder assembles the system prompt:
          agent instructions + deferred-tool guidance + subagent guidance + OpenUI rules + …
      • tools = [ list_tools, get_tool_info, call_tool,      ← deferred meta-tools
                  create_sub_agent, ask_user_question,
                  get_current_datetime, get_openui_instructions ]
        (Linear's 30 tools are NOT in context — preload:false)
      • ILLM.create(...) streams
      • yield model.message  → dual-written (DB + EventSubscription) → SSE flush
      • yield model.message.delta × N → SSE only
      • final assembled model.message with tool_calls:[ list_tools({mcp_server:'linear'}) ]
      state = 'tool-response-required'
      • executeToolCalls → DeferredTool returns Linear's tool list
      • ToolResponseProcessors: LargeToolResponse checks size (small, passes through)
      • yield tool.response → persisted → SSE
      state = 'llm-call-required'
      • model calls call_tool({mcp_server:'linear', tool_name:'list_issues', input:{…}})
      • ToolSelectorPolicy: 'list_issues' is @read-only → no approval needed
      • RemoteMCP executes it over HTTP
        (had it been 'create_issue' → @write → ApprovalRequiredResponse →
         state='user-input-required' → tool.approval_required event → the loop PAUSES,
         the turn stays open, the UI renders Approve/Deny, and a
         POST .../turns with user.tool_approval resumes it)
      • yield tool.response → persisted → SSE
      state = 'llm-call-required'
      • model produces a final answer with no tool calls
      • yield internal.agent.done → TurnHandle maps it to thread.done → persisted → SSE
 7. TurnHandle writes the terminal turn state, emits turn.done, then resolver.close()
 8. SSE stream ends; the client has every event with dense sequence numbers
```

**If the browser disconnects at step 6**, execution continues server-side. On reconnect the client
calls `GET /sessions/{sid}/turns/{tid}/subscribe?after=<last sequence>` and resumes exactly where
it left off — routed to the owning replica via the executor id embedded in `tid`.

---

## 12. Context engineering — the differentiating features

The context window is the scarce resource. Every feature here buys context back.

### Deferred tool loading

**Problem:** ten MCP servers × 30 tools × ~200 tokens of JSON Schema = 60 000 tokens gone before
the user types anything.

**Solution:** with `preload: false` (the default), a server contributes only its *name and
description*. Four meta-tools do discovery on demand:

| Tool | Purpose |
| --- | --- |
| `list_tools(mcp_server)` | names + truncated descriptions (≤200 chars) for one server |
| `get_tool_info(mcp_server, tool_name)` | the full input schema for one tool |
| `get_tool_output_schema(mcp_server, tool_name)` | the output schema |
| `call_tool(mcp_server, tool_name, input)` | execute it |

Cost: one or two extra round-trips when a tool is first needed. Benefit: the context holds only
the schemas actually used. `preload_tools` lets you keep a few hot tools eager while the rest stay
deferred.

### Subagents

`create_sub_agent(...)` spawns a child `AgentThread` with a fresh context, the same tools, and
`SUB_AGENT_IDENTITY` prepended to its instructions ("focus on the delegated task, return a concise
result, you cannot ask the user questions"). Up to 5 run in parallel. Only the child's final
message returns to the parent as the tool result. Recommended explicitly for "large unstructured
output like web search, documentation lookup, or broad search results."

### Large tool responses

A `ToolResponseProcessor` inspects every raw tool result *before it enters context*. Over
`individual_tool_response_token_threshold` (6 000) or a batch over
`total_tool_response_token_threshold` (10 000) → the full payload is written to a sandbox file and
context receives a ~100-character preview plus the path, along with guidance telling the model it
can `grep`/`jq`/parse the file in the sandbox. Requires a sandbox.

### Code Mode

Instead of `call_tool` → giant JSON → model reads it all, the agent writes a **Python script in
the sandbox** that calls MCP tools directly and prints only a summary.

The mechanism is the interesting part. Sandboxed code must reach MCP servers whose credentials
live on the *host*, so the harness injects `mcp_client.py` and opens a bridge:

```
sandbox python  ──>  mcp_client.py  ──NATS/UDS──>  CodeModeTransport  ──>  CodeModeDispatcher
                                                                              │
                                                       host-side IToolSet.callTool()
```

Wire protocol is a tiny discriminated union (`{op:'list_tools'|'call_tool', …}` /
`{ok:true,result} | {ok:false,error,source}`). Per-server allow-lists are passed as
`TFY_MCP_SERVERS`, destructive tools stay blocked in code mode (they must go through the approval
flow), and W3C trace context (`traceparent`) is propagated through NATS headers so sandbox-side
MCP calls nest under the originating `Sandbox: exec` span.

**Credentials never enter the sandbox** — the sandbox asks the host to make the call.

### Context compaction

A `preLLM` processor. When estimated input tokens exceed the threshold (explicit
`config.…compaction.trigger.value`, else 80% of the model's context length, else 50 000), it asks
the model to summarize older history and emits an `agent.context.overwrite` event replacing that
history with a structured summary. Because it's an *event*, the rewrite is persisted with the same
persist-before-mutate discipline as everything else and is fully replayable.

### Skills

Git-backed instruction packs (`SKILL.md`). Only the skill's **name and description** enter the
input context. When the model decides a skill is relevant it reads the full body from the sandbox
("progressive disclosure"). Installing a skill sparse-clones a subdirectory of a repo into the
sandbox via `git_downloader.py`, with process-scoped git credentials (`GIT_CONFIG_*`, never a
mutated `~/.gitconfig`). Requires `config.sandbox.enabled`.

---

## 13. Sandbox architecture

```
AgentThread ──> Sandbox (an IToolSet) ──> SandboxProvider ──> Daytona | TrueFoundry | local
                    │
                    ├── exec tool                (shell/python execution)
                    ├── file read/write/download
                    ├── SkillMounter             (git sparse clone of SKILL.md packs)
                    └── CodeModeDispatcher       (MCP bridge back to the host)
```

Design points:

- **Lazily provisioned.** No VM exists until the first sandbox tool call; that call emits
  `sandbox.created`.
- **Reattached across turns.** `resolveSandbox({ existing })` reuses the previous turn's
  `sandbox_id` so files persist within a session. Providers auto-stop (5 min), auto-archive
  (60 min) and auto-delete (7200 min) idle sandboxes.
- **Scripts are injected safely.** `buildWriteAndRunScriptCommand` base64-encodes content, writes
  it, `chmod a-w`s it, then runs it — never interpolating raw text into a shell command.
- **Path traversal is validated** (`validateNoPathTraversal`), and downloads are capped by
  `SANDBOX_FILE_MAX_BYTES_FOR_DOWNLOAD`.
- **A local provider exists** (`src/sandbox/local/`) for development without a cloud account, with
  its own contract tests and a Lima-based smoke test on macOS.

---

## 14. Deployment topologies

| | Local | Hosted |
| --- | --- | --- |
| Command | `npx @truefoundry/trueforge` | Docker Compose / Helm / Railway |
| `STANDALONE` | `true` | `false` |
| Storage | SQLite file | Postgres |
| Extra infra | none | Postgres + Redis |
| Replicas | 1 | N behind a load balancer, peered via Redis |
| Controller | in-process | separate process |
| Auth | none by default | OIDC |

`docker-compose.yml` builds the full stack for `pnpm smoke` (host ports offset to `5433`/`6380`/
`8791` so they never collide with dev infra on `5432`/`6379`/`8790`). `docker-compose.dev.yml`
brings up only Postgres + Redis for `pnpm dev`. `AGENTS.md` requires the shared settings in both
files to stay synchronized, with intentional differences kept explicit.

Two server entry points: `dist/main.js` (env-only boot; Docker, `pnpm start`) and `dist/cli.js`
(the `npx` CLI with `--help`/`--port`, which then loads main).

> Local mode has **no login** and a local SQLite file. It is explicitly documented as
> localhost-only and not production-safe.

---

## 15. Engineering conventions and why they exist

`AGENTS.md` is the law of the repo (`CLAUDE.md` is a one-line `@AGENTS.md` include, and every
nested `AGENTS.md` has a sibling `CLAUDE.md` doing the same, so Cursor and Claude Code load
identical rules). The rules are unusually good; here's what they buy:

| Rule | Why |
| --- | --- |
| No `as T`, `as unknown as T`, `!`, `as never` to silence type errors | Assertions hide the real modeling bug; fix the contract instead |
| Rethrown errors must set `{ cause: caught }` | The original stack survives to the logs |
| One canonical owner per type/schema/helper; no forwarding shims | Kills the "which of these three definitions is real?" problem |
| Runtime types must be `z.infer<typeof Schema>` | A hand-written interface *will* drift from its validator |
| `z.discriminatedUnion` whenever members share a literal tag | Better errors and much better narrowing |
| Static `import` only — no `require`, no lint suppressions | Keeps the ESM/CJS dual build tractable |
| Dead code must be deleted in the same change that orphans it | No "just in case" shims |
| Tests live in a top-level `test/` mirroring `src/`, never inline | Test files never ship in `dist` |
| Multi-param functions of the same type take one options object | `f(a, b)` with two strings is a bug waiting to happen |
| `snake_case` for all wire and DB identifiers | One convention across HTTP, JSON, jsonb and SQL |
| `rem` for UI lengths (1rem = 16px) | Respects user font scaling |
| No `process.env` outside `config.ts` | Config is validated once, at boot, in one place |
| Comments explain intent/trade-offs, never restate code; no file-level banners; no `{@link}` chains | Comments that restate code rot; comments that explain *why* don't |
| Route/schema descriptions ≤ 50 words, caller-visible contract only | Descriptions ship in the public OpenAPI doc |

**Process:**

- **Changesets** are required for any change to published-package code (`trueforge-core`,
  `trueforge`, `trueforge-ui`, `trueforge-sdk`, and `frontend` because it ships inside
  `@truefoundry/trueforge`). Docs, CI, charts and compose changes are exempt.
- **Generated artifacts are committed but never hand-edited**: `packages/trueforge-sdk`,
  `.github/fern/openapi/openapi.json`, `docs/openapi.json` (the two OpenAPI copies must stay
  byte-identical).
- **CI path filters, matrix package ids, and root `test:*` scripts must stay synchronized** when
  a package is added, renamed or moved.
- **Every workspace task is a `package.json` script** — no ad-hoc commands in docs. If a workflow
  is repeatable, it becomes a script.
- Husky + lint-staged run Prettier on commit; ESLint (flat config, typescript-eslint, React hooks
  plugin) runs in CI.

---

## 16. Build your own: a staged blueprint

Here's how I'd build a TrueForge-shaped system from zero, in the order that keeps you shipping.
Each stage is runnable.

### Stage 0 — Decide your seams first (half a day, pays for itself immediately)

Write these five contracts before any implementation:

```ts
// 1. The model. One shape, model identity bound to the instance.
interface ILLM {
  create(body: Omit<ChatCompletionCreateParamsStreaming, 'model'>):
    AsyncGenerator<Chunk, AssistantMessageWithUsage>;
}

// 2. Tools. Everything is a toolset — remote, local, client-side, sandbox.
interface IToolSet {
  name: string; id: string; preload: boolean;
  listTools(): Promise<ListToolsResponse>;
  callTool(params, approval?): Promise<CallToolResponse>;  // discriminated union!
}

// 3. Persistence. One interface, many backends, one contract test suite.
interface ISessionStore { createSession; getSession; createTurn; appendEvent; listEvents; … }

// 4. Wiring. spec (names) → live objects (with secrets). Your product's seam.
interface ITurnResourceResolver { resolveAgentDefinition; resolveSandbox; close; }

// 5. Extensions. Plugins, not flags.
interface AgentCapability { systemToolSets?; preLLMProcessors?; toolResponseProcessors?; … }
```

If you get these right, everything after is fill-in-the-blanks. If you get them wrong, you'll be
rewriting in month three.

### Stage 1 — The loop (1–2 days)

Build the state machine. Nothing else.

```ts
type State = 'llm-call-required' | 'tool-response-required' | 'user-input-required';

function deriveState(context: Message[]): State {
  const open = openToolCallIds(context);            // assistant tool_calls with no tool reply
  if (open.size === 0) return 'llm-call-required';
  const last = lastAssistant(context);
  const needsHuman = last?.tool_calls?.some(tc =>
    open.has(tc.id) && (tc.needsApproval || tc.isClientSide));
  return needsHuman ? 'user-input-required' : 'tool-response-required';
}

async function* execute(signal?: AbortSignal) {
  try {
    for (;;) {
      switch (deriveState(this.context)) {
        case 'llm-call-required':
          if (signal?.aborted) return;
          if (this.iterations++ >= this.limit) { yield errorEvent('iteration limit'); return; }
          if ((yield* this.stepLLM()) === 'exit') return;
          break;
        case 'tool-response-required':
          if (signal?.aborted) return;
          if ((yield* this.stepTools()) === 'exit') return;
          break;
        case 'user-input-required':
          if ((yield* this.stepUserInput()) === 'exit') return;
          break;
      }
    }
  } catch (e) {
    yield errorEvent(describeUnknownError(e));   // NEVER throw out of the loop
  }
}
```

Non-negotiables even at this stage:

- **Async generator, not callbacks.** It gives you backpressure, cancellation and
  persist-before-yield for free.
- **Derive state from context.** Don't store a state field.
- **Yield errors, never throw them.**
- **`yield` before you mutate.** Even without a database yet, build the habit.

Test it with a fake `ILLM` that returns scripted responses. You now have a working agent.

### Stage 2 — Durability (2–3 days)

Add `ISessionStore` + an in-memory implementation, then `SessionHandle` / `TurnHandle` around the
loop:

- `createTurn` rebuilds threads from the previous turn's snapshot, sends the new input, drains
  appends into memory, and **writes everything in one atomic store call**.
- `stream()` persists each event before yielding it. Deltas pass through unpersisted.
- Write the store **contract test suite** now, while there's one implementation. When you add
  SQLite and Postgres later, they just bind to the suite.

Restart-mid-turn should now be recoverable, and "resume this conversation" should be a snapshot
load rather than a replay.

### Stage 3 — Tools (2–3 days)

Implement `LocalToolMCP` first (`defineTool({ name, description, schema, handler })` over Zod),
then `RemoteMCP` over the MCP SDK. Make `callTool` return the discriminated union from day one,
even if you only implement `{ result }` — the pause cases are much harder to retrofit than to
anticipate.

Add the tool selector policy (`@all` / `@read-only` / `@write` / `@destructive` + literal names)
and wire approvals through `ApprovalRequiredResponse`. Human-in-the-loop is now free.

### Stage 4 — The server (3–5 days)

- Hono + `@hono/zod-openapi`. Split **route definitions** from **handlers** from the start.
- All env through one validated config module. Fail at import time.
- SSE for turn creation. Get the dual write right: persist → publish to the subscription →
  flush to the socket.
- Add `/subscribe?after=<seq>` with dense sequence numbers. Do this *now*, not later — retrofitting
  resumability into a streaming API is miserable.
- Generate the OpenAPI doc from the same route objects; serve Swagger at `/docs`.

### Stage 5 — Persistence backends (2–3 days)

Kysely (or Drizzle) with parallel migration directories per engine. SQLite for local, Postgres for
hosted. Dynamic-import the driver so only the active engine loads. Run the contract suite against
both in CI.

### Stage 6 — The UI (1–2 weeks)

Generate the client from OpenAPI (Fern, `openapi-typescript-codegen`, or Orval) — don't write it.
Build the UI against **ports**, with your server as one adapter. Even if you never write a second
adapter, the discipline keeps API assumptions out of components.

### Stage 7 — Context engineering (ongoing)

Add capabilities one at a time, each as a plugin, in this order of value-per-effort:

1. **Deferred tool loading** — biggest context win, no infra needed.
2. **Compaction** — a `preLLM` processor plus a summarize call.
3. **Subagents** — needs the orchestrator; huge for research-shaped tasks.
4. **Sandbox** — start with Docker locally; add a cloud provider behind the same interface.
5. **Large tool response offloading** — requires the sandbox.
6. **Code Mode** — requires the sandbox plus a bridge; do it last.

### Stage 8 — Scale-out (when you actually need it)

Redis-backed event subscriptions (identical semantics to in-memory), executor ids embedded in
resource ids, request-reply for cross-replica commands, and a single-replica controller process
for periodic work. Don't build this before you have two replicas.

### A minimal skeleton to start from

```
my-harness/
├── packages/
│   ├── core/
│   │   ├── src/runtime/    AgentThread.ts, Orchestrator.ts, AgentDefinition.ts
│   │   ├── src/llm/        ILLM.ts, OpenAIAdapter.ts
│   │   ├── src/tools/      IToolSet.ts, LocalTool.ts, RemoteMCP.ts
│   │   ├── src/events/     schema.ts
│   │   ├── src/capabilities/  AgentCapability.ts, builtins/
│   │   └── src/session/    Sessions.ts, SessionHandle.ts, TurnHandle.ts, store/
│   ├── server/
│   │   ├── src/config.ts   ← the only process.env reader
│   │   ├── src/routes/     ← definitions
│   │   ├── src/apis/       ← handlers
│   │   ├── src/db/         ← migrations + stores
│   │   └── src/main.ts
│   └── ui/
├── AGENTS.md
└── package.json            ← every task is a script
```

### Time budget for a working v1

| Stage | Effort |
| --- | --- |
| 0. Contracts | 0.5 day |
| 1. Loop | 1–2 days |
| 2. Durability | 2–3 days |
| 3. Tools | 2–3 days |
| 4. Server | 3–5 days |
| 5. Backends | 2–3 days |
| 6. UI | 1–2 weeks |
| 7. Context features | ongoing |
| **Usable v1** | **~3–4 weeks solo** |

---

## 17. Design decisions worth stealing (and ones to skip)

### Steal these

1. **Derive loop state from message history.** Crash recovery, branching and multi-replica handoff
   all become free.
2. **Async generators end to end.** The loop yields; the consumer persists and transports. One
   pattern gives you streaming, backpressure, cancellation and durability ordering.
3. **`yield` before `mutate`.** The whole consistency story in one rule.
4. **Names in the spec, secrets in the resolver.** Makes configuration safe to store, log, diff
   and display, and keeps credentials out of every layer that doesn't need them.
5. **Everything is a toolset.** Sandbox, builtins, remote MCP and client-side tools behind one
   interface means the loop has one code path.
6. **Every pause is a tool-call outcome.** Approvals, client-side tools, OAuth prompts and
   subagent spawns are all just non-`result` returns from `callTool`.
7. **Capabilities as plugins with named hook points.** Beats a growing pile of `if (config.x)`.
8. **One store interface, one contract suite, many backends.** Adding Postgres after SQLite
   becomes a day, not a month.
9. **Route definitions separate from handlers, OpenAPI generated from both.** Docs and SDK cannot
   drift.
10. **Generate the client.** Hand-written API clients are a permanent tax.
11. **Catalogs (presets) ≠ settings (configuration).** Users discover from the first, configure the
    second.
12. **Dense sequence numbers on the event stream.** Resumability is a one-line client change.
13. **The `trueforge-dev` export condition.** Source resolution in dev, `dist` for consumers,
    no build step in the inner loop.
14. **An `AGENTS.md` that's actually enforced.** The type-assertion ban and the "one canonical
    owner" rule alone prevent whole categories of rot.
15. **Deferred tool loading.** The cheapest large context win available.

### Skip these unless you need them

- **Multi-replica peering** (Redis request-reply, executor ids) — real complexity; only worth it
  above one replica.
- **Code Mode's NATS bridge** — powerful but the most intricate subsystem here.
- **Enterprise control-plane mode** (`src/truefoundry/`) — an integration, not a pattern.
- **Dual CJS/ESM publishing** — pure ESM is fine if you control your consumers.
- **A separate controller process** — an in-process interval is fine at one replica.
- **Generative UI (OpenUI)** — a large instruction surface for a niche win; add it late.

### Traps to avoid

| Trap | Consequence |
| --- | --- |
| Storing loop state instead of deriving it | Crash recovery and branching become bespoke, buggy code |
| Throwing out of the agent loop | Users see 500s instead of a visible error step |
| Persisting after yielding | A crash between the two leaves the DB behind the client |
| Putting credentials in the agent spec | Specs stop being safe to log, store, copy or display |
| Preloading every tool schema | 40 000 tokens of context gone before the first message |
| Hand-writing types that mirror Zod schemas | Guaranteed drift |
| Offset pagination on an append-only log | Duplicates and gaps under concurrent writes |
| Re-executing a turn on client reconnect | Duplicate tool side effects — subscribe, don't re-run |
| Running the agent inside the sandbox | A VM per conversation, and secrets in the blast radius |

---

## 18. Glossary

| Term | Meaning |
| --- | --- |
| **Harness** | The runtime around an LLM: loop, tools, context, persistence, safety |
| **AgentSpec** | Static agent configuration (JSON, Zod-validated, secret-free) |
| **AgentDefinition** | The *resolved* runtime form of a spec: live model client, toolsets, limits |
| **Session** | A conversation; owns turns |
| **Turn** | One user request + the agent's full response; the unit of execution and cancellation |
| **Thread** | One agent's message history within a turn (root, or one per subagent) |
| **Context** | The messages sent to the model on a given call |
| **Capability** | A plugin adding tools, instruction sections, and context processors |
| **ToolSet** | Anything exposing tools: remote MCP, local functions, client-side, sandbox |
| **MCP** | Model Context Protocol — the standard for exposing tools to models |
| **Deferred tool** | A tool whose schema is fetched on demand rather than preloaded |
| **Code Mode** | The agent scripting MCP calls in a sandbox, with a bridge back to the host |
| **Compaction** | Replacing older history with a summary when context grows too large |
| **Skill** | A git-backed `SKILL.md` instruction pack, loaded on demand in the sandbox |
| **Sandbox** | An isolated execution environment exposed to the agent *as a tool* |
| **Executor** | One server replica; its id is embedded in every turn id it owns |
| **Request-reply** | Redis RPC that routes a command to the replica owning a resource |
| **Controller** | The single process running periodic control loops (e.g. schedules) |
| **Catalog** | Shipped YAML presets for discovery (never executed against) |
| **Settings** | What the user actually configured, stored in the DB, with secrets |
| **Standalone** | Single-process mode: SQLite, no Redis, no login |

---

## Where to look when you want to understand X

| Question | File |
| --- | --- |
| How does the agent loop work? | `packages/trueforge-core/src/core/runtime/AgentThread.ts` (start at `execute()`, line ~1302) |
| How do subagents run in parallel? | `core/runtime/AgentThreadOrchestrator.ts` |
| What can I configure on an agent? | `agent-session/schemas/agentSpec.ts` |
| How is a turn persisted? | `agent-session/SessionHandle.ts` → `createTurn()`, `TurnHandle.ts` → `stream()` |
| What does the storage layer require? | `agent-session/store/ISessionStore.ts` |
| How do I plug in my own models/tools/sandbox? | `agent-session/ITurnResourceResolver.ts` |
| How do I add a feature to the loop? | `core/capabilities/AgentCapability.ts` + `builtins/` |
| What HTTP endpoints exist? | `packages/trueforge/src/routes/` (definitions), `src/apis/` (handlers) |
| How is streaming resumable? | `src/runtime/event-subscription/` + `src/runtime/peeringIds.ts` |
| What env vars exist? | `packages/trueforge/src/config.ts`, `packages/trueforge/.env.example` |
| How is the DB laid out? | `src/db/postgres/types.ts` + `migrations/` |
| How do I run it locally? | `CONTRIBUTING.md` → "Running from source" |
| What are the coding rules? | `AGENTS.md` (root, and nested per package) |

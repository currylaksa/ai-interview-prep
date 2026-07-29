# 03 · Tool Use & MCP

Tool use is the mechanism that turns an LLM from a text generator into
something that can actually do work — look things up, call APIs, take
action. Almost every Applied AI Engineer and FDE interview will probe
whether you understand the loop mechanically (not just "the model calls
functions") and whether you've thought about the failure modes that show up
once tool use hits production traffic. MCP (Model Context Protocol) gets its
own section because it's increasingly the interface layer interviewers
expect you to know — not as trivia, but as an architectural choice with real
tradeoffs against writing a plain function.

---

## Multiple choice

### Q3.1 · The tool-use loop · [Recall]

You're explaining to a junior engineer how "the model calls a tool." Which
of these is the most accurate description of what actually happens?

- **A.** The model directly invokes a function in your codebase, and the
  return value is inserted into its next token prediction
- **B.** The model emits structured output naming a tool and arguments; your
  code executes the tool and sends the result back as a new message in the
  conversation, then you call the model again
- **C.** The model runs the tool inside its own sandboxed environment and
  only reports the final answer to your application
- **D.** Your application pre-selects which tool the model must use for a
  given input, and the model just fills in the arguments

<details>
<summary>Answer</summary>

**B**

**Why B:** The model has no execution capability of its own — it predicts
tokens. "Tool calling" means the model outputs a structured request (tool
name + arguments) as its generation. Your application code parses that,
runs the actual tool, and appends the result to the conversation as a new
message. You then call the model again with the updated history so it can
use the result. This is a full request/response round trip per tool call
(or batch of parallel calls), not an in-model operation.

**Why not A:** The model never touches your codebase directly — there's no
execution boundary crossing at the model level, only structured text
generation that your code interprets.

**Why not C:** Some hosted "computer use" or code-execution products *do*
run in a sandbox, but that's a specific product feature, not what "tool
calling" means generically. Standard tool use is: model requests → your
code executes → you return the result → you call the model again.

**Why not D:** That describes forced tool choice (a real feature — you can
constrain which tool the model must pick), but it's not the default
behaviour and it doesn't describe the general loop.

**Interviewer's likely follow-up:** *"So if the model calls three tools in
a row, how many round trips to the API does that take?"* (At minimum three
additional calls, one after each tool result is appended — unless the model
requests multiple tools in parallel within a single turn, in which case you
execute them concurrently and return all results before the next round
trip.)
</details>

---

### Q3.2 · Why descriptions matter more than names · [Applied]

Your team ships a tool called `search` with parameters `q` (string) and
`n` (integer). In testing, the model frequently passes natural-language
sentences into `q` when a structured filter query was expected, and picks
arbitrary values for `n`. What's the most likely root cause?

- **A.** The model wasn't fine-tuned for tool use
- **B.** The parameter and tool descriptions don't tell the model what
  `search` searches over, what query syntax `q` expects, or what `n` means
  or bounds to
- **C.** The tool name `search` is too short for the model to parse reliably
- **D.** The temperature is too high during the tool-selection step

<details>
<summary>Answer</summary>

**B**

**Why B:** The model only knows what you tell it in the schema. A bare name
like `search` and cryptic parameter names like `q`/`n` with no description
force the model to guess intent from naming conventions alone, and it will
often guess wrong — passing free text into a field meant for structured
syntax, or picking a plausible-looking number for an undocumented limit.
Rich descriptions ("free-text search over product titles, not full
sentences" / "max results to return, 1–50") directly fix this. This is the
standard interview point: tool *descriptions* carry more information
density than tool *names*, and are usually the first thing to fix when
tool selection or argument quality is bad.

**Why not A:** All current frontier models used in interview-relevant work
are already trained on tool use; the failure here is a schema-authoring
problem, not a model-capability problem.

**Why not C:** Name length isn't the mechanism — `search` is a perfectly
ordinary word. The problem is what's missing around it, not the name
itself.

**Why not D:** Temperature affects sampling diversity, but a systematic,
repeated misunderstanding of what a parameter means is a schema
information problem, not a randomness problem. Lowering temperature
wouldn't fix a model that genuinely doesn't know what `q` is for.

**Interviewer's likely follow-up:** *"You fix the descriptions and it's
still inconsistent — what do you check next?"* (Whether the parameter
typing itself is loose — e.g. `q: string` accepting anything vs. an enum
or constrained schema — whether the description includes an example, and
whether the tool name or purpose collides with another tool in the same
toolset.)
</details>

---

### Q3.3 · Parallel tool calls · [Recall]

A model is given three independent tools in a single turn —
`get_weather`, `get_stock_price`, `get_time` — and the user asks for all
three at once. What determines whether the model calls them in parallel
rather than one at a time?

- **A.** Your integration executes tool calls sequentially or concurrently
  based on how many the model requests in one turn — the model decides how
  many calls to request together based on whether it judges them
  independent
- **B.** The order the tools are listed in the tools array
- **C.** Whether the tools share the same return type
- **D.** The number of tokens remaining in the context window

<details>
<summary>Answer</summary>

**A**

**Why A:** When calls are independent, a well-prompted model will typically
request multiple tool calls within the same turn, and your application
code chooses to execute them concurrently (e.g. via `asyncio.gather` or
`Promise.all`) rather than one round trip per call. Parallelism is a
property of how many calls the model batches into one turn *and* how your
integration code chooses to execute that batch — not a special API mode
that automatically parallelises unrelated single calls.

**Why not B:** Array ordering has no bearing on execution strategy — it's
just how the tools are presented in the schema.

**Why not C:** Return type has nothing to do with whether calls are
independent; two tools with the same return type can have a strict
ordering dependency (e.g. "book the flight" then "confirm the booking"),
and two tools with different return types can be fully parallel.

**Why not D:** Remaining context budget affects how much you *can* fit, not
whether execution happens in parallel.

**Interviewer's likely follow-up:** *"What's a case where the model
requests calls in parallel that actually need to run sequentially?"*
(Classic example: `check_inventory` and `place_order` for the same item —
if the model treats them as independent and you execute them concurrently,
you can place an order before confirming stock. Tool descriptions and
system prompting need to make ordering dependencies explicit, or the
integration layer needs to enforce them regardless of what the model
requests.)
</details>

---

### Q3.4 · Returning tool errors to the model · [Applied]

A tool call to an internal pricing API times out. Your integration code
catches the exception. What should happen next?

- **A.** Silently retry the call up to 3 times with no visibility to the
  model, and only surface an error if all retries fail
- **B.** Return a structured error result to the model (e.g. `{"error":
  "pricing_api_timeout"}`) as the tool result, so the model can decide how
  to proceed — retry, try an alternative, or tell the user
- **C.** Terminate the conversation and return an HTTP 500 to the caller
- **D.** Substitute a cached or default price silently so the flow
  continues uninterrupted

<details>
<summary>Answer</summary>

**B**

**Why B:** The model is the orchestrator of the conversation — if a tool
fails, it needs to know, in the same structured channel it gets successful
results in, so it can reason about next steps: retry, try a different tool,
ask the user for more information, or apologise and stop. Treating errors
as just another tool result (not a special hidden-from-model failure) keeps
the model's reasoning grounded in what actually happened.

**Why not A:** Retrying silently before the model ever sees a failure is
often reasonable *as an infrastructure concern* (transient timeouts), but
"no visibility to the model" framed as the complete answer is wrong — if
retries are exhausted, the model must be told, and even a first-time
timeout is often worth surfacing if latency matters to the user experience.
This option describes hiding the failure entirely, which isn't correct.

**Why not C:** Killing the whole conversation over one tool failure is a
poor user experience and wastes the context already built up — the model
might have a perfectly good fallback (ask for a different SKU, offer to
follow up later) if it's simply told the call failed.

**Why not D:** Silently substituting a default price is dangerous — it
fabricates data the model and user believe is real, which is especially
bad for anything involving money. If a fallback value is used, that must be
explicit in the tool result, not silent.

**Interviewer's likely follow-up:** *"How do you stop the model from
retrying the same failing tool forever?"* (Cap retries at the application
layer — e.g. track call count per tool per turn and refuse to execute
beyond a limit, returning a result like `{"error": "max_retries_exceeded"}`
that tells the model to stop trying and escalate instead.)
</details>

---

### Q3.5 · Hallucinated arguments · [Applied]

You have a tool `refund_order(order_id: str, amount: float)`. In
production logs, you find a call where the model invoked
`refund_order` with an `order_id` that was never mentioned anywhere in the
conversation. What's the most likely explanation?

- **A.** The model is malicious and is attempting fraud
- **B.** The model pattern-matched on a plausible-looking ID format from its
  training data or from a similar ID earlier in context, because nothing in
  the tool schema or prompt required the ID to be sourced from a prior tool
  result or user message
- **C.** This is expected behaviour — the model always fabricates IDs when
  none are given
- **D.** The API silently substituted a placeholder value

<details>
<summary>Answer</summary>

**B**

**Why B:** Models will produce a plausible-shaped value for a required
argument even when no legitimate value exists in context — this is the
"hallucinated argument" failure mode, and it's a direct consequence of
not constraining *where* an argument value is allowed to come from. The
fix isn't "the model is broken," it's a system design gap: the argument
should have come from a prior `lookup_order` tool call result, and nothing
enforced that dependency. Load-bearing arguments (order IDs, account IDs,
anything tied to money or access) should never be trusted from a bare tool
call without independent verification against something you already know
to be real.

**Why not A:** Attributing intent like "malicious" to next-token prediction
is a category error — the model isn't reasoning about deception, it's
producing a statistically plausible completion for a required field.

**Why not C:** It's a known failure mode, not universal or guaranteed
behaviour — well-designed schemas and prompting sharply reduce (but don't
eliminate) its frequency.

**Why not D:** Nothing in a standard tool-calling API silently fills in
arguments — argument values come entirely from the model's generation.

**Interviewer's likely follow-up:** *"How would you actually prevent the
refund from executing?"* (Validate the `order_id` server-side against
orders that exist in the current conversation's verified context — e.g.
require it to match an ID that was returned by an earlier `lookup_order`
call in the same turn history, and reject/return an error otherwise, never
trusting the model's argument as ground truth for anything irreversible.)
</details>

---

### Q3.6 · Tool result size and context budget · [Applied]

A `search_documents` tool returns full document text for the top 20
matches, averaging 2,000 tokens each. After a few turns of tool use, the
conversation blows past the context window and the model starts losing
track of earlier instructions. What's the best fix?

- **A.** Switch to a model with a larger context window and leave the tool
  unchanged
- **B.** Truncate the returned documents to titles and short snippets by
  default, and give the model a separate tool to fetch full text for a
  specific document it decides it needs
- **C.** Reduce the number of results from 20 to 10
- **D.** Compress the tool results using gzip before returning them

<details>
<summary>Answer</summary>

**B**

**Why B:** The actual problem is that most retrieved content is never
needed at full length — the model needs enough to decide relevance, then
can selectively drill into what matters. This is the same principle as
RAG retrieval hygiene applied to tool design: return a lean summary by
default, and let the model make a second, targeted call for full content
only when it's actually going to use it. This bounds context growth
regardless of how large the underlying corpus is.

**Why not A:** A bigger context window delays the problem without fixing
the underlying design flaw, and it doesn't solve the second-order problem
mentioned in the prompt — that the model starts *losing track of
instructions* as unrelated bulk content crowds the context, which is a
"lost in the middle" attention problem, not purely a token-count-limit
problem.

**Why not C:** Halving the result count reduces total tokens but keeps the
per-result token cost the same; it's a band-aid that will re-break as soon
as usage or corpus size grows, and it may hurt recall for legitimate
broad queries.

**Why not D:** The model consumes the tool result as text tokens — you
can't hand it gzip-compressed bytes and expect it to read them; the
transport-level compression trick doesn't apply to what the model actually
processes.

**Interviewer's likely follow-up:** *"How do you decide what counts as
'enough to decide relevance' in the summary?"* (Title, source, a short
extract around the highest-scoring match, and any structured metadata
useful for filtering — enough for the model to judge fit without paying
full-document token cost for results it won't use.)
</details>

---

### Q3.7 · What problem MCP solves · [Recall]

What problem does the Model Context Protocol (MCP) primarily solve?

- **A.** It makes LLMs more accurate at reasoning tasks
- **B.** It standardises how AI applications discover and connect to
  external tools, data sources, and prompts, so integrations aren't
  reimplemented per model/client combination
- **C.** It replaces the need for tool schemas entirely
- **D.** It provides built-in memory and long-term storage for agents

<details>
<summary>Answer</summary>

**B**

**Why B:** Before MCP, every AI application that wanted to integrate with,
say, GitHub, Slack, and a filesystem had to write bespoke integration code
per tool and often per model provider's tool-calling conventions. MCP
defines a standard client/server protocol: an MCP server exposes tools,
resources, and prompts in a consistent way, and any MCP-compatible client
(an AI application) can connect to it without custom glue code. It's
fundamentally an integration standardisation problem, analogous to what
LSP did for editor/language-server integrations.

**Why not A:** MCP doesn't change model reasoning capability — it changes
how the model's application gets connected to external capabilities.

**Why not C:** MCP servers still define schemas for their tools — MCP
standardises the *transport and discovery* layer around those schemas, it
doesn't remove the need to describe what a tool does and what arguments it
takes.

**Why not D:** Memory is a separate architectural concern; MCP has a
"resources" concept for exposing data, but it isn't a memory system, and
implementing long-term memory is still up to the application.

**Interviewer's likely follow-up:** *"If MCP didn't exist, what would you
have built instead, and what would you lose?"* (You'd hand-write
per-integration tool wrappers directly in your agent code — functionally
similar, but not portable across projects or reusable by others, and
you'd rebuild auth/connection handling per integration instead of once
per MCP server.)
</details>

---

### Q3.8 · Tools vs resources vs prompts · [Recall]

In MCP, what's the distinction between a **tool**, a **resource**, and a
**prompt**?

- **A.** They're three names for the same underlying concept, chosen based
  on developer preference
- **B.** Tools are model-invoked actions with side effects or computation;
  resources are read-only data the client can attach to context (like a
  file or record); prompts are reusable, parameterised prompt templates
  the server exposes for the client to surface to the user
- **C.** Tools are for the model, resources are for the client's UI, and
  prompts don't run on the server at all
- **D.** Resources are deprecated in favour of tools

<details>
<summary>Answer</summary>

**B**

**Why B:** This is a core MCP concept distinction. **Tools** are
model-callable functions — the model decides to invoke them as part of
reasoning, and they can have side effects (send an email, create a ticket)
or just fetch data. **Resources** are addressable, typically read-only
content (a file, a database record, a URI) that the *client application*
(not necessarily the model autonomously) can choose to attach to context —
think "here's the data available to browse," more like an attachment than
an action. **Prompts** are server-defined, parameterised prompt templates —
reusable instructions the server exposes so a client can present them to
users as a quick-start ("summarise this repo," "triage this ticket") rather
than each user writing the prompt from scratch.

**Why not A:** They're deliberately distinct primitives with different
invocation models — conflating them loses the reason MCP defines all
three.

**Why not C:** Prompts *are* server-defined content, they just aren't
"invoked" the way tools are; and resources absolutely can be relevant to
the model, not just the UI — a client commonly attaches a resource into the
model's context.

**Why not D:** Resources are a first-class, actively used primitive, not
deprecated.

**Interviewer's likely follow-up:** *"Give an example of something that
could be modelled as either a tool or a resource — how would you decide?"*
(E.g. "get the contents of file X." As a resource, the client attaches it
to context ahead of time or on user request. As a tool, the model
autonomously decides to fetch it mid-reasoning. Decide based on whether you
want the model to have agency over *when* to fetch it — dynamic,
in-the-loop need → tool; known-upfront context → resource.)
</details>

---

### Q3.9 · stdio vs streamable HTTP · [Applied]

You're building an MCP server that will run as a **local, single-user**
integration — for example, giving a desktop AI coding assistant access to
the user's local filesystem and git repo. Which transport should you pick,
and why?

- **A.** Streamable HTTP, because it's the more modern transport
- **B.** stdio, because the server runs as a local subprocess of the client
  with no need for network exposure, auth, or multi-client support — it's
  simpler and has no attack surface beyond the local process boundary
- **C.** It doesn't matter — both transports have identical operational
  characteristics
- **D.** WebSockets, because they support bidirectional streaming

<details>
<summary>Answer</summary>

**B**

**Why B:** stdio transport runs the MCP server as a child process
communicating over stdin/stdout — ideal for local, single-user tools where
the client spawns the server directly, there's no need for network
listeners, authentication, or handling concurrent remote clients. It's the
lowest-overhead choice precisely because the deployment topology (one
client, one local server, same machine) doesn't need any of what HTTP
transport is built for.

**Why not A:** Streamable HTTP is the right choice for **remote or
multi-client** scenarios — a server that multiple users or multiple client
applications need to connect to over a network. "More modern" isn't the
deciding factor; deployment topology is. Picking HTTP for a local
single-user tool adds unnecessary complexity (a listener, auth
considerations, network config) for no benefit.

**Why not C:** They have materially different operational characteristics
— stdio has no network exposure and no built-in multi-client support;
streamable HTTP does, at the cost of needing to handle auth, connection
management, and remote deployment.

**Why not D:** MCP doesn't use raw WebSockets as a defined transport;
streamable HTTP is the standard option for network-based MCP
communication, not a hand-rolled WebSocket layer.

**Interviewer's likely follow-up:** *"Now say you need to expose the same
capability to your whole engineering team, not just one local user — what
changes?"* (Move to streamable HTTP, add authentication/authorization per
user or per API key, think about rate limiting and multi-tenant isolation
between users' sessions, and consider whether the server needs to be
horizontally scalable.)
</details>

---

### Q3.10 · MCP server vs plain function · [Applied]

Your team is building an internal agent that needs to query one internal
Postgres database. A teammate suggests wrapping the query logic as an MCP
server. When would that actually be the right call, versus just writing a
plain function the agent calls directly?

- **A.** Always wrap it as an MCP server — it's the modern standard
- **B.** Write it as an MCP server if multiple different agents or AI
  clients (not just this one codebase) need the same capability, or if
  it needs to be deployed/versioned/secured independently of any single
  agent; write it as a plain function if it's used by exactly one agent in
  one codebase with no reuse or cross-team need
- **C.** Never use MCP for internal-only tools — it's meant for third-party
  integrations
- **D.** The choice should be based purely on which one is faster to call
  at runtime

<details>
<summary>Answer</summary>

**B**

**Why B:** MCP earns its complexity when there's a real integration
boundary to standardise across — multiple consumers, independent
deployment/versioning, or a need for the same capability to be reusable
outside the one codebase that first needed it. If it's genuinely a single
agent calling a single internal function with no reuse case on the
horizon, a plain function call is simpler, faster to build, easier to
debug, and avoids maintaining a separate server process for no payoff.
This is a "use the simplest thing that works, add structure when it earns
its keep" question, same principle as workflow-vs-agent in file 05.

**Why not A:** "Modern standard" isn't sufficient justification —
introducing a protocol boundary and a separate server process has real
operational cost (deployment, auth, monitoring) that should be paid for
by an actual reuse or isolation need, not adopted reflexively.

**Why not C:** MCP is commonly used for internal tools too, especially once
more than one internal agent or team needs the same capability — it isn't
scoped to third-party integrations only.

**Why not D:** Runtime call latency is rarely the deciding factor between
a plain function call and an MCP server call over stdio/HTTP — the
decision is about integration surface and reuse, not micro-latency, and
in most agent workloads the LLM call itself dominates latency anyway.

**Interviewer's likely follow-up:** *"Six months later, two more teams want
the same Postgres query capability in their own agents. What do you do?"*
(That's exactly the trigger to extract it into an MCP server — now there's
a real reuse and independent-versioning need that justifies the protocol
overhead you'd have paid for unnecessarily on day one.)
</details>

---

### Q3.11 · Wrong tool selection · [Applied]

Your agent has both `search_web` and `search_internal_kb` available. A
user asks about your company's refund policy — internal, documented
information — but the model calls `search_web` instead. What's the most
likely fix, in order of what to try first?

- **A.** Remove `search_web` entirely so there's no wrong option
- **B.** Sharpen both tool descriptions so their scope is mutually
  exclusive and unambiguous (e.g. `search_internal_kb`: "authoritative
  company policy, product, and process documentation — always prefer this
  for questions about our own company, products, or policies"), and only
  if that doesn't resolve it, consider a routing/classification step before
  tool selection
- **C.** Lower the model's temperature to zero
- **D.** Add a system prompt instruction repeating "use internal search"
  ten times for emphasis

<details>
<summary>Answer</summary>

**B**

**Why B:** Wrong tool selection is very often a description-ambiguity
problem, same root cause as Q3.2 — if both tools' descriptions could
plausibly apply, the model has no strong signal to prefer one. Making
scope boundaries explicit and mutually exclusive in the description text
is the cheapest, most maintainable fix and should be tried first. If tool
descriptions are already tight and the confusion persists (e.g. because the
distinction genuinely requires understanding intent, not just keywords), a
lightweight upstream classification/routing step is a reasonable escalation
— but it's a second resort, not the first thing to reach for.

**Why not A:** Removing `search_web` is a sledgehammer that breaks
legitimate use cases (the agent presumably needs web search for other
questions) to fix a selection-boundary problem that's solvable without
losing functionality.

**Why not C:** Temperature affects sampling randomness broadly; a
systematic, repeated wrong-tool choice reflects the model's best
interpretation of ambiguous descriptions, not noise that a lower
temperature would fix.

**Why not D:** Repeating an instruction for emphasis is a weak, brittle
lever — it doesn't scale to more tools, and it fights the actual problem
(ambiguous scope) instead of fixing it, plus verbose repeated instructions
waste context budget for a marginal, unreliable effect.

**Interviewer's likely follow-up:** *"What if the ambiguity is genuinely
unavoidable — e.g. the user asks something that could be either internal
or external knowledge?"* (Let the model call both, if latency allows, and
synthesise; or explicitly instruct a priority order — internal first,
fall back to web only if internal search returns nothing relevant.)
</details>

---

### Q3.12 · Sequential dependency across tool calls · [Design]

You're designing a tool set for an agent that books a meeting room: it
needs to (1) look up the user's calendar availability, (2) check room
availability for that slot, (3) reserve the room. What's the most robust
way to enforce that these happen in the right order, given that models
aren't perfectly reliable at respecting implied ordering from tool names
alone?

- **A.** Trust the model to call them in a sensible order based on tool
  names like `check_calendar`, `check_room`, `book_room`
- **B.** Design the tools so each step's output is a required input to the
  next (e.g. `book_room` requires a `room_hold_id` that only
  `check_room_availability` can produce), so the model *cannot* successfully
  call a later step without having called the earlier one — the ordering is
  enforced by data dependency, not by convention
- **C.** Merge all three steps into a single tool call with no
  intermediate visibility, so ordering can't go wrong
- **D.** Add a comment in the system prompt listing the correct order

<details>
<summary>Answer</summary>

**B**

**Why B:** Naming conventions and prompt instructions are soft signals —
models mostly follow them but not with the reliability you want for
something with real-world side effects like booking a room. The strongest
guarantee comes from data dependency: if `book_room` requires an ID that
literally does not exist until `check_room_availability` has run and
returned it, the model is structurally unable to skip the step, regardless
of what order it "intends" to call things in. This is the same principle
as Q3.5's fix for hallucinated order IDs, applied proactively at design
time instead of reactively after a bug.

**Why not A:** This is exactly the unreliable-by-convention approach the
question is asking you to improve on — it works most of the time, which is
precisely the problem for something that can double-book a room or skip a
step.

**Why not C:** Collapsing three steps into one tool call removes the
model's ability to react between steps (e.g. tell the user "that slot's
unavailable, want me to check the next one?"), and pushes all three
side-effecting operations behind a single opaque call, which makes
partial-failure handling much harder (what happens if calendar check
succeeds but room booking fails halfway through your merged function?).

**Why not D:** Prompt-level ordering instructions are the weakest of these
options — same soft-convention problem as A, just moved to the system
prompt instead of the tool names.

**Interviewer's likely follow-up:** *"What if the model is honest and
correct about ordering 99% of the time — is the data-dependency design
still worth the extra complexity?"* (Yes, for anything with real-world
side effects or cost — a 1% failure rate on room double-booking or,
worse, financial transactions, compounds badly at volume and erodes trust
fast; the data-dependency guarantee is cheap relative to the blast radius
of getting it wrong.)
</details>

---

### Q3.13 · Idempotency in tool design · [Design]

You're designing a `charge_customer(customer_id, amount)` tool for an
agent. The model occasionally calls a tool twice for the same logical
action — e.g. because a network timeout made it think the first call
failed, or because of an internal retry after an ambiguous tool result.
How should the tool be designed to make this safe?

- **A.** Trust that the model won't call it twice if the prompt says not to
- **B.** Require an idempotency key (e.g. a client-generated request ID) as
  a parameter, and have the underlying charge API reject or return the
  original result for a duplicate key instead of creating a second charge
- **C.** Add a manual confirmation step in the UI before every charge, and
  consider the tool itself safe by extension
- **D.** Rate-limit the tool to one call per conversation

<details>
<summary>Answer</summary>

**B**

**Why B:** This is the same idempotency principle from API design (see
file 01) applied specifically to tool design — anything with real-world,
non-repeatable side effects needs to be safe to call more than once with
the same logical intent. An idempotency key lets the underlying system
detect "this is the same charge request as before" and return the
existing result instead of double-charging, regardless of *why* the model
called it twice (timeout confusion, retry logic, model error). This is a
system-level guarantee, not a behavioural hope.

**Why not A:** Prompt instructions are a soft constraint on model
behaviour, not a system guarantee — and the scenario explicitly describes
non-malicious, plausible reasons a duplicate call happens (timeout
confusion) that no amount of prompting eliminates.

**Why not C:** A UI confirmation step is a reasonable *additional* safety
layer for high-stakes actions, but it doesn't make the tool itself
idempotent — if the confirmation step itself gets triggered twice, or is
bypassed in an automated/headless flow, you're back to the same risk. It
solves a different problem (human oversight) than duplicate execution
safety.

**Why not D:** Rate-limiting to one call per conversation is both too
blunt (a user might legitimately need two separate charges in one
conversation) and doesn't actually solve the retry/duplicate-call problem
— it just caps a symptom rather than making the operation safe to repeat.

**Interviewer's likely follow-up:** *"Where does the idempotency key come
from — the model, or your application code?"* (Best practice: your
application code generates and tracks it per logical user action, not the
model — you don't want to depend on the model reliably reusing the same
key across a confused retry; the application layer, which has more
reliable state than the model's generation, should own key generation and
mapping to the specific business transaction.)
</details>

---

### Q3.14 · Streaming and tool calls · [Applied]

A user-facing agent streams its text responses token-by-token for
responsiveness. When the model decides to call a tool mid-response, what
typically has to happen to the stream?

- **A.** The tool call streams token-by-token like text, and the user sees
  the raw tool call JSON appear live
- **B.** The text stream pauses (or the turn ends) while the tool call is
  assembled, executed, and its result is fed back — the user typically sees
  a pause or a loading/status indicator, not the tool call internals,
  before streaming resumes with the model's next response
- **C.** Streaming and tool use are mutually exclusive features — you must
  pick one per conversation
- **D.** The tool executes instantly with zero added latency because it
  runs client-side in the browser

<details>
<summary>Answer</summary>

**B**

**Why B:** Once the model's generation resolves into a tool-call request
rather than plain text, there's necessarily a gap: your code has to
receive the complete tool call, execute it (which takes real time — a
network call, a DB query), and feed the result back before the model can
continue generating. Well-built UIs surface this as a status indicator
("Searching…", "Checking availability…") rather than exposing raw tool-call
JSON, and resume token streaming once the model's next turn begins
generating actual response text.

**Why not A:** Exposing raw tool-call JSON to end users is a poor UX choice
almost nobody makes deliberately — the *mechanism* of streaming applies to
generated tokens, but production apps translate tool-call events into
human-readable status, not raw payloads.

**Why not C:** They're commonly combined — you stream the assistant's
natural-language turns and handle the (non-streamed, or differently
handled) tool-call turns with their own UI treatment in between.

**Why not D:** Tool execution latency is real — a database query, an
external API call, or a computation all take actual time; "zero added
latency" ignores this and would be a strange claim in an interview about
mechanics.

**Interviewer's likely follow-up:** *"How would you design the loading
indicator to be genuinely useful rather than a generic spinner?"* (Surface
what's actually happening where possible — "Checking room availability for
3pm…" rather than a bare spinner — using the tool name/arguments you
already have client-side the moment the tool call is emitted, before the
result comes back.)
</details>

---

### Q3.15 · Time-to-first-token and tool use · [Applied]

Your product team is concerned that a customer support agent "feels slow."
Investigating, you find that the agent calls an average of 2.3 tools per
turn before producing a final answer, and each tool call round-trip
(execute + re-call the model) takes ~800ms. Which lever addresses the root
cause of the perceived slowness most directly?

- **A.** Switch to a lower-latency model for the final response generation
  only
- **B.** Reduce the number of sequential tool round-trips — e.g. by
  batching independent lookups into parallel calls, caching frequently
  repeated lookups, or trimming unnecessary tool calls from the flow —
  since each round trip adds its own model-inference latency on top of the
  tool execution time
- **C.** Increase the context window so more information fits per call
- **D.** Add a streaming loading spinner so the wait feels shorter without
  changing actual latency

<details>
<summary>Answer</summary>

**B**

**Why B:** The stated bottleneck is architectural: ~2.3 sequential tool
round trips, each paying both tool-execution latency *and* a fresh model
inference latency to decide the next step. The direct fix is reducing the
number of sequential round trips — parallelising genuinely independent
lookups (Q3.3), caching repeat queries, or cutting tool calls that don't
change the outcome. This attacks the actual multiplier (2.3 round trips ×
per-round-trip cost) rather than shaving cost off one piece of it.

**Why not A:** Speeding up only the *final* generation ignores that most of
the latency described comes from the 2.3 tool round trips before that
final response — this optimises the smallest part of the critical path.

**Why not C:** A bigger context window doesn't reduce the number of
round trips or their latency; it's an unrelated lever (capacity, not
speed) and wouldn't move the metric described.

**Why not D:** A spinner is a legitimate perceived-performance tactic (see
Q3.14) and worth doing regardless, but the question asks for what
addresses the *root cause* — actual latency, not perceived latency — and a
spinner does nothing to the underlying 2.3×800ms.

**Interviewer's likely follow-up:** *"You cut it to 1 round trip on
average. What did you actually change architecturally?"* (Likely answer
shape: merged what used to be 2-3 sequential lookups into tools the model
can call in parallel in one turn, pre-fetched commonly-needed context
before the model even starts reasoning so it doesn't need a tool call for
it, or cached hot lookups so repeat calls skip the external round trip
entirely.)
</details>

---

### Q3.16 · Which is NOT a reason to use MCP · [Recall]

Which of the following is **NOT** a legitimate reason to build an MCP
server rather than a plain in-codebase function?

- **A.** Multiple different AI clients/agents across your org need the same
  capability
- **B.** You want the integration to be independently deployable and
  versioned from any single agent that consumes it
- **C.** You believe MCP will make the underlying model's reasoning more
  accurate on this task
- **D.** You want to expose the capability to third-party AI tools your
  team doesn't control the codebase of

<details>
<summary>Answer</summary>

**C**

**Why C:** MCP is a protocol for standardising *integration and
discovery* — it has no effect on the model's underlying reasoning quality.
Wrapping a tool as an MCP server versus a plain function makes zero
difference to how well the model reasons about when and how to use it; the
schema and description quality (Q3.2) is what drives that, and that's
identical whether the tool is exposed via MCP or a direct function call.

**Why A, B, D are legitimate:** These are the real drivers discussed in
Q3.10 — reuse across multiple consumers (A), independent
deployment/versioning lifecycle (B), and enabling third-party or
external-codebase clients to use your capability without them needing
your source code (D) are all genuine reasons the protocol boundary earns
its cost.

**Interviewer's likely follow-up:** *"So if MCP doesn't improve reasoning,
what actually would?"* (Better tool/parameter descriptions, tighter
schemas and typing, fewer overlapping/ambiguous tools, and — separately —
model choice and prompting quality; the integration layer and the
reasoning-quality layer are genuinely orthogonal concerns.)
</details>

---

### Q3.17 · Context isolation via subagents and tools · [Design]

You're building a research agent that needs to read through dozens of long
documents to answer one question. Reading all of them directly into the
main agent's context would blow the budget and bury the actual question in
noise. What tool-design pattern addresses this?

- **A.** Give the main agent a single `read_all_documents` tool that
  concatenates everything into one giant tool result
- **B.** Give the main agent a tool that dispatches each document (or a
  batch) to an isolated sub-call — e.g. a summarisation/extraction step
  with its own small context — and only returns the extracted, relevant
  information back to the main agent's context, not the raw documents
- **C.** Increase the main agent's context window until all documents fit
- **D.** Have the user manually pre-filter which documents are relevant
  before the agent runs

<details>
<summary>Answer</summary>

**B**

**Why B:** This is the tool-use analogue of context isolation via
subagents (covered more fully in file 05) — instead of pulling raw,
high-volume content into the orchestrating agent's context, you push the
*processing* of that content down into isolated calls that each work with
a small, bounded context and return only the distilled, relevant output.
The main agent's context stays small and signal-dense regardless of how
many documents exist in the corpus.

**Why not A:** This is exactly the failure mode from Q3.6 at a larger
scale — dumping raw bulk content into one context blows the budget and
degrades the model's ability to track the actual question amid the noise.

**Why not C:** Context window size is a red herring here — even with a
huge window, dumping dozens of long documents into one context measurably
degrades attention/recall on the actually-relevant parts ("lost in the
middle"), and it's far more token-expensive than extracting only what's
needed.

**Why not D:** Manual pre-filtering defeats the purpose of building an
autonomous research agent, and doesn't scale — the whole point is the
agent should be able to handle an unbounded/unknown-ahead-of-time set of
documents.

**Interviewer's likely follow-up:** *"What information do you pass back
from each isolated call to make sure the main agent doesn't lose important
nuance?"* (Extracted claims/quotes with enough surrounding context to be
verifiable, source attribution back to the specific document, and
explicit "not relevant" signals rather than silently omitting a document —
so the main agent knows what was actually checked, not just what came
back positive.)
</details>

---

### Q3.18 · Enum vs free-text parameters · [Applied]

A tool `set_ticket_priority(priority: string)` is meant to accept
`"low"`, `"medium"`, `"high"`, or `"urgent"`. In production, you
occasionally see the model pass `"Medium"`, `"high priority"`, or
`"ASAP"`. What's the most robust fix?

- **A.** Add a note in the description saying "must be exactly one of low,
  medium, high, urgent"
- **B.** Change the parameter's schema type to an enum with those four
  exact values, so the model is constrained at the schema level rather than
  relying on it correctly following a text instruction
- **C.** Accept any string and normalise/fuzzy-match it server-side to the
  closest valid value
- **D.** Reject any call that doesn't exactly match and don't return an
  error — just silently drop it

<details>
<summary>Answer</summary>

**B**

**Why B:** Enum-constrained schema fields restrict what the model can even
generate for that parameter — it's a schema-level guarantee rather than a
best-effort instruction the model might not follow precisely. This is
strictly more reliable than a text note, because it removes the failure
mode at the source instead of trying to catch it after the fact.

**Why not A:** A text instruction in the description is a soft constraint
— the model usually follows it, but "usually" is exactly the gap that
produced `"Medium"` and `"ASAP"` in production. It's a reasonable
supplement to an enum, not a substitute for one.

**Why not C:** Server-side fuzzy-matching can work as a defensive layer,
but it adds an unnecessary translation step and hidden logic for a problem
that's fully solvable by constraining the schema itself — why build and
maintain a normalisation function when the schema can make the invalid
values ungenerateable in the first place?

**Why not D:** Silently dropping an invalid call gives neither the model
nor the user any signal that something went wrong — the ticket priority
silently never gets set, which is worse than a visible failure the model
could react to (see Q3.4's argument for always surfacing tool errors).

**Interviewer's likely follow-up:** *"The model still occasionally passes
something outside the enum despite the schema — how, and what do you do?"*
(Some models/integrations are stricter about enum enforcement than others,
and a model can still emit invalid output at the raw text level before
your SDK validates it; validate server-side regardless of what the schema
"should" prevent, and return a structured error listing the valid values
so the model can self-correct on the next call.)
</details>

---

### Q3.19 · Least-context tool design · [Design]

You're designing a `create_calendar_event` tool for an agent that has
access to the full user profile (name, email, timezone, role, manager,
department, office location, phone number...). Which parameters should the
tool schema expose to the model?

- **A.** All fields from the user profile, in case the model needs them
- **B.** Only what's actually needed to create the event — title, start/end
  time, attendees, and description — pulling context like the user's
  timezone or email from the authenticated session server-side rather than
  asking the model to supply it
- **C.** None — collect everything from the user profile automatically and
  give the model no parameters at all
- **D.** Whatever fields the underlying calendar API's create-event
  endpoint accepts, unfiltered

<details>
<summary>Answer</summary>

**B**

**Why B:** Every parameter you expose to the model is something it has to
correctly reason about and potentially get wrong (see Q3.5) — fields that
your application already knows deterministically (the authenticated user's
timezone, email) shouldn't be re-derived by model generation. This mirrors
least-privilege thinking from tool/security design: expose the minimum
surface the model actually needs to make a genuine decision about, and let
your code fill in what it already knows with certainty.

**Why not A:** Exposing every profile field invites the model to
mis-populate fields it doesn't need to reason about, increases the chance
of leaking irrelevant personal data into a tool call unnecessarily, and
bloats the schema for no benefit.

**Why not C:** Some fields (title, time, attendees, description) are
genuinely things only the conversation/user request determines — the model
has to supply those, they can't be inferred server-side. Zero parameters
would make the tool unusable for its actual purpose.

**Why not D:** Blindly mirroring the underlying API's full parameter
surface conflates "what the API can accept" with "what the model should
decide" — many API fields are infrastructure concerns (internal IDs,
API version flags) that have nothing to do with the model's job and
shouldn't be part of its decision space.

**Interviewer's likely follow-up:** *"What's the security argument for
this beyond just cleanliness?"* (Fewer model-controlled fields means a
smaller attack surface for prompt-injection-driven misuse — e.g. if an
attacker can influence the model's output via injected content, they can
only manipulate the fields you actually let the model set; server-derived
fields like the authenticated user's identity stay outside the model's
reach entirely.)
</details>

---

### Q3.20 · Tool call limits and cost runaway · [Applied]

An agent with access to 12 tools occasionally gets into a loop: it calls
`search`, doesn't find what it needs, refines the query, calls `search`
again, and repeats 40+ times before a request timeout kills it. What
should have prevented this in production?

- **A.** Nothing — this is rare enough not to design for
- **B.** A hard cap on tool calls (or total steps) per turn, enforced by
  your application code independent of the model's own judgement, with a
  graceful "I wasn't able to find this — here's what I tried" fallback once
  the cap is hit
- **C.** Removing the `search` tool from the agent's toolset entirely
- **D.** Increasing the request timeout so the loop can finish naturally

<details>
<summary>Answer</summary>

**B**

**Why B:** Unbounded retry/refinement loops are a well-known agent failure
mode (covered more in file 05) — the model's own judgement about "when to
give up" isn't reliable enough to be the only safeguard, especially
because each individual retry looks locally reasonable to the model ("that
query didn't work, let me refine it"). A hard, application-enforced step
cap with a defined fallback behaviour turns an open-ended cost/latency
runaway into a bounded, predictable worst case — and gives the user a
useful response ("couldn't find it, here's what I searched for") instead
of a timeout.

**Why not A:** This is exactly the kind of failure mode you should design
for proactively — a request timeout killing the process mid-loop is a
worse user and cost outcome than catching it with an explicit limit, and
"rare" doesn't mean the cost (in latency, spend, and a broken user
experience) is acceptable when it happens.

**Why not C:** Removing the tool eliminates the capability entirely, not
just the failure mode — the agent needs `search` for legitimate cases; the
problem is the *unbounded retry*, not the tool's existence.

**Why not D:** A longer timeout just lets the loop run longer (and cost
more, if each call has a token/API cost) before eventually still failing
or succeeding by chance — it doesn't address why the loop happens or give
the model/user a defined exit path.

**Interviewer's likely follow-up:** *"How do you choose the right cap
number, and what happens if you set it too low?"* (Base it on observed
p95/p99 successful-task step counts from real usage data, with headroom;
set too low, you'll cut off legitimately multi-step tasks partway through
and give users a "gave up" response on things that would have succeeded —
so it should be informed by data, not picked arbitrarily.)
</details>

---

### Q3.21 · Tool schema typing precision · [Recall]

A tool parameter for a date is defined as `date: string` with no further
constraint. What's the main risk of this compared to a tighter
specification (e.g. an explicit ISO-8601 format note, or separate
year/month/day integer fields)?

- **A.** There's no real risk — strings can hold any date format fine
- **B.** The model may generate dates in inconsistent formats (MM/DD/YYYY
  vs DD/MM/YYYY vs "next Tuesday") across different calls, and your
  downstream code has to guess which format it received, risking silent
  misinterpretation (e.g. 03/04/2026 read as March 4th vs April 3rd)
- **C.** Strings are always slower to parse than integers
- **D.** The API will reject any tool with an unconstrained string
  parameter

<details>
<summary>Answer</summary>

**B**

**Why B:** Loose typing pushes format ambiguity downstream to your parsing
code, and date formats are a classic, high-stakes example — DD/MM vs MM/DD
ambiguity silently produces a *valid-looking but wrong* date rather than an
obvious error, which is worse than a clean failure. Specifying the exact
expected format in the description, or better, having the model return
structured year/month/day fields, removes the ambiguity at the source
instead of hoping the model is consistent.

**Why not A:** This is the misconception the question is testing — "it's
just a string, it'll be fine" is exactly how silent date-misinterpretation
bugs get shipped; the risk isn't hypothetical, it's a common real-world
tool-design mistake.

**Why not C:** Parsing speed is not the relevant concern here — the risk
is correctness/ambiguity, not performance; a date-format bug that silently
swaps day and month is far more costly than a few extra microseconds of
string parsing.

**Why not D:** Tool-calling APIs don't reject loosely-typed string
parameters — they'll happily accept the schema as written; the risk is a
runtime correctness problem, not a schema-validation-time rejection.

**Interviewer's likely follow-up:** *"What's your actual fix — pick one and
justify it?"* (Specify ISO-8601 explicitly in both the type description and
an example value in the description text, since that format is
unambiguous and machine-parseable; some teams go further and use separate
typed fields to remove any string-parsing step entirely.)
</details>

---

### Q3.22 · Multi-tenant tool scoping · [Design]

You're building an internal support agent used by multiple customer
support reps, each of whom should only be able to access tickets for
customers in their assigned region. The agent has a `get_ticket(ticket_id)`
tool. How should region-scoping be enforced?

- **A.** Tell the model in the system prompt which region the current rep
  is assigned to, and trust it not to fetch tickets outside that region
- **B.** Enforce the scoping server-side inside the tool's implementation —
  look up the authenticated rep's region from the session (not from
  anything the model says) and reject or filter `get_ticket` calls for
  tickets outside that region, regardless of what the model requests
- **C.** Give each region its own separate agent deployment with a
  hardcoded ticket ID range
- **D.** Ask the rep to manually confirm their region before every ticket
  lookup

<details>
<summary>Answer</summary>

**B**

**Why B:** This is the Zero Trust principle applied to tool design (see
file 07 for the fuller treatment): never trust the model output or prompt
content as the enforcement mechanism for an authorization boundary.
Access control has to be enforced in the tool's actual implementation,
using session/auth data your application controls, completely independent
of what the model was told or what it requests. The model can be
manipulated (via prompt injection, confused context, or just an error) into
requesting the wrong ticket — the server-side check is what actually
prevents unauthorized access, not the prompt.

**Why not A:** Prompt-level instructions are not a security boundary — a
system prompt telling the model to respect region scoping is a behavioural
hint the model usually follows, not an enforcement mechanism, and it's
exactly the kind of thing that fails silently under prompt injection or
model error, with real access-control consequences.

**Why not C:** Region-specific deployments with hardcoded ID ranges is
operationally heavy (N deployments to maintain) and brittle (ticket ID
ranges changing, a rep temporarily covering multiple regions), when the
same guarantee is achievable with a single deployment and a server-side
authorization check per call.

**Why not D:** A manual confirmation step adds friction without adding
real security — nothing stops the rep (or a compromised session, or a
model that's been manipulated) from confirming the wrong thing; it's a UX
speed bump, not an access-control mechanism.

**Interviewer's likely follow-up:** *"Where exactly does the region check
happen in your architecture — same process as the tool, or somewhere
else?"* (Strong answer: as close to the data access as possible — ideally
the database query itself is scoped by the authenticated rep's region via
row-level security or an equivalent filter, so even a bug in the tool
wrapper code doesn't create an exposure; defense in depth rather than one
checkpoint.)
</details>

---

### Q3.23 · Tool result relevance vs completeness · [Applied]

A `get_customer_history` tool returns a customer's entire 3-year support
ticket history (200+ tickets) every time it's called, regardless of what
the current conversation is about. What's the problem, and what's the
better design?

- **A.** There's no problem — more information is always better for the
  model
- **B.** Most of that history is irrelevant to any single conversation,
  wastes context budget, and can bury the actually-relevant recent tickets;
  better design: return a small default (e.g. last 5 tickets + a summary),
  with a separate, more targeted tool or parameter (e.g. `filter_by_topic`,
  `date_range`) for the model to request more when the conversation
  actually needs deeper history
- **C.** Cache the 200 tickets client-side so at least the network transfer
  is faster
- **D.** Reduce it to just the single most recent ticket, always

<details>
<summary>Answer</summary>

**B**

**Why B:** This is the same "return lean by default, let the model drill in
on demand" principle from Q3.6 and Q3.17, applied to a different kind of
bulk tool result. Dumping the full 3-year history into every call wastes
tokens on irrelevant content and risks the genuinely relevant recent
context getting lost among 195 unrelated tickets. A default summary plus
an on-demand deeper-fetch path scales far better and keeps the signal
dense.

**Why not A:** "More information is always better" is precisely the
misconception this file keeps testing against — irrelevant volume actively
degrades the model's ability to find and use what matters (see Q3.6,
Q3.17); it isn't a neutral cost, it's a negative one past a point.

**Why not C:** Client-side caching addresses network transfer speed, not
the actual problem, which is that the *model* receives too much irrelevant
content per call regardless of how fast it arrived.

**Why not D:** Only the single most recent ticket is too aggressive in the
other direction — some conversations genuinely need broader history (a
recurring issue pattern, a long-running case); the fix should be adaptive
(default small, expandable on demand), not a fixed minimal cutoff that
loses legitimate cases.

**Interviewer's likely follow-up:** *"How does the model know it needs
more history than the default gives it?"* (The default summary should
include enough signal for the model to judge — e.g. "5 most recent
tickets shown; 195 more exist, use `date_range` or `filter_by_topic` to
retrieve" — so the model has an explicit, actionable cue rather than
silently working with an incomplete picture it doesn't know is
incomplete.)
</details>

---

### Q3.24 · Cost-aware tool orchestration · [Design]

You're building a coding assistant using a dual-model setup: a cheaper,
faster "worker" model handles routine file edits and tool calls, while a
more capable "thinking" model is invoked only for planning and
architecture decisions (a real pattern used in tools like Claude Code via
cc-switch-style configuration). From a tool-design perspective, what has
to be true for this split to actually save cost/latency rather than just
adding complexity?

- **A.** Both models need identical tool access with no differentiation
- **B.** The routing logic that decides which model handles a given step
  needs to be cheap and reliable itself, and the handoff between models
  needs to preserve exactly the context each one needs — otherwise you pay
  coordination overhead (extra calls, redundant context re-transmission)
  that can eat into or exceed the savings from using a cheaper model for
  routine work
- **C.** The worker model should have more tools than the thinking model,
  since it does more of the work
- **D.** This only works if both models are from the same provider

<details>
<summary>Answer</summary>

**B**

**Why B:** A cost-saving architecture like this only pays off if the
orchestration overhead is smaller than the savings. If deciding "which
model handles this step" itself requires an expensive call, or if handing
off between models means re-sending large amounts of context redundantly,
you can end up paying more in coordination than you saved by downgrading
routine work to a cheaper model. The design has to keep the routing
decision lightweight and the context handoff efficient for the split to be
a net win rather than just added system complexity.

**Why not A:** Identical tool access defeats the point of differentiating
the models by role — the worker model doing routine edits typically needs
a narrower, execution-focused toolset, while the thinking model needs
whatever supports planning/architecture reasoning; giving both everything
adds unnecessary decision surface to each.

**Why not C:** Tool count isn't allocated by "who does more work" — it's
allocated by what each model's role actually requires; a thinking model
might need broader read/search tools to gather context for planning even
if it calls tools less frequently overall than the worker model does.

**Why not D:** There's no technical requirement that both models share a
provider — the pattern is about capability/cost tiering, and mixed-provider
setups are viable as long as the orchestration layer can talk to both
APIs; provider-matching isn't what makes the architecture work.

**Interviewer's likely follow-up:** *"How would you measure whether this
split is actually saving money in practice, rather than just assuming it
is?"* (Track cost and latency per task end-to-end — including
orchestration/routing overhead and any redundant context re-transmission —
against a single-model baseline doing the same tasks, not just compare
raw per-token pricing between the two models in isolation.)
</details>

---

### Q3.25 · Diagnosing a tool-use regression · [Applied]

After a prompt update, your team notices the model's tool-selection
accuracy dropped from 96% to 78% on your eval set — it's now frequently
choosing between two tools (`update_record` and `patch_record`) somewhat
randomly. What's the most likely cause to check first?

- **A.** The model provider silently degraded model quality
- **B.** The prompt update likely introduced or exposed genuine ambiguity
  between the two tools — check whether their names/descriptions became
  more similar, whether one was recently added and overlaps in scope with
  the other, or whether the system prompt update removed disambiguating
  context that used to exist
- **C.** This is normal variance and doesn't need investigation
- **D.** Random tool selection means the temperature setting was changed

<details>
<summary>Answer</summary>

**B**

**Why B:** A sharp accuracy drop correlated with a specific change (the
prompt update) and localized to confusion between two *specific* tools
points strongly at a schema/description regression, not a model-quality
issue — especially since `update_record` and `patch_record` are named
similarly enough to be a plausible source of genuine new ambiguity. The
right move is to diff what changed in the prompt/schema around that update
and check whether disambiguating context was lost, rather than assuming
external causes.

**Why not A:** Silent model degradation is possible in principle but is a
much lower-probability explanation than "we just changed the prompt and
now these two similarly-named tools are confused" — Occam's razor points
at your own recent change first, and you should rule that out with a diff
before escalating to a provider-side explanation.

**Why not C:** An 18-point accuracy drop concentrated on two specific
tools, immediately following a change, is a clear regression signal, not
noise — dismissing it as normal variance would mean shipping a real
degradation to production.

**Why not D:** Temperature affects sampling diversity broadly across
generation, but a systematic pattern of confusion between two *specific*
tools (not general randomness across all tool choices) points at a
content/schema ambiguity problem, not a sampling-parameter change — and
the scenario doesn't mention temperature being touched.

**Interviewer's likely follow-up:** *"Your eval set didn't catch this
before you shipped the prompt update — what does that tell you about your
eval coverage?"* (It likely means the eval set didn't have enough cases
that specifically exercise the `update_record` vs `patch_record`
boundary, or that eval wasn't re-run as a gate before the prompt change
shipped — points to needing prompt/schema changes to go through the same
regression-eval gate as model or code changes, covered further in file
06.)
</details>

---

## Explain prompts

### E3.1 · Explain: the tool-use loop end to end

**Prompt:** *"Walk me through, mechanically, what happens from the moment
a user asks a question that requires a tool call to the moment they see
the final answer."*

**Target:** 60–90 seconds spoken. Answer out loud before opening the
rubric.

<details>
<summary>Rubric — score yourself</summary>

**Must hit (1 point each):**
- [ ] States that the model receives the conversation plus the available
      tool schemas as part of the request
- [ ] Explains that the model's "tool call" is structured output (name +
      arguments), not code execution
- [ ] States that the application code executes the actual tool and gets a
      real result
- [ ] Explains the result is appended to the conversation and the model is
      called again with the updated history
- [ ] Notes this can repeat multiple times (or in parallel) before a final
      text answer is produced

**Bonus — signals strength:**
- [ ] Mentions that each round trip pays fresh inference latency, not just
      tool execution time
- [ ] Distinguishes forced vs free tool choice
- [ ] Mentions parallel tool calls as an optimisation when calls are
      independent

**Red flags — deduct:**
- [ ] Describes the model as "running" or "executing" the tool itself
- [ ] Can't explain why a second model call is needed after the tool
      result comes back
- [ ] Treats it as a single opaque step with no round-trip structure

**Score: ___ / 5**

**Model answer:**
So basically, the model never actually runs anything — it's just
predicting text, and one of the things it can predict is a structured
request that says "call this tool with these arguments." My code sees
that, actually goes and does the thing — hits an API, queries a database,
whatever — and gets a real result back. Then I take that result, stick it
into the conversation as a new message, and call the model again with the
whole updated history. The model reads the tool result like it would read
anything else in context, and decides what to do next — maybe it's done
and gives a final answer, maybe it needs another tool call, and the whole
thing repeats. So if a question needs three tool calls, that's three
separate round trips to the model minimum, each one paying both the tool's
own latency and a fresh inference call on top.
</details>

---

### E3.2 · Explain: schema design philosophy

**Prompt:** *"A junior engineer asks you: 'why does it matter so much how
I write the tool description? Isn't the function signature enough?' How do
you answer?"*

**Target:** 60–90 seconds spoken.

<details>
<summary>Rubric — score yourself</summary>

**Must hit (1 point each):**
- [ ] States that the model only knows what's in the schema/description —
      it has no access to your code, comments, or internal docs
- [ ] Explains that ambiguous names/descriptions cause wrong tool
      selection or wrong arguments (not a hypothetical — a real, common
      failure mode)
- [ ] Gives a concrete example of a description improvement (vague name →
      specific scope, or untyped param → constrained/enum type)
- [ ] Distinguishes "the function signature enough for a human reading the
      code" from "enough for a model with zero other context"
- [ ] States that better descriptions are cheaper to fix than adding
      validation/retry logic downstream

**Bonus — signals strength:**
- [ ] Mentions writing descriptions the way you'd write API docs for a
      new engineer who's never seen the codebase
- [ ] Mentions including an example value in the description
- [ ] Notes that this scales — a bad description compounds across every
      call, not just one

**Red flags — deduct:**
- [ ] Treats this as a stylistic nice-to-have rather than a functional
      correctness issue
- [ ] Can't produce a concrete before/after example
- [ ] Suggests the fix is always "add more validation" instead of fixing
      the description first

**Score: ___ / 5**

**Model answer:**
Yeah, so the thing is, a function signature is enough for you, because you
can read the code, see how it's called elsewhere, check the docs, ask a
teammate. The model has none of that — literally the only information it
gets is whatever's in the tool name, the description, and the parameter
descriptions. If your description just says `search(q: string)`, the model
is guessing what kind of search this is, what format `q` should be in, all
of it. And it'll guess wrong in ways that look random but aren't — it's
pattern-matching off whatever weak signal it has. So a good description is
basically like writing a quick API doc for a new hire who's never seen your
codebase and can't ask questions — what does this actually search, what
format does the query need to be in, give an example if it's not obvious.
It's way cheaper to fix a vague description than to build a bunch of
downstream validation and retry logic to catch the bad calls that
ambiguity causes.
</details>

---

### E3.3 · Explain: handling a tool that returns an error

**Prompt:** *"You're building an agent and one of its tools — a payment
lookup — starts intermittently failing. Talk through how you'd design the
error handling, end to end."*

**Target:** 90 seconds spoken.

<details>
<summary>Rubric — score yourself</summary>

**Must hit (1 point each):**
- [ ] States that the error should be returned to the model as a
      structured tool result, not hidden
- [ ] Distinguishes transient (retry-able) errors from permanent ones and
      handles them differently
- [ ] Mentions capping retries at the application layer, not trusting the
      model to self-limit
- [ ] Describes a defined fallback/graceful-degradation behaviour once
      retries are exhausted (tell the user, don't silently continue)
- [ ] Doesn't propose fabricating or defaulting sensitive data (payment
      info) silently to keep the flow going

**Bonus — signals strength:**
- [ ] Distinguishes error handling for a read (lookup) vs a write
      (charge) — the stakes and idempotency concerns differ
- [ ] Mentions logging/observability for the failure independent of what
      the model does with it
- [ ] Notes that surfacing partial success/failure clearly matters more
      than always fully succeeding

**Red flags — deduct:**
- [ ] Proposes silently retrying forever
- [ ] Proposes silently substituting fake/default payment data
- [ ] Has no application-level retry cap, relies entirely on the model

**Score: ___ / 5**

**Model answer:**
First thing, whatever fails, the model needs to actually see that it
failed — I'm not going to swallow the error and pretend nothing happened,
because then the model just confidently tells the user something that
isn't true. So the tool result comes back as, like, an explicit error
object — "payment lookup timed out" — and the model can react to that. Now,
is it worth retrying automatically before even bothering the model? Sure,
for something transient like a timeout, I'd retry once or twice at the
application level, not leave that decision to the model, because I don't
trust it to know when to stop. Cap it — say three tries — and if it still
fails, that's when the model actually gets told "this failed, retries
exhausted," and it should tell the user something honest, like "I'm having
trouble pulling that up right now, can you try again in a bit." The one
thing I'd never do is have it silently fall back to some placeholder value
for payment data — that's the kind of thing where a quiet wrong answer is
way worse than a visible failure.
</details>

---

### E3.4 · Explain: MCP vs a plain function

**Prompt:** *"Your manager asks: 'why are we spending time building an MCP
server for this instead of just writing a function?' Justify the
decision, or push back if you don't think it's justified."*

**Target:** 60–90 seconds spoken.

<details>
<summary>Rubric — score yourself</summary>

**Must hit (1 point each):**
- [ ] States clearly that MCP's value is standardised integration/reuse
      across multiple consumers, not reasoning quality or raw speed
- [ ] Asks/answers whether more than one agent, team, or external client
      actually needs this capability
- [ ] Willing to push back and say "just a function" if there's no real
      reuse case — doesn't default to justifying MCP reflexively
- [ ] Mentions the operational cost of MCP (a server to deploy, secure,
      version) as a real tradeoff, not free
- [ ] Frames the decision as "does the integration boundary earn its
      cost," not "is MCP the modern choice"

**Bonus — signals strength:**
- [ ] Distinguishes local/stdio vs remote/HTTP deployment cost as part of
      the tradeoff
- [ ] Gives a concrete trigger for revisiting the decision later (e.g. a
      second team wants the same capability)
- [ ] Avoids treating this as a binary permanent decision — frames it as
      "right for now, can change"

**Red flags — deduct:**
- [ ] Says MCP is always the right choice because it's the modern standard
- [ ] Can't articulate any operational cost of MCP
- [ ] Confuses MCP with a reasoning/capability improvement for the model

**Score: ___ / 5**

**Model answer:**
Honestly, depends — I'd ask first whether anyone besides this one agent
actually needs this capability. If it's genuinely just us, one codebase,
no other team or client waiting on it, then no, I don't think MCP's
justified yet — you're taking on a server to deploy, version, and secure,
for zero reuse benefit right now. A plain function is simpler, easier to
debug, ships faster. But if I know two other teams are already asking for
the same integration, or we want external tools to be able to plug into
this without needing our codebase, then yeah, that's exactly the case MCP
is for — standardising the integration so it's not five different
bespoke wrappers. It's not really about MCP being modern or not, it's
about whether the reuse or isolation need is real yet. If it's not, I'd
rather ship the simple version now and migrate later when there's an
actual second consumer forcing the question.
</details>

---

### E3.5 · Explain: stdio vs streamable HTTP

**Prompt:** *"When would you choose stdio transport for an MCP server, and
when would you choose streamable HTTP? Give a concrete example of each."*

**Target:** 60 seconds spoken.

<details>
<summary>Rubric — score yourself</summary>

**Must hit (1 point each):**
- [ ] States stdio is for local, single-client, subprocess-based
      deployment
- [ ] States streamable HTTP is for remote, multi-client, network-exposed
      deployment
- [ ] Gives a concrete example for each (e.g. local dev-tool integration
      vs. a shared team/company service)
- [ ] Mentions that HTTP transport brings auth/access-control
      considerations that stdio largely avoids by virtue of being local
- [ ] Doesn't frame the choice as "which is newer/better" but as
      "which matches the deployment topology"

**Bonus — signals strength:**
- [ ] Mentions that a server could plausibly need to move from stdio to
      HTTP later if its usage scope grows
- [ ] Notes stdio has minimal attack surface precisely because it doesn't
      listen on a network port

**Red flags — deduct:**
- [ ] Says one transport is universally better
- [ ] Can't give a concrete example for either
- [ ] Doesn't mention auth/access control as a distinguishing factor

**Score: ___ / 5**

**Model answer:**
It really comes down to who's connecting and from where. If it's a local
tool — like giving a coding assistant access to your own filesystem or git
repo on your machine — stdio's the obvious choice, the client just spawns
the server as a subprocess and talks to it over stdin/stdout, no network
exposure at all, nothing to secure beyond your own machine. But if you're
exposing something like an internal API to your whole engineering team, or
to external clients, you need streamable HTTP, because now you've got
multiple people connecting remotely, and that means you actually have to
think about authentication, who's allowed to connect, rate limiting, all
of that. It's not that one's better, it's that they match completely
different deployment situations, and honestly a server might start as
stdio for one person's local use and get promoted to HTTP once more people
need it.
</details>

---

### E3.6 · Explain: preventing hallucinated tool arguments

**Prompt:** *"A tool call executed with an argument value that was never
mentioned anywhere in the conversation. Walk me through how you'd both
diagnose this and prevent it going forward."*

**Target:** 90 seconds spoken.

<details>
<summary>Rubric — score yourself</summary>

**Must hit (1 point each):**
- [ ] Identifies this as the "hallucinated argument" failure mode, not a
      one-off fluke
- [ ] Explains the mechanism: the model produces a plausible value because
      nothing constrained where the value had to come from
- [ ] Proposes tracing the argument back to whether it should have come
      from a prior tool result and checking if that dependency was enforced
- [ ] Proposes a concrete fix — validating the argument server-side against
      known-real values, or restructuring the tool so the value can only
      come from a prior step's output
- [ ] Distinguishes this from a security exploit — it's a schema/design
      gap, not the model "trying" to do something wrong

**Bonus — signals strength:**
- [ ] Notes this matters most for load-bearing arguments (IDs tied to
      money, access, or irreversible actions)
- [ ] Mentions logging/alerting on arguments that fail a
      known-value-provenance check
- [ ] References the same fix pattern as sequential-dependency tool design
      (data-dependency enforcement)

**Red flags — deduct:**
- [ ] Attributes intent/malice to the model
- [ ] Proposes only prompt-level fixes ("tell it not to do that") with no
      structural fix
- [ ] Can't explain the actual generation mechanism behind it

**Score: ___ / 5**

**Model answer:**
So first thing I'd do is not panic and assume something malicious happened
— this is a known thing, models will produce a plausible-looking value for
a required field even when there's genuinely nothing in context to base it
on, because it's just predicting the most likely next tokens, not
reasoning about whether the value is real. So I'd trace back — where was
this ID supposed to come from? If the answer is "it should've come from an
earlier lookup call," then the actual bug is that nothing enforced that
dependency — the model could call the second tool without ever having
called the first. The real fix is structural: make the second tool require
something — like a token or ID — that literally only exists if the first
tool actually ran and returned it, so the model physically can't skip the
step and invent a value instead. And for anything tied to money or access,
I'd also validate server-side that whatever ID comes in actually matches
something we know is real in this conversation, not just trust the tool
call at face value.
</details>

---

### E3.7 · Explain: bounding tool result size

**Prompt:** *"You're designing a tool that could plausibly return anywhere
from 1 to 10,000 rows depending on the query. How do you design it so it
doesn't blow the context budget?"*

**Target:** 60–90 seconds spoken.

<details>
<summary>Rubric — score yourself</summary>

**Must hit (1 point each):**
- [ ] Proposes a default cap/limit on rows returned rather than returning
      everything matched
- [ ] Proposes summarising or truncating (not just cutting off arbitrarily)
      when the result set is large
- [ ] Gives the model a way to know more exists and how to get more
      (pagination, a count, a narrower filter) rather than silently
      truncating with no signal
- [ ] Mentions this is the same tradeoff as retrieval/chunking in RAG —
      lean by default, expand on demand
- [ ] Considers what "relevant" looks like for a default view (e.g. most
      recent, highest-scoring) rather than an arbitrary first N

**Bonus — signals strength:**
- [ ] Mentions token-cost implications explicitly, not just "it'll be a
      lot of text"
- [ ] Distinguishes a bounded default from a hard error on oversized
      results
- [ ] Notes that a "lost in the middle" attention problem exists even
      within the context window, not just a hard token-limit problem

**Red flags — deduct:**
- [ ] Proposes returning everything and relying on a big context window
- [ ] Proposes an arbitrary silent truncation with no signal to the model
- [ ] Doesn't consider what happens when the model actually needs more
      than the default

**Score: ___ / 5**

**Model answer:**
I wouldn't ever return all 10,000 rows, obviously — even if the context
window technically fits it, that's a huge chunk of mostly-irrelevant
tokens crowding out everything else, and it costs a lot per call. So I'd
cap the default response — say, top 20 or 50 by whatever ordering actually
matters, most recent or highest relevance — and include a count, like
"showing 20 of 4,312 results." That way the model knows there's more and
can decide whether it actually needs to narrow the query or page through
more, instead of silently thinking it saw everything when it only saw a
slice. It's basically the same idea as chunking in RAG — you don't hand
the model the whole corpus, you hand it a relevant slice and give it a way
to ask for more if the first slice isn't enough.
</details>

---

### E3.8 · Explain: choosing between a tool and a resource in MCP

**Prompt:** *"You're building an MCP server that exposes a company's
internal wiki. Would you model 'get a wiki page by title' as a tool, a
resource, or does it depend? Explain your reasoning."*

**Target:** 60–90 seconds spoken.

<details>
<summary>Rubric — score yourself</summary>

**Must hit (1 point each):**
- [ ] Correctly distinguishes tools as model-invoked, dynamic-need actions
      from resources as client-attachable, more static content
- [ ] Says "it depends" and identifies the deciding factor: does the model
      need agency over *when* to fetch it mid-reasoning, or is it known
      upfront
- [ ] Gives a concrete case for treating it as a tool (model doesn't know
      which page it needs until mid-conversation)
- [ ] Gives a concrete case for treating it as a resource (user explicitly
      attaches a specific page to give the model context upfront)
- [ ] Doesn't claim there's one universally correct answer

**Bonus — signals strength:**
- [ ] Notes both could coexist in the same server for different use cases
- [ ] Mentions that resources are more appropriate when a client UI wants
      to let a human browse/select content, not just the model

**Red flags — deduct:**
- [ ] Picks one option confidently with no acknowledgment of the tradeoff
- [ ] Confuses the tool/resource distinction with a permissions distinction
- [ ] Can't give a concrete scenario for either side

**Score: ___ / 5**

**Model answer:**
Honestly it depends on the use case, and I'd probably build both. If
someone's chatting with an assistant and mid-conversation the model
realizes it needs to check the onboarding page to answer a question, that
should be a tool — `get_wiki_page(title)` — because the model doesn't know
ahead of time which page it'll need, it's deciding that dynamically as the
conversation unfolds. But if the client app has a "attach a doc" feature
where a user explicitly picks the onboarding page and drops it into the
chat before asking anything, that's a resource — it's known upfront, the
client is attaching known content, not the model deciding to go fetch
something. So the real question I'd ask is: does the model need agency
over when this gets pulled in, or is it something a human or the app
already knows it wants in context ahead of time. Those are genuinely
different jobs even though the underlying content — a wiki page — is the
same either way.
</details>

---

### E3.9 · Explain: enforcing authorization at the tool layer

**Prompt:** *"Someone on your team says 'we told the model in the system
prompt which records this user is allowed to see, so we're covered.' What
do you say back?"*

**Target:** 60–90 seconds spoken.

<details>
<summary>Rubric — score yourself</summary>

**Must hit (1 point each):**
- [ ] States clearly that a system prompt instruction is not a security
      boundary
- [ ] Explains that the model can be manipulated (prompt injection,
      confusion, or plain error) into ignoring or misapplying that
      instruction
- [ ] States the correct enforcement point: server-side, inside the tool
      implementation, using the authenticated session — not model output
- [ ] Frames this as a Zero Trust principle — never trust model output for
      an authorization decision
- [ ] Says this directly and doesn't hedge or soften it into "it's mostly
      fine"

**Bonus — signals strength:**
- [ ] Gives a concrete failure scenario (injected content causing the
      model to request an out-of-scope record)
- [ ] Mentions defense in depth — enforcing it as close to the data as
      possible, e.g. row-level filtering
- [ ] Doesn't just criticise — offers the concrete fix in the same breath

**Red flags — deduct:**
- [ ] Agrees that prompt-level instruction is sufficient
- [ ] Treats this as a low-priority nice-to-have rather than a real gap
- [ ] Can't explain a concrete way the prompt-only approach fails

**Score: ___ / 5**

**Model answer:**
I'd push back on that pretty directly — a system prompt is not
enforcement, it's a suggestion the model usually follows. It's text, and
text can be overridden — through prompt injection, through the model just
getting confused in a long conversation, or honestly just through model
error. If that's the only thing standing between a user and a record they
shouldn't see, that's not "covered," that's one bad generation away from a
real access-control incident. What actually has to happen is the check
happens in the tool itself, server-side, using the authenticated session —
not anything the model said or was told — so it doesn't matter what the
model requests, the wrong record just can't come back. Ideally that check
lives as close to the data as possible, like row-level filtering in the
query itself, so even if something upstream goes wrong, there's still a
real wall there. This is basically Zero Trust applied to agents — you
don't trust the model's output for anything that gates access, ever.
</details>

---

### E3.10 · Explain: tool sprawl and overlapping scope

**Prompt:** *"Your agent has grown to 30 tools over six months, added
one at a time by different engineers. Tool-selection accuracy has been
quietly declining. How do you approach fixing this?"*

**Target:** 90 seconds spoken.

<details>
<summary>Rubric — score yourself</summary>

**Must hit (1 point each):**
- [ ] Identifies tool sprawl / overlapping scope as the likely root cause,
      not model degradation
- [ ] Proposes auditing for overlapping or ambiguous tool descriptions
      across the full set
- [ ] Proposes consolidating or removing redundant tools rather than only
      adding more disambiguation text
- [ ] Mentions using eval data (which tools get confused with which) to
      prioritise the audit instead of guessing
- [ ] Treats this as an ongoing maintenance problem, not a one-time fix —
      more tools will keep getting added

**Bonus — signals strength:**
- [ ] Mentions establishing a review process/checklist for new tools
      before they're added, to prevent recurrence
- [ ] Distinguishes "too many tools" from "poorly scoped tools" as
      related but separate problems
- [ ] Considers whether some tools could be grouped behind a single
      parameterised tool instead of many narrow ones

**Red flags — deduct:**
- [ ] Jumps to "switch to a better model" without investigating the tool
      set itself
- [ ] Proposes only adding more tools/instructions to fix confusion caused
      by too many tools
- [ ] No mention of using actual failure data to prioritise the fix

**Score: ___ / 5**

**Model answer:**
Thirty tools added incrementally by different people over six months —
honestly, my first guess isn't the model, it's tool sprawl. Different
engineers naming things slightly differently, probably some real overlap
in scope that's built up without anyone noticing because each addition
looked fine in isolation. I'd start by pulling eval or production logs for
which tools the model actually confuses with each other, so I'm not
guessing — that tells you exactly where the ambiguity is instead of
auditing all thirty blind. Then for each overlapping pair, decide: can
these actually be merged into one tool with a parameter, or genuinely need
to stay separate with much tighter, non-overlapping descriptions. I'd
probably cut a handful of tools entirely if they turn out to be redundant.
And longer-term, I'd want some kind of lightweight review before a new
tool gets added — does this overlap with something that already exists —
because otherwise we're back here in another six months with thirty-five
tools instead of thirty.
</details>


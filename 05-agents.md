# 05 · Agents

Agent questions are where interviewers separate candidates who've shipped
something from candidates who've read a blog post. The single most common
trap at your level is reaching for "agent" as the default architecture —
this file weights "when NOT to use an agent" heavily on purpose, because a
candidate who can articulate *restraint* reads as more senior than one who
can only describe capability. The rest covers the ReAct loop, multi-agent
patterns, memory, failure modes, and the operational concerns (cost
bounding, observability, human-in-the-loop) that separate a toy agent from
one you'd trust in production.

---

## Multiple choice

### Q5.1 · Workflow vs agent · [Recall]

A colleague says "we built an agent that processes refund requests." Looking
at the code, you see: extract fields from the request → call a fixed
validation function → if valid, call a fixed payment-reversal API → send a
fixed confirmation template. There's an LLM call at the extraction step
only. What is this, precisely?

- **A.** A single-agent system with one tool
- **B.** A workflow — an LLM call embedded in a fixed, predetermined sequence of steps
- **C.** A multi-agent system, since it involves multiple stages
- **D.** A ReAct agent, since it reasons about the extracted fields before acting

<details>
<summary>Answer</summary>

**B**

**Why B:** The defining feature of an agent is that the *model* decides what
happens next — which tool to call, whether to loop, when to stop. Here the
control flow is entirely fixed by the code; the LLM is doing one bounded
extraction task inside a pipeline it doesn't control. That's a workflow, not
an agent.

**Why not A:** "Agent" implies the model is choosing the path, not just
occupying one step in someone else's fixed path.

**Why not C:** Multiple stages don't make something multi-agent — multi-agent
means multiple LLM-driven decision-makers coordinating, not multiple
processing steps.

**Why not D:** ReAct requires an observe-reason-act loop where the model
chooses actions based on intermediate results. A single extraction call with
no loop and no choice of next action isn't reasoning about what to do next —
it's doing the one thing it was asked to do.

**Interviewer's likely follow-up:** *"Does it matter that this is 'just' a
workflow? Isn't calling it an agent basically harmless marketing?"* (Answer:
it matters for engineering decisions — a workflow is easy to test
deterministically, cheap to run, and easy to reason about failure modes for;
calling it an agent invites people to add speculative flexibility, retries-
via-reasoning, and non-deterministic branching it doesn't need, which adds
risk with no corresponding benefit.)

</details>

---

### Q5.2 · When not to use an agent, angle 1: bounded classification · [Applied]

You're asked to build a system that reads incoming support tickets and
assigns them to one of six fixed teams (Billing, Technical, Onboarding,
Security, Sales, Other). Someone on the team proposes an agent that can
"investigate the ticket, look up the customer's account, and decide the
best team." What's the strongest argument against the agent framing here?

- **A.** LLMs are bad at classification tasks
- **B.** Routing to one of six fixed categories is a bounded classification problem — a single structured-output call, possibly with a lookup tool available, solves it without needing the model to choose its own path
- **C.** Agents can't call lookup tools
- **D.** The six categories should be reduced to fewer options first

<details>
<summary>Answer</summary>

**B**

**Why B:** The output space is small and fixed, and the "investigation" —
even if it needs an account lookup — is one predictable step, not an
open-ended sequence the model needs to plan through. A single call (structured
output, with a tool available if the lookup is genuinely needed) gets you the
same result with far less latency, cost variance, and failure surface than
letting the model decide how many steps to take and in what order.

**Why not A:** LLMs are actually good at this kind of classification —
that's not the problem. The problem is architectural overkill, not
capability.

**Why not C:** Agents absolutely can call lookup tools — that's not the
distinguishing issue. A single structured-output call can *also* have a tool
available to it without becoming an "agent" in the sense of open-ended
looping.

**Why not D:** Reducing categories doesn't address the actual question of
whether this needs agentic control flow — it's a separate, orthogonal
design choice.

**Interviewer's likely follow-up:** *"What would make you change your mind
and actually build an agent for this?"* (Answer: if routing correctness
depends on a variable-length investigation — e.g., checking multiple
systems in an order that depends on what's found in the first one — then
the path is genuinely unknown ahead of time, and that's when agentic
control flow starts paying for itself.)

</details>

---

### Q5.3 · When not to use an agent, angle 2: fixed pipeline mistaken for agentic · [Applied]

A document-processing system does: OCR the PDF → LLM call to extract five
named fields → validate fields against a schema → if validation fails, LLM
call to re-extract with the validation errors appended to the prompt → write
to database. The team calls this "our extraction agent" and wants to give it
tools to browse the filesystem in case the schema-based approach
fails outright. What should you push back on first?

- **A.** The filesystem-browsing tool, because that expands scope without addressing why extraction is failing
- **B.** The OCR step, because OCR should never be the first step
- **C.** Nothing — adding a filesystem tool is a natural next step for any extraction agent
- **D.** The retry-with-errors step, because retries should never use the same model

<details>
<summary>Answer</summary>

**A**

**Why A:** This is a workflow with a bounded, sensible retry loop (re-extract
with validation errors) — that's a good pattern, not the problem. Giving it
open-ended filesystem access to "fail more gracefully" is scope creep: it
turns a well-understood, testable pipeline into something with an
unpredictable action space, without first diagnosing *why* extraction is
failing on the cases it's failing on. Fix the actual failure mode before
expanding what the system is allowed to do.

**Why not B:** OCR-first is a completely standard and appropriate first step
for PDF field extraction — nothing wrong with it.

**Why not C:** This is the trap the question is testing — "it's called an
agent, so agent-like additions feel natural" is exactly the reasoning that
leads to scope creep without a diagnosed need.

**Why not D:** Retrying with the same model plus the specific validation
errors appended is a legitimate, common pattern — it gives the model
concrete feedback on what to fix, which is more useful than switching
models.

**Interviewer's likely follow-up:** *"So what would you actually do about
the failing extractions?"* (Answer: pull the failing examples, categorize
the failure modes — bad OCR text, ambiguous field, format the model doesn't
recognize — and fix each category with a targeted change: better OCR
preprocessing, a clarified schema description, or few-shot examples, rather
than a generic capability expansion.)

</details>

---

### Q5.4 · When not to use an agent, angle 3: cost of agentic control flow · [Design]

You're deciding between a fixed 3-step workflow and an agent for a data
enrichment task where, in practice, 95% of records follow the exact same
three-step path and 5% need a variable extra lookup. Your team lead wants to
default to the agent "since it handles both cases." What's the strongest
case for the workflow, with an escape hatch for the 5%, instead?

- **A.** Agents cannot handle conditional logic at all
- **B.** For the 95% common case, an agent adds non-determinism, higher latency (multiple model round-trips vs. one path), and higher cost variance, for a benefit that only 5% of traffic actually needs — a workflow with a conditional branch to a small agentic sub-step for the 5% gets you both determinism where it's cheap and flexibility where it's needed
- **C.** Workflows are always cheaper than agents regardless of the task
- **D.** The 5% edge case should simply be dropped since it's a minority of traffic

<details>
<summary>Answer</summary>

**B**

**Why B:** This is the general principle in miniature: match the control
flow to how much of the problem actually varies. Paying the agent tax (extra
model calls to decide what to do next, harder-to-predict latency and cost,
more surface area for things to go wrong) on 95% of traffic that doesn't
need it is a bad trade when you can isolate the variability into a small
branch and only pay that cost where it's earned.

**Why not A:** Agents can absolutely do conditional logic — that's not a
real limitation, and claiming it is would be a red flag in an interview.

**Why not C:** Not universally true — for genuinely unpredictable-path tasks,
an agent can be cheaper than trying to hand-code every branch. The
comparison has to be made per-task, based on how much of the traffic is
actually variable.

**Why not D:** Dropping 5% of traffic to avoid an engineering decision is not
a real answer an interviewer wants to hear — it dodges the tradeoff instead
of making it.

**Interviewer's likely follow-up:** *"How would you decide where to draw the
line between 'workflow with a branch' and 'just make the whole thing an
agent'?"* (Answer: roughly, if the number of possible branches is small and
enumerable, hand-code them; once the number of paths or their ordering is
combinatorial or genuinely unknown ahead of time, the cost of enumerating
them exceeds the cost of letting the model decide.)

</details>

---

### Q5.5 · When not to use an agent, angle 4: "uses an LLM" ≠ "is an agent" · [Recall]

Which statement most accurately describes the difference between "a system
that uses an LLM" and "an agent"?

- **A.** They're the same thing — any system with an LLM call in it is an agent
- **B.** An agent additionally requires the LLM to control what happens next — choosing actions, deciding when to stop, potentially looping — not just producing output that a fixed pipeline consumes
- **C.** An agent is any system that uses more than one LLM call
- **D.** An agent is defined by having access to external tools

<details>
<summary>Answer</summary>

**B**

**Why B:** The distinguishing feature isn't the presence of an LLM, the
number of calls, or tool access — it's *who controls the control flow*. If
the code decides what happens next regardless of what the model said, it's
not agentic, no matter how many LLM calls are in the pipeline.

**Why not A:** This conflation is exactly the misconception the question is
testing — it's extremely common and worth explicitly correcting in an
interview, since claiming it yourself is a red flag.

**Why not C:** Call count is irrelevant to the definition — a multi-step
fixed pipeline with five sequential LLM calls is still a workflow if the
code, not the model, decides the sequence.

**Why not D:** Tool access is necessary for many agents but isn't the
defining feature — you can give a single non-agentic call a tool (e.g., one
structured-output call that may invoke one lookup) without the system
becoming agentic in the control-flow sense.

**Interviewer's likely follow-up:** *"Where's the gray area, then — is
there a case where this distinction gets genuinely fuzzy?"* (Answer: yes —
a "workflow with one conditional LLM-driven branch" sits on a spectrum
between the two; the useful question isn't "is this technically an agent"
but "how much of the control flow is model-determined, and does that
amount of flexibility earn its cost.")

</details>

---

### Q5.6 · When not to use an agent, angle 5: structured output suffices · [Applied]

A team building a meeting-notes tool wants an agent that "decides which
action items to extract, who to assign them to based on the transcript, and
formats them for Slack." What's the most likely correct architecture?

- **A.** A single structured-output call: pass the transcript, ask for a list of {action_item, assignee, due_date} objects in a fixed schema — no agent needed, since this is one well-defined extraction/formatting task with no genuinely unknown path
- **B.** A ReAct agent that can call a Slack API tool, a calendar tool, and a database tool in whatever order it decides
- **C.** A multi-agent system with a separate "extraction agent" and "formatting agent"
- **D.** An agent with unlimited tool access so it can look up any context it might need

<details>
<summary>Answer</summary>

**A**

**Why A:** "Extract structured items from text" is the canonical
structured-output use case — there's no branching decision to make about
*what to do next*, just a well-specified transformation. Formatting for
Slack is a deterministic post-processing step on the structured output, not
something requiring model-driven control flow.

**Why not B:** Giving it open-ended tool choice for a task with a fixed,
known shape adds latency, cost, and failure modes (wrong tool order,
redundant calls) with zero corresponding benefit — there's nothing to
"decide" here that a fixed pipeline can't handle.

**Why not C:** Splitting one well-defined extraction task across two "agents"
adds coordination overhead (state handoff, consistency between the two)
for a task that doesn't need decomposition — classic premature multi-agent
architecture.

**Why not D:** Unlimited tool access to a task that's fundamentally a
transcript-to-structured-data mapping just increases the chances of the
model doing something unnecessary or wrong; it doesn't add real capability
here.

**Interviewer's likely follow-up:** *"What if the assignee needs to be
looked up by matching a name mentioned in the transcript against a company
directory that isn't in the transcript?"* (Answer: that's a legitimate
reason to add exactly one lookup tool to the structured-output call — still
not agentic, since there's no branching decision, just one deterministic
enrichment step before or during the extraction call.)

</details>

---

### Q5.7 · ReAct loop · [Recall]

What does the ReAct pattern actually add on top of a plain "give the model
tools and let it call them" setup?

- **A.** ReAct requires the model to write out an explicit reasoning step before each action, interleaving Thought → Action → Observation, rather than just emitting a tool call directly
- **B.** ReAct is a different model architecture optimized for tool calling
- **C.** ReAct means the model can only take one action per conversation
- **D.** ReAct is a framework-specific term with no meaningful difference from standard tool use

<details>
<summary>Answer</summary>

**A**

**Why A:** ReAct (Reason + Act) is specifically about interleaving explicit
reasoning traces with actions and observations — the model articulates why
it's about to do something, does it, observes the result, and reasons again
before the next action. This is a prompting/loop pattern, not a different
underlying capability.

**Why not B:** It's not an architecture — it's a pattern for structuring the
prompt/loop around any tool-calling-capable model.

**Why not C:** ReAct is explicitly iterative — it's designed for
multi-step loops, not single actions.

**Why not D:** There is a real, if sometimes overstated, difference:
explicit reasoning traces tend to improve reliability on multi-step tasks
by giving the model (and you, when debugging) a visible record of *why* it
chose an action, versus a bare sequence of tool calls with no stated intent.

**Interviewer's likely follow-up:** *"When does the extra reasoning step
stop being worth it?"* (Answer: for very simple, low-branching tool
sequences, the reasoning tokens add latency and cost without changing the
outcome — ReAct earns its keep on tasks where the next-best-action genuinely
depends on reasoning about what was just observed.)

</details>

---

### Q5.8 · Task decomposition · [Applied]

An agent tasked with "prepare a competitive analysis of three companies"
immediately starts searching for information on Company A without first
producing any kind of plan. Two hours and forty tool calls later, it hasn't
started on Companies B or C and has re-searched overlapping queries for
Company A three times. What's the most likely root cause?

- **A.** The model isn't capable of running more than one search
- **B.** The agent has no upfront decomposition step — it's improvising the whole task step-by-step instead of first producing a plan (three companies, N dimensions each) that it could execute against and check off
- **C.** The tool descriptions are unclear
- **D.** The model's context window is too small for this task

<details>
<summary>Answer</summary>

**B**

**Why B:** Without an explicit planning step, an agent tends to greedily
pursue whatever seems locally useful next, which produces exactly this
pattern — deep, redundant investigation of the first sub-task with no
tracked global structure. An explicit plan (even a simple checklist the
agent maintains and updates) gives it something to execute against and
notice when it's done with one part.

**Why not A:** Nothing here suggests a capability limit on searching — the
problem is the *lack of structure*, not an inability to search.

**Why not C:** Tool clarity would produce wrong tool calls or errors, not
this specific pattern of narrow, repetitive, un-decomposed effort.

**Why not D:** A too-small context window would show up as forgotten early
instructions or truncated history, not as a failure to ever plan the task
in the first place — this is a planning-step absence, not a memory
constraint.

**Interviewer's likely follow-up:** *"How would you actually fix this in
the prompt or architecture?"* (Answer: force an explicit planning step
before any tool calls — e.g., require the agent to output a structured task
list first, then execute against it, checking items off and re-reading the
list periodically so it doesn't drift; some architectures make this a
distinct planning tool call the agent must complete before other tools
unlock.)

</details>

---

### Q5.9 · Single-agent vs multi-agent · [Applied]

A team proposes a "research agent," a "writing agent," and a "editing
agent" that pass a document between them for a report-generation task.
Output quality is worse than a single well-prompted agent doing the same
task end to end, and it costs more. What's the most likely explanation?

- **A.** Multi-agent systems are always worse than single-agent systems
- **B.** The task decomposition doesn't match real boundaries — research, writing, and editing aren't cleanly separable without shared context, so each "agent" is operating with less information than a single agent would have, while the system pays coordination overhead (handoffs, potential information loss between agents) for no compensating benefit
- **C.** The models used for each subagent must have been different sizes
- **D.** Three agents should always outperform one, so this is a bug in the framework being used

<details>
<summary>Answer</summary>

**B**

**Why B:** Multi-agent decomposition pays off when subtasks have genuinely
separable context needs (e.g., isolating a large tool-heavy investigation
from a final synthesis step so the synthesis isn't drowned in raw tool
output). Report writing, research, and editing are tightly coupled — the
writer needs the research's nuance, the editor needs to know what was
prioritized and why — so splitting them loses information at each handoff
without buying real isolation benefit. This is the textbook case of
"multi-agent is usually premature."

**Why not A:** Not universally true — multi-agent has legitimate use cases,
this question is about a *specific* bad decomposition, not multi-agent
architecture in general.

**Why not C:** Model size isn't given as a variable here and isn't the
governing factor in this failure — the issue is architectural, not about
model capability.

**Why not D:** This assumes more agents inherently means more capability,
which is the misconception the question is testing directly.

**Interviewer's likely follow-up:** *"When would this exact
research/write/edit split actually make sense?"* (Answer: if research
involves so much raw tool output that it would blow the writer's context
budget, isolating it into a subagent that returns a condensed synthesis —
not raw output — can genuinely help; the key is that the subagent's *output*
is a distilled artifact, not that the task was split for its own sake.)

</details>

---

### Q5.10 · Multi-agent coordination overhead · [Design]

You're designing a system where multiple agents need to negotiate a shared
plan (e.g., a "scheduling agent" and a "budget agent" that need to agree on
a trip itinerary). What's the primary engineering risk this introduces
that a single agent wouldn't have?

- **A.** The individual agents will be slower at their sub-tasks
- **B.** Coordination failure modes — inconsistent shared state between agents, deadlock/ping-pong exchanges where neither agent has authority to finalize a decision, and difficulty tracing *why* a final answer emerged when it came from an inter-agent negotiation rather than one reasoning trace
- **C.** Multi-agent systems cannot use tools
- **D.** Each agent will require its own separate model provider

<details>
<summary>Answer</summary>

**B**

**Why B:** Negotiation between autonomous decision-makers is exactly where
multi-agent systems introduce failure modes that don't exist in a single
reasoning trace: state can diverge, there can be no clear resolution
authority causing back-and-forth loops, and debugging becomes harder because
the "why" of a decision is now split across two independent traces instead
of one.

**Why not A:** Individual sub-tasks aren't inherently slower — the risk is
in the *coordination*, not each agent's local performance.

**Why not C:** Nothing about multi-agent architecture restricts tool
access — this isn't a real constraint.

**Why not D:** There's no requirement that different agents use different
providers — this is an implementation detail, not an inherent risk of
multi-agent coordination.

**Interviewer's likely follow-up:** *"How would you mitigate the deadlock
risk specifically?"* (Answer: give one agent (or a lightweight orchestrator)
final authority to resolve disagreements after a bounded number of
exchange rounds, rather than letting two peers negotiate indefinitely —
plus a hard cap on negotiation rounds as a safety net.)

</details>

---

### Q5.11 · Subagent context isolation · [Recall]

Explaining a dual-model orchestration setup (a "thinking" model that plans
and a separate "worker" model that executes each step) to an interviewer,
what's the strongest technical justification for *why* this beats a single
model doing both?

- **A.** Two models are always more accurate than one
- **B.** It isolates the planning context (which needs to stay clean, high-level, and coherent across the whole task) from the execution context (which gets filled with verbose tool output, file contents, and step-level noise) — the planner never sees the execution noise, so its judgment isn't degraded by clutter, and the worker doesn't need to "remember" the full task history, only its current instruction
- **C.** It's required because no single model can call tools and reason at the same time
- **D.** It halves the number of tokens used overall

<details>
<summary>Answer</summary>

**B**

**Why B:** This is the real justification for planner/worker or
orchestrator/subagent splits generally: context pollution. A single model
doing both planning and step-by-step execution accumulates tool output,
error messages, and low-level noise in the same context that's supposed to
hold the high-level plan — degrading its ability to track the plan
coherently over a long task. Splitting the roles keeps each context
appropriately scoped.

**Why not A:** Not a real guarantee — two models can also compound errors or
add coordination overhead; the benefit here is architectural (context
hygiene), not an automatic accuracy boost.

**Why not C:** False — a single model can plan and call tools in the same
loop; this isn't a hard capability limitation.

**Why not D:** It can *increase* total token usage (you're running two
model contexts instead of one) even though it may reduce cost per
step-model if the worker uses a cheaper model — the point is context
isolation, not token savings.

**Interviewer's likely follow-up:** *"What's the downside of this split?"*
(Answer: added latency from the extra round-trip between planner and
worker, added complexity in defining the hand-off contract between them,
and a new failure mode — the worker executing a step correctly but in a way
the planner didn't anticipate, requiring the planner to re-check outcomes
rather than assuming its plan executed as imagined.)

</details>

---

### Q5.12 · When to spawn a subagent vs inline tool call · [Design]

You're designing an agent that occasionally needs to do a deep, multi-step
investigation (e.g., "figure out why this deploy failed" involving reading
several log files and correlating timestamps) as part of a broader task.
Should that investigation be a subagent call or a sequence of inline tool
calls in the main agent's loop?

- **A.** Always inline — subagents add unnecessary complexity
- **B.** Subagent, if the investigation would otherwise dump a large volume of raw intermediate output (log excerpts, grep results) into the main agent's context that it doesn't need once it has the answer — the subagent absorbs that noise and returns a condensed finding
- **C.** Always subagent — any multi-step task should be delegated
- **D.** It doesn't matter, since context budget isn't a design consideration for agents

<details>
<summary>Answer</summary>

**B**

**Why B:** This is the general rule for subagent boundaries: delegate when
the sub-task's process generates far more intermediate volume than its
result, so the main loop's context stays focused on the task-level plan
rather than being flooded with grep output and log excerpts it will never
need again once the root cause is identified.

**Why not A:** Blanket avoidance ignores the real context-budget benefit
subagents provide for exactly this kind of noisy investigation.

**Why not C:** Blanket delegation adds coordination overhead and latency
for sub-tasks that are short enough not to need isolation — not every
multi-step task benefits from being pulled out.

**Why not D:** Context budget is one of the central design considerations
for agents — ignoring it is precisely how agents degrade over long
sessions (see Q5.17–Q5.20 on failure modes).

**Interviewer's likely follow-up:** *"How does the subagent communicate its
finding back without just re-including all the raw log data anyway?"*
(Answer: the subagent's return contract should be a structured or
prose summary — "root cause: X, evidence: Y, Z" — not a dump of every log
line it read; enforcing that contract is what actually delivers the context
savings.)

</details>

---

### Q5.13 · Context management: compaction · [Applied]

A long-running agent session (many tool calls over an hour) starts
producing worse decisions in the second half of the session than the
first, even though nothing about the task changed. What's the most likely
cause, and what's the standard mitigation?

- **A.** The model has gotten "tired" — restart with a fresh instance
- **B.** The context window has filled with accumulated tool output and intermediate reasoning, degrading the model's ability to attend to the original task instructions buried early in the transcript — mitigate with periodic compaction: summarizing or pruning older, no-longer-relevant context while preserving the task goal and key findings
- **C.** The temperature setting must have drifted upward over the session
- **D.** This is expected and there's no useful mitigation short of ending the session

<details>
<summary>Answer</summary>

**B**

**Why B:** This is a well-documented failure pattern — as context fills with
verbose intermediate content, models can lose track of instructions given
early in the conversation ("lost in the middle" effects), and irrelevant
accumulated content crowds out what's actually still useful. Compaction —
periodically summarizing what's been done and pruning raw tool output that's
served its purpose — keeps the working context focused.

**Why not A:** Models are stateless per call — there's no fatigue mechanism;
attributing this to "tiredness" misdiagnoses a context-management problem as
something else entirely.

**Why not C:** Temperature is a fixed parameter per call, not something that
drifts during a session — this isn't a real mechanism.

**Why not D:** There is a well-known, standard mitigation (compaction/context
pruning) — claiming there's none would be a notable gap in an interview
answer.

**Interviewer's likely follow-up:** *"What's the risk of compacting too
aggressively?"* (Answer: you can prune information the agent turns out to
still need, causing it to repeat work or make a decision without context
it should have had — compaction strategy needs to preserve task-critical
state, not just shrink token count for its own sake.)

</details>

---

### Q5.14 · Forgotten early instructions · [Applied]

Partway through a long agent session, the agent violates a constraint that
was stated clearly in its very first instruction (e.g., "never modify files
outside `/src`"). What's the most robust fix, beyond just "add compaction"?

- **A.** Increase the temperature so the model pays more attention
- **B.** Re-inject critical constraints periodically (e.g., in a system reminder or at the start of each new tool-call cycle) rather than relying on them surviving from the first message alone across a long, noisy context
- **C.** Switch to a model with a larger context window and stop worrying about it
- **D.** Constraints stated once should always hold; if they don't, the model is unreliable and no architectural fix applies

<details>
<summary>Answer</summary>

**B**

**Why B:** Rather than trusting a single early statement to survive
arbitrarily long, noisy contexts, robust agent systems re-assert critical
constraints at regular intervals or key decision points — this is standard
practice precisely because "stated once at the start" is known to degrade
in reliability as context grows.

**Why not A:** Temperature affects sampling randomness in output generation,
not the model's attention to earlier context — this isn't a mechanism that
would fix instruction adherence.

**Why not C:** A larger context window helps but doesn't solve the
underlying attention degradation — the constraint can still get lost amid
volume even if it technically "fits."

**Why not D:** This is defeatist and factually wrong — the reinjection
pattern is a known, practical mitigation, not something to shrug off as
unfixable.

**Interviewer's likely follow-up:** *"Isn't reinjecting the same constraint
repeatedly wasteful of tokens?"* (Answer: yes, there's a real cost tradeoff
— the mitigation is to reinject concisely and only for constraints where a
violation is costly, not to repeat the entire original prompt every turn.)

</details>

---

### Q5.15 · Memory: what's actually retrieval · [Recall]

A product describes its agent as having "long-term memory" that lets it
recall facts from conversations weeks ago. Under the hood, what is this
almost always actually implemented as?

- **A.** The model's weights are being fine-tuned after every conversation
- **B.** Past conversation content is stored externally (e.g., in a database or vector store) and relevant pieces are retrieved and injected into context at the start of a new session — the model itself has no persistent state between calls
- **C.** The context window has been permanently expanded to include all prior sessions
- **D.** The model has an internal cache that persists between API calls

<details>
<summary>Answer</summary>

**B**

**Why B:** LLM APIs are stateless between calls — there is no mechanism for
the model itself to "remember" anything from a prior session unless it's
re-supplied in the prompt. "Long-term memory" products are, functionally, a
RAG system over conversation history: store, retrieve relevant pieces,
inject at the top of the new context.

**Why not A:** Fine-tuning after every conversation is not how these
products work in practice — it's far too slow, expensive, and risky
(catastrophic forgetting, per-user model management) compared to retrieval.

**Why not C:** Context windows have fixed limits per model/API tier — they
aren't dynamically expanded per user, and even if they were, unbounded
inclusion of all history would blow past any practical limit quickly.

**Why not D:** There's no persistent per-conversation cache at the model
level in standard API usage — each call is independent and stateless.

**Interviewer's likely follow-up:** *"So is 'agent memory' just RAG with
extra steps?"* (Answer: largely yes, for long-term memory — the interesting
engineering is in what gets stored, how it's summarized/structured for
retrieval, and when it's surfaced, not in some fundamentally different
mechanism from retrieval-augmented generation.)

</details>

---

### Q5.16 · Memory scenario · [Applied]

A user reports that an agent "remembered" their preference for concise
answers across two completely separate sessions three days apart, with no
explicit memory feature enabled in the product. What's the most likely
actual explanation?

- **A.** The model inherently remembers all users it has ever talked to
- **B.** The preference wasn't remembered by the model at all — it's more likely the product silently re-injects a saved user-preference profile into the system prompt on every session, which the user is misattributing to the model "remembering"
- **C.** This is impossible and the user is mistaken
- **D.** The model was fine-tuned specifically on this user's prior conversations

<details>
<summary>Answer</summary>

**B**

**Why B:** Given the API is stateless, any cross-session personalization
that "no explicit memory feature" doesn't quite capture is most plausibly a
product-level mechanism the user isn't aware of — a stored preference
profile injected into every new session's system prompt — rather than any
property of the model itself.

**Why not A:** Models don't have persistent per-user memory built in — this
would require an explicit storage-and-retrieval mechanism, which is exactly
what the answer describes, just one the user isn't aware exists.

**Why not C:** The behavior is real and explicable, not evidence of user
error — dismissing it that way skips the actual engineering question.

**Why not D:** Per-user fine-tuning after every session is not standard
practice for this kind of lightweight personalization — retrieval/injection
is dramatically cheaper and faster to implement.

**Interviewer's likely follow-up:** *"How would you verify your
explanation instead of just asserting it?"* (Answer: check the system
prompt/logs sent on the second session for injected preference content, or
test by clearing any product-level user data and seeing if the behavior
disappears — verify the mechanism rather than guessing.)

</details>

---

### Q5.17 · Failure mode: infinite loop · [Applied]

An agent gets stuck alternating between two tool calls — checking a
condition, taking an action that should satisfy it, checking again, finding
it still unsatisfied, repeating — for 80 iterations before someone notices.
What's the most direct architectural safeguard against this class of
failure, distinct from just fixing this one bug?

- **A.** Prompt the model more strongly to "not get stuck in loops"
- **B.** A hard step limit (max iterations) enforced by the surrounding code, independent of the model's own judgment, that terminates the session and surfaces a "not resolved after N steps" state rather than trusting the model to notice it's looping
- **C.** Switch to a larger model, which won't get stuck in loops
- **D.** Remove the tool that's involved in the loop

<details>
<summary>Answer</summary>

**B**

**Why B:** Loop failures happen precisely because the model's own judgment
about whether it's making progress is what's failing — you can't rely on
the same judgment to also detect the failure. A code-enforced step cap is a
safety net that doesn't depend on the model recognizing its own
malfunction, which is why it's the standard mitigation regardless of what
caused any specific loop.

**Why not A:** Prompting is a soft mitigation that can reduce frequency but
provides no guarantee — it's not a safeguard, it's a hope.

**Why not C:** Larger models can still loop; model capability doesn't
eliminate this failure class, it just changes the odds.

**Why not D:** Removing the tool fixes this one instance but isn't a general
safeguard against the failure *class* — the next loop will just involve a
different tool.

**Interviewer's likely follow-up:** *"What do you do once the step limit is
hit — just fail silently?"* (Answer: no — surface it as an explicit
"exceeded step budget, did not complete" state, ideally with the partial
trace, so a human or a fallback path can take over, rather than either
silently failing or letting it run unbounded.)

</details>

---

### Q5.18 · Failure mode: cost runaway · [Applied]

You deploy an agent that occasionally, on certain inputs, spawns many more
tool calls than expected — one incident cost 40x the typical session. What
combination of safeguards addresses this, beyond the step-limit from the
previous question?

- **A.** A step limit alone is sufficient, since step count and cost are the same thing
- **B.** A per-session cost/token budget tracked independently of step count (since steps can vary wildly in token cost — one tool call can return 50 tokens, another 50,000), with a hard cutoff, plus alerting when a session approaches the budget so you can catch systemic issues, not just single incidents
- **C.** Rely on the model to self-report when it's using too many resources
- **D.** Rate-limit the API key and consider the problem solved

<details>
<summary>Answer</summary>

**B**

**Why B:** Step count and cost aren't the same thing — a small number of
steps can still be expensive if individual tool results or reasoning are
token-heavy. A real cost safeguard tracks actual token/dollar spend per
session with its own cutoff, and alerting matters because a single 40x
incident should also prompt investigation into whether it's a one-off or a
systemic issue with a particular input class.

**Why not A:** This conflates two different resources — a session could hit
a low step count but still be very expensive per step, so step limits alone
don't bound cost.

**Why not C:** Self-reporting relies on the same model whose judgment may be
compromised in the failure scenario — not a reliable external safeguard.

**Why not D:** Rate-limiting the API key bounds request *frequency*, not
cost per session — a single session can still blow through budget within
the rate limit.

**Interviewer's likely follow-up:** *"How would you investigate the root
cause of the 40x session after the fact?"* (Answer: trace/log review of
that specific session — what input triggered it, where the token usage
concentrated, whether it was a loop, a single tool returning excessive
output, or repeated large context re-transmission — and check whether other
sessions with similar inputs show the same pattern.)

</details>

---

### Q5.19 · Failure mode: tool misuse · [Recall]

An agent has access to both a `search_customer_by_email` tool and a
`search_customer_by_partial_name` tool. Given a request containing a full
email address, it calls the partial-name search with the email string as
the name argument, gets no results, and reports "customer not found" —
when the correct tool would have found them immediately. What's the most
likely root cause, and the fix?

- **A.** The model is fundamentally incapable of choosing between similar tools
- **B.** The tool descriptions likely don't clearly differentiate when each applies (e.g., don't specify that one expects an exact email and the other a fuzzy name match) — fix by making tool descriptions explicit about expected input format and when to prefer one over the other, not just what each tool technically does
- **C.** There should only ever be one search tool to avoid this
- **D.** This is unfixable without switching to a different model provider

<details>
<summary>Answer</summary>

**B**

**Why B:** Tool misuse like this is almost always a description problem, not
a capability problem — if the descriptions don't clarify input format
expectations and disambiguate overlapping tools, the model has to guess,
and guesses are sometimes wrong. Explicit guidance ("use this when you have
a full email address; use the other when you only have a partial name") is
exactly the kind of thing that eliminates this class of error.

**Why not A:** Models handle tool disambiguation well *when the schema and
descriptions make the distinction clear* — this isn't a hard capability
limit, it's a specification gap.

**Why not C:** Collapsing to one tool loses real functionality (you do
sometimes only have a partial name) — the fix is better differentiation, not
elimination.

**Why not D:** This is a prompt/schema engineering problem solvable within
any capable model — not a model-provider limitation.

**Interviewer's likely follow-up:** *"How would you catch this kind of
misuse before it reaches production?"* (Answer: eval cases specifically
targeting tool-selection ambiguity — inputs that could plausibly trigger
either tool — checked against expected tool choice, not just final output
correctness.)

</details>

---

### Q5.20 · Failure mode: silent failure · [Applied]

An agent tasked with "update all customer records with the new address
format" reports "task completed successfully," but a spot check shows only
12 of 200 records were actually updated — the rest were silently skipped
because a tool call failed partway and the agent moved on without flagging
it. What's the core design flaw?

- **A.** The agent should have used a bigger model
- **B.** The agent (or the surrounding harness) doesn't distinguish between "step succeeded" and "step attempted" — tool failures are being swallowed rather than surfaced, and there's no verification step confirming the claimed outcome actually matches reality before reporting success
- **C.** 200 records is too many for an agent to handle in one session
- **D.** The customer database schema must be the problem

<details>
<summary>Answer</summary>

**B**

**Why B:** This is a textbook silent-failure pattern: an error occurred, the
agent didn't propagate or escalate it (or the harness swallowed it), and
nothing verified the end state against the claimed outcome. The fix is
two-fold — tool errors should be explicit and visible to the model (and
logged), and any "completed" claim on a bulk task should be checked against
actual verification (e.g., count updated vs. expected) rather than trusted
at face value.

**Why not A:** Model size doesn't address the structural gap — a bigger
model given the same silent tool-failure signal and no verification step
would very plausibly make the same mistake.

**Why not C:** Volume isn't inherently the issue — a well-designed batch
process with proper error handling and verification can handle 200 or
200,000 records; the failure here is about error visibility, not scale.

**Why not D:** Nothing points to a schema problem — the described failure
is about error handling and reporting, not data structure.

**Interviewer's likely follow-up:** *"How would you design the verification
step concretely?"* (Answer: after the batch runs, query the actual updated
count/state from the source of truth and compare against the expected
count — report the *verified* number, not the number the agent believes it
processed, and flag any discrepancy explicitly rather than defaulting to
"success.")

</details>

---

### Q5.21 · Failure mode: no recovery path · [Design]

You're designing an agent for a multi-step booking task (search flights →
hold a seat → collect payment → confirm). A tool call in step 3 fails after
step 2 already succeeded (a seat is now held). What must the design account
for that a naive "retry the failed step" doesn't?

- **A.** Nothing — retrying step 3 alone is always sufficient
- **B.** Partial-completion state — the system needs to know a seat is already held (to avoid double-holding, and to release it if the overall task ultimately fails or times out) rather than treating each step as independent and stateless; recovery requires reasoning about what's already been committed, not just re-running the failed piece
- **C.** The agent should restart the entire multi-step task from scratch every time any step fails
- **D.** This is only a concern for financial transactions, not booking flows

<details>
<summary>Answer</summary>

**B**

**Why B:** Multi-step tasks with side effects create partial-completion
states that a stateless retry doesn't account for — if step 3 fails, simply
retrying it might work, but the system also needs a way to know the seat
hold from step 2 exists so it isn't duplicated, and a way to clean it up
(release the hold) if the task is ultimately abandoned. This is the general
shape of "no recovery path" as a failure mode: the agent (or its harness)
has no model of what's already committed versus what's still pending.

**Why not A:** Retrying step 3 alone ignores that step 2's side effect
(the held seat) is real and needs to be tracked and reconciled, not just
assumed away.

**Why not C:** Restarting from scratch risks duplicating already-committed
side effects (e.g., holding a second seat) rather than fixing the actual
problem — it's not a real solution, just a way to make the failure worse.

**Why not D:** This concern applies to any multi-step process with side
effects — booking flows, inventory reservations, multi-stage writes — not
uniquely to financial transactions, though financial ones make the stakes
more visible.

**Interviewer's likely follow-up:** *"How would you actually track this
partial-completion state?"* (Answer: an explicit state machine or ledger
per task instance — recording which steps have succeeded and what side
effects they had — that both the retry logic and any cleanup/rollback logic
can consult, rather than relying on the agent's in-context memory of what
it already did.)

</details>

---

### Q5.22 · Human-in-the-loop checkpoints · [Design]

You're building an agent that can issue customer refunds autonomously. Where
should you place a human-approval checkpoint, and why there specifically?

- **A.** Nowhere — full autonomy is the goal, and checkpoints defeat the purpose of automation
- **B.** Before the irreversible action (issuing the actual refund), gated on risk signals — e.g., refund amount above a threshold, or the case not matching a clean, well-precedented pattern — rather than either no checkpoint at all or a checkpoint on every single action regardless of risk
- **C.** After every tool call, regardless of what the tool does
- **D.** Only at the very end of the entire conversation, after the refund has already been issued

<details>
<summary>Answer</summary>

**B**

**Why B:** The right place for a checkpoint is where the cost of a wrong
action is high and irreversible — right before the refund actually
executes — and the right *condition* for triggering it is risk-based, not
blanket: low-value, clearly-precedented refunds can proceed autonomously,
while high-value or unusual cases route to a human. This balances autonomy
(where risk is low) against safety (where it isn't).

**Why not A:** No checkpoint at all on an irreversible financial action is a
real operational risk most companies wouldn't accept — "full autonomy" isn't
free of consequences when errors are costly and hard to undo.

**Why not C:** Checkpointing every single tool call (including harmless
ones like a lookup) destroys the efficiency benefit of automation and
creates approval fatigue, making humans rubber-stamp everything — which
defeats the safety purpose too.

**Why not D:** A checkpoint *after* the refund has already happened isn't a
checkpoint at all — it's just an audit log; it doesn't prevent the bad
outcome, only records it after the fact.

**Interviewer's likely follow-up:** *"How do you decide the risk threshold
that triggers human review?"* (Answer: usually a combination of dollar
amount, deviation from historical/precedented cases, and confidence signals
from the agent's own reasoning — tuned initially conservatively and loosened
as you build evidence the autonomous path is reliable on a given case
category.)

</details>

---

### Q5.23 · Observability and tracing · [Recall]

Debugging why an agent gave a wrong answer on a specific customer
interaction, you find the logs only contain the final response — no record
of which tools were called, what they returned, or what the model's
intermediate reasoning was. What's the actual cost of this gap, beyond "it's
annoying to debug"?

- **A.** None — as long as the final output is logged, that's what matters for a production system
- **B.** You can't distinguish between different root causes that produce the same symptom (wrong tool called vs. right tool but misread result vs. right information but flawed reasoning) — without the full trace, every debugging session becomes guesswork, and you can't build a reliable eval set from real failures because you don't know what actually went wrong
- **C.** The cost is purely about latency, since fuller logging always slows down responses
- **D.** This only matters for agents that use more than one tool

<details>
<summary>Answer</summary>

**B**

**Why B:** Multiple distinct failure modes can produce an identical wrong
final answer — tracing (tool calls, their inputs/outputs, and the model's
reasoning at each step) is what lets you tell them apart. Without it, fixing
a bug becomes trial-and-error rather than diagnosis, and you can't turn real
production failures into targeted eval cases, because you don't actually
know what category of failure you're looking at.

**Why not A:** Final-output-only logging is exactly the gap the question
describes as costly — it's the premise being questioned, not a valid
defense of it.

**Why not C:** Logging intermediate steps has a storage/cost overhead, not
primarily a latency one (logging can happen async, after the response is
already returned) — this mischaracterizes the tradeoff.

**Why not D:** Even single-tool agents benefit from tracing — you still need
to see the reasoning and the exact tool input/output to diagnose why a
single call went wrong.

**Interviewer's likely follow-up:** *"What would you actually want in the
trace, concretely?"* (Answer: every tool call with its exact arguments and
raw return value, the model's stated reasoning at each decision point if
using something like ReAct, timestamps for latency analysis, and ideally a
way to replay the same input against a fixed model version to check whether
a fix actually resolves the issue.)

</details>

---

### Q5.24 · Cost bounding and step limits · [Design]

Design the cost-control layer for an agent that will run in production for
an FDE-style customer pilot. What's the minimum viable set of controls you'd
insist on before letting it run unattended overnight?

- **A.** Just a good system prompt telling it to be efficient
- **B.** A hard per-session step limit, a hard per-session token/cost cap independent of step count, and an explicit terminal state ("stopped: budget exceeded") that's surfaced rather than silently truncating — plus alerting if sessions are hitting these limits more often than expected, since that signals a systemic problem, not just an edge case
- **C.** Rate-limiting the underlying API key is sufficient on its own
- **D.** None of these matter if you trust the model provider's own safety systems

<details>
<summary>Answer</summary>

**B**

**Why B:** This combines the mitigations from the loop and cost-runaway
failure modes into a concrete minimum: a step cap for control-flow safety, a
cost cap for the case where individual steps are unexpectedly expensive,
explicit surfacing of the terminal state rather than a silent cutoff (so
someone actually notices), and alerting on the *rate* of limit-hits, since
that's the signal that something is systemically wrong rather than a
one-off.

**Why not A:** Prompting alone is a soft control with no guarantee — you
need code-enforced limits that don't depend on the model choosing to
comply.

**Why not C:** Rate-limiting the API key bounds request frequency across
all traffic, not per-session cost or step count — a single session can
still run away within an otherwise-compliant rate limit.

**Why not D:** Provider-level safety systems address different concerns
(content policy, abuse) — they're not a substitute for your own
application-level cost and step controls specific to your task and budget.

**Interviewer's likely follow-up:** *"What do you do when a session hits
the limit mid-task, for a customer-facing pilot specifically?"* (Answer:
fail gracefully and visibly — surface a "didn't complete, here's what was
done so far" state to whoever's monitoring, rather than either silently
stopping or (worse) letting the customer see a broken half-finished
result with no explanation.)

</details>

---

### Q5.25 · When is agentic control flow actually justified? · [Applied]

Which of the following is **NOT** a valid reason to choose agentic control
flow over a fixed workflow?

- **A.** The number and order of steps genuinely can't be known ahead of time and depends on what's discovered along the way
- **B.** The task requires the system to recover from unexpected intermediate failures by trying a different approach, not just retrying the same step
- **C.** The team is more familiar with agent frameworks than with writing conditional logic by hand
- **D.** The investigation depth needed varies enormously by input, and hand-enumerating every possible path would be impractical

<details>
<summary>Answer</summary>

**C**

**Why C is the invalid reason (the answer to "NOT"):** Team familiarity with
a framework is an implementation-convenience argument, not a task-shape
argument — it says nothing about whether the *problem itself* has an
unknown or highly variable path. Choosing architecture based on tooling
comfort rather than what the task actually needs is exactly the kind of
reasoning that leads to agent-shaped solutions for workflow-shaped problems.

**Why A is valid:** This is the core legitimate justification — genuinely
unknown-ahead-of-time paths are what agentic control flow is for.

**Why B is valid:** Adaptive recovery (trying a different approach, not just
retrying) requires the kind of judgment agentic loops provide — a fixed
retry-the-same-step workflow can't do this.

**Why D is valid:** When the space of possible paths is too large or
variable to hand-code, letting the model determine the path per-input is a
legitimate efficiency argument, not just convenience.

**Interviewer's likely follow-up:** *"How do you push back on a team that's
choosing agents for reason C without sounding like you're just against the
technology?"* (Answer: reframe the conversation around the task's actual
shape — ask "what's the fewest number of fixed steps that would work for
90% of cases," and show that the agent's flexibility is only earning its
cost on the genuinely variable slice, same as the workflow-with-a-branch
argument from Q5.4.)

</details>

---

### Q5.26 · ReAct failure: correct reasoning, hallucinated arguments · [Applied]

An agent's reasoning trace correctly identifies "I need to look up this
customer's order history using their order ID," but the tool call it emits
passes a plausible-looking but entirely fabricated order ID rather than one
that actually appeared anywhere in the conversation. What does this tell
you, and what's the fix?

- **A.** The reasoning step is pointless since it didn't prevent the error
- **B.** Correct high-level reasoning doesn't guarantee correct low-level argument grounding — the model can identify the right *action* while still fabricating a *parameter* it doesn't actually have; the fix is validating tool arguments against known-good sources (e.g., check the order ID exists in context or the database) before executing, and/or requiring the model to quote or cite where a parameter value came from
- **C.** This means the model should never be trusted with any tool that takes an ID as an argument
- **D.** Increasing the reasoning verbosity will automatically fix argument hallucination

<details>
<summary>Answer</summary>

**B**

**Why B:** Reasoning about *what to do* and correctly *populating the
arguments to do it* are separate failure surfaces — a model can nail the
former and still hallucinate the latter, especially when a plausible-format
value (like an order ID) wasn't actually supplied anywhere in context. The
practical fix is a validation layer that checks arguments against
ground-truth before the tool executes, rather than trusting that correct
reasoning implies correct grounding.

**Why not A:** The reasoning step still has value (it makes the intended
action visible and auditable) even though it didn't catch this particular
class of error — the fix is adding argument validation, not discarding the
reasoning step.

**Why not C:** Blanket distrust of any ID-taking tool overcorrects — the fix
is validating the specific argument, not abandoning a whole category of
tool.

**Why not D:** More verbose reasoning doesn't inherently fix grounding —
the model can produce a longer, still-hallucinated explanation just as
easily; validation against ground truth is what actually catches this, not
verbosity.

**Interviewer's likely follow-up:** *"How would you design the validation
without just re-implementing the whole lookup yourself?"* (Answer: a
lightweight check — does this order ID appear anywhere in the conversation
history or a quick existence check against the database — before the
mutating or expensive tool call runs, rather than a full parallel
implementation of the tool's logic.)

</details>

---

### Q5.27 · Planning failure: no replanning · [Applied]

An agent produces a reasonable initial plan for a multi-step task, but two
steps in, a tool result reveals an assumption in the plan was wrong (e.g.,
"assume the customer has one active subscription" turns out false — they
have three). The agent continues executing the original plan anyway,
producing a wrong result. What's missing?

- **A.** A bigger model that wouldn't have made the wrong initial assumption
- **B.** A replanning step — the agent needs to periodically check whether new information invalidates the current plan and revise it, rather than treating the initial plan as fixed once formed; this is a known weakness of naive plan-then-execute agents versus ones that interleave planning and execution
- **C.** The tool that revealed the surprising information should be removed
- **D.** More detailed upfront planning would have prevented this, so replanning isn't necessary if the initial plan is good enough

<details>
<summary>Answer</summary>

**B**

**Why B:** Even a well-reasoned initial plan can be built on assumptions
that turn out wrong once real data comes in — the failure here isn't bad
initial planning, it's the *absence of a mechanism to revise the plan* when
new evidence contradicts it. Interleaving planning and execution (check:
does what I just learned change what I should do next) is the standard
fix.

**Why not A:** No amount of model size prevents an initial assumption from
occasionally being wrong when the real answer wasn't knowable until the
tool call ran — the fix is architectural (replanning), not raw capability.

**Why not C:** Removing the tool that surfaced the true state doesn't fix
anything — it just makes the agent blind to the discrepancy instead of
reactive to it, which is worse.

**Why not D:** No amount of upfront planning quality eliminates the need for
replanning — some information is genuinely only available once you act
(that's the whole reason for tool calls in the first place), so a purely
upfront-planned system will always have this exposure.

**Interviewer's likely follow-up:** *"Doesn't constant replanning risk the
agent thrashing and never committing to anything?"* (Answer: yes, that's a
real tension — the mitigation is to replan only when new information
actually contradicts a plan assumption, not on every single tool result,
and to bound how many times a plan can be revised before escalating rather
than looping indefinitely.)

</details>

---

### Q5.28 · Multi-agent handoff overhead · [Design]

Two agents in a pipeline — a "research" agent and a "summarizer" agent —
communicate by the research agent producing a full transcript of its tool
calls and reasoning, which is then passed entirely into the summarizer's
context. What's the design flaw, and the fix?

- **A.** There's no flaw — passing the full transcript maximizes the information available to the summarizer
- **B.** The full transcript includes noise (redundant tool calls, dead-end reasoning, raw tool output) that the research agent has already resolved into a conclusion — passing all of it defeats the purpose of splitting the work, since the summarizer now has to wade through the same volume the research agent did; the fix is a defined handoff contract where the research agent returns a condensed finding, not its raw working notes
- **C.** The two agents should be merged into one, since any handoff is inherently wasteful
- **D.** The fix is to give the summarizer agent tool access too, so it can independently verify the research

<details>
<summary>Answer</summary>

**B**

**Why B:** This connects back to the subagent-boundary principle (Q5.12):
the value of delegating a noisy sub-task is that it *absorbs* the noise and
returns a condensed result. Passing the raw transcript through negates that
value entirely — the summarizer inherits the same context bloat the
research agent was supposed to isolate, just one hop later.

**Why not A:** More raw information isn't free — it costs context budget and
can dilute the summarizer's attention on what actually matters, which is
exactly the failure this question is testing.

**Why not C:** Merging isn't the only fix, and if the research step
genuinely produces a lot of noisy intermediate work, keeping it separate
with a *proper* condensed handoff is still the better architecture — the
flaw is in the handoff contract, not the existence of two agents.

**Why not D:** Giving the summarizer independent tool access doesn't address
the actual problem (an overloaded handoff) — it adds a different
capability without fixing the noise-passing issue, and duplicates work the
research agent already did.

**Interviewer's likely follow-up:** *"What should the handoff contract
actually specify?"* (Answer: a fixed structure — e.g., "key findings,"
"confidence/caveats," "sources checked" — that forces the research agent to
distill rather than dump, and gives the summarizer a predictable shape to
work from instead of an open-ended transcript.)

</details>

---

### Q5.29 · Cost-aware orchestration with a dual-model split · [Recall]

In a thinking-model/worker-model orchestration setup, the worker model is
deliberately a smaller, cheaper model than the thinking model. What's the
main risk this introduces that you need to design around?

- **A.** The worker model can't call tools at all
- **B.** The worker model may misinterpret or imperfectly execute a step the thinking model intended, and because the thinking model doesn't see the worker's full raw execution — only a summary or result — a subtle execution error can slip through unnoticed unless the handoff includes enough signal for the thinking model to sanity-check the outcome
- **C.** Using two different models is against API terms of service
- **D.** The overall system becomes slower than using the thinking model for everything

<details>
<summary>Answer</summary>

**B**

**Why B:** The cost/latency benefit of a cheaper worker model comes with a
real tradeoff: less capable execution, combined with a thinking model that
isn't watching every token of what the worker actually did — only the
outcome it reports. If the handoff contract doesn't surface enough of what
happened for the thinking model to catch a plausible-looking but wrong
result, errors can propagate silently. Designing the handoff to include
verifiable detail (not just "done") is how you manage this risk.

**Why not A:** Model size doesn't determine tool-calling capability — many
smaller models support tool use fine; this isn't the relevant constraint.

**Why not C:** Mixing models across providers or tiers within one
application is common practice and not a terms-of-service issue in
general.

**Why not D:** The point of using a cheaper worker model is usually to
*reduce* latency and cost per step, not increase it — the tradeoff is
accuracy/capability risk, not speed.

**Interviewer's likely follow-up:** *"How would you decide which steps are
safe to hand to the cheaper worker versus which need the thinking model
directly?"* (Answer: steps that are mechanical and easily verifiable
(formatting, straightforward tool calls with clear success/failure signals)
are safe for the worker; steps requiring judgment calls or where a wrong
result is costly and hard to detect after the fact should stay with the
more capable model, or at least get an explicit verification step before
being trusted.)

</details>

---

### Q5.30 · Design: agent architecture for support-ticket escalation · [Design]

Design an agent that autonomously handles Tier-1 support tickets and
escalates to a human when it can't resolve one. What are the key components
you'd include, beyond "an LLM with tools"?

- **A.** Just the LLM, a knowledge-base search tool, and a "reply to customer" tool — additional components add unnecessary complexity
- **B.** A step/cost budget with a hard cap; a confidence or escalation-trigger mechanism (not just "ran out of steps" but genuine uncertainty signals, e.g., contradictory information found, or the ticket matching a category historically prone to error); a human-handoff path that passes full context (not a bare "I'm stuck" message) so the human isn't starting cold; tracing/logging for every resolved *and* escalated ticket so you can build an eval set from real outcomes; and a way to distinguish "resolved" from "replied and hoped for the best," i.e., some verification the issue is actually fixed, not just that a plausible-sounding reply was sent
- **C.** A separate agent for every possible ticket category, coordinated by a router agent
- **D.** Full autonomy with no escalation path, since the goal is to minimize human involvement

<details>
<summary>Answer</summary>

**B**

**Why B:** This pulls together the failure-mode and safeguard material from
the rest of the file into one design: cost/step bounding (Q5.17/Q5.18/Q5.24),
a real escalation trigger rather than a fallback that only fires on
resource exhaustion, a rich handoff so escalation doesn't just dump the
problem on a human with no context, tracing for building better evals over
time (Q5.23), and — critically — outcome verification, since "sent a reply"
and "solved the problem" are different things (echoing the silent-failure
pattern in Q5.20).

**Why not A:** This is the minimum viable *capability* set but omits every
safety and observability component that makes the system trustworthy in
production — it would work in a demo and fail in exactly the ways this file
describes once it meets real traffic.

**Why not C:** Per-category agents is premature multi-agent decomposition
(see Q5.9) for a task that a single well-designed agent with good tools and
escalation logic can likely handle — added coordination cost without an
established need for it.

**Why not D:** Removing the escalation path entirely for the sake of
minimizing human involvement is exactly the kind of choice that turns a
manageable failure rate into unmitigated customer-facing harm — escalation
*is* the safety valve, not an admission of failure.

**Interviewer's likely follow-up:** *"How would you measure whether this
system is actually working well after launch?"* (Answer: track resolution
rate with verified outcomes (not just replies sent), escalation rate and
whether escalated tickets were genuinely hard vs. things the agent should
have handled, customer satisfaction on resolved tickets, and cost per
ticket — then use the traced failures to build a growing eval set rather
than relying on a static one from launch day.)

</details>

---

## Explain prompts

### E5.1 · Explain: when not to use an agent — expense approval

**Prompt:** *"We're building a system to approve or flag employee expense
reports for review. Should this be an agent? Walk me through your
reasoning."*

**Target:** 60–90 seconds spoken. Answer out loud before opening the rubric.

<details>
<summary>Rubric — score yourself</summary>

**Must hit (1 point each):**
- [ ] Identifies that "approve, flag, or reject" is a bounded classification
      with a small output space
- [ ] Notes that policy checks (amount thresholds, category rules, receipt
      presence) are largely deterministic and don't need model-driven
      control flow
- [ ] States that a structured-output call (possibly with one lookup tool
      for policy limits) handles the common case without agentic looping
- [ ] Names a concrete cost of defaulting to an agent here: unpredictable
      latency/cost per report, harder to audit why a specific decision was
      made
- [ ] Describes what would change the answer — e.g., a report requiring
      cross-referencing multiple systems (travel booking, prior approvals,
      manager hierarchy) in a variable order before a decision can be made

**Bonus — signals strength:**
- [ ] Distinguishes "the task involves multiple checks" from "the task
      requires the model to decide what to check next" — only the latter
      justifies agentic flow
- [ ] Mentions that even the "flag for review" path is itself a fixed
      outcome, not something requiring further agent autonomy
- [ ] Raises auditability: finance/compliance contexts often need a
      deterministic, explainable decision trail

**Red flags — deduct:**
- [ ] Reaches for "agent" immediately without examining the task's actual
      shape
- [ ] Can't articulate any cost of using an agent here
- [ ] Treats "uses an LLM to read the receipt/description" as automatically
      meaning "is an agent"

**Score: ___ / 5**

**Model answer:**
"Honestly, my first instinct is — this doesn't need to be an agent. Approve,
flag, or reject is a small, fixed set of outcomes, and most of the actual
logic — is it over the threshold, is the category allowed, is there a
receipt — that's deterministic policy stuff, not something that needs the
model deciding what to check next. I'd do one structured-output call: feed
it the report, maybe give it one tool to look up the relevant policy limit,
and have it return a decision plus a reason. If I made it an agent instead,
I'm paying for unpredictable latency and cost on every single report, and
honestly it gets harder to explain to finance why a specific report got
flagged — which matters a lot in this kind of context. Where I'd actually
reach for an agent is if resolving a report meant checking multiple systems
in an order you can't know ahead of time — like cross-referencing a travel
booking against manager approvals against a policy exception list, where
the path really does depend on what you find along the way."

</details>

---

### E5.2 · Explain: when not to use an agent — code review comments

**Prompt:** *"A teammate proposes an 'agent' that reads a pull request and
posts review comments. Is this the right framing? What would you actually
build?"*

**Target:** 60–90 seconds spoken.

<details>
<summary>Rubric — score yourself</summary>

**Must hit:**
- [ ] Recognises that "read a diff and produce comments" is largely a
      single well-defined transformation, not an open-ended task
- [ ] Proposes a workflow: pull the diff, one (or a few parallel, not
      sequential-agentic) LLM calls to generate comments against fixed
      criteria, post them — no model-driven looping needed for the core
      path
- [ ] Identifies a genuine reason agentic behavior *might* help: e.g.,
      needing to look up related code elsewhere in the repo to understand
      context before commenting, where what to look up isn't knowable
      upfront
- [ ] Notes the cost of over-engineering this as an agent: slower CI
      feedback loops, less predictable comment quality/volume
- [ ] Distinguishes "agent" from "uses an LLM with a repo-search tool" —
      having a tool available for the occasional lookup doesn't require
      full agentic control flow if the core task is still one bounded
      pass over the diff

**Bonus:**
- [ ] Mentions parallelizing comment-generation across files/hunks as a
      workflow-level optimization, distinct from agentic sequencing
- [ ] Raises that comment quality is easier to eval and regression-test
      against a fixed pipeline than an open-ended agent
- [ ] Notes that even the "needs to look up related code" case might be
      handled by a single call with retrieval, not a full multi-step agent

**Red flags:**
- [ ] Accepts the "agent" framing uncritically because "reading code and
      reasoning about it" sounds agent-like
- [ ] Can't name what would actually justify more autonomy here
- [ ] Ignores CI/latency implications entirely

**Score: ___ / 5**

**Model answer:**
"I'd push back a little on calling it an agent, at least for the core case.
Reading a diff and producing review comments is basically one
transformation — you're not really choosing between different paths, you're
generating structured output against a fixed set of criteria. I'd build it
as a workflow: pull the diff, run it through comment-generation, maybe
parallelize across files since they don't depend on each other, post the
results. Where it gets more agent-shaped is if understanding a change
actually requires looking at other parts of the codebase — like, this
function changed, and I need to check who calls it before I know if the
change is safe. That's a case where what you need to look up isn't knowable
ahead of time. Even then, I'd probably start with just giving the workflow a
repo-search tool it can use once or twice, rather than jumping straight to
a fully agentic loop — see how far that gets you before adding more
autonomy than the task's actually asking for."

</details>

---

### E5.3 · Explain: the workflow-chain-agent spectrum

**Prompt:** *"How do you personally decide where something falls on the
spectrum from 'fixed workflow' to 'full agent'? Walk me through your mental
model."*

**Target:** 90 seconds spoken.

<details>
<summary>Rubric — score yourself</summary>

**Must hit:**
- [ ] Articulates the spectrum: single LLM call → chain of fixed calls
      (workflow) → LLM-controlled branching/looping (agent)
- [ ] States the governing question: who decides what happens next — the
      code, or the model?
- [ ] Connects the choice to how *knowable* the task's path is ahead of
      time — enumerable paths favor workflows, genuinely unknown paths favor
      agents
- [ ] Names at least two concrete costs that increase as you move toward
      full agent: cost/latency variance, harder debugging/auditability,
      non-determinism
- [ ] Gives a concrete example of a task on each end of the spectrum (not
      just abstract description)

**Bonus:**
- [ ] Mentions that you can mix — a workflow with one agentic sub-step
      isolated to the genuinely variable part, rather than treating it as
      all-or-nothing
- [ ] Notes that this decision should be revisited as you learn more about
      real traffic patterns, not fixed once at design time
- [ ] Distinguishes "the task is complex" from "the task's path is unknown"
      — complexity alone doesn't require agentic flow if the complex logic
      is still enumerable

**Red flags:**
- [ ] Treats "agent" as simply "the more advanced/impressive option"
- [ ] Can't give a concrete example, only abstract description
- [ ] Implies the decision is purely about model capability rather than
      task shape

**Score: ___ / 5**

**Model answer:**
"For me it comes down to one question — who's deciding what happens next,
the code or the model? If I can enumerate the steps and the order ahead of
time, that's a workflow, even if there's an LLM call in there doing
something smart at one step — like extracting fields, or classifying
something. If the path genuinely depends on what you find along the way —
like, an investigation where step three depends entirely on what step two
turned up, and you can't predict that shape upfront — that's when you need
the model actually choosing the next action, which is what makes it an
agent. And I try not to treat it as all-or-nothing — a lot of the time the
right answer is a workflow with one agentic piece isolated to just the part
that's genuinely unpredictable, rather than making the whole thing agentic
because one piece of it is. The cost of going full-agent when you don't
need to is real — latency gets less predictable, cost varies more, and
honestly it's just harder to debug when something goes wrong, because now
you're debugging a decision the model made instead of a step you wrote."

</details>

---

### E5.4 · Explain: the ReAct loop

**Prompt:** *"Explain the ReAct pattern to someone who understands LLMs but
has never built an agent."*

**Target:** 60 seconds spoken.

<details>
<summary>Rubric — score yourself</summary>

**Must hit:**
- [ ] Explains the Thought → Action → Observation loop structure
- [ ] Clarifies that "Thought" means the model explicitly states its
      reasoning before acting, not just emitting a tool call silently
- [ ] Explains why this helps: reasoning grounded in the most recent
      observation tends to make better next-step decisions than acting
      without articulating why
- [ ] Notes it's iterative — the loop repeats until the model decides it has
      enough information to produce a final answer
- [ ] Mentions the practical debugging benefit: the reasoning trace is
      visible, so you can see *why* a wrong action was taken, not just that
      it was

**Bonus:**
- [ ] Notes ReAct is a prompting/loop pattern applicable to many models, not
      a special model feature
- [ ] Mentions the added latency/token cost of the explicit reasoning step
      as a real tradeoff
- [ ] Gives a concrete mini-example (e.g., "what's the weather where our
      newest customer is based" requiring lookup then lookup)

**Red flags:**
- [ ] Describes ReAct as a model architecture rather than a loop/prompting
      pattern
- [ ] Can't explain why the reasoning step is useful beyond "it sounds more
      thorough"

**Score: ___ / 5**

**Model answer:**
"ReAct is basically a loop: the model thinks out loud about what it should
do next, takes an action — usually a tool call — gets back an observation,
and then thinks again before deciding the next action, and it keeps doing
that until it's got enough to answer. The 'thought' part matters because
you're forcing the model to articulate its reasoning grounded in what it
just observed, instead of just chaining tool calls with no stated intent.
Practically, that's huge for debugging — if it picks a wrong tool, you can
actually see why it thought that was the right move, instead of just seeing
a wrong call with no context. It's not some special model capability, by
the way — it's a prompting and loop-structure pattern, you could implement
it with basically any model that can follow instructions and call tools.
The tradeoff is it costs more tokens and adds latency per step, since you're
generating that reasoning text every time, not just the action."

</details>

---

### E5.5 · Explain: why multi-agent is usually premature

**Prompt:** *"A stakeholder is excited about building a 'team of agents'
for a new feature. How do you have that conversation?"*

**Target:** 90 seconds spoken.

<details>
<summary>Rubric — score yourself</summary>

**Must hit:**
- [ ] States the general principle: start with a single agent (or workflow)
      and only split when you have a concrete reason
- [ ] Names the main legitimate reason to split: isolating a noisy,
      context-heavy sub-task so its raw output doesn't pollute the main
      loop's context
- [ ] Names the main cost of splitting prematurely: coordination overhead,
      information loss at handoffs, harder debugging across multiple
      traces
- [ ] Reframes the stakeholder's excitement productively — asks what
      specific problem they think multiple agents solves, rather than just
      saying no
- [ ] Notes that "multi-agent" is often reached for because it sounds more
      sophisticated, not because the task actually needs it

**Bonus:**
- [ ] Distinguishes subagents-for-context-isolation from
      multi-agent-negotiation (peer agents coordinating) as different
      patterns with different risk profiles
- [ ] Mentions that you can always add decomposition later once you've
      observed where a single agent's context actually gets overloaded
- [ ] Offers a concrete diagnostic question: "what would go wrong with one
      well-prompted agent doing this end to end?"

**Red flags:**
- [ ] Dismisses the idea without offering a constructive alternative
- [ ] Can't name a real scenario where multi-agent would be justified
- [ ] Agrees to build it without pushing back at all

**Score: ___ / 5**

**Model answer:**
"I'd start by asking what specifically they think a team of agents solves
that one well-designed agent doesn't — not to shut it down, just to get
concrete. Usually what comes out is either 'it feels more thorough' or
'each part seems like a different job,' and neither of those is actually a
reason for multiple agents on their own. The real reason to split is when
one part of the task generates a ton of noisy intermediate output — logs,
search results, whatever — that you don't want polluting the context of the
part that makes the final call. That's a legitimate reason to isolate a
subagent. But splitting things that are tightly coupled — like research and
writing, where the writer really needs the nuance from the research, not
just a summary — that usually makes output worse, not better, because you
lose information at the handoff and you're paying coordination cost for it.
So my answer is usually: let's build one agent first, see where it actually
struggles or gets overloaded, and split only the part that's genuinely
earning it."

</details>

---

### E5.6 · Explain: context management and compaction

**Prompt:** *"Your agent's context window is filling up over a long
session. What are your options, and how do you choose between them?"*

**Target:** 90 seconds spoken.

<details>
<summary>Rubric — score yourself</summary>

**Must hit:**
- [ ] Names compaction/summarization of older content as the primary
      mitigation
- [ ] Distinguishes what's safe to prune (raw tool output that's already
      been distilled into a conclusion) from what must be preserved (the
      original task goal, key decisions/constraints)
- [ ] Mentions re-injecting critical constraints periodically rather than
      trusting them to survive from early in a long context
- [ ] Notes the risk of over-pruning: losing information the agent turns
      out to still need, causing repeated work or wrong decisions
- [ ] Mentions using subagents to isolate noisy sub-tasks as a
      complementary strategy to reduce what enters the main context at all

**Bonus:**
- [ ] Mentions structured/external state (a task list, a scratch file) as
      an alternative to keeping everything in the conversational context
- [ ] Notes that compaction itself costs a model call/latency, so it's not
      free and shouldn't happen on every single turn
- [ ] Distinguishes proactive compaction (scheduled) from reactive
      (triggered when nearing a token threshold)

**Red flags:**
- [ ] Only names "use a bigger context window" as the solution, ignoring
      the underlying attention-degradation problem
- [ ] Can't articulate what's safe to prune vs. what isn't

**Score: ___ / 5**

**Model answer:**
"The main tool is compaction — periodically summarizing or pruning older
content so the working context stays focused. The judgment call is what's
safe to cut: raw tool output that's already served its purpose and gotten
folded into a conclusion, that's usually safe to drop. What's not safe to
drop is the original task goal and any hard constraints — and honestly, for
those, I don't even rely on compaction to preserve them, I'd rather
re-inject them explicitly at intervals, because I don't trust something
stated once at the very start to survive a long, noisy context reliably.
There's also a complementary move, which is just not letting noisy stuff
into the main context in the first place — if a sub-task is going to
generate a lot of raw output, push it into a subagent and only bring back
the distilled result. The risk on the other side is pruning too
aggressively and losing something the agent actually needed later, so I'd
rather be a little conservative about what counts as safe to cut than
optimize purely for token count."

</details>

---

### E5.7 · Explain: memory vs retrieval

**Prompt:** *"Someone says their product has 'agent memory.' What questions
do you ask to understand what that actually means technically?"*

**Target:** 60–75 seconds spoken.

<details>
<summary>Rubric — score yourself</summary>

**Must hit:**
- [ ] States that LLM APIs are stateless, so any persistence must be
      implemented externally
- [ ] Identifies that "memory" is almost always a store-and-retrieve
      mechanism — functionally a RAG system over past interactions
- [ ] Asks what's actually stored (raw transcripts? summarized facts?
      structured preferences?)
- [ ] Asks how/when retrieval happens (every session? triggered by
      relevance matching?) and what gets injected into context
- [ ] Distinguishes short-term "memory" (just the current conversation
      history being resent) from genuine long-term/cross-session memory

**Bonus:**
- [ ] Asks about staleness/update handling — what happens when a stored
      fact becomes outdated
- [ ] Asks about per-user isolation/access control on stored memory
- [ ] Notes that "memory" framing can obscure a fairly conventional
      retrieval architecture, which is fine, but worth naming accurately

**Red flags:**
- [ ] Assumes the model itself is persisting state between calls
- [ ] Doesn't ask any technical follow-up, just accepts "memory" at face
      value

**Score: ___ / 5**

**Model answer:**
"First thing I'd clarify — the model itself doesn't remember anything
between calls, the API's stateless, so whatever 'memory' means here has to
be built externally. So I'd ask: what's actually being stored — is it raw
conversation transcripts, or is something summarizing it into facts or
preferences first? And then, how does it come back — is it re-injected
into every new session automatically, or does it get retrieved based on
relevance to what the user's currently asking? Because functionally, that's
just RAG over past interactions, which is totally fine, I just want to know
if that's what we're talking about versus something more exotic. I'd also
want to know how it's scoped per user, and what happens when a stored fact
goes stale — like, if someone's preference changes, does the old one get
overwritten or does it just accumulate. None of this is a criticism of the
approach, retrieval-based memory works well, I just want the actual
mechanism named accurately instead of left vague."

</details>

---

### E5.8 · Explain: the major agent failure modes

**Prompt:** *"What are the ways an agent fails in production that you
wouldn't necessarily catch in a demo?"*

**Target:** 90 seconds spoken.

<details>
<summary>Rubric — score yourself</summary>

**Must hit:**
- [ ] Names infinite/repetitive loops as a failure mode, distinct from a
      single wrong action
- [ ] Names cost runaway as distinct from step-count issues (token cost per
      step can vary enormously)
- [ ] Names tool misuse — wrong tool chosen, or hallucinated arguments to a
      correctly-chosen tool
- [ ] Names silent failure — the agent reports success without the outcome
      actually being verified
- [ ] Names "no recovery path" — partial completion of a multi-step task
      with side effects, with no mechanism to reconcile or roll back

**Bonus:**
- [ ] Notes these often don't show up in demos because demos are short,
      low-volume, and use clean/expected inputs — production surfaces edge
      cases and scale
- [ ] Connects each failure mode to a concrete mitigation, not just naming
      the failure
- [ ] Mentions that these failure modes compound — e.g., a silent failure
      that also has no recovery path is worse than either alone

**Red flags:**
- [ ] Can only name one or two failure modes
- [ ] Describes failures vaguely ("it sometimes doesn't work") without
      naming the actual mechanism

**Score: ___ / 5**

**Model answer:**
"A few big ones. Infinite or repetitive loops — the agent gets stuck
alternating between the same couple of actions because it can't tell it's
not making progress. Cost runaway, which is related but not the same thing
— a session can be expensive without looping, if individual tool calls just
return a lot of tokens. Tool misuse — picking the wrong tool, or picking the
right tool but hallucinating an argument that looks plausible but was never
actually in the data. Silent failure is a big one — the agent says 'done'
but nobody actually checked that the claimed outcome matches reality, so you
find out three days later that most of a batch job silently failed partway
through. And then no recovery path — for multi-step tasks with real side
effects, if step three fails after step two already committed something,
you need to know what's already been done, not just retry blindly. None of
these show up in a demo because demos are short, clean, and low-volume —
they show up once you've got real traffic hitting edge cases at scale."

</details>

---

### E5.9 · Explain: placing human-in-the-loop checkpoints

**Prompt:** *"How do you decide where to put human approval steps in an
otherwise autonomous agent?"*

**Target:** 75 seconds spoken.

<details>
<summary>Rubric — score yourself</summary>

**Must hit:**
- [ ] States the general rule: gate irreversible or high-cost actions, not
      every action
- [ ] Mentions risk-based triggering (amount thresholds, deviation from
      precedented cases) rather than blanket checkpointing
- [ ] Names the failure mode of over-checkpointing: approval fatigue, where
      humans start rubber-stamping without real review
- [ ] Names the failure mode of under-checkpointing: irreversible harm with
      no safety valve
- [ ] Notes the checkpoint should happen before the irreversible action, not
      as an after-the-fact log

**Bonus:**
- [ ] Mentions that thresholds should be tunable and revisited as you build
      evidence about the autonomous path's reliability
- [ ] Distinguishes "low confidence" as a separate valid trigger from pure
      dollar-amount thresholds
- [ ] Notes the tension between safety and the efficiency benefit of
      automation, and that this is a genuine tradeoff, not a solved problem

**Red flags:**
- [ ] Proposes checkpointing everything, treating this as "safer is always
      better" with no cost consideration
- [ ] Proposes no checkpoints at all for an action with real-world
      consequences

**Score: ___ / 5**

**Model answer:**
"The rule I use is: gate the things that are expensive to get wrong and
hard to undo, not everything. So for something like issuing refunds, I
wouldn't put a human in the loop for every single case — that just leads to
approval fatigue, where the human stops actually reviewing and starts
rubber-stamping, which defeats the point. Instead I'd trigger review based
on risk — refund amount above some threshold, or the case not matching a
pattern we've seen a lot before. And the checkpoint has to sit right before
the action actually executes, not as a log afterward — a record after the
fact doesn't prevent anything, it just documents that something already
happened. There's a real tension here between safety and the whole point of
automating this in the first place, and I don't think there's a clean
answer — you start conservative, watch how the autonomous path performs on
the low-risk cases, and loosen the threshold as you build actual evidence
it's reliable."

</details>

---

### E5.10 · Explain: observability for agents

**Prompt:** *"What would you want logged for every agent session before
you'd trust this system in production?"*

**Target:** 75 seconds spoken.

<details>
<summary>Rubric — score yourself</summary>

**Must hit:**
- [ ] Names every tool call with exact arguments and raw return value
- [ ] Names the model's reasoning/intermediate steps, not just the final
      output
- [ ] Explains why final-output-only logging is insufficient: can't
      distinguish between different root causes producing the same wrong
      symptom
- [ ] Mentions timestamps/latency data per step for performance debugging
- [ ] Connects tracing to eval-building: real failures, once traced, become
      targeted eval cases

**Bonus:**
- [ ] Mentions the ability to replay a traced session against a fixed model
      version to verify a fix actually resolves the issue
- [ ] Notes cost/token usage per step as part of the trace, tying back to
      cost-runaway diagnosis
- [ ] Mentions redacting/handling sensitive data in traces as a real
      constraint, not an afterthought

**Red flags:**
- [ ] Says final-output logging is sufficient
- [ ] Can't explain why intermediate steps matter, only that "more logging
      is generally good"

**Score: ___ / 5**

**Model answer:**
"At minimum, every tool call with its exact arguments and the raw result it
got back — not a paraphrase, the actual return value. And the model's
reasoning at each step, if we're doing anything ReAct-like, because that's
what tells you *why* it made a decision, not just what it decided. The
reason final-output-only logging isn't enough is that totally different
root causes can produce the same wrong answer — wrong tool picked, versus
right tool but misread the result, versus right information but bad
reasoning on top of it — and without the trace you're just guessing which
one happened. I'd also want timestamps per step for latency debugging, and
honestly the biggest long-term value is that every traced failure becomes a
candidate eval case — that's how you actually build a real eval set instead
of one you made up before you'd seen any production traffic."

</details>

---

### E5.11 · Explain: cost bounding for agents

**Prompt:** *"You're about to let an agent run unattended overnight for the
first time. What cost controls do you insist on first?"*

**Target:** 75 seconds spoken.

<details>
<summary>Rubric — score yourself</summary>

**Must hit:**
- [ ] Names a hard step limit, enforced by code not the model's judgment
- [ ] Names a separate hard cost/token budget, distinct from step count
      (since per-step cost can vary a lot)
- [ ] States that hitting a limit should produce an explicit surfaced
      state, not a silent stop
- [ ] Mentions alerting on the *rate* of limit-hits across sessions as a
      signal of systemic problems, not just single-incident handling
- [ ] Notes these controls need to be independent of the model recognizing
      its own malfunction — the model's judgment is exactly what's failing
      in these scenarios

**Bonus:**
- [ ] Mentions setting the initial limits conservatively and loosening
      based on observed normal-session cost distributions
- [ ] Distinguishes per-session limits from account/org-wide spend caps as
      a second layer of protection
- [ ] Notes that unattended/overnight runs specifically raise the stakes,
      since no human will notice a runaway session until morning

**Red flags:**
- [ ] Relies only on prompting ("be efficient") with no code-enforced
      control
- [ ] Conflates step count and cost as the same thing

**Score: ___ / 5**

**Model answer:**
"Two separate hard limits, both enforced in code, not just prompted for.
One's a step cap — max number of tool calls or loop iterations. The other's
a cost or token budget, and that has to be separate from step count, because
a session can hit very few steps and still be expensive if one tool call
returns a huge amount of data. And critically, when either limit gets hit,
that needs to surface as an explicit 'stopped, didn't finish' state —
not silently truncate and pretend everything's fine. For something running
unattended overnight specifically, I'd also want alerting if sessions are
hitting these limits more often than some baseline, because that's telling
you something systemic is going wrong with a class of input, not that one
session had a bad night. And the whole reason these need to be
code-enforced rather than the model just being told to watch its own
budget is — if the model's judgment is what's failing in the first place,
you can't trust that same judgment to notice it's failing."

</details>

---

### E5.12 · Explain: defending the dual-model orchestration decision

**Prompt:** *"Tell me about a time you made a deliberate architecture
decision around cost, latency, or reliability trade-offs — walk me through
your reasoning."*

**Target:** 90 seconds spoken. (This doubles as behavioural practice — see
also file 11.)

<details>
<summary>Rubric — score yourself</summary>

**Must hit:**
- [ ] Describes the actual decision concretely: a thinking/planning model
      paired with a separate, cheaper worker model for execution
- [ ] States the problem it solves: keeping the planning context clean and
      uncluttered by execution-level noise (tool output, step-level detail)
- [ ] Names a real cost/tradeoff of the decision, not just the benefit —
      e.g., added handoff complexity, or the risk of the worker
      misexecuting a step in a way the planner doesn't catch
- [ ] Explains how the tradeoff was managed (e.g., what the handoff
      contract looks like, or when the cheaper model was/wasn't trusted)
- [ ] Speaks about it as a genuine engineering decision with reasoning, not
      just "I used a cool tool"

**Bonus:**
- [ ] Connects the decision to a measurable outcome or observed benefit
      (even informally — faster iteration, fewer context-overflow issues)
- [ ] Shows awareness of when this pattern wouldn't be worth the added
      complexity (e.g., for short, simple tasks)
- [ ] Demonstrates this was a considered choice among alternatives, not the
      only option considered

**Red flags:**
- [ ] Can't name any downside or tradeoff — every self-described
      "architecture decision" should have a real cost attached
- [ ] Describes the setup without explaining *why* it was chosen over
      simpler alternatives

**Score: ___ / 5**

**Model answer:**
"So — in my own workflow, I use a setup where a 'thinking' model handles
planning and a separate, cheaper 'worker' model actually executes each
step. The reason I went that way is context hygiene, mostly — if one model's
doing both the high-level planning and the step-by-step execution, its
context fills up with tool output and low-level noise, and I noticed that
degraded how well it tracked the overall plan on longer tasks. Splitting it
means the planner stays focused on the plan, and the worker just gets one
clear instruction at a time. The real cost is that the planner doesn't see
everything the worker did in detail — just the outcome it reports — so if
the worker does something subtly wrong, that can slip through unless the
handoff includes enough for the planner to sanity-check it. I manage that by
keeping the worker's tasks fairly mechanical — steps where success or
failure is pretty verifiable — and not handing it anything that needs real
judgment calls. It's not something I'd bother with for a short task, but for
longer sessions it's made a real difference in not losing the thread."

</details>

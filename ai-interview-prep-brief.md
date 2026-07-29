# Build Brief — AI Interview Prep Question Bank

**Handoff target:** Claude Code (autonomous build)
**Owner:** Chan (`currylaksa`)
**Deliverable:** A portable markdown question bank for interview preparation
**Estimated build:** One overnight autonomous run

---

## 1. Goal

Build a self-contained markdown study system that prepares a candidate for **entry-level interviews** in these role families:

- Applied AI Engineer
- AI-native / AI product developer
- Forward Deployed Engineer (FDE)
- AI Solutions Engineer
- Software Engineer (AI-adjacent)

**Target market:** Singapore. Candidate is a Malaysian CS (Networks & Security) graduate requiring Employment Pass sponsorship.

**Explicit non-goal:** This is NOT certification prep. Do not write vendor trivia — no exact pricing tables, no API version numbers, no "which AWS service does X" recall questions. Every question must be something a working engineer would plausibly ask in an interview.

**Critical design constraint:** The candidate needs to *explain* concepts out loud, not just recognise correct answers. MCQs alone will produce false confidence. The bank must train recall AND articulation.

---

## 2. Output structure

Create this directory tree:

```
ai-interview-prep/
├── README.md                      # How to use + 4-week study plan
├── 00-scoring-guide.md            # How to self-grade free-text answers
├── 01-software-engineering.md
├── 02-llm-api-mechanics.md
├── 03-tool-use-and-mcp.md
├── 04-rag.md
├── 05-agents.md
├── 06-evaluation.md
├── 07-guardrails-and-security.md
├── 08-ml-foundations.md
├── 09-integration-and-deployment.md
├── 10-scenarios.md                # FDE case questions
├── 11-behavioural.md
├── 12-mock-interview-prompts.md   # Paste-into-Claude blocks
└── progress.md                    # Self-tracking table
```

Plain markdown only. No build step, no dependencies. Must render correctly on GitHub and in any markdown editor.

---

## 3. Question formats

Three formats. Use `<details>` blocks so answers stay hidden until deliberately opened.

### Format A — Multiple choice

```markdown
### Q4.12 · Chunking strategy · [Applied]

A support-docs RAG pipeline returns answers that are factually correct but
consistently miss information that spans section boundaries. Retrieval recall@10
is 0.91. What is the most likely cause?

- **A.** The embedding model is undertrained on domain vocabulary
- **B.** Chunk size is too small relative to the semantic units in the source
- **C.** The reranker is over-weighting lexical overlap
- **D.** Temperature is set too high on the generation call

<details>
<summary>Answer</summary>

**B**

**Why B:** High recall@10 means retrieval is finding the right *documents*. The
failure is that individual chunks don't contain complete thoughts — information
spanning a boundary gets split, so no single chunk carries the full answer.
Increasing chunk size or overlap, or moving to semantic/recursive chunking, addresses this.

**Why not A:** A weak embedding model would depress recall. Recall is 0.91.

**Why not C:** A reranking problem would also show up as lower effective recall on
the retrieved set, and wouldn't specifically correlate with boundary-spanning content.

**Why not D:** Temperature affects phrasing and creativity, not what information
is available to the model.

**Interviewer's likely follow-up:** *"Okay — so you increase chunk size. What
breaks?"* (Answer: precision drops, more irrelevant context per chunk, higher
token cost, and you may exceed the context budget once you retrieve k chunks.)
</details>
```

**Required elements:** every MCQ must include *why the correct answer is correct* AND *why each distractor is wrong*, plus one realistic follow-up question. The distractor explanations are where most of the learning happens — do not skip them.

### Format B — Explain prompt with rubric

```markdown
### E5.3 · Explain: when not to use an agent

**Prompt:** *"We're building a system that processes incoming support tickets and
routes them to the right team. Should this be an agent? Walk me through your reasoning."*

**Target:** 60–90 seconds spoken. Answer out loud before opening the rubric.

<details>
<summary>Rubric — score yourself</summary>

**Must hit (1 point each):**
- [ ] Identifies that routing is a *classification* task with a bounded output space
- [ ] States that a single structured-output call is sufficient — no agent needed
- [ ] Articulates the general principle: use the simplest thing that works;
      escalate to agents only when the path is genuinely unknown ahead of time
- [ ] Names at least one concrete cost of agentic architecture (latency,
      non-determinism, cost variance, harder to evaluate, harder to debug)
- [ ] Describes what *would* change the answer (e.g. ticket requires looking up
      customer history across systems before routing → now you need tool use,
      and possibly an agent if the lookup sequence varies)

**Bonus — signals strength:**
- [ ] Distinguishes workflow (fixed path, LLM at steps) from agent (LLM chooses path)
- [ ] Mentions how you'd evaluate the routing decision
- [ ] Raises the fallback question: what happens on low confidence?

**Red flags — deduct:**
- [ ] Reaches for an agent framework immediately
- [ ] Cannot articulate any downside to agents
- [ ] Confuses "uses an LLM" with "is an agent"

**Score: ___ / 5**

**Model answer:**
[Write a natural, spoken-register model answer here — roughly 150 words.
Not a textbook definition. What a strong candidate would actually say.]
</details>
```

**Rubric design rules:**
- Must-hit points are *concepts*, not phrasings. The candidate should be able to tick a box even if they used different words.
- 4–6 must-hit points. Fewer is too coarse, more is unscoreable.
- Red flags are as important as must-hits — they catch confident wrong answers.
- Model answers must be written in spoken register, not written register. Contractions, natural hedging, the way a person actually talks.

### Format C — Scenario / case

```markdown
### S3 · Pilot is failing

**Setup:** You're three weeks into a six-week pilot with a Singapore insurance
client. Their internal champion messages you: the document-extraction accuracy is
"nowhere near good enough" and their CTO is asking whether to kill the pilot. You
have not seen the failing examples.

**Your task:** What do you do in the next 24 hours? Talk through it.

<details>
<summary>Rubric — score yourself</summary>

**Must hit:**
- [ ] Gets the actual failing examples before proposing any fix
- [ ] Distinguishes "accuracy is low" from "accuracy is low on the cases they care about"
- [ ] Asks what "good enough" means numerically — was a threshold ever agreed?
- [ ] Addresses the champion's political position, not just the technical problem
- [ ] Proposes a concrete, time-boxed next step rather than an open-ended investigation

**Bonus:**
- [ ] Recognises the root failure may be scoping, not model performance
- [ ] Suggests a shared eval set so "good enough" stops being subjective
- [ ] Plans for the CTO conversation, not just the champion conversation

**Red flags:**
- [ ] Jumps straight to "let's try a bigger model" or "let's fine-tune"
- [ ] Defensive framing — blames the client's data or expectations
- [ ] No plan for communicating upward

**Score: ___ / 5**

**Commentary:** [200 words on what this question is really testing and how
strong vs weak candidates differ.]
</details>
```

---

## 4. Content scope and volume

### Technical files (01–09)

| File | Topic | MCQ | Explain |
|---|---|---|---|
| 01 | Software engineering fundamentals | 30 | 8 |
| 02 | LLM API mechanics | 30 | 10 |
| 03 | Tool use & MCP | 25 | 10 |
| 04 | RAG | 35 | 12 |
| 05 | Agents | 30 | 12 |
| 06 | Evaluation | 25 | 10 |
| 07 | Guardrails & security | 25 | 8 |
| 08 | ML foundations | 20 | 8 |
| 09 | Integration & deployment | 25 | 8 |

Plus: **20 scenarios** (file 10), **18 behavioural** (file 11), **8 mock interview prompt blocks** (file 12).

### Difficulty distribution

Tag every question `[Recall]`, `[Applied]`, or `[Design]`. Target mix: **25% Recall, 50% Applied, 25% Design**. Applied questions are scenario-framed — they give a situation and ask what's happening or what to do. Design questions ask the candidate to architect something.

### Topic detail

**01 · Software engineering fundamentals**
Python and TypeScript idioms; async/await and concurrency; API design (REST, pagination, idempotency, versioning); error handling and retry semantics; exponential backoff and jitter; testing (unit vs integration, mocking external APIs, testing non-deterministic systems); git workflow; SQL basics; time and space complexity for common operations. Skew toward *integration* code over algorithms.

**02 · LLM API mechanics**
Tokenisation and why token counts differ from word counts; context windows and what happens at the limit; temperature vs top-p vs top-k; system/user/assistant message roles; multi-turn conversation state (the API is stateless — you resend history); streaming and time-to-first-token; prompt caching (what's cacheable, cache hit conditions, cost impact); batch APIs; rate limits and quota handling; structured output and JSON mode; **cost and latency arithmetic** — include at least 5 questions requiring actual calculation given token counts, request volumes, and per-token rates.

**03 · Tool use & MCP**
The tool-use loop end to end; tool schema design (naming, descriptions, parameter typing, when descriptions matter more than names); parallel vs sequential tool calls; handling tool errors and returning them to the model; what happens when the model picks the wrong tool or hallucinates arguments; tool result size and context budget. MCP: what problem it solves; client/server architecture; stdio vs streamable HTTP transports; tools vs resources vs prompts; when you'd write an MCP server vs a plain function.

**04 · RAG**
RAG vs fine-tuning vs long-context — the decision framework; chunking strategies (fixed, recursive, semantic, document-structure-aware) and the size/overlap tradeoff; embedding models and dimensionality; cosine similarity and why not euclidean; vector stores (pgvector, Qdrant, Pinecone, Weaviate) and selection criteria; hybrid search (BM25 + dense) and why it beats either alone; reranking; metadata filtering; query rewriting and HyDE; **diagnosing retrieval failure vs generation failure**; retrieval metrics (recall@k, precision@k, MRR, NDCG); handling documents that update; multi-tenant retrieval and access control.

**05 · Agents**
The workflow → chain → agent spectrum; **when NOT to use an agent** (weight this heavily — multiple angles); ReAct loop; task decomposition and planning; single-agent vs multi-agent, and why multi-agent is usually premature; subagent patterns and context isolation; context management and compaction strategies; memory (short-term, long-term, what's actually just retrieval); agent failure modes (infinite loops, cost runaway, tool misuse, silent failure, no recovery path); human-in-the-loop checkpoints; observability and tracing for agents; cost bounding and step limits.

**06 · Evaluation**
Why LLM outputs can't be unit-tested deterministically; building a golden dataset (size, sourcing, maintenance); LLM-as-judge and its known biases (position bias, verbosity bias, self-preference); pairwise comparison vs absolute scoring; rubric design for judges; offline vs online evaluation; A/B testing generative features; regression suites and CI for prompts; tracing and observability tooling; task-specific metrics; measuring hallucination and groundedness; what to do when eval and user feedback disagree.

**07 · Guardrails & security**
Prompt injection (direct and indirect) and why it isn't solved; the trust boundary between model output and system action; jailbreaks; PII detection and redaction; output validation and schema enforcement; sandboxing tool execution; least-privilege tool design; OWASP LLM Top 10; data leakage through logs and traces; multi-tenant isolation; **applying Zero Trust principles to agent architecture** — never trust model output, verify every tool call, scope credentials per-call. Include questions that let a security-background candidate demonstrate depth.

**08 · ML foundations**
Conceptual only — no training, no maths derivations. What attention does and why it matters; transformer architecture at a block-diagram level; pretraining vs post-training vs RLHF at a conceptual level; why hallucination happens mechanistically; embeddings vs generative models; what fine-tuning actually changes and when it beats RAG; LoRA and parameter-efficient tuning conceptually; distillation; quantisation and its tradeoffs; overfitting, train/test split, and why they still matter when evaluating LLM systems.

**09 · Integration & deployment**
Auth patterns (OAuth 2.0 flows, SAML, API keys, service accounts, token rotation); webhooks vs polling; REST vs GraphQL vs gRPC; data pipelines and batch vs stream; containers and orchestration basics; cloud IAM; VPC and network isolation; deployment models (SaaS, VPC-hosted, on-prem) and what drives the choice; data residency; **Singapore-specific: PDPA, and MAS TRM guidelines for financial-services clients**; SOC 2 and what it means in a sales conversation; latency budgets across a distributed system.

**10 · Scenarios** — FDE/Solutions Engineer case questions. Cover: failing pilot; scope creep; customer wants a feature that's technically wrong for them; discovery call with a customer who can't articulate their problem; demo breaks live; customer's data is far messier than they claimed; procurement/security review blocking deployment; customer asks "why not just use ChatGPT"; estimating effort with incomplete information; handing off to a customer's internal team.

**11 · Behavioural** — Standard STAR-format prompts, but rubrics should be tuned to what these roles screen for: **autonomy, comfort with ambiguity, customer empathy, and shipping over perfecting**. Include: why AI, why this company, why Singapore, a time you shipped under ambiguity, a time you disagreed with a stakeholder, a failure and what changed afterward, a time you had to learn something fast, how you handle being the least experienced person in the room.

Also include a short section on the **sponsorship conversation** — how to handle "do you need a work pass?" directly and without apology, and how to preempt it by leading with what makes the hire worth the paperwork.

**12 · Mock interview prompts** — Copy-paste blocks the candidate feeds into Claude to run a live mock. One per role family plus one hostile-interviewer variant. Each block must instruct the interviewer persona to: ask one question at a time, probe follow-ups rather than accepting first answers, not reveal scoring until the end, and finish with specific feedback on articulation quality (not just correctness).

---

## 5. Personalisation

Anchor examples and behavioural rubrics to the candidate's actual material where it fits naturally. Do not force it.

- **SecureExam UTM** — Zero-Trust exam platform, Node/Express + React + MySQL, JWT MFA, TOTP, RBAC across four roles, 25 mapped security controls, Isolation Forest anomaly-detection microservice. DIGITEX 2026 Silver Medal. Strongest asset for security and system-design questions.
- **Daily Sparks Events** — current role; Next.js 15 + Tailwind + Sanity + Vercel rebuild for a real client with a non-technical stakeholder. This is the FDE story: requirements from a business owner, shipped production system, measurable outcome.
- **Huawei Malaysia internship** — Project Engineer, built seven automation tools. Enterprise context.
- **Claude Code dual-model workflow** — thinking/worker model split via cc-switch. Genuine, defensible orchestration decision. Good material for agent-architecture questions.
- **CCNA + Google Cybersecurity** — networking and security depth most AI candidates lack. Leverage in integration and deployment questions.

**Known gaps to target harder** — the bank should over-index on these because the candidate has no production experience in them:
1. RAG at production scale
2. Evaluation and eval harnesses
3. Cost and latency optimisation

---

## 6. Quality rules

Enforce these strictly. A bank of 250 mediocre questions is worse than 150 sharp ones.

1. **No trivia.** If the answer is a memorised fact with no reasoning, cut it. Exception: a small number of genuinely load-bearing definitions.
2. **Plausible distractors.** Every wrong option must be something a reasonable candidate might actually believe. Draw distractors from real misconceptions. No filler options, no joke options.
3. **No length tells.** The correct answer must not be systematically longer or more hedged than the distractors.
4. **Limit negation stems.** At most 10% of MCQs should use "which is NOT" framing.
5. **Scenario-framed stems preferred.** "A pipeline does X and you observe Y — what's happening?" beats "What is Y?"
6. **Spoken register in model answers.** Model answers should read like a person talking, not a textbook. This is the whole point.
7. **No duplicate concepts.** Track concepts covered; if two questions test the same idea, cut or differentiate.
8. **Every question earns its place.** Ask: would a real interviewer plausibly ask this? If no, cut it.

---

## 7. `00-scoring-guide.md` content

Explain the two-layer grading system:

**Layer 1 — rubric self-scoring.** Answer out loud or in a scratch file *before* opening the rubric. Tick concepts you actually hit. Score honestly. Anything below 4/5 goes on the re-review list. Emphasise: the point is not the score, it's noticing which concepts you consistently drop under pressure.

**Layer 2 — Claude grading.** Provide a copy-paste prompt template:

```
Grade my interview answer strictly, as a senior engineer would.

QUESTION: [paste]
RUBRIC: [paste]
MY ANSWER: [paste]

Score me against each rubric point. Then, separately from the rubric,
assess: was this a coherent explanation or a pile of correct keywords?
Would this answer land in a real interview? Be blunt. If I hit the
concepts but explained them badly, say so.
```

Explain *why* both layers exist: the rubric catches missing knowledge, Claude catches bad articulation. Rubric-passing answers can still fail interviews.

**Layer 3 — spoken practice.** For the 20 highest-value explain-prompts, record yourself. Listen back. Flag: filler, circular explanation, trailing off, answering a different question than the one asked.

---

## 8. `README.md` content

Include a **4-week study plan** with daily blocks, assuming ~1 hour/day and active job applications running in parallel:

- Week 1: files 01–03 (SWE fundamentals, API mechanics, tool use/MCP)
- Week 2: files 04–06 (RAG, agents, evaluation) — heaviest week
- Week 3: files 07–09 + scenarios (security, ML foundations, integration, cases)
- Week 4: behavioural, mock interviews, re-review of everything scored below 4/5

Plus: how to use the `<details>` blocks honestly, how to log in `progress.md`, and a note that scenario and behavioural files should be practised *out loud*, not read.

`progress.md` should contain a simple markdown table: file, date attempted, MCQ score, explain average, concepts to re-review.

---

## 9. Build sequence

1. Scaffold directory and all file skeletons with headers and section structure.
2. Write `00-scoring-guide.md` and `README.md` first — they define the contract everything else follows.
3. Build technical files in order 01 → 09. **Complete each file fully before starting the next.** Do not draft all files shallowly and backfill.
4. Build scenarios (10), then behavioural (11), then mock prompts (12).
5. Final pass: check for duplicate concepts across files, verify difficulty distribution matches target, verify every MCQ has distractor explanations and a follow-up, verify every explain-prompt has must-hits, bonus, red flags, and a spoken-register model answer.
6. Generate `progress.md` with all files pre-listed.

**Commit granularity:** one commit per completed file. Message format: `add: <topic> question bank (<n> MCQ, <n> explain)`.

---

## 10. Definition of done

- [ ] All 14 files exist and are complete
- [ ] ~245 MCQs, ~86 explain-prompts, 20 scenarios, 18 behavioural, 8 mock blocks
- [ ] Every MCQ has: correct-answer reasoning, per-distractor reasoning, one follow-up
- [ ] Every explain-prompt has: must-hit list, bonus list, red-flag list, spoken-register model answer
- [ ] Difficulty tags present on every question, distribution within 5% of target
- [ ] No duplicate concepts across files
- [ ] Renders cleanly on GitHub — all `<details>` blocks collapse correctly
- [ ] `progress.md` scaffolded and ready to use

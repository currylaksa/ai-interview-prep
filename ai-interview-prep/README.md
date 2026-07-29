# AI Interview Prep Question Bank

A self-contained markdown study system for entry-level interviews in:

- Applied AI Engineer
- AI-native / AI product developer
- Forward Deployed Engineer (FDE)
- AI Solutions Engineer
- Software Engineer (AI-adjacent)

**Target market:** Singapore, as a candidate requiring Employment Pass
sponsorship. The content, examples, and behavioural rubrics are anchored to
that context throughout — including a dedicated section on how to handle the
sponsorship conversation.

This is **not** certification prep. There is no vendor trivia, no pricing
tables, no "which AWS service does X" recall. Every question is something a
working engineer would plausibly ask in a real interview. The design
constraint driving everything: you need to be able to *explain* concepts out
loud, not just recognise the correct answer in a list. See
[`00-scoring-guide.md`](00-scoring-guide.md) for why MCQs alone produce false
confidence, and how the three-layer grading system fixes that.

---

## How the bank is organised

| File | Topic |
|---|---|
| [`00-scoring-guide.md`](00-scoring-guide.md) | How to self-grade — read this first |
| [`01-software-engineering.md`](01-software-engineering.md) | SWE fundamentals: Python/TS, async, API design, testing, git, SQL |
| [`02-llm-api-mechanics.md`](02-llm-api-mechanics.md) | Tokens, context windows, sampling params, streaming, caching, cost/latency math |
| [`03-tool-use-and-mcp.md`](03-tool-use-and-mcp.md) | Tool-use loop, schema design, MCP architecture |
| [`04-rag.md`](04-rag.md) | Chunking, embeddings, hybrid search, reranking, failure diagnosis |
| [`05-agents.md`](05-agents.md) | Workflow vs agent, ReAct, multi-agent, failure modes, cost bounding |
| [`06-evaluation.md`](06-evaluation.md) | Golden datasets, LLM-as-judge, offline/online eval, regression suites |
| [`07-guardrails-and-security.md`](07-guardrails-and-security.md) | Prompt injection, Zero Trust for agents, OWASP LLM Top 10 |
| [`08-ml-foundations.md`](08-ml-foundations.md) | Attention, transformers, fine-tuning vs RAG, LoRA, quantisation — conceptual only |
| [`09-integration-and-deployment.md`](09-integration-and-deployment.md) | Auth, deployment models, PDPA, MAS TRM |
| [`10-scenarios.md`](10-scenarios.md) | FDE/Solutions Engineer case questions |
| [`11-behavioural.md`](11-behavioural.md) | STAR prompts + the sponsorship conversation |
| [`12-mock-interview-prompts.md`](12-mock-interview-prompts.md) | Paste-into-Claude blocks to run a live mock interview |
| [`progress.md`](progress.md) | Your self-tracking log |

Three question formats appear throughout, always with the answer hidden
behind a `<summary>Answer</summary>` or `<summary>Rubric</summary>` block so
you're not tempted to peek:

- **MCQ** — scenario-framed multiple choice, with reasoning for the correct
  answer *and* for every distractor, plus a realistic interviewer follow-up.
- **Explain prompt** — a spoken prompt with a scoring rubric (must-hit /
  bonus / red flags) and a spoken-register model answer.
- **Scenario** — an FDE-style case with the same rubric structure.

---

## The 4-week study plan

Assumes ~1 hour/day, running in parallel with active job applications. Adjust
pacing to your own calendar — the sequencing matters more than the exact
day count.

### Week 1 — Foundations (files 01–03)
Get the baseline technical vocabulary solid before layering on RAG/agents,
which assume you already have this.
- Days 1–2: `01-software-engineering.md` — MCQs first, then explain-prompts
- Days 3–4: `02-llm-api-mechanics.md` — don't skip the cost/latency
  calculations, they come up more than you'd expect in FDE interviews
- Days 5–7: `03-tool-use-and-mcp.md` + review anything scored below 4/5

### Week 2 — The heaviest week (files 04–06)
RAG, agents, and evaluation are where you have the least production
experience — this is deliberate, budget extra time here rather than rushing
to keep pace.
- Days 8–10: `04-rag.md` (35 MCQs — split across two sessions if needed)
- Days 11–13: `05-agents.md`, with extra attention on "when NOT to use an
  agent" — this comes up in almost every FDE-style interview
- Day 14: `06-evaluation.md` explain-prompts — say your rubric-design
  answers out loud, this is a weak spot for most candidates at your level

### Week 3 — Security, ML depth, deployment, cases (files 07–09 + 10)
- Days 15–16: `07-guardrails-and-security.md` — this is your strongest file
  given your SecureExam UTM background; use it, don't undersell it
- Day 17: `08-ml-foundations.md`
- Days 18–19: `09-integration-and-deployment.md`
- Days 20–21: `10-scenarios.md` — **practise these out loud**, not silently.
  Reading a scenario and nodding along teaches you nothing about how you'll
  actually sound improvising a 24-hour action plan under pressure.

### Week 4 — Behavioural, mock interviews, re-review
- Days 22–23: `11-behavioural.md`, including the sponsorship conversation
  section — rehearse this one specifically, it will come up
- Days 24–26: `12-mock-interview-prompts.md` — run at least 3 full mocks,
  including the hostile-interviewer variant
- Days 27–28: re-review everything logged below 4/5 in `progress.md`,
  re-answer out loud, re-score

---

## Using the `<details>` blocks honestly

The entire value of this bank depends on one discipline: **answer before you
open the block.** Out loud if you can, in a scratch file if you can't. If you
open the rubric or answer first "just to see the format," you're training
recognition, not recall — and recognition is exactly the false confidence
this bank exists to avoid.

Scenario and behavioural files (10, 11) should always be **spoken**, never
just read. Reading a good answer silently and being able to *deliver* it
under pressure are different skills, and the gap between them is usually
what separates a pass from a "we'll be in touch."

---

## Logging your progress

Update [`progress.md`](progress.md) after every session:

| File | Date attempted | MCQ score | Explain avg | Concepts to re-review |
|---|---|---|---|---|

Log honestly, including the sessions that went badly. The re-review list in
week 4 is only useful if it's real.

# 06 · Evaluation

Evaluation is the file most entry-level AI candidates are weakest on, because
it rarely gets taught and almost never gets built in a personal project — you
ship a demo that works on the five examples you tried and call it done. In a
real job, "does it work" has to become a repeatable, defensible number, and
building that number is its own discipline: golden datasets, judge design,
regression suites, and the uncomfortable moment when your eval says one thing
and your users say another. This file leans hard into that discipline —
mechanics over vibes.

---

## Multiple choice

### Q6.1 · Why unit tests don't work here · [Recall]

A new grad on your team writes `assert response == "The capital of France is Paris."`
to test an LLM-backed endpoint, and it fails intermittently even though the
answer is correct every time a human reads it. What's the most fundamental
reason this testing approach is wrong for LLM outputs?

- **A.** The test is checking the wrong endpoint
- **B.** LLM outputs are non-deterministic and semantically-equivalent-but-textually-different valid outputs are common, so exact string equality is the wrong invariant to test
- **C.** The test needs a longer timeout
- **D.** The model needs a lower temperature setting to pass

<details>
<summary>Answer</summary>

**B**

**Why B:** Even at temperature 0, phrasing can vary run to run (batching,
floating-point non-associativity, provider-side changes), and "Paris is the
capital of France" is just as correct as the exact string the test expects.
Testing free-text generation with exact-match assertions confuses "wrong" with
"differently phrased." The right invariant is usually a property check
(contains "Paris", or an LLM-judge call, or a structured-output field) — not
string equality.

**Why not A:** Nothing in the scenario suggests the wrong endpoint is being
hit; the endpoint responds, just with varying phrasing.

**Why not C:** Timeout has nothing to do with intermittent *content*
mismatches — a timeout failure looks like no response, not a wrong-string
failure.

**Why not D:** Lowering temperature reduces but doesn't eliminate variation,
and treating temperature as the fix hides the actual problem: the test
methodology, not the model's settings, is broken.

**Interviewer's likely follow-up:** *"So what do you assert instead?"*
(Answer: property-based checks — does it contain required facts, does it
parse as valid JSON, does an LLM-judge score it above a threshold, does it
avoid banned content — rather than exact string equality.)
</details>

### Q6.2 · Golden dataset sourcing · [Applied]

You're building the first eval set for a customer-support RAG assistant that
has been live for two months. You have access to two months of real
conversation logs. What's the strongest way to source your golden dataset?

- **A.** Write 50 questions from scratch based on the product spec, since real logs may contain low-quality user phrasing
- **B.** Sample real user queries from logs, stratified across query types and including known-hard cases flagged by support agents, then have a human write or verify the correct answer for each
- **C.** Use an LLM to generate 200 synthetic questions and answers covering the whole knowledge base
- **D.** Only use the queries that currently get thumbs-down feedback, since those are the ones that matter

<details>
<summary>Answer</summary>

**B**

**Why B:** Real logs are the ground truth for what users actually ask —
including the messy phrasing, typos, and ambiguity a spec-written question
would never capture. Stratifying across query types (not just hard cases)
gives you a representative baseline, not just a failure-hunting set, and
human-verified answers give you a trustworthy reference to score against.

**Why not A:** Spec-written questions are cleaner than reality and will
systematically miss the phrasing and ambiguity that causes real failures —
your eval set ends up testing an easier task than production.

**Why not C:** Fully synthetic Q&A tends to be self-consistent with whatever
the generating model already believes, which means it under-represents the
actual failure modes of your specific system and can bake in the generating
model's own blind spots.

**Why not D:** Thumbs-down-only sampling gives you a hard-cases-only set with
no signal on whether you're regressing on cases that currently work — you'd
have no idea if a change made easy cases fail.

**Interviewer's likely follow-up:** *"How big does this set need to be to
trust the number?"* (Answer: it depends on the required sensitivity — often
100-300 well-stratified examples is enough to detect a meaningful regression,
more than that has diminishing returns versus just adding categories you're
missing.)
</details>

### Q6.3 · Golden set maintenance · [Applied]

Six months after launch, your product's documentation has changed
significantly, and your golden eval set's expected answers are now stale for
about 15% of examples. Your eval score has been quietly dropping. What should
you do?

- **A.** Ignore it — the eval score dropping is directly attributable to worse retrieval, so investigate retrieval first
- **B.** Delete the stale examples so the score recovers
- **C.** Treat the golden set as a living artifact: audit it against current source-of-truth docs on a schedule, update expected answers where the underlying truth changed, and re-baseline the score
- **D.** Freeze the golden set permanently once created, since changing it defeats the purpose of a stable baseline

<details>
<summary>Answer</summary>

**C**

**Why C:** A golden set that never gets refreshed against a moving ground
truth silently becomes a test of "does the model agree with outdated docs,"
which is a different (and useless) thing to optimize for. Eval sets need
ownership and a maintenance cadence just like the product does — audit
against current truth, update, and be transparent that the baseline shifted
so you don't misread a legitimate update as a regression.

**Why not A:** You'd be debugging retrieval on stale ground truth — you could
spend days "fixing" a system that's actually correctly reflecting updated
docs the eval set doesn't know about yet.

**Why not B:** Silently deleting inconvenient examples is a data-integrity
problem — it inflates your score without fixing anything and hides real
coverage gaps.

**Why not D:** A frozen golden set drifts from reality and becomes actively
misleading; "stable" and "static" aren't the same thing — you want a stable
*process* for updating it, not a static, immutable file.

**Interviewer's likely follow-up:** *"How do you avoid this turning into
constant eval-set churn that makes your metrics incomparable over time?"*
(Answer: version the golden set, log which version each eval run used, and
batch updates on a schedule rather than ad hoc — so you can still compare
runs within a version and understand when/why a baseline shifted.)
</details>

### Q6.4 · LLM-as-judge, position bias · [Recall]

You're using an LLM to do pairwise comparison of two candidate responses (A
vs B) to pick the better one. What is "position bias" in this context?

- **A.** The judge systematically prefers whichever response is placed first (or second) in the prompt, regardless of actual quality
- **B.** The judge is biased toward responses that mention its own model family
- **C.** The judge scores responses lower when the prompt is long
- **D.** The judge prefers responses generated by the same model architecture as itself

<details>
<summary>Answer</summary>

**A**

**Why A:** This is the standard definition — LLM judges have been shown to
favor the response in a particular slot (often the first) at a rate above
chance, independent of content. The standard mitigation is running the
comparison twice with the order swapped and checking for a consistent
verdict, or averaging.

**Why not B:** That's closer to self-preference bias (Q6.5), a related but
distinct known bias — favoring outputs that resemble the judge's own style,
not literal self-mentions.

**Why not C:** Length sensitivity is verbosity bias, a separate known
failure mode, not position bias.

**Why not D:** That's a form of self-preference bias applied to model
identity rather than a positional artifact — different mechanism, different
mitigation.

**Interviewer's likely follow-up:** *"If you swap the order and get a
different verdict each time, what does that tell you?"* (Answer: the
judge doesn't have a reliable preference — treat it as a tie or flag it for
human review rather than picking one arbitrarily.)
</details>

### Q6.5 · Verbosity and self-preference bias · [Applied]

You switch your production model from Model X to Model Y and use Model Y
itself as the judge to score the switch. Model Y's responses score
dramatically higher than Model X's did under the old eval. A colleague is
about to approve the rollout based on this number. What's the concern?

- **A.** Nothing — the eval was run consistently, so the comparison is valid
- **B.** Using Model Y to judge Model Y's own outputs risks self-preference bias — the judge may favor outputs that match its own style/phrasing tendencies independent of actual quality, inflating the apparent improvement
- **C.** The eval set is too small to detect any real difference
- **D.** Model Y is definitely better and this just confirms it

<details>
<summary>Answer</summary>

**B**

**Why B:** LLM judges have a documented tendency to rate outputs from their
own model family more favorably, likely because stylistic familiarity reads
as quality to the judge. When the judge and the system-under-test are the
same model, that bias directly inflates the score for the very change you're
trying to evaluate — exactly the situation described.

**Why not A:** Consistency of process doesn't rule out a systematic bias
baked into that process — you can run something consistently and still get a
consistently wrong number.

**Why not C:** Set size is a real concern in general, but it's not what this
scenario is testing — the described risk is a specific, known bias
mechanism, not a sample-size problem.

**Why not D:** This begs the question — the whole point of the eval was to
determine whether Y is actually better, and the scenario gives you a
specific reason to distrust the number that "confirms" it.

**Interviewer's likely follow-up:** *"How would you actually validate the
switch?"* (Answer: use a judge from a different model family than either
candidate, or a human-adjudicated sample, to remove the self-preference
confound; ideally corroborate with an online metric too.)
</details>

### Q6.6 · Pairwise vs absolute scoring · [Applied]

You need to decide whether two prompt variants for a summarization feature
should be compared with pairwise (A vs B, pick the better one) or absolute
scoring (rate each 1-5 independently). You plan to run this comparison
repeatedly as you iterate on the prompt over the next few months. Which is
generally more reliable for this use case, and why?

- **A.** Absolute scoring, because it gives you a single number you can track as a trend line over time without needing to always compare against something
- **B.** Pairwise comparison, because LLM judges are more consistent at relative "which is better" judgments than at anchoring an absolute number to a fixed scale, and you can always compare a new variant against the current best
- **C.** They're equivalent — the choice doesn't matter for judge consistency
- **D.** Absolute scoring, because it lets you set a fixed pass/fail bar independent of any comparison

<details>
<summary>Answer</summary>

**B**

**Why B:** LLM judges tend to be noisier at absolute scale anchoring — a "4"
today and a "4" next week may not mean the same thing, because the judge has
no fixed internal reference. Pairwise comparisons are typically more
reliable because "is A better than B" is an easier and more stable judgment
than "is A a 4 out of 5." For iterative prompt development, you can run each
new candidate against your current best.

**Why not A:** This is the tempting-but-wrong answer — a trend line built on
an unreliable absolute scale is not actually trustworthy over time, since
scale drift can look like a real trend when it isn't.

**Why not C:** They are not equivalent; this is a well-documented difference
in judge reliability, not a coin flip.

**Why not D:** A fixed bar sounds appealing operationally, but if the
underlying absolute scores are noisy, the bar itself is unreliable — you'd
be enforcing a threshold on a wobbly number.

**Interviewer's likely follow-up:** *"What's the downside of pairwise
comparison?"* (Answer: it doesn't give you an absolute sense of "is this
actually good enough" — you can win every pairwise comparison while every
candidate is still mediocre, so you usually want some absolute or
human-anchored check too, especially before a launch decision.)
</details>

### Q6.7 · Rubric design for judges · [Design]

You're designing a rubric for an LLM judge to grade customer support
responses for tone. Which rubric design is most likely to produce reliable,
actionable scores?

- **A.** "Rate the tone from 1-10" with no further guidance, so the judge has maximum flexibility
- **B.** A rubric with specific, checkable criteria (e.g., "acknowledges the customer's frustration before offering a solution," "avoids blaming the customer," "uses the customer's name if provided") that the judge scores independently, summed into a total
- **C.** "Rate the tone as either 'good' or 'bad'" for maximum inter-rater simplicity
- **D.** Ask the judge to compare the tone to a list of 50 example responses and pick the closest match

<details>
<summary>Answer</summary>

**B**

**Why B:** Specific, checkable criteria reduce ambiguity in what "good tone"
means and let you decompose a fuzzy judgment into components a judge (or
human) can apply more consistently. It also gives you diagnostic
information — you learn *which* aspect of tone is failing, not just an
opaque aggregate score.

**Why not A:** An open-ended 1-10 scale with no criteria is exactly the
absolute-scoring reliability problem from Q6.6, amplified — "tone" is highly
subjective without anchors, so the judge effectively invents its own
criteria each time, inconsistently.

**Why not C:** Binary good/bad is simple but throws away almost all
diagnostic value — you can't tell what to fix, and edge cases get forced
into a bucket that doesn't fit.

**Why not D:** Comparing against 50 examples is expensive per call, doesn't
scale, and still requires someone to have judged those 50 examples correctly
in the first place — it just moves the rubric-design problem one level back.

**Interviewer's likely follow-up:** *"How many criteria is too many?"*
(Answer: past roughly 6-8 checkable criteria, judges get noisier and it gets
harder to act on the results — you want enough to be diagnostic, not so many
it becomes unscoreable, same principle as the explain-prompt rubrics in this
bank.)
</details>

### Q6.8 · Offline vs online evaluation · [Recall]

What's the core distinction between offline and online evaluation for a
generative AI feature?

- **A.** Offline evaluation runs on a fixed dataset before deployment; online evaluation measures behavior against real traffic/users after deployment
- **B.** Offline evaluation is manual; online evaluation is automated
- **C.** Offline evaluation only applies to RAG systems; online evaluation applies to agents
- **D.** Offline evaluation measures latency; online evaluation measures accuracy

<details>
<summary>Answer</summary>

**A**

**Why A:** This is the standard distinction — offline eval (golden sets,
regression suites) happens before or outside of production traffic and gives
you a controlled, repeatable signal; online eval (A/B tests, live monitoring,
user feedback) happens against real usage and captures things offline eval
structurally can't, like actual user satisfaction and distributional drift.

**Why not B:** Both can be manual or automated — the distinction is about
when/against-what data the eval runs, not who runs it.

**Why not C:** Both apply generally across RAG, agents, and any generative
feature — nothing here is architecture-specific.

**Why not D:** Both can measure either metric category; the offline/online
split is about data source and timing, not metric type.

**Interviewer's likely follow-up:** *"Why can't you just rely on offline
eval if it's cheaper and faster?"* (Answer: your golden set can't capture
every real-world input distribution, and some failure modes — like
long-term user trust erosion or edge cases your set-builders didn't
anticipate — only show up in production.)
</details>

### Q6.9 · A/B testing generative features · [Applied]

You want to A/B test a new prompt version for a chatbot against the current
one, targeting an increase in "conversation resolved without escalation."
Two weeks in, variant B shows a higher resolution rate, but also a
statistically significant increase in average response length. What should
you check before declaring B the winner?

- **A.** Nothing — resolution rate is the primary metric and it improved, ship it
- **B.** Whether the increase in resolution rate is actually caused by better answers, or whether longer responses are gaming a resolution-detection heuristic (e.g., a heuristic that infers "resolved" from the absence of a follow-up message, which longer answers might suppress by overwhelming or discouraging follow-up)
- **C.** Whether the response length increase alone is a launch blocker regardless of resolution rate
- **D.** Whether the test ran on a weekday or weekend

<details>
<summary>Answer</summary>

**B**

**Why B:** Metric gaming is a real risk any time your success metric is a
proxy rather than a direct measure of the thing you care about. A longer,
more overwhelming response could suppress follow-up messages (and thus trip
a "resolved" heuristic) without actually resolving anything better — you'd
be optimizing for silence, not satisfaction. Before shipping, you'd want to
corroborate with a more direct signal (CSAT, human review of a sample of
"resolved" B conversations).

**Why not A:** Shipping on a primary metric without checking whether it's
measuring what you think it's measuring is exactly the trap the scenario is
describing — the metric moved, but you don't yet know why.

**Why not C:** Length alone isn't necessarily a blocker — the question is
whether it's *causing* the metric to look better without a real
improvement, not whether length itself is bad.

**Why not D:** Day-of-week could be a confound for many things, but it's not
the specific, flagged anomaly (length correlating with the metric change) in
this scenario.

**Interviewer's likely follow-up:** *"How would you corroborate that the
improvement is real?"* (Answer: sample a set of B's "resolved" conversations
and have a human check if they were genuinely resolved, or look at a
downstream metric like repeat-contact-within-24h that a gamed heuristic
wouldn't fool.)
</details>

### Q6.10 · Regression suites and CI for prompts · [Design]

You're setting up a regression suite so prompt changes don't silently break
things. Which design is most appropriate for catching regressions before a
prompt change merges?

- **A.** Run the full production traffic through both prompt versions and compare user complaint volume after each deploys
- **B.** On every prompt change, run the fixed golden eval set against the new prompt, compare scores (and specific example diffs) against the last known-good baseline, and block the merge if score drops beyond a set threshold or specific known-critical examples regress
- **C.** Have an engineer manually spot-check five examples before merging
- **D.** Only test in production and roll back if metrics degrade after launch

<details>
<summary>Answer</summary>

**B**

**Why B:** This mirrors standard software CI: a fixed, versioned test set run
automatically on every change, with a comparable baseline and a concrete
gating threshold, catches regressions *before* they reach users — which is
the whole point of a regression suite. Diffing specific examples (not just
the aggregate score) also catches cases where an average holds steady but a
specific important case broke.

**Why not A:** That's an online/production test, not a pre-merge regression
check — by definition it can't catch anything before it ships, and it
exposes real users to a possibly-broken prompt.

**Why not C:** Manual spot-checking of five examples doesn't scale, isn't
repeatable, and will miss regressions outside whatever five examples the
engineer happened to pick.

**Why not D:** This is the "ship and see" approach the whole exercise is
trying to avoid — you want to catch regressions before launch when the
tooling exists to do so cheaply.

**Interviewer's likely follow-up:** *"What do you do when the eval score
looks fine in aggregate but you suspect a specific regression?"* (Answer:
diff individual example outputs between versions, not just the aggregate
score — aggregates can mask a bad regression on a small but critical
subset.)
</details>

### Q6.11 · Tracing and observability · [Recall]

In the context of LLM application evaluation, what is the primary purpose of
tracing/observability tooling (e.g., logging full prompts, tool calls,
intermediate steps, and outputs per request)?

- **A.** It replaces the need for a golden eval set entirely
- **B.** It lets you inspect what actually happened on a specific request — the full prompt sent, any tool calls and their results, intermediate reasoning steps, and the final output — which is essential for debugging failures that aggregate metrics can't explain
- **C.** It's only useful for measuring latency, not correctness
- **D.** It's a compliance requirement with no debugging value

<details>
<summary>Answer</summary>

**B**

**Why B:** An aggregate eval score tells you *that* something is wrong; a
trace tells you *what happened* on a specific request so you can figure out
*why*. Without tracing, debugging a single bad output in a multi-step agent
or RAG pipeline means guessing which stage failed — tracing gives you the
actual intermediate state to inspect.

**Why not A:** Tracing and golden-set evaluation solve different problems —
tracing is per-request debugging visibility, evaluation is aggregate quality
measurement. Neither replaces the other.

**Why not C:** Latency is one thing traces capture, but the scenario's core
value (and the standard use case) is inspecting request-level behavior for
correctness debugging, not just timing.

**Why not D:** Tracing can support compliance, but framing it as having "no
debugging value" is backwards — debugging is its primary day-to-day use.

**Interviewer's likely follow-up:** *"How do you avoid tracing becoming a
data-leakage risk?"* (Answer: redact or avoid logging PII/secrets in traces,
control access to trace data, and be deliberate about retention — this
connects directly to file 07's guardrails content.)
</details>

### Q6.12 · Task-specific metrics · [Applied]

You're evaluating an AI feature that extracts structured line-item data
(product, quantity, price) from scanned invoices. Which metric is most
appropriate as your primary quality signal?

- **A.** BLEU score comparing the raw extracted text to the raw invoice text
- **B.** Field-level accuracy — for each expected field (product, quantity, price) per line item, did the extraction match the ground truth, aggregated as something like exact-match rate per field and per whole line item
- **C.** Overall response length compared to invoice length
- **D.** A single LLM-judge score rating "how good does this extraction look"

<details>
<summary>Answer</summary>

**B**

**Why B:** Structured extraction has a well-defined ground truth per field,
so the most direct and actionable metric is field-level accuracy — it tells
you exactly which field type is failing (e.g., prices are fine but
quantities are frequently off-by-one due to unit confusion), which a
holistic score can't.

**Why not A:** BLEU measures n-gram overlap for free-text generation
tasks like translation — it's a poor fit for structured extraction where you
have exact expected values per field, not a free-text similarity question.

**Why not C:** Length has no relationship to whether the extracted fields
are correct — it's not a quality signal at all here.

**Why not D:** A holistic LLM-judge score is vaguer and less diagnostic than
direct field comparison when you have exact ground truth values available —
use the judge for genuinely subjective quality, not for something you can
just check field-by-field.

**Interviewer's likely follow-up:** *"What if the extraction gets the right
values but in a slightly different format, like '10' vs '10.00'?"* (Answer:
normalize before comparing — define canonicalization rules per field type
so formatting differences don't get counted as extraction errors.)
</details>

### Q6.13 · Measuring groundedness · [Applied]

You want to measure whether your RAG system's answers are actually grounded
in the retrieved documents, as opposed to the model adding plausible-sounding
but unsupported claims. What's a reasonable way to measure this at scale?

- **A.** Manually read every single production response forever
- **B.** Use an LLM-judge call that's given the retrieved context and the generated answer, and asked specifically whether every claim in the answer is supported by the context — flagging unsupported claims — run on a sampled or full basis, corroborated periodically with human spot-checks
- **C.** Check whether the answer's length roughly matches the context's length
- **D.** Assume groundedness is fine as long as recall@k was high on the retrieval step

<details>
<summary>Answer</summary>

**B**

**Why B:** Groundedness is specifically about the *generation* step, not
retrieval — did the model stick to what the context actually supports. An
LLM-judge prompted specifically for claim-level support checking (not a
general quality rating) is a scalable proxy, and periodic human corroboration
catches cases where the judge itself is unreliable.

**Why not A:** Manual review of every response doesn't scale and isn't
sustainable as a permanent process, though it's reasonable as a periodic
spot-check to validate the automated method.

**Why not C:** Length has no logical connection to whether claims are
supported — a short answer can hallucinate and a long one can be perfectly
grounded.

**Why not D:** This is the classic retrieval-vs-generation conflation from
file 04 — good retrieval only means the right information was *available*,
not that the model actually used it faithfully. A model can retrieve the
right chunk and still hallucinate past it.

**Interviewer's likely follow-up:** *"What's a concrete failure mode where
recall@10 is high but groundedness is low?"* (Answer: the right chunk was
retrieved, but the model filled a gap in the chunk with plausible-sounding
outside knowledge instead of saying "the context doesn't specify.")
</details>

### Q6.14 · When eval and user feedback disagree · [Design]

Your offline eval score for a new prompt version is meaningfully higher than
the previous version. After launch, user thumbs-down feedback increases.
What's the most defensible next step?

- **A.** Trust the offline eval since it's a controlled, repeatable measurement, and dismiss the thumbs-down increase as noise
- **B.** Immediately roll back without investigating, since users are always right
- **C.** Treat the disagreement as a signal that your golden set doesn't represent the real distribution of what users are asking or care about — investigate the thumbs-down cases specifically, check if they cluster around a pattern missing from your golden set, and use that to improve the eval set itself
- **D.** Average the two signals into a single composite score and use that going forward

<details>
<summary>Answer</summary>

**C**

**Why C:** When your controlled eval and real-world signal disagree, the
eval is the thing to interrogate, not the thing to trust blindly — it's a
proxy, and this disagreement is direct evidence the proxy has a blind spot.
Digging into the actual thumbs-down cases to find the pattern (a query type,
a topic, an edge case) your golden set doesn't cover turns this into an
opportunity to close the gap, not just a one-off rollback decision.

**Why not A:** Dismissing real user signal because it's less "controlled"
gets the epistemics backwards — the offline eval exists to *predict* real
user experience; when it fails to predict correctly, that's a failure of the
eval, not the users.

**Why not B:** An immediate rollback without investigation might be the right
short-term mitigation, but skipping the investigation means you'll walk into
the exact same blind spot next time you ship a change the eval approves of.

**Why not D:** Averaging two disagreeing signals produces a number that
doesn't mean anything specific — it papers over the disagreement instead of
resolving what's actually causing it.

**Interviewer's likely follow-up:** *"Should you roll back while you
investigate, or leave it live?"* (Answer: depends on severity and blast
radius — for a meaningful thumbs-down increase on a live product, the
common answer is roll back to the known-good version first, then
investigate offline against a fixed reproduction of the bad cases, rather
than debugging live.)
</details>

### Q6.15 · Sample size and statistical confidence · [Design]

You ran an eval with only 12 golden examples and variant B scored 83% versus
variant A's 75%. A colleague wants to ship B based on this. What's the
concern?

- **A.** No concern — 83% is clearly higher than 75%
- **B.** With only 12 examples, that 8-point gap could easily be one additional example flipping — the sample is too small to distinguish a real effect from noise, and you should either grow the eval set or run a proper significance check before treating the difference as meaningful
- **C.** The concern is that 83% isn't a passing grade
- **D.** The concern is that variant B is more expensive to run, not the sample size

<details>
<summary>Answer</summary>

**B**

**Why B:** 12 examples means each one is worth over 8 percentage points —
a single example flipping from wrong to right (or the reverse, due to
non-determinism) swings the score by a large amount. An 8-point gap on that
sample size has wide, overlapping confidence intervals and is very plausibly
noise, not a real quality difference.

**Why not A:** This is exactly the trap — "clearly higher" is an illusion
created by ignoring how unstable a percentage computed on 12 items actually
is.

**Why not C:** Whether 83% is "passing" is a separate question from whether
the *difference* between 83% and 75% is statistically meaningful on this
sample size — the scenario is about comparison validity, not an absolute
bar.

**Why not D:** Cost is a real launch consideration in general, but it's not
the concern this scenario is highlighting — the flagged issue is sample-size
reliability of the comparison itself.

**Interviewer's likely follow-up:** *"How big does the set need to be to
trust an 8-point gap?"* (Answer: depends on variance, but as a rule of
thumb you want enough examples that a one-item flip moves the score by a
fraction of a point, not multiple points — often 100+ for this kind of
gap to be trustworthy, though a formal significance test is the rigorous
answer.)
</details>

### Q6.16 · Hallucination measurement design · [Design]

You want to build an automated hallucination-detection check for a
customer-facing AI assistant that answers general product questions (not
strictly RAG-grounded — it can also reason). What makes this harder to
measure than groundedness in a RAG system?

- **A.** It isn't harder — the same claim-support check against retrieved context works identically
- **B.** Without a fixed retrieved context to check claims against, "hallucination" has to be checked against a broader, harder-to-define notion of ground truth (external knowledge, product documentation, or verifiable facts), which requires either a knowledge source to check against or a different verification strategy like requiring citations or restricting scope
- **C.** Hallucination doesn't apply to assistants that reason, only to pure retrieval
- **D.** It's harder only because the assistant is customer-facing, not because of anything about groundedness measurement

<details>
<summary>Answer</summary>

**B**

**Why B:** RAG groundedness has a clean reference point — the retrieved
context — so "is this claim supported" is a bounded check. A general
reasoning assistant without a fixed context has no such anchor; you either
need to check claims against an external source of truth (docs, a knowledge
base, a search call), or change the system design itself (require citations,
restrict the assistant to only answer from a defined scope) so the
verification problem becomes tractable again.

**Why not A:** The "same check" doesn't apply because there's no fixed
context to check against in this scenario — that's precisely the added
difficulty.

**Why not C:** Hallucination absolutely applies to reasoning outputs — a
model can confidently state an incorrect fact with no retrieval step
involved at all; this option gets the mechanism backwards.

**Why not D:** Being customer-facing raises the *stakes* of hallucination,
but it's not what makes *measurement* harder — the measurement difficulty
comes from the lack of a fixed reference to check against.

**Interviewer's likely follow-up:** *"What's a practical mitigation if you
can't fully solve measurement?"* (Answer: narrow the assistant's scope to
things you can verify — e.g., require it to cite a specific doc section, or
restrict free-reasoning answers to non-factual conversational content —
trading some capability for verifiability.)
</details>

### Q6.17 · CI regression on non-deterministic output · [Applied]

Your regression suite runs the golden set through the model at temperature 0
and diffs outputs against a stored baseline. Even with no prompt changes,
about 3% of examples show a different output on re-run and fail the diff.
What's the most likely explanation, and what should you do?

- **A.** The model is broken and needs to be replaced
- **B.** Temperature 0 doesn't guarantee bit-for-bit determinism across provider infrastructure (batching effects, hardware non-determinism, minor backend updates); an exact-output diff is too strict a check — compare against the same property/metric-based evaluation used elsewhere (LLM-judge, field match, contains-required-facts) instead of raw string equality
- **C.** This is expected and can be safely ignored with no changes to the pipeline
- **D.** The golden set itself is corrupted

<details>
<summary>Answer</summary>

**B**

**Why B:** Temperature 0 reduces but does not guarantee perfect
determinism in practice, due to factors outside your control on the provider
side. Diffing raw strings against a frozen baseline inherits the exact same
flaw as Q6.1's unit test — it's checking an invariant (textual identity)
that isn't actually the thing you care about. Swapping to a property-based
or judge-based check aligns the regression suite with what "correct" really
means.

**Why not A:** Nothing about small non-deterministic drift indicates the
model is broken — this is a known characteristic of hosted LLM inference,
not a defect.

**Why not C:** Ignoring it outright means every CI run has flaky failures
that erode trust in the suite — "ignore the noise" without fixing the
underlying comparison method just trains the team to ignore CI, which is
worse than fixing the check.

**Why not D:** There's no indication the golden set's content is wrong —
the failure is in the comparison method, not the data.

**Interviewer's likely follow-up:** *"Doesn't switching to an LLM-judge
check introduce its own non-determinism into the CI gate?"* (Answer: yes,
somewhat — mitigate by using a stable, low-temperature judge model, judging
against clear pass/fail criteria rather than a fine-grained score, and
treating borderline judge calls as a manual-review flag rather than an
automatic block.)
</details>

### Q6.18 · Judge model choice · [Applied]

You're setting up LLM-as-judge for your regression suite. Should you use the
same model as your production system, or a different one?

- **A.** Always use the exact same model as production, since it best understands the task
- **B.** Prefer a different, typically stronger or at least differently-sourced model than the one under test, specifically to avoid self-preference bias — and keep the judge model version pinned/stable so your regression baseline doesn't shift underneath you when the judge itself updates
- **C.** It doesn't matter as long as the judge prompt is well written
- **D.** Use a randomly selected model each run for diversity

<details>
<summary>Answer</summary>

**B**

**Why B:** Using the system-under-test as its own judge reintroduces the
self-preference bias from Q6.5. A separate, typically stronger judge model
avoids that specific confound. Pinning the judge's version matters
separately — if the judge model changes underneath you, your scores can
shift for reasons that have nothing to do with your actual system changing,
breaking the comparability of your regression baseline over time.

**Why not A:** This directly reintroduces the self-preference bias problem —
the model grading its own homework is the exact failure mode to avoid.

**Why not C:** Prompt quality matters, but it doesn't cancel out a
structural bias like self-preference — a well-written prompt run through a
biased setup is still biased.

**Why not D:** Random judge selection would make your baseline
non-reproducible run to run for reasons unrelated to your system — you'd
lose the ability to trust score changes as meaningful.

**Interviewer's likely follow-up:** *"What if the stronger judge model is
too expensive to run on every CI trigger?"* (Answer: run the full judge-based
suite on a schedule or pre-merge gate rather than every commit, and use a
cheaper deterministic check — like schema validation — for faster, more
frequent signal.)
</details>

### Q6.19 · Evaluating multi-step agent outputs · [Design]

You need to evaluate an agent that takes 5-10 tool-call steps to complete a
task (e.g., researching and drafting a report). Scoring only the final output
misses what?

- **A.** Nothing — the final output is all that matters to the user
- **B.** Whether the agent took an efficient, correct path to get there — wasted steps, redundant tool calls, or a lucky recovery from an early mistake can all produce a good final output while hiding a process that's expensive, fragile, or won't generalize
- **C.** The final output score already accounts for path efficiency automatically
- **D.** Multi-step agents can't be evaluated with any additional step-level signal

<details>
<summary>Answer</summary>

**B**

**Why B:** A good final answer can mask a bad process — the agent might have
called the wrong tool three times before stumbling onto the right one, or
looped unnecessarily, none of which shows up if you only grade the end
result. That process is exactly what drives cost, latency, and reliability
in production, and it's often what breaks first when the task distribution
shifts slightly. Step-level signals (number of steps, redundant/failed tool
calls, whether it recovered gracefully) give you visibility the final-output
score can't.

**Why not A:** Users care about the final output for that one task, but you
as the system owner also need to care about cost, latency, and
generalization — which the final output alone doesn't tell you.

**Why not C:** Nothing about a final-output score inherently captures path
efficiency; they're independent axes unless you explicitly measure both.

**Why not D:** Step-level tracing (tool-call counts, error/retry patterns,
redundant calls) is a standard, addable signal — this connects directly to
the tracing/observability content in Q6.11 and the agent failure-mode content
in file 05.

**Interviewer's likely follow-up:** *"What's a concrete step-level metric
you'd track?"* (Answer: number of tool calls per task relative to a
reasonable minimum, rate of tool-call errors, and whether the same tool
call with the same arguments gets repeated — a strong signal of a loop or
confused state.)
</details>

### Q6.20 · Evaluation vs monitoring · [Recall]

What's the distinction between "evaluation" and "monitoring" in an LLM
system's lifecycle?

- **A.** They're the same thing with different names
- **B.** Evaluation is a deliberate, structured measurement (often against a golden set or via A/B test) run to answer "is this change good," typically before or around a launch decision; monitoring is continuous observation of live system health and behavior over time, catching drift or degradation you didn't specifically test for
- **C.** Evaluation only happens in production; monitoring only happens offline
- **D.** Monitoring is a subset of tracing with no relationship to evaluation

<details>
<summary>Answer</summary>

**B**

**Why B:** Evaluation answers a specific question at a specific point
(should we ship this change), typically with a controlled comparison.
Monitoring is ongoing and open-ended — watching key metrics, error rates, and
user feedback continuously to catch problems (model provider updates,
distribution shift, a slow degradation) that no single eval run was designed
to catch. They're complementary layers of the same overall quality system.

**Why not A:** They serve different purposes at different points in time —
treating them as identical loses the distinction that matters for how you'd
design each.

**Why not C:** This reverses the more common association (though not a hard
rule) — evaluation is more often offline/pre-launch and monitoring is
inherently about the live system, though online A/B testing is a form of
evaluation that does happen in production.

**Why not D:** Monitoring depends on good tracing/observability
infrastructure, but it's a broader practice — dashboards, alerting,
feedback aggregation — not merely a subset of trace logging.

**Interviewer's likely follow-up:** *"Give an example of something
monitoring would catch that a pre-launch eval wouldn't."* (Answer: a
third-party model provider silently updates the underlying model and
behavior shifts, or a new class of user query becomes common post-launch
that wasn't in your golden set at all.)
</details>

### Q6.21 · Judge prompt sensitivity · [Applied]

You notice that rephrasing your LLM-judge's instructions slightly (without
changing the actual criteria) shifts the pass rate on your golden set by 6
points. What does this suggest, and what should you do?

- **A.** Nothing — small prompt rephrasing shouldn't affect anything, so this is likely a code bug
- **B.** The judge is sensitive to phrasing, not just criteria — treat the judge prompt itself as something that needs to be version-controlled, tested, and stabilized like any other part of the pipeline, and validate judge reliability (e.g., agreement with human ratings) before trusting its scores
- **C.** This confirms the judge is unreliable and LLM-as-judge should be abandoned entirely
- **D.** Pick whichever phrasing gives the higher pass rate

<details>
<summary>Answer</summary>

**B**

**Why B:** LLM judges, like any LLM call, are sensitive to how they're
prompted — this is a real and well-documented property, not a bug. The
correct response is to treat judge-prompt design with the same rigor as any
other prompt: version it, don't casually edit it, and periodically validate
that its scores correlate with human judgment (a calibration check) so you
have actual confidence in what the number means.

**Why not A:** This sensitivity is a known characteristic of LLM judges, not
evidence of a code bug — dismissing it means you'll keep getting surprised
by future rephrasings.

**Why not C:** Abandoning the technique entirely is an overreaction — the
fix is to harden and validate the judge setup, not throw out a generally
useful and industry-standard tool.

**Why not D:** Cherry-picking the phrasing that produces the higher score is
exactly backwards — you'd be optimizing the measurement to flatter you
rather than to be accurate, defeating the entire purpose of having an eval.

**Interviewer's likely follow-up:** *"How do you validate that a judge is
'good enough' to trust?"* (Answer: periodically sample judge verdicts and
have a human rate the same examples, then measure agreement — if human and
judge diverge often, the judge prompt or model needs work before you lean on
it for gating decisions.)
</details>

### Q6.22 · Evaluating a chatbot with subjective quality · [Design]

You're building eval for a coding-assistant chatbot where "good" partly means
"stylistically fits how a senior engineer would explain it" — a fairly
subjective bar. How do you build a defensible eval for this?

- **A.** Skip evaluation for subjective qualities since they can't be measured
- **B.** Decompose the subjective quality into more concrete, checkable sub-criteria (accuracy of the technical content, appropriate level of detail for the audience, absence of unnecessary hedging, whether it answers the actual question asked) that can each be judged more reliably, then combine them — rather than asking a judge to rate "seniority" directly
- **C.** Only use human evaluators for this file, since LLMs can't judge subjective quality at all
- **D.** Use word count as a proxy for seniority of explanation

<details>
<summary>Answer</summary>

**B**

**Why B:** This is the same principle as Q6.7's rubric design — a vague,
holistic quality dimension becomes more measurable when you break it into
specific, checkable components. "Does it answer the actual question" and
"is technical content accurate" are far more reliably judged (by a human or
an LLM) than an abstract "does this sound senior" rating.

**Why not A:** Skipping evaluation because a quality is subjective gives up
too early — the fix is better decomposition, not abandoning measurement,
especially for a quality this central to the product's value.

**Why not C:** LLM judges can meaningfully assess several of the decomposed
sub-criteria (technical accuracy, whether the question was answered) even if
a human check remains valuable for the most subjective slice — it's not an
all-or-nothing choice.

**Why not D:** Word count has no reliable relationship to explanation
quality or seniority — it's a spurious proxy that would likely reward
padding.

**Interviewer's likely follow-up:** *"What's still hard to fully capture
even after decomposition?"* (Answer: overall coherence and "does this feel
right" gestalt judgments resist full decomposition — that residual is where
periodic human review still earns its keep.)
</details>

### Q6.23 · Cost of evaluation itself · [Design]

Running your full LLM-judge-based regression suite on every pull request
costs $40 in API calls and takes 25 minutes. Engineers have started
skipping it locally and only running it in CI right before merge, and a few
have started merging without waiting for it. What eval-system design change
would most directly address this?

- **A.** Make the suite mandatory via a stricter policy with no technical changes
- **B.** Build a tiered suite: a fast, cheap, deterministic subset (schema/format checks, key fact-presence checks) that runs on every commit locally or in seconds, and reserve the full expensive judge-based suite for pre-merge or nightly runs — so the feedback loop engineers actually experience day-to-day is fast
- **C.** Remove the eval suite from CI entirely and rely on production monitoring instead
- **D.** Increase the cost budget so the suite can run more thoroughly, which will make engineers value it more

<details>
<summary>Answer</summary>

**B**

**Why B:** The behavior described (skipping a slow, expensive check) is a
predictable response to bad feedback-loop ergonomics, not a discipline
problem to be policy-solved. Tiering the suite — fast/cheap checks for rapid
iteration, expensive/thorough checks at the gate that actually matters —
keeps the cost and time where they're tolerable while still catching real
regressions before merge.

**Why not A:** A policy without addressing the actual friction (cost, time)
tends to get worked around anyway, as the scenario already shows happening.

**Why not C:** Removing the pre-merge safety net and relying only on
production monitoring means regressions reach real users before you catch
them — a step backwards from the whole point of a regression suite.

**Why not D:** Making the expensive thing more expensive doesn't fix the
underlying friction that's causing people to skip it — it very plausibly
makes the avoidance worse.

**Interviewer's likely follow-up:** *"What goes in the fast tier vs the
slow tier?"* (Answer: fast tier — deterministic, cheap checks: schema
validity, required-field presence, banned-content regex, latency budget.
Slow tier — anything requiring an LLM judge call or a large golden set pass,
reserved for pre-merge/nightly.)
</details>

### Q6.24 · Interpreting a flat eval score · [Applied]

You shipped a prompt change three weeks ago intended to reduce a specific
failure mode (the assistant refusing valid requests too often). Your overall
eval score on the golden set is unchanged before and after. A teammate
concludes the change had no effect. What should you check before agreeing?

- **A.** Nothing further — an unchanged aggregate score is conclusive
- **B.** Whether the golden set actually contains enough examples of the specific failure mode you were targeting — an aggregate score across a broad set can stay flat even if you meaningfully fixed (or broke) a narrow subcategory, because that subcategory is a small fraction of the total
- **C.** Whether the eval was run on the correct date
- **D.** Whether the prompt change was actually deployed, since an unchanged score always means the deploy failed

<details>
<summary>Answer</summary>

**B**

**Why B:** An aggregate metric can hide a real, meaningful change to a
narrow subcategory if that subcategory is a small slice of the total set —
this is the same failure mode as averaging masking a regression in Q6.10.
The right move is to slice the golden set by category (specifically, the
over-refusal cases) and check the targeted subcategory's score directly,
not just the headline number.

**Why not A:** Treating the aggregate as conclusive is exactly the trap —
you need to check whether the eval set even has statistical power on the
specific thing you changed.

**Why not C:** Nothing in the scenario suggests a scheduling or dating
issue — this is a distraction not supported by the given information.

**Why not D:** An unchanged score doesn't specifically imply the deploy
failed; there are more likely explanations (dilution in the aggregate) worth
checking first, and this option asserts a false certainty ("always means").

**Interviewer's likely follow-up:** *"What would you do differently next
time you make a targeted fix like this?"* (Answer: tag/categorize golden set
examples by failure mode up front, so you can slice and track a targeted
subcategory's score directly instead of relying on an aggregate that dilutes
the signal.)
</details>

### Q6.25 · Evaluation ownership and process · [Design]

Six months into a project, nobody owns the eval set — different engineers
add examples ad hoc when they hit a bug, there's no review process, and two
engineers have started keeping their own private eval scripts because they
don't trust the shared one. What's the most direct fix?

- **A.** Tell engineers to stop making private eval scripts via a Slack message
- **B.** Assign clear ownership of the shared eval set and pipeline (even if informal — a rotating owner or a specific team), establish a lightweight review process for adding/changing golden examples (so quality and coverage stay intentional, not accidental), and consolidate the private scripts' useful examples back into the shared set
- **C.** Delete all the ad hoc examples and start over with a clean, engineer-approved set
- **D.** This is a natural and healthy outcome of engineers caring about quality — no action needed

<details>
<summary>Answer</summary>

**B**

**Why B:** An eval set with no owner and no review process degrades into
exactly what's described — inconsistent coverage and fragmented trust. The
fix mirrors how you'd treat any other shared, critical piece of
infrastructure: assign accountability, add lightweight review so changes are
intentional rather than accidental, and actively recover the value locked in
the private scripts rather than leaving it siloed.

**Why not A:** A policy message without giving people a trustworthy shared
alternative doesn't address *why* they forked off in the first place — they
built private scripts because the shared one wasn't reliable enough for
them to depend on.

**Why not C:** Wholesale deletion throws away potentially valuable
ad hoc examples (often the ones that caught real bugs) instead of curating
and consolidating them — most of that content is worth keeping, just not
worth keeping ungoverned.

**Why not D:** Fragmentation into private, untrusted-by-others eval scripts
is a coordination failure, not a healthy sign — it means the team can no
longer have a shared conversation about whether quality improved, since
everyone's measuring something slightly different.

**Interviewer's likely follow-up:** *"How lightweight should the review
process be, realistically, for a small team?"* (Answer: heavy enough to
catch obviously wrong or duplicate examples and require a second pair of
eyes on category coverage, light enough that it doesn't become its own
bottleneck — a PR review on the golden-set file itself is often enough.)
</details>

---

## Explain prompts

### E6.1 · Explain: why can't you unit-test an LLM feature the normal way

**Prompt:** *"Our QA lead is asking why we can't just write normal unit tests
for the new AI feature the way we do for the rest of the codebase. How do you
explain that to her, and what do you propose instead?"*

**Target:** 60–90 seconds spoken. Answer out loud before opening the rubric.

<details>
<summary>Rubric — score yourself</summary>

**Must hit (1 point each):**
- [ ] States that LLM outputs are non-deterministic / free-text, so exact-match assertions are the wrong invariant
- [ ] Distinguishes "wrong" from "differently phrased but equally correct"
- [ ] Proposes a concrete alternative: property-based checks, structured-output validation, or LLM-as-judge against a rubric
- [ ] Explains that this doesn't mean "no testing" — it means testing at a different layer/invariant
- [ ] Mentions golden datasets or a regression suite as the practical replacement for a unit-test suite

**Bonus — signals strength:**
- [ ] Notes that some parts of the system (retrieval logic, tool schemas, API glue code) absolutely can and should still use normal deterministic unit tests
- [ ] Mentions non-determinism can persist even at temperature 0
- [ ] Frames it in terms QA lead would care about: "same rigor, different technique"

**Red flags — deduct:**
- [ ] Implies AI features simply can't be tested at all
- [ ] Suggests exact-match testing is fine if you just lower temperature enough
- [ ] Dismisses the QA lead's question instead of engaging with it

**Score: ___ / 5**

**Model answer:**
So the honest answer is — normal unit tests assume the same input always
produces the same, exactly-matching output, and that's just not true for a
generative model. Two different phrasings can both be completely correct, so
if I assert on an exact string, I'm going to get flaky failures on outputs
that are actually fine. That's not a testing problem I can solve by tweaking
the model — it's the wrong kind of check for this kind of system. What we do
instead is build a golden set of real examples with known-good answers, and
check properties — does it contain the required facts, does it pass an
LLM-judge rubric, does it validate against a schema — rather than exact
text. And to be clear, that's not "less rigorous" than unit testing, it's
just testing at a different layer. Anything deterministic underneath the
model call — our retrieval code, our tool schemas — still gets normal unit
tests exactly like today.
</details>

### E6.2 · Explain: building your first golden dataset

**Prompt:** *"You're the first person setting up evaluation for a new AI
feature at your company. Walk me through how you'd build the initial golden
dataset from scratch."*

**Target:** 90 seconds spoken.

<details>
<summary>Rubric — score yourself</summary>

**Must hit (1 point each):**
- [ ] Sources real examples (production logs, support tickets, or realistic simulated queries if pre-launch) rather than only spec-derived questions
- [ ] Mentions stratifying across query types/difficulty, not just picking easy or only-hard cases
- [ ] Describes getting human-verified correct answers/labels for each example
- [ ] Gives a reasonable ballpark size and acknowledges it grows over time, not a one-shot deliverable
- [ ] Mentions the set needs ongoing maintenance as the product/docs evolve

**Bonus — signals strength:**
- [ ] Notes including known-hard or previously-failed cases specifically
- [ ] Mentions version-controlling the set
- [ ] Distinguishes "building it" from "who owns keeping it current"

**Red flags — deduct:**
- [ ] Proposes generating the entire set synthetically with no human verification
- [ ] Treats the golden set as a one-time deliverable with no maintenance plan
- [ ] Gives no sense of size or gives an unjustified precise number

**Score: ___ / 5**

**Model answer:**
First thing I'd do is go find real examples, not write them from the spec —
if we've got any production traffic or even a beta group, I want actual user
queries, because real phrasing is messier and more representative than
anything I'd write myself. I'd stratify those across the query types we
expect — easy, common cases, and known-hard edge cases — so the set tells me
both "does the baseline work" and "where does it break." For each one I'd
get a human — probably me plus whoever's closest to the domain — to write or
verify the correct answer, so I've got a real reference to score against.
Size-wise I'd start around a hundred to a few hundred, enough to be
statistically meaningful, and grow it over time rather than treating it as
done. And honestly the part people skip — this needs an owner and a
refresh cadence, because the product changes and the "correct" answers
change with it. A stale golden set quietly becomes useless.
</details>

### E6.3 · Explain: LLM-as-judge biases

**Prompt:** *"We're about to roll out LLM-as-judge for our eval pipeline.
What biases should we be worried about, and how do we mitigate each one?"*

**Target:** 90 seconds spoken.

<details>
<summary>Rubric — score yourself</summary>

**Must hit (1 point each):**
- [ ] Names position bias and its mitigation (swap order / average)
- [ ] Names verbosity bias (favoring longer responses regardless of quality)
- [ ] Names self-preference bias and its mitigation (use a different model as judge than the system under test)
- [ ] Mentions prompt/phrasing sensitivity of the judge itself
- [ ] Recommends periodic calibration against human ratings to validate the judge is trustworthy

**Bonus — signals strength:**
- [ ] Distinguishes pairwise vs absolute scoring reliability
- [ ] Mentions version-pinning the judge model
- [ ] Gives a concrete mitigation recipe, not just a list of bias names

**Red flags — deduct:**
- [ ] Lists biases with no mitigation for any of them
- [ ] Claims LLM judges are basically unbiased if the prompt is good enough
- [ ] Confuses position bias and self-preference bias

**Score: ___ / 5**

**Model answer:**
There's a handful of known ones worth naming specifically. Position bias —
judges tend to favor whichever response is placed first or second regardless
of quality, so for pairwise comparisons I'd run it both orders and check for
a consistent verdict. Verbosity bias — longer responses tend to score higher
even when they're not actually better, so I'd want the rubric to explicitly
not reward length. Self-preference bias — if the judge is the same model
family as what's being judged, it tends to favor its own style, so I'd
always use a different, ideally stronger model as the judge. And then there's
just plain prompt sensitivity — rewording the judge instructions can shift
scores meaningfully even with the same criteria, so I'd version-control the
judge prompt like any other piece of the pipeline. None of these mean don't
use LLM-as-judge — they mean don't trust it blind. I'd periodically sample
judge verdicts and have a human check agreement, so we actually know how
much to trust the number day to day.
</details>

### E6.4 · Explain: pairwise vs absolute — which and when

**Prompt:** *"When would you use pairwise comparison versus absolute
scoring for an LLM eval, and why does it matter?"*

**Target:** 60–90 seconds spoken.

<details>
<summary>Rubric — score yourself</summary>

**Must hit (1 point each):**
- [ ] States pairwise is generally more reliable/consistent for judges than absolute scale anchoring
- [ ] States absolute scoring gives you a trackable number/threshold independent of a specific comparison
- [ ] Gives a concrete scenario favoring pairwise (iterating prompt variants against a current best)
- [ ] Gives a concrete scenario favoring or requiring absolute (needing a fixed launch bar, or trend tracking over a long period)
- [ ] Notes a downside of relying purely on pairwise (no absolute sense of "good enough")

**Bonus — signals strength:**
- [ ] Mentions combining both — pairwise for iteration, periodic absolute/human check for launch bar
- [ ] Notes scale drift as a specific risk of pure absolute scoring over time

**Red flags — deduct:**
- [ ] Claims one is strictly better with no tradeoff
- [ ] Can't give a concrete scenario for either

**Score: ___ / 5**

**Model answer:**
Honestly I lean pairwise for day-to-day iteration, because judges are just
more consistent at "is A better than B" than they are at anchoring an
absolute one-to-five score — a four today and a four next month might not
mean the same thing if there's no fixed reference. So if I'm iterating on a
prompt and comparing each new candidate against my current best, pairwise
is the more trustworthy signal. But pairwise alone doesn't tell you if
you've actually hit "good enough" — you could win every comparison while
every candidate is still mediocre. So for a launch decision, or if I need a
trend line I can track over months, I do want some absolute or
human-anchored bar too. In practice I'd use both — pairwise to drive
iteration, and a periodic absolute or human check before anything ships.
</details>

### E6.5 · Explain: designing a rubric for a subjective quality

**Prompt:** *"Product wants to measure whether our assistant's tone is
'empathetic.' That's pretty subjective. How would you turn that into
something you can actually evaluate?"*

**Target:** 90 seconds spoken.

<details>
<summary>Rubric — score yourself</summary>

**Must hit (1 point each):**
- [ ] Proposes decomposing "empathetic" into specific, checkable sub-behaviors rather than judging the abstract quality directly
- [ ] Gives at least two concrete example sub-criteria (e.g., acknowledges the user's stated frustration, doesn't dismiss the concern, doesn't over-apologize to the point of sounding insincere)
- [ ] Notes this makes scoring more consistent/reliable across a judge or multiple human raters
- [ ] Mentions this also makes the score diagnostic — you learn what specifically to fix
- [ ] Acknowledges some residual subjectivity remains even after decomposition

**Bonus — signals strength:**
- [ ] Mentions calibrating the rubric against real examples humans agree are/aren't empathetic before trusting it
- [ ] Notes red flags matter as much as positive criteria (e.g., dismissive phrasing, false urgency)

**Red flags — deduct:**
- [ ] Proposes just asking a judge to rate "empathy" 1-10 with no decomposition
- [ ] Claims subjective qualities can't be measured at all

**Score: ___ / 5**

**Model answer:**
I wouldn't try to score "empathetic" directly — that's too fuzzy to be
consistent, whether it's a human or an LLM judge doing the rating. What I'd
do is break it into specific things I can actually check: does it
acknowledge what the user said before jumping to a solution, does it avoid
language that sounds dismissive, does it use an appropriately warm but not
over-the-top tone. Each of those is something a judge — or a human — can
check much more reliably than an abstract "rate the empathy" question. And
the nice side effect is it's diagnostic — if the score's low, I know exactly
which behavior is missing instead of just knowing "tone bad." I'd still want
to sanity-check the rubric itself against some examples a human clearly
agrees are or aren't empathetic, because even with decomposition there's a
residual subjective judgment call at the edges that I can't fully engineer
away.
</details>

### E6.6 · Explain: setting up a regression suite for prompts

**Prompt:** *"Walk me through how you'd design a CI regression suite so
that prompt changes can't silently break the product."*

**Target:** 90 seconds spoken.

<details>
<summary>Rubric — score yourself</summary>

**Must hit (1 point each):**
- [ ] Runs the fixed golden set against every prompt change automatically
- [ ] Compares against a stored baseline, not just an absolute pass/fail
- [ ] Sets a concrete gating threshold or flags specific critical-example regressions
- [ ] Notes using property/judge-based comparison rather than exact-string diff, given non-determinism
- [ ] Addresses cost/speed tradeoff — mentions tiering fast deterministic checks vs slower judge-based checks

**Bonus — signals strength:**
- [ ] Mentions diffing individual examples, not just the aggregate score
- [ ] Mentions this needs an owner/maintenance process, tying back to golden-set upkeep

**Red flags — deduct:**
- [ ] Proposes exact string-match diffing as the comparison method
- [ ] No mention of a concrete gate/threshold — just "look at the score"

**Score: ___ / 5**

**Model answer:**
Every prompt change would run the fixed golden set automatically and compare
against the last known-good baseline — not just spitting out a raw number,
but flagging both an aggregate score drop past some threshold and any
specific example that used to pass and now fails, because averages can hide
a real regression in a small but important slice. And since outputs aren't
deterministic, I wouldn't diff exact strings — I'd use the same kind of
property or judge-based check we use for eval generally. The other thing
I'd think about up front is cost — a full judge-based run on every commit
gets expensive and slow fast, so I'd split it: fast, cheap, deterministic
checks like schema validation on every commit, and the full judge-based
suite pre-merge or nightly. Otherwise people just start skipping it, which
defeats the whole point.
</details>

### E6.7 · Explain: online vs offline evaluation, and when each fails you

**Prompt:** *"Why isn't offline evaluation enough on its own? What does
online evaluation catch that it can't?"*

**Target:** 60–90 seconds spoken.

<details>
<summary>Rubric — score yourself</summary>

**Must hit (1 point each):**
- [ ] Defines offline eval as controlled, fixed-dataset measurement
- [ ] Defines online eval as measurement against real traffic/users post-deployment
- [ ] States offline eval is bounded by what's in the golden set — can't cover unanticipated real-world distribution
- [ ] Gives a concrete example of something only online eval would catch (real user satisfaction, distributional drift, an edge case not anticipated)
- [ ] States they're complementary, not either/or

**Bonus — signals strength:**
- [ ] Mentions A/B testing as the standard online evaluation method for generative features
- [ ] Notes offline eval is cheaper/faster/pre-launch, online is slower but more representative

**Red flags — deduct:**
- [ ] Claims a big enough golden set makes online eval unnecessary
- [ ] Can't give a concrete example of an online-only failure mode

**Score: ___ / 5**

**Model answer:**
Offline eval is great because it's controlled and repeatable — same golden
set, same comparison, every time. But it's bounded by whatever's in that
set, and no matter how good your set is, real users are going to ask things
you didn't anticipate, in phrasing you didn't anticipate, in combinations
you didn't test. Online eval — A/B tests, live monitoring, actual user
feedback — is how you catch the stuff your golden set structurally can't
cover, like a model provider quietly updating behavior, or a whole new
category of query becoming common after launch. I wouldn't treat them as
competing — offline is your fast, cheap, pre-launch gate, and online is your
slower but more honest signal for whether things are actually going well in
the real world. You want both layers, not one instead of the other.
</details>

### E6.8 · Explain: what to do when eval and user feedback disagree

**Prompt:** *"Your offline eval says a new prompt version is better. Users
are complaining more since it shipped. Walk me through how you'd handle
this."*

**Target:** 90 seconds spoken.

<details>
<summary>Rubric — score yourself</summary>

**Must hit (1 point each):**
- [ ] Treats the disagreement as evidence the eval set has a blind spot, not evidence users are wrong
- [ ] Proposes investigating the actual complaint cases for a pattern
- [ ] Considers a rollback as a mitigation step separate from the investigation
- [ ] Proposes using findings to improve/expand the golden set going forward
- [ ] Doesn't dismiss either signal outright

**Bonus — signals strength:**
- [ ] Distinguishes "roll back now, investigate after" from "investigate before deciding," and picks one with reasoning based on severity
- [ ] Mentions this is a recurring, expected part of running an eval system, not a one-off failure

**Red flags — deduct:**
- [ ] Trusts the offline number over real user signal with no further investigation
- [ ] Immediately blames users' feedback as unreliable/noisy without checking

**Score: ___ / 5**

**Model answer:**
First instinct is not to trust the eval number over what real users are
telling me — if there's a real disagreement, that's a sign my golden set
has a blind spot, not that the users are wrong. Depending on how bad the
complaints are, I'd probably roll back to the known-good version first just
to stop the bleeding, and then go dig into the actual complaint cases
separately — look for a pattern, a query type or topic the golden set didn't
have good coverage of. Once I find that pattern, I'd add representative
examples to the golden set so this exact gap gets caught next time before it
ships, not after. The eval set failing to predict this isn't a one-off
embarrassment, it's just what happens when a set doesn't fully match the
real distribution — the response is to close the gap, not to defend the
number.
</details>

### E6.9 · Explain: evaluating a multi-step agent, not just its output

**Prompt:** *"We only score our research agent on its final report quality.
What are we missing, and how would you fix the eval?"*

**Target:** 90 seconds spoken.

<details>
<summary>Rubric — score yourself</summary>

**Must hit (1 point each):**
- [ ] States final-output-only scoring misses process quality (efficiency, redundant/wrong tool calls, recovery from mistakes)
- [ ] Notes a good final output can mask an expensive or fragile process
- [ ] Proposes concrete step-level metrics (tool-call count vs minimum needed, error/retry rate, repeated identical calls as a loop signal)
- [ ] Connects this to cost/latency/reliability concerns, not just correctness
- [ ] States both final-output and step-level signals are needed together

**Bonus — signals strength:**
- [ ] Notes this connects to agent failure modes (infinite loops, cost runaway) from file 05
- [ ] Mentions tracing/observability as the infrastructure prerequisite for step-level eval

**Red flags — deduct:**
- [ ] Claims final-output quality is sufficient on its own
- [ ] Can't propose a concrete step-level metric

**Score: ___ / 5**

**Model answer:**
Final report quality alone hides a lot — an agent can stumble through three
wrong tool calls, get lucky, and still produce a great report, and that
final score won't show you any of that. But that hidden process is exactly
what drives your cost and latency in production, and it's the thing most
likely to break when the task shifts slightly. So I'd add step-level
metrics alongside the output score — how many tool calls did it take versus
a reasonable minimum, how often did calls error or need retrying, and
specifically whether it's calling the same tool with the same arguments
repeatedly, which is usually a strong loop signal. None of that replaces
scoring the final report — you need both. Final output tells you if the
user got a good result; step-level tells you whether that result was
efficient and reliable enough to trust at scale.
</details>

### E6.10 · Explain: your eval philosophy for a system you don't own yet

**Prompt:** *"You're joining a team that's shipped an AI feature with zero
formal evaluation — just vibes and a few Slack screenshots of good outputs.
What's your first-month plan?"*

**Target:** 90 seconds spoken.

<details>
<summary>Rubric — score yourself</summary>

**Must hit (1 point each):**
- [ ] Starts by building a golden set from real usage/logs before making any changes
- [ ] Establishes a baseline measurement of current behavior before proposing improvements
- [ ] Sets up at minimum a lightweight regression check so future changes are measured, not just shipped on vibes
- [ ] Prioritizes based on risk/impact rather than trying to build a perfect eval system immediately
- [ ] Frames this as incremental — doesn't propose boiling the ocean in month one

**Bonus — signals strength:**
- [ ] Mentions talking to whoever's closest to users/support tickets to source real failure examples
- [ ] Mentions establishing eval-set ownership from the start, learning from Q6.25's failure mode

**Red flags — deduct:**
- [ ] Proposes a huge, comprehensive eval framework as a month-one deliverable with no incremental path
- [ ] Dismisses the existing "vibes" process without proposing something concrete to replace it

**Score: ___ / 5**

**Model answer:**
Honestly the first thing I'd do isn't build anything fancy — it's get a
golden set together from whatever real usage or logs exist, even if it's
small, and use it to establish a baseline of how the system's actually
doing right now, not how we think it's doing from a few good screenshots.
Then I'd get a minimal regression check running — even a handful of
automated property checks — so that from that point forward, at least, no
change ships without some signal. I wouldn't try to build the full eval
system in month one — I'd prioritize by risk, so if there's a category of
query that's clearly high-stakes or high-volume, that gets covered first. I'd
also want to talk to whoever's closest to users or support, because they'll
know the actual failure patterns way faster than I'll find them by guessing.
The goal for month one realistically is "we can now tell if a change made
things better or worse," not "we have a perfect eval harness."
</details>

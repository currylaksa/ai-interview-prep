# 02 · LLM API Mechanics

This file covers the mechanics of actually calling an LLM API in production: how
tokenisation, context windows, sampling, streaming, caching, and rate limits behave,
and — because this is one of the areas you'll be judged hardest on with no
production experience to lean on — the cost and latency arithmetic that turns
"it works in the demo" into "it works at 50,000 requests/day." Do the calculation
questions on paper or out loud; skipping straight to the answer defeats the point.

---

## Multiple choice

### Q2.1 · Tokenisation basics · [Recall]

A teammate says "the input is 500 words, so it should be about 500 tokens." Why is
this a bad rule of thumb for English text sent to a modern LLM?

- **A.** Tokenisers merge punctuation into the previous word, so token counts are always lower than word counts
- **B.** Common subword tokenisers typically produce roughly 1.3–1.5 tokens per word for English, because many words split into multiple subword pieces
- **C.** Tokens only count whitespace-separated words, so the estimate is actually exact
- **D.** Token counts depend on the model's training data size, not on the input text

<details>
<summary>Answer</summary>

**B**

**Why B:** Subword tokenisers (BPE-style) break words into pieces based on
frequency in training data — common short words often stay whole, but longer or
rarer words, numbers, punctuation-heavy text, and non-English text split into
several tokens. For typical English prose, ~1.3 tokens/word is a workable
estimate, not 1:1.

**Why not A:** Punctuation is often its own token or attached inconsistently —
there's no rule that it always reduces the count below the word count.

**Why not C:** Tokens are subword units, not word units — this is the whole point
of the question.

**Why not D:** Tokenisation is fixed by the tokeniser/vocabulary the model was
trained with, applied deterministically to input text — it doesn't vary by
training data size at inference time.

**Interviewer's likely follow-up:** *"Where does this bite you in practice?"*
(Answer: cost and context-budget estimates that are computed from word counts
undercount actual usage, sometimes badly enough to blow a context window you
thought had headroom — especially with code, JSON, or non-English text, which
tokenise less efficiently than prose.)
</details>

---

### Q2.2 · Tokenisation and code · [Applied]

You're estimating token usage for a feature that sends chunks of source code to an
LLM. Your estimate, based on the same words-to-tokens ratio you use for prose, comes
in consistently low. What's the most likely reason?

- **A.** Code uses more whitespace than prose, and whitespace is free
- **B.** Code has a higher density of symbols, indentation, and rare identifiers, which tokenise less efficiently than natural-language words
- **C.** The model applies a separate, cheaper tokeniser for code inputs
- **D.** Your estimate is fine — the discrepancy is measurement error in the token counter

<details>
<summary>Answer</summary>

**B**

**Why B:** Prose ratios are calibrated on natural language, where common words
are often single tokens. Code has camelCase/snake_case identifiers, brackets,
operators, and indentation whitespace — much of it rare or irregular relative to
the tokeniser's training distribution — so it tends to tokenise at a *higher*
tokens-per-character rate than prose.

**Why not A:** Whitespace still consumes tokens (often merged with adjacent
characters, but not free) — and indentation-heavy code has more of it, not less.

**Why not C:** There's one tokeniser per model, applied uniformly regardless of
content type.

**Why not D:** A consistent, one-directional gap is a systematic estimation
error, not noise — dismissing it as "measurement error" would be exactly the
kind of unexamined answer that loses points here.

**Interviewer's likely follow-up:** *"How would you get a better estimate before
shipping?"* (Answer: count tokens on representative real samples using the
provider's tokenizer/count endpoint rather than a words × ratio heuristic,
especially for any input type — code, JSON, non-English — that isn't plain
English prose.)
</details>

---

### Q2.3 · Context window limit behaviour · [Applied]

Your application sends a growing conversation history with every turn. On turn 40,
the API call fails with an error instead of silently truncating. What does this
tell you about how the context window is enforced?

- **A.** The model automatically drops the oldest messages once the limit is hit, and the error is unrelated
- **B.** The provider validates total input tokens against the model's context limit before generation and rejects the request rather than silently truncating your input
- **C.** Context windows only apply to output tokens, so this must be a rate-limit error
- **D.** The API randomly samples which tokens to keep when the limit is exceeded

<details>
<summary>Answer</summary>

**B**

**Why B:** Context limits are enforced as a hard boundary on total tokens
(input + expected output budget, depending on the API). Exceeding it produces an
explicit error — the API does not silently drop or reorder your input, because
doing so invisibly would be far more dangerous than failing loudly.

**Why not A:** There's no built-in "forget the oldest turns" behaviour — that's
application-level logic you'd have to implement yourself (windowing, summarising,
or truncating history).

**Why not C:** Context limits bound total tokens across input and output
together (or input alone, depending on the provider) — not output only. A
rate-limit error would carry different error semantics (e.g. HTTP 429) than a
context-length error.

**Why not D:** No provider does silent random sampling of your prompt — that
would make behaviour non-reproducible in a way no one could debug.

**Interviewer's likely follow-up:** *"So how do you handle turn 40 in
production?"* (Answer: proactively manage context before hitting the limit —
sliding window of recent turns, periodic summarisation of older turns, or
retrieval of only relevant history — rather than reacting to the error.)
</details>

---

### Q2.4 · Temperature vs top-p · [Applied]

A teammate sets `temperature=0` for a classification task and separately sets
`top_p=0.1` for a creative-writing task, and asks you which one actually matters
more once the other is at a "restrictive" setting. What's the accurate answer?

- **A.** They stack independently and multiplicatively, so both always matter equally
- **B.** With `temperature=0`, the model becomes effectively deterministic (always picks the highest-probability token), making `top_p` irrelevant at that call; with a low `top_p` at non-zero temperature, sampling is restricted to a small nucleus of high-probability tokens before temperature is applied
- **C.** `top_p` only affects output length, not token selection, so it's unrelated to temperature
- **D.** Setting both parameters at once is invalid and most APIs will reject the request

<details>
<summary>Answer</summary>

**B**

**Why B:** `temperature=0` collapses sampling to greedy/argmax decoding — there's
no distribution left for `top_p` to restrict, so it becomes a no-op at that call.
At non-zero temperature, `top_p` (nucleus sampling) truncates the candidate token
set to the smallest set whose cumulative probability exceeds `p` *before*
temperature reshapes that distribution — the two parameters interact rather than
apply independently.

**Why not A:** They don't stack multiplicatively — one can make the other
irrelevant, as shown above.

**Why not C:** `top_p` restricts which tokens are eligible for sampling at each
step, not the number of steps (length) generated.

**Why not D:** Most APIs accept both parameters simultaneously; it's just that
one can dominate depending on values chosen.

**Interviewer's likely follow-up:** *"When would you actually want to tune both
instead of just temperature?"* (Answer: rarely — most teams pick one knob.
`top_p` can help when you want to exclude a long tail of low-probability tokens
while still allowing some randomness among plausible options, e.g. varied but
sane phrasing in user-facing copy.)
</details>

---

### Q2.5 · Message roles · [Recall]

Why do most chat-completion APIs distinguish `system`, `user`, and `assistant`
roles instead of just sending one flat block of text?

- **A.** It's purely a UI convention — the model receives roles as metadata but ignores them during generation
- **B.** The roles let the model distinguish standing instructions from conversational turns, and let you reconstruct multi-turn state on every call since the API itself doesn't retain memory between requests
- **C.** Each role is processed by a separate specialised model internally
- **D.** Roles exist only to support function/tool calling and have no effect otherwise

<details>
<summary>Answer</summary>

**B**

**Why B:** The model is trained to treat `system` as standing behavioural
instructions, `user` as the human's input, and `assistant` as its own prior
turns — this structure is baked into training, not just a labelling convention.
Because the API is stateless, roles are also how you reconstruct "what has been
said so far" on every single call.

**Why not A:** Roles materially affect how the model weighs instructions vs
conversational content — a system-role instruction and the same text as a
user-role message can produce different behaviour.

**Why not C:** One model processes the whole formatted context; there's no
role-specific sub-model.

**Why not D:** Roles are foundational to ordinary chat behaviour, independent of
whether tools are involved at all.

**Interviewer's likely follow-up:** *"What happens if you put your system prompt
in the user role instead?"* (Answer: it usually still works to a degree, since
the model can infer intent from content, but it competes with conversational
content for weight and is less reliably followed — especially for instructions
you want to hold firm across many turns.)
</details>

---

### Q2.6 · Statelessness · [Applied]

Your chatbot backend stores conversation history in a database and, on every user
message, sends the full history array back to the LLM API. A junior engineer asks
why you don't just send the new message, since "the model already saw the earlier
ones." How do you explain this?

- **A.** The API is stateless — it has no memory of prior calls, so every call must include the full conversation context you want the model to reason over
- **B.** The model does retain memory between calls, but only for 5 minutes, so this is just a safety margin
- **C.** Sending only the new message is correct and more efficient — the extra history is redundant
- **D.** The history needs to be resent only if the conversation includes tool calls

<details>
<summary>Answer</summary>

**A**

**Why A:** Each API call is independent and self-contained. Nothing persists
server-side tied to "this conversation" unless the provider offers an explicit
stateful/conversation feature (and even then, that's the provider storing and
replaying history for you — the underlying generation call is still fed full
context). If you don't resend prior turns, the model has no way to know they
happened.

**Why not B:** There is no default memory window between ordinary completion
calls — this describes a feature that doesn't exist by default.

**Why not C:** This would cause the model to respond to the new message with no
awareness of anything said before it — breaking multi-turn coherence entirely.

**Why not D:** Statelessness applies to all calls uniformly, tool calls or not.

**Interviewer's likely follow-up:** *"What's the cost implication of this?"*
(Answer: as a conversation grows, you're re-sending — and being billed for —
the entire history on every turn, which is exactly the problem prompt caching is
designed to reduce; see Q2.11.)
</details>

---

### Q2.7 · Streaming and TTFT · [Applied]

Your product team wants the chat UI to "feel fast." The generation itself takes 8
seconds end-to-end whether or not you stream. What does enabling streaming actually
improve?

- **A.** Total generation time — streaming makes the model produce tokens faster
- **B.** Time-to-first-token — the user sees output start appearing almost immediately, even though total completion time is unchanged, which improves perceived latency
- **C.** Token cost — streamed responses are billed at a lower per-token rate
- **D.** Output quality — streaming allows the model to revise earlier tokens as it generates later ones

<details>
<summary>Answer</summary>

**B**

**Why B:** Streaming changes *when* you receive tokens, not how fast the model
computes them. Total wall-clock time to the last token is roughly the same; what
improves is time-to-first-token (TTFT) — the user perceives responsiveness
because something appears on screen almost immediately instead of after an 8-second
blank wait.

**Why not A:** Generation speed is a function of model size, hardware, and
output length — streaming doesn't change the underlying compute.

**Why not C:** Streaming and non-streaming calls are billed the same way per
token in virtually all providers; streaming is a delivery mechanism, not a
pricing tier.

**Why not D:** Generation is still autoregressive and left-to-right regardless of
streaming — earlier tokens are not revised once emitted, streamed or not.

**Interviewer's likely follow-up:** *"When would you deliberately NOT stream?"*
(Answer: when you need to post-process, validate, or parse the complete response
before showing anything — e.g. enforcing structured output/JSON schema
validation, since a partial JSON blob mid-stream isn't valid on its own.)
</details>

---

### Q2.8 · Structured output · [Applied]

You ask the model to "return JSON" via prompt instructions alone (no schema
enforcement feature), and about 3% of production responses fail to parse. What's
the most robust fix?

- **A.** Lower the temperature to 0 — that alone guarantees valid JSON
- **B.** Use the provider's structured-output/JSON-mode or schema-constrained generation feature, which constrains the token sampling to only produce schema-valid output, rather than relying on the model to follow free-text instructions
- **C.** Add the word "IMPORTANT" in capital letters before the JSON instruction
- **D.** Retry every request twice regardless of whether the first one succeeded, to average out failures

<details>
<summary>Answer</summary>

**B**

**Why B:** Prompt-only instructions rely on the model choosing to comply — it
can still emit stray prose, trailing commas, or unescaped characters. Schema-
constrained decoding (JSON mode / structured output / tool-call-style forced
output) restricts the token sampling process itself so invalid tokens can't be
selected, which is a much stronger guarantee than instruction-following.

**Why not A:** Lowering temperature makes output more deterministic but doesn't
enforce syntactic validity — a temperature-0 model can still confidently produce
malformed JSON if nothing constrains its token choices.

**Why not C:** Emphasis in the prompt is still just an instruction the model can
ignore under some inputs — it doesn't change the generation mechanism.

**Why not D:** Blind retries waste cost and latency on requests that would have
succeeded anyway, and don't address inputs that reliably fail.

**Interviewer's likely follow-up:** *"What do you do for the residual failures
even with structured output enabled?"* (Answer: validate the parsed response
against your schema anyway, and have a retry-with-error-feedback path — most
providers still allow edge-case failures, e.g. from truncation at a token limit,
so validate-then-retry, don't trust blindly.)
</details>

---

### Q2.9 · Rate limits · [Applied]

Your service starts receiving `429` responses from the LLM API during a traffic
spike. Your current code treats this the same as a `500` and retries immediately in
a tight loop. What's wrong with that, and what should you do instead?

- **A.** Nothing is wrong — immediate retries are correct because rate limits reset instantly
- **B.** Immediate retries add more load exactly when the provider is telling you to back off, which can worsen the throttling and burn your quota faster; you should back off (ideally reading a `Retry-After` header if provided) and add jitter, plus consider request queuing/concurrency limits on your side
- **C.** 429s should never be retried at all — the request should be dropped permanently
- **D.** 429 means the API key is invalid and needs to be rotated

<details>
<summary>Answer</summary>

**B**

**Why B:** A 429 is an explicit signal that you're exceeding a rate or quota
limit — hammering it with immediate retries is the opposite of the correct
response and can extend the throttling window. Respecting `Retry-After` (if
present), backing off with jitter, and controlling your own outbound
concurrency (e.g. a semaphore or queue) are the standard fixes.

**Why not A:** Rate limits are typically windowed (e.g. per minute), not
instantaneous — retrying immediately usually just hits the same limit again.

**Why not C:** 429s are often transient and legitimately retryable once you
back off — dropping the request outright is unnecessarily lossy for a spike that
may pass in seconds.

**Why not D:** A 429 is a distinct status from `401`/`403` (auth errors) — it
specifically indicates rate/quota limiting, not invalid credentials.

**Interviewer's likely follow-up:** *"How would you design for this
proactively, not just reactively?"* (Answer: client-side rate limiting /
request queuing that stays under your known quota, plus monitoring on 429 rate
as a leading indicator you're near capacity — don't wait to discover the limit
via user-facing failures.)
</details>

---

### Q2.10 · Batch APIs · [Recall]

A data pipeline needs to classify 200,000 historical support tickets overnight, with
no requirement for a response within seconds. Why would a batch API be the right
choice over the standard synchronous endpoint?

- **A.** Batch APIs use a fundamentally different, more accurate model
- **B.** Batch APIs trade guaranteed low latency (results may take hours) for typically substantial cost discounts and higher aggregate throughput, which fits workloads with no real-time requirement
- **C.** Batch APIs are required whenever a request contains more than one message
- **D.** Batch APIs bypass context window limits entirely

<details>
<summary>Answer</summary>

**B**

**Why B:** Batch processing lets the provider schedule your requests flexibly
against spare capacity, which is why it's typically offered at a meaningful
discount versus the synchronous API — in exchange, you give up latency
guarantees (results often land within a window measured in hours, not seconds).
That trade is exactly right for a large, non-interactive overnight job.

**Why not A:** Batch and synchronous APIs generally hit the same underlying
model — the difference is scheduling and pricing, not model quality.

**Why not C:** Multi-message conversations run fine on the synchronous endpoint;
batch is about volume and latency tolerance, not message count.

**Why not D:** Context window limits are a property of the model, unaffected by
which API surface (batch vs synchronous) you call it through.

**Interviewer's likely follow-up:** *"What's the failure mode you'd design
around with a batch job?"* (Answer: partial failures within a large batch —
you need per-item error handling and idempotent reprocessing of just the failed
subset, not an all-or-nothing rerun of 200,000 items.)
</details>

---

### Q2.11 · Prompt caching mechanics · [Recall]

Your system prompt and a large block of reference documentation together total
6,000 tokens, sent identically on every one of thousands of daily requests, with
only a short user question changing each time. What makes this a strong candidate
for prompt caching?

- **A.** Caching works on any input regardless of whether it repeats, so this detail is irrelevant
- **B.** The large, unchanging prefix can be cached after its first use, so subsequent requests that share that exact prefix are billed at a reduced rate for the cached portion instead of full price for those tokens every time
- **C.** Caching only reduces latency, never cost
- **D.** Caching requires the entire request, including the user's question, to be byte-identical to a previous request

<details>
<summary>Answer</summary>

**B**

**Why B:** Prompt caching keys off a shared, identical prefix — here, the system
prompt + reference docs. Once that prefix has been processed once, subsequent
calls that reuse it exactly can skip reprocessing it and are billed at a
significantly reduced rate for the cached tokens, while the changing suffix
(the user's question) is processed and billed normally. This scenario — large
static prefix, small varying suffix, high call volume — is the textbook case
caching is built for.

**Why not A:** Caching specifically depends on repetition of an identical
prefix — a wholly unique prompt every time gets no benefit.

**Why not C:** Reduced reprocessing typically improves both cost *and*
latency (TTFT), since cached tokens don't need to be reprocessed through the
model — cost reduction is usually the headline benefit providers advertise.

**Why not D:** Only the shared prefix needs to match exactly; the varying
suffix (e.g. the user question) is expected to differ and is priced separately.

**Interviewer's likely follow-up:** *"What breaks the cache?"* (Answer: any
change to the cached prefix — even one character in the system prompt or
reordering the reference docs — invalidates the match, so caching rewards
keeping shared context stable and put first/unchanged across calls.)
</details>

---

### Q2.12 · Cache invalidation cost · [Applied]

A team stores a 10,000-token knowledge-base snippet as a cached prefix, but a
teammate keeps a habit of appending a live timestamp to that same prefix "for
debugging," believing it's harmless since it's a small addition. What's actually
happening to their caching strategy?

- **A.** Nothing — small additions don't affect caching, since the cache is fuzzy-matched
- **B.** The timestamp makes the prefix different on every single call, so it never matches a previous cache entry — the entire 10,000-token block is reprocessed at full price on every request, silently destroying the intended savings
- **C.** The cache simply ignores the timestamp field automatically
- **D.** Caching is only affected by changes in the last 100 tokens, so a prefix-level timestamp is safe

<details>
<summary>Answer</summary>

**B**

**Why B:** Prompt caching typically requires an exact prefix match. A
timestamp — by definition different every call — placed inside or before the
cached block breaks the match every time, so the "cached" prefix is fully
reprocessed and billed at full rate on every request. The team believes they're
caching; they're paying full price silently, which is a worse outcome than never
having built caching at all, because no one notices.

**Why not A / C:** Caching is exact-match, not fuzzy or field-aware — there's no
mechanism that "ignores" a varying substring inside the cached region.

**Why not D:** Match sensitivity isn't about position within the last N tokens —
any change anywhere in the intended-cached prefix can break the match, depending
on where the cache boundary is set.

**Interviewer's likely follow-up:** *"How would you catch this in production
before it costs real money?"* (Answer: monitor cache hit rate as an explicit
metric, not just total cost — a sudden or persistent drop to near-zero hit rate
is a clear signal something in the "static" prefix isn't actually static.)
</details>

---

### Q2.13 · Sampling determinism · [Applied]

QA reports that the same prompt, sent twice with `temperature=0`, occasionally
produces slightly different output. A teammate says this proves temperature=0
"doesn't really work." How do you respond?

- **A.** They're right — temperature=0 is documented as fully deterministic with zero exceptions on every provider
- **B.** Temperature=0 makes sampling greedy (always the highest-probability token), but it doesn't guarantee bit-for-bit determinism across calls — floating-point non-associativity in parallel GPU computation and backend/infrastructure variance can still cause tiny numerical differences that occasionally flip which token has the highest probability
- **C.** This can only happen if the prompt itself changed between calls
- **D.** Determinism is only guaranteed if `top_p` is also set to 0

<details>
<summary>Answer</summary>

**B**

**Why B:** `temperature=0` removes *intentional* randomness from token
selection, but doesn't eliminate all sources of variation — batched inference,
parallel GPU execution, and floating-point rounding are not strictly
order-independent, so on rare occasions two near-tied logits can resolve
differently between runs. It's a real, known limitation, not a myth.

**Why not A:** This overstates the guarantee — providers generally describe
temperature=0 as "mostly deterministic" or note best-effort determinism, not an
absolute guarantee, precisely because of infrastructure-level variance.

**Why not C:** The scenario states the prompt is identical — the variation
described here has a different root cause (compute-level, not input-level).

**Why not D:** `top_p=0` is generally invalid/undefined in most APIs (nucleus of
size zero), and isn't the mechanism controlling this behaviour — greedy decoding
comes from temperature, not top_p.

**Interviewer's likely follow-up:** *"Does this matter for your evaluation
strategy?"* (Answer: yes — it means you can't unit-test LLM output for exact
string equality even at temperature=0; you need tolerance-based or
property-based checks, which connects directly to file 06's evaluation
content.)
</details>

---

### Q2.14 · Context rot / long-context degradation · [Design]

You're deciding whether to stuff an entire 80-page manual into context on every
call versus retrieving only relevant sections. The model's context window
technically fits the whole manual. Why might "it fits" not be a good enough reason
to do it?

- **A.** Context windows silently shrink over time as a model ages
- **B.** Model attention isn't uniform across a very long context — relevant details can get diluted or under-weighted among a large volume of irrelevant text, and every token in the stuffed context still costs money and adds latency on every call
- **C.** Sending more than 10,000 tokens is against most providers' terms of service
- **D.** Long contexts always get truncated silently by the API regardless of the stated limit

<details>
<summary>Answer</summary>

**B**

**Why B:** "Fits in the context window" is a necessary but not sufficient
condition. Real-world behaviour (often summarised as "lost in the middle" /
context-rot effects) shows retrieval-relevant accuracy can degrade as
irrelevant volume grows, even within the stated limit — plus every token you
include, relevant or not, is billed and adds latency on every single call,
compounding over high request volume. Retrieving only what's relevant (RAG) is
often both cheaper and more accurate than "just include everything."

**Why not A:** Context windows are a fixed model property, not something that
degrades over time.

**Why not C:** This is a fabricated constraint — providers publish explicit
token limits; there's no separate informal policy against using them.

**Why not D:** Silent truncation isn't standard behaviour (see Q2.3) — most
APIs reject over-limit requests explicitly rather than truncating quietly.

**Interviewer's likely follow-up:** *"So when WOULD you just stuff the whole
document in, instead of retrieving?"* (Answer: when the corpus is small enough
that the cost/latency hit is negligible, when you need the model reasoning
across the whole document rather than isolated chunks, or when retrieval quality
itself is the harder problem to get right — long-context and RAG are tools, not
a strict hierarchy.)
</details>

---

### Q2.15 · System prompt vs first user message · [Design]

You're designing an API wrapper for a customer-facing product. Should
product-wide behavioural rules ("always respond in the customer's language,
never discuss pricing of competitors") live in the system prompt or be prepended
to every user message?

- **A.** It doesn't matter functionally — pick whichever is easier to template
- **B.** The system prompt is the intended mechanism for standing, product-wide instructions — models are trained to weight it more heavily and consistently than equivalent text placed in the user turn, and it keeps user-turn content focused on the actual per-request input
- **C.** Behavioural rules must go in the user message because system prompts are stripped before generation
- **D.** Both must be duplicated in both places or the model will ignore them entirely

<details>
<summary>Answer</summary>

**B**

**Why B:** This is exactly the design purpose of the role separation from
Q2.5 — standing instructions belong in `system`, because that's the channel
models are trained to treat as higher-priority, persistent behaviour, and it
keeps your per-request user content clean and easy to reason about
independently.

**Why not A:** The two placements are not functionally equivalent — instruction
adherence and robustness differ, which is the whole reason the roles exist.

**Why not C:** System prompts are actively used in generation, not stripped —
this option describes something that isn't true of any mainstream provider.

**Why not D:** Duplication isn't required, and doing it needlessly wastes
tokens (and cache-hit potential — see Q2.11) without a compensating benefit.

**Interviewer's likely follow-up:** *"What's a case where you'd still put
some rules in the user turn instead?"* (Answer: per-request context that
varies by call — e.g. "the customer's detected language is French" — belongs in
the user turn or a dynamic prefix, since it changes per request rather than
being a fixed product-wide rule.)
</details>

---

### Q2.16 · Calculation — monthly cost estimate · [Applied]

Your feature sends an average of 1,200 input tokens and receives 300 output tokens
per request. Input is priced at $3 per million tokens, output at $15 per million
tokens. You expect 40,000 requests/day. What's the approximate monthly cost
(30-day month)?

- **A.** ~$620/month
- **B.** ~$1,890/month
- **C.** ~$4,860/month
- **D.** ~$9,720/month

<details>
<summary>Answer</summary>

**D**

**Why D:** Input tokens/day = 1,200 × 40,000 = 48,000,000 → cost = 48 ×
$3 = $144/day. Output tokens/day = 300 × 40,000 = 12,000,000 → cost = 12 ×
$15 = $180/day. Total = $324/day. Over 30 days: $324 × 30 = **$9,720/month**.

**Why not A:** ~$620/month undercounts by roughly 15×, consistent with
mistakenly using per-thousand-token pricing math on a per-million-token rate.

**Why not B:** ~$1,890/month is roughly what you'd get from a single day's
figure multiplied incorrectly, or from swapping which volume the input/output
rates apply to.

**Why not C:** ~$4,860/month is exactly half of the correct total — the size of
error you'd get by, for example, forgetting to include either the input or the
output cost component entirely (in this case, omitting one of the two roughly
$144–$180/day legs and only scaling the other, or mis-applying the 30-day
multiplier to a half-day figure).

**Interviewer's likely follow-up:** *"Your PM asks you to cut this by 40%
without changing the product experience. What's your first lever?"* (Answer:
output tokens are 5× more expensive per token here, and 12M output tokens/day
already account for more than half the daily cost despite being a quarter of
the volume — check whether output is more verbose than needed (a `max_tokens`
cap or brevity instructions), whether a cheaper/smaller model suffices for parts
of the flow, and whether the input side has a stable prefix worth caching.)
</details>

---

### Q2.17 · Calculation — caching savings · [Applied]

You send a 5,000-token static system prompt + docs prefix on every call, plus a
200-token unique user question, at 10,000 requests/day. Uncached input price is $3/M
tokens; cached-read price is $0.30/M tokens (a 90% discount). Assume the prefix is
cached after the first call each day and hits on all others. Roughly how much do you
save on input cost per day versus not caching at all?

- **A.** About $13/day saved
- **B.** About $135/day saved
- **C.** About $270/day saved
- **D.** No meaningful savings — the user question still needs full-price processing

<details>
<summary>Answer</summary>

**B**

**Why B:** Without caching: 5,200 tokens/call × 10,000 calls = 52,000,000
tokens/day, all at $3/M ≈ $156/day. With caching: the 5,000-token prefix is
billed at the cached rate ($0.30/M) for essentially all 10,000 calls
(50,000,000 tokens × $0.30/M = $15/day), plus the 200-token unique suffix stays
at full price for all calls (2,000,000 tokens × $3/M = $6/day). Cached total ≈
$15 + $6 = $21/day. Savings ≈ $156 − $21 ≈ **$135/day**.

**Why not A:** ~$13/day undercounts the effect — roughly what you'd get if you
mistakenly applied the discount to only a small fraction of the static prefix's
daily volume instead of nearly all of it.

**Why not C:** ~$270/day overstates it — that would require caching to also
discount the unique 200-token suffix, which it doesn't (only the static,
repeated prefix is cacheable).

**Why not D:** There is meaningful savings — the static portion dominates total
token volume (5,000 of 5,200 tokens per call), so caching it has an outsized
effect even though the unique suffix is unaffected.

**Interviewer's likely follow-up:** *"At what request volume does building the
caching logic stop being worth the engineering effort?"* (Answer: there's a
break-even point — at very low daily volume, the savings may not justify the
complexity of managing cache-friendly prompt structure; at the volumes typical
of a production feature, it almost always is. Frame this as cost/benefit
judgement, not a fixed rule.)
</details>

---

### Q2.18 · Calculation — latency budget · [Design]

A customer-facing flow must respond within a 3-second p95 latency budget. Your
current design makes 3 sequential LLM calls (extract → validate → format), each
with a measured p95 latency of 900ms. What's the p95 latency of the sequential
chain, and what does that tell you about your design?

- **A.** ~900ms — p95s don't add up across sequential calls
- **B.** ~2,700ms, which is dangerously close to the 3-second budget with zero margin for network overhead or downstream work — the sequential design is a latency risk
- **C.** ~300ms — chaining calls averages their latencies
- **D.** The chain's latency is unrelated to individual call latency; only total token count matters

<details>
<summary>Answer</summary>

**B**

**Why B:** Sequential dependent calls have latencies that roughly sum (each
call must complete before the next starts) — 3 × 900ms = 2,700ms just for LLM
calls, before any network round-trip, serialization, or business logic overhead
is added. Against a 3-second budget, that's a design with almost no margin —
one call landing at p99 instead of p95 blows the budget.

**Why not A:** p95s of independent sequential steps don't cap at the single-step
value — worst cases can (and statistically will, at scale) stack.

**Why not C:** Chaining is additive, not averaging — each step still has to
happen in full before the next begins.

**Why not D:** Token count is *one driver* of per-call latency, but the
question is specifically about how per-call latencies compose across a
sequential chain — that composition is real and matters regardless of what's
driving each individual call's time.

**Interviewer's likely follow-up:** *"Given this budget, how would you
redesign the flow?"* (Answer: look for steps that can run in parallel instead
of sequentially, collapse multiple steps into a single call via structured
output if the model can do extract+validate+format together, or evaluate
whether a faster/smaller model is acceptable for lower-stakes steps in the
chain.)
</details>

---

### Q2.19 · Calculation — batch vs synchronous break-even · [Design]

You need to process 500,000 documents. The synchronous API costs $2/M input tokens;
the batch API costs $1/M input tokens (50% discount) but adds up to 12 hours of
latency. Each document averages 800 input tokens. If your deadline is "results
needed within 2 hours," what does the arithmetic actually tell you, independent of
the cost difference?

- **A.** Batch is cheaper, so it's the right choice regardless of the deadline
- **B.** The cost saving is irrelevant here — batch's latency profile (up to 12 hours) doesn't fit a 2-hour deadline, so the decision is constrained by latency requirements, not cost, and synchronous is the only viable option
- **C.** You should split the job: half synchronous, half batch, to average the latency to 6 hours
- **D.** 500,000 documents at 800 tokens each exceeds the batch API's total volume limit, making the comparison moot

<details>
<summary>Answer</summary>

**B**

**Why B:** This tests whether you let cost optimisation override a hard
requirement. Batch is materially cheaper (500,000 × 800 = 400,000,000 tokens,
$1/M vs $2/M is a $400 difference for this run), but its latency isn't
guaranteed within your 2-hour window. A deadline is a constraint, not a
preference; when a cheaper option can't meet a hard constraint, it's
disqualified regardless of the savings.

**Why not A:** This is the trap — cost is not the only variable, and picking
batch here would risk missing the deadline entirely for a savings that doesn't
matter if the output arrives too late to be useful.

**Why not C:** Splitting doesn't average latency — the batch half still isn't
guaranteed within 2 hours; you'd still be exposed to missing the deadline on
that portion.

**Why not D:** No such fixed universal document-count limit is implied by the
numbers given — this introduces an assumption unsupported by the scenario, which
is itself a mistake worth flagging in an interview (don't invent constraints
that aren't given).

**Interviewer's likely follow-up:** *"The deadline turns out to be soft —
'ideally 2 hours, but end of day is fine.' Does your answer change?"* (Answer:
yes — this reopens the cost/latency tradeoff, and batch becomes the strong
default choice given the $400 saving on this run alone.)
</details>

---

### Q2.20 · Cost/latency tradeoff — model tiering · [Design]

Your app currently sends every request to the most capable (and most expensive)
model available. A cost review flags this as the top line item. What's the standard
first move, and what's the risk in doing it carelessly?

- **A.** Switch every call to the cheapest available model — quality differences are marginal in practice
- **B.** Route requests by task difficulty: use a smaller/cheaper model for easy, well-defined subtasks (classification, extraction, formatting) and reserve the expensive model for steps that genuinely need its reasoning strength — the risk is picking the split based on guesswork rather than measured quality on your actual tasks
- **C.** Keep the expensive model everywhere, but reduce `max_tokens` to cut cost
- **D.** Cost and quality are fixed tradeoffs set by the provider — there's no engineering lever here

<details>
<summary>Answer</summary>

**B**

**Why B:** Model tiering — matching model capability to task difficulty — is
the standard lever for cost without a blanket quality hit. The risk isn't the
strategy itself, it's implementing it on intuition ("this task feels easy")
rather than measured evaluation on your actual data (see file 06) — a subtask
that looks simple can still need the stronger model if it turns out to have
edge cases the cheaper model handles badly.

**Why not A:** "Marginal in practice" is an unverified assumption — quality
differences vary hugely by task, and a blanket downgrade risks silently
degrading the parts of the flow that actually needed the stronger model.

**Why not C:** Reducing `max_tokens` caps output length, but doesn't address
the largest lever available here (model choice) and can truncate legitimate
longer outputs, causing a different class of failure.

**Why not D:** This ignores model tiering, prompt caching, batch APIs, and
prompt-length reduction — all genuine engineering levers covered elsewhere in
this file.

**Interviewer's likely follow-up:** *"How do you decide where the cut line
is between 'cheap model' and 'expensive model' tasks?"* (Answer: run the
cheaper model against a labelled evaluation set for that specific subtask and
compare accuracy/quality against the expensive model's output — don't decide by
eyeballing a few examples.)
</details>

---

### Q2.21 · Rate limits — quota scope · [Recall]

Two features on your team both call the same LLM provider under one shared API
key. Feature A starts getting rate-limited during a spike caused entirely by
Feature B. Why does this happen, and what does it imply about how you should
architect quota management?

- **A.** This shouldn't happen — rate limits are scoped per code path, not per credential
- **B.** Rate limits are typically scoped to the API key/account (sometimes per-model), not per feature — a shared key means the features share one quota bucket, so an unrelated feature's traffic spike can throttle you even if your own request volume hasn't changed
- **C.** Feature A must have a bug causing it to send duplicate requests
- **D.** Rate limits reset per feature automatically as long as each feature has distinct request headers

<details>
<summary>Answer</summary>

**B**

**Why B:** Providers enforce limits against the credential (and sometimes
per-model) making the call, with no built-in awareness of your internal feature
boundaries. Sharing one key across unrelated features means they compete for
the same quota — a spike anywhere in that shared pool can throttle everyone
using it.

**Why not A:** There's no code-path-level scoping unless you build it yourself
client-side — the provider only sees the credential.

**Why not C:** Nothing in the scenario indicates a bug in Feature A; the
described mechanism (shared quota) fully explains the symptom without assuming
one.

**Why not D:** Custom headers don't create separate quota buckets on the
provider side unless the provider explicitly supports and meters that
(uncommon) — this is a fabricated mechanism.

**Interviewer's likely follow-up:** *"How would you fix this without asking
the provider for a quota increase?"* (Answer: separate API keys/projects per
feature if the provider supports it, so quota is isolated; or implement
internal per-feature rate limiting/prioritisation so one feature can't starve
another even on a shared key.)
</details>

---

### Q2.22 · Calculation — concurrency under a rate limit · [Applied]

Your API key is limited to 500 requests per minute. Each request takes an average
of 1.5 seconds to complete. What's the maximum sustainable concurrency (number of
requests in flight at once) you can run without exceeding the rate limit?

- **A.** 500 concurrent requests
- **B.** About 12–13 concurrent requests
- **C.** 1 concurrent request, since rate limits require fully sequential calls
- **D.** Concurrency is irrelevant to rate limits — only total daily volume matters

<details>
<summary>Answer</summary>

**B**

**Why B:** Requests/minute = concurrency × (requests per worker per minute).
Each request takes 1.5s, so one worker completes 60/1.5 = 40 requests/minute.
To stay at or under 500 requests/minute: concurrency = 500 / 40 = **12.5**, so
about 12 concurrent workers keeps you safely under the limit.

**Why not A:** Running 500 requests concurrently, each taking 1.5s, would
attempt far more than 500 requests in that first minute alone (500 concurrent ×
40 completions/minute per worker if sustained ≈ 20,000/minute) — wildly over
the limit.

**Why not C:** Fully sequential (concurrency = 1) would only achieve ~40
requests/minute, well under the 500/minute you're allowed — leaving throughput
on the table unnecessarily.

**Why not D:** Concurrency directly determines your request rate given a fixed
per-request duration — this option ignores the mechanism the question is
testing.

**Interviewer's likely follow-up:** *"What happens if request latency is
variable, not a clean 1.5s average?"* (Answer: use a token-bucket or leaky-
bucket rate limiter keyed on actual request timestamps rather than a static
concurrency count, so you adapt to real latency instead of relying on an
average that can spike.)
</details>

---

### Q2.23 · max_tokens truncation · [Applied]

A production incident: structured-output JSON responses are failing to parse, but
only for a specific subset of inputs — the ones that require longer answers. What's
the most likely root cause, and how do you confirm it?

- **A.** The model is malfunctioning specifically on longer inputs
- **B.** The response is likely being cut off mid-generation by a `max_tokens` limit set too low for the longer responses this subset needs, producing a truncated (and therefore invalid) JSON string — confirm by checking the response's stop/finish reason for "length" rather than "stop"
- **C.** JSON mode only supports responses under 500 tokens by design
- **D.** The tokeniser fails on inputs above a certain length

<details>
<summary>Answer</summary>

**B**

**Why B:** Every completion response carries a stop/finish reason. If it reads
something like `"length"` or `max_tokens` instead of a natural stop, the
response was cut off before the model finished — for JSON output, that almost
always produces an unparseable fragment (an unclosed brace, a dangling value).
This precisely explains why only the longer-answer subset fails: those are the
ones actually hitting the cap.

**Why not A:** Nothing in the scenario suggests a model malfunction — the
pattern (only long-answer inputs fail) points specifically at a length-related
mechanism, not general unreliability.

**Why not C:** There's no such fixed 500-token ceiling built into JSON mode —
the limit is whatever `max_tokens` you configured for the call.

**Why not D:** Tokenisation itself doesn't "fail" based on input length within
supported limits — this isn't a real failure mode for the tokeniser.

**Interviewer's likely follow-up:** *"What's your fix, beyond just raising
max_tokens?"* (Answer: raise the cap to a value with real headroom for the
task, but also treat a "length" finish reason as an explicit error condition
in your code — don't silently attempt to parse a response you already know was
truncated.)
</details>

---

### Q2.24 · Calculation — multi-turn cost growth · [Design]

A support chatbot resends full conversation history every turn (per Q2.6). Turn 1
costs $0.01 in input tokens. Each turn adds roughly 150 tokens of new content to the
history, and input is priced at $3/M tokens. By turn 20, roughly how much more
expensive (in input cost alone) is that single turn compared to turn 1, and what
architectural implication does that have?

- **A.** Turn 20 costs the same as turn 1 — history doesn't affect per-turn cost
- **B.** Turn 20's input cost is meaningfully higher than turn 1's because it resends ~19 turns' worth of accumulated history (~2,850 extra tokens ≈ $0.0086 more) on top of the original content — conversations left unbounded make later turns progressively, and needlessly, more expensive
- **C.** Turn 20 is exactly 20× more expensive than turn 1
- **D.** Cost growth from history only matters once a conversation exceeds 100 turns

<details>
<summary>Answer</summary>

**B**

**Why B:** By turn 20, roughly 19 additional turns' worth of history (≈19 ×
150 = 2,850 tokens) is being resent on top of turn 1's original content, at $3/M
→ ≈ $0.0086 in additional input cost for that turn alone — on top of the
baseline $0.01. That's not dramatic in isolation, but it compounds: every later
turn in every long conversation pays this growing tax, and at scale (thousands
of conversations, some running much longer than 20 turns) the aggregate is
real. The implication is architectural: unbounded history growth is a cost
problem, not just a context-window problem, which motivates windowing or
summarisation strategies (see Q2.3, Q2.14) purely on cost grounds, independent
of quality concerns.

**Why not A:** This ignores the stated behaviour (full history resent every
turn) and Q2.6's premise entirely.

**Why not C:** Cost growth here is additive per accumulated history tokens, not
a clean linear multiple of the turn number — the relationship depends on
tokens-added-per-turn, not turn count alone.

**Why not D:** Cost grows from turn 1 onward, incrementally — there's no
threshold below which it "doesn't matter"; it's just smaller earlier.

**Interviewer's likely follow-up:** *"At what point would you introduce
summarisation instead of just truncating old turns?"* (Answer: truncation
risks losing information the user or model still needs — summarisation
preserves the gist at much lower token cost; the tradeoff is summarisation
itself costs an extra LLM call, so it's worth it once conversations are long/
frequent enough that the savings outweigh that added cost.)
</details>

---

### Q2.25 · Structured output schema overhead · [Applied]

You add a JSON schema for structured output enforcement to a high-volume endpoint.
Someone on the team notices token usage per request went up slightly even though
the actual response content didn't get longer. Why?

- **A.** JSON mode always doubles output token cost as a platform fee
- **B.** The schema definition itself (field names, types, descriptions) is typically part of what's sent to or processed by the model to constrain generation, so a large or verbose schema adds real token overhead on top of the response content
- **C.** This is a billing bug and should be reported to the provider
- **D.** Structured output has no token cost implications — the observed increase must be unrelated

<details>
<summary>Answer</summary>

**B**

**Why B:** The schema you define — field names, types, nested structure,
descriptions — typically has to be communicated to the model as part of the
call (as an input-side cost) so it knows the exact shape to constrain its
output to. A verbose schema with long field descriptions adds real tokens on
top of whatever the response content itself would have cost, even though the
answer isn't any longer than before.

**Why not A:** There's no flat "doubling" platform fee for JSON mode — the
overhead scales with the size and verbosity of the schema you actually define,
not a fixed multiplier.

**Why not C:** This isn't a billing error — it's an expected, explainable
consequence of how schema-constrained generation works, not a bug to report.

**Why not D:** The scenario describes a real, measurable token increase with an
identifiable cause (the added schema) — dismissing it as unrelated ignores the
mechanism the question is testing.

**Interviewer's likely follow-up:** *"How would you keep this overhead under
control on a high-volume endpoint?"* (Answer: keep schemas as lean as
possible — concise field names and descriptions, avoid deeply nested or
redundant structure — and, if the schema is static across calls, check whether
it can be part of a cached prefix so its overhead isn't paid at full price on
every single request.)
</details>

---

### Q2.26 · Batch job partial failure · [Recall]

A 100,000-item batch job to the LLM batch API completes, but the results show 300
items failed (e.g. malformed input, content filtering) while the rest succeeded.
What's the correct way to handle this?

- **A.** Discard the entire batch and resubmit all 100,000 items, since the job as a whole is "failed"
- **B.** Treat the batch as mostly successful: process the 99,700 successful results, isolate and inspect the 300 failures individually, and only resubmit that small failed subset (after fixing whatever caused the failure, if fixable)
- **C.** Silently drop the 300 failed items with no follow-up, since 99.7% success is a good rate
- **D.** Batch APIs guarantee 100% per-item success or the whole job errors out, so partial failure like this can't actually occur

<details>
<summary>Answer</summary>

**B**

**Why B:** Batch jobs are large enough that some per-item failure rate is
normal and expected (bad input formatting, content policy triggers, occasional
transient errors) — treating the whole batch as failed wastes the 99.7% that
succeeded. The correct pattern is per-item error handling: process what
succeeded, isolate what failed with its specific error reason, and selectively
resubmit only the failed subset.

**Why not A:** This is wasteful and unnecessary — nothing in a partial-failure
result invalidates the successful items.

**Why not C:** Silently dropping failures without investigation risks missing a
systematic issue (e.g. a whole category of malformed input) that will recur on
every future batch.

**Why not D:** Batch APIs don't provide all-or-nothing guarantees — per-item
failure within an otherwise-successful batch is a normal, expected outcome you
must design for.

**Interviewer's likely follow-up:** *"How would you design the pipeline so
this doesn't require manual intervention every time?"* (Answer: build
automated per-item failure classification and retry logic into the pipeline —
distinguish retryable failures (transient) from ones needing a fix upstream
(consistently malformed input), and alert only on the latter.)
</details>

---

### Q2.27 · Temperature for deterministic tasks · [Applied]

You're building a feature that classifies support tickets into one of eight fixed
categories. A teammate suggests `temperature=0.7` "so it doesn't get stuck picking
the same category too often." Do you agree?

- **A.** Yes — higher temperature improves classification accuracy by exploring more options
- **B.** No — classification into a fixed, correct category isn't a creative task; there's a right answer, and `temperature=0` (or very low) is appropriate to get the model's best single judgment consistently, rather than introducing arbitrary randomness into what should be a deterministic decision
- **C.** Yes — without randomness, the model will always output the same category regardless of ticket content
- **D.** Temperature has no effect on classification tasks, only on open-ended generation

<details>
<summary>Answer</summary>

**B**

**Why B:** Classification has a correct-or-incorrect answer per item — it's not
a task where variety in phrasing or approach is valuable, unlike creative
writing. Introducing randomness via higher temperature just adds noise to a
decision that should reflect the model's most confident judgment given the
input, and makes results non-reproducible for the same ticket content.

**Why not A:** Higher temperature doesn't improve accuracy — it increases the
chance of sampling a lower-probability (and often lower-quality) token/answer
instead of the model's top choice.

**Why not C:** This confuses "deterministic" with "input-insensitive" — at
temperature=0 the model still changes its output based on the actual ticket
content; it just stops varying its output for the *same* content across repeat
calls.

**Why not D:** Temperature affects token selection in every generation task,
classification included — the model is still choosing among candidate tokens
representing category labels.

**Interviewer's likely follow-up:** *"When might you actually want some
randomness in a classification-adjacent task?"* (Answer: if you're
deliberately sampling multiple candidate labels to build an ensemble/voting
mechanism, or generating diverse synthetic training examples — but that's a
different goal than "classify this one ticket correctly.")
</details>

---

### Q2.28 · TTFT vs total latency — UX design · [Recall]

For a long-form report-generation feature (30+ seconds to fully complete), which
matters more to perceived user experience: time-to-first-token or total completion
time, and why?

- **A.** Total completion time — users only care about the final result, so first-token timing is irrelevant
- **B.** Time-to-first-token — for long generations, showing the user that something is actively happening (via streaming) as early as possible substantially improves perceived responsiveness, even though it doesn't shorten the actual total wait
- **C.** They're equally important and interchangeable metrics
- **D.** Neither matters for report generation — only accuracy matters

<details>
<summary>Answer</summary>

**B**

**Why B:** For long-running generations, an unresponsive-feeling blank wait is
one of the biggest UX complaints regardless of eventual output quality —
getting visible progress on-screen quickly (low TTFT via streaming) reassures
the user the system is working, even though total completion time is unchanged.
This is a well-established perceived-performance principle, not unique to LLMs.

**Why not A:** Total completion time matters too, but dismissing TTFT ignores
the specific UX problem long-running generations create — silence during a
30-second wait reads as "broken," even if the eventual output is fine.

**Why not C:** They're not interchangeable — they measure different things and
matter for different reasons (perceived responsiveness vs actual throughput),
which is exactly why both concepts exist as separate metrics.

**Why not D:** Accuracy is a separate axis entirely — the question is
specifically about UX/latency perception, not output quality, and both can
matter simultaneously without being the same concern.

**Interviewer's likely follow-up:** *"What would you show the user during
that 30-second wait beyond just streamed tokens?"* (Answer: intermediate
progress signals if the pipeline has discrete stages — e.g. "researching...",
"drafting...", "formatting..." — gives a sense of progress beyond raw text
appearing, especially useful if there's a slow non-streaming step like a tool
call in the middle.)
</details>

---

### Q2.29 · Design — dual-model cost/latency architecture · [Design]

Some engineers run a "dual-model" workflow: a cheaper/faster model handles routine
execution steps, while a stronger (slower, pricier) model is invoked only for
planning or reasoning-heavy steps — for example, splitting "thinking" and "worker"
responsibilities across two different models in an agentic coding tool. What's the
underlying cost/latency argument for this pattern, and what's the main risk?

- **A.** It's purely a cost hack with no latency benefit, since total work done is unchanged
- **B.** Most steps in a typical agentic or multi-step workflow are mechanical (formatting, straightforward tool calls, simple edits) and don't need top-tier reasoning — reserving the expensive model for the subset of steps that actually require it cuts blended cost and often latency, since the cheaper model responds faster too; the risk is misrouting a step that actually needed strong reasoning to the weaker model, producing a subtly wrong result that's more expensive to catch than it would have been to just use the strong model
- **C.** This pattern only saves money if both models are from the same provider
- **D.** Dual-model setups always increase total latency because of the extra routing decision overhead

<details>
<summary>Answer</summary>

**B**

**Why B:** This is model tiering (Q2.20) applied to an agentic loop
specifically: most steps in a multi-step task are routine, and routine steps
don't need the most capable (and most expensive/slowest) model. Splitting
responsibility by step type reduces blended cost and often latency, since
smaller models also tend to respond faster. The real risk is a routing
misjudgement — sending a step that actually needed careful reasoning to the
cheaper model produces a wrong or subtly flawed result, and debugging *that*
(figuring out a silent quality regression happened at all) can cost more than
the savings, especially if it ships before anyone notices.

**Why not A:** There is a latency benefit in practice, not just cost — smaller
models typically have lower per-token latency in addition to lower per-token
price.

**Why not C:** Nothing about this pattern requires same-provider models — the
split is about capability/cost tiers, not vendor.

**Why not D:** Routing overhead (a cheap classification or heuristic decision)
is typically negligible compared to the latency difference between model tiers
— the net effect is usually a latency win, not a loss.

**Interviewer's likely follow-up:** *"How do you decide, concretely, which
steps get routed to which model?"* (Answer: define the split based on
measured task difficulty/failure rate per step type against an eval set, not
intuition — similar to the answer in Q2.20 — and monitor for regressions after
the split ships, since task difficulty in production can drift from what you
measured offline.)
</details>

---

### Q2.30 · Multi-tenant cache invalidation · [Design]

You serve multiple enterprise customers from one product, each with a customised
system prompt (different company name, tone, and a customer-specific knowledge
snippet). You want prompt caching to work across all of them. What's the design
implication, and what's the tradeoff?

- **A.** Caching can't work at all in a multi-tenant setup — skip it
- **B.** Each customer's distinct system prompt creates its own separate cache entry (since caching is exact-prefix-match per Q2.11/Q2.12) — you still get caching benefits within a customer's repeated calls, but you don't get cross-customer sharing, and the number of distinct cached prefixes now scales with your customer count, which has its own cost/memory-lifetime implications on the provider side
- **C.** You should merge all customers onto one shared system prompt to maximise cache hits, even if it reduces personalisation
- **D.** Caching automatically detects the customer-specific portion and caches only the shared boilerplate around it

<details>
<summary>Answer</summary>

**B**

**Why B:** Since caching matches on exact prefix, each distinct
customer-specific system prompt is its own cache key — customer A's calls hit
customer A's cache entry, customer B's hit a separate one. You still get the
core benefit (each customer's repeated calls stay cheap), but there's no
cross-customer sharing, and you now have N cache entries instead of one, which
is a real scaling consideration (cache entries typically have a
time-to-live and per-entry constraints worth checking against your provider's
docs as customer count grows).

**Why not A:** Caching still works — it just operates per distinct prefix
rather than globally, which is a design detail, not a disqualifier.

**Why not C:** Sacrificing genuine per-customer personalisation to chase a
caching optimisation gets the priority backwards — the product requirement
(customisation) should drive the design; caching strategy adapts to it, not the
reverse.

**Why not D:** Caching doesn't do partial-prefix intelligent splitting
automatically — it's exact-match on what you send, so if you want a genuinely
shared *and* cacheable portion, you have to structure the prompt yourself (e.g.
shared instructions first, customer-specific block after) — see the follow-up.

**Interviewer's likely follow-up:** *"How would you restructure the prompt to
get partial caching benefit even across customers?"* (Answer: put the
universal, customer-independent instructions in the first part of the prefix
and the customer-specific block right after it — depending on how the
provider's caching handles partial-prefix matches, you may get a cache hit on
the shared leading portion across all customers, with only the trailing
customer-specific segment processed fresh each time.)
</details>

---

## Explain prompts

### E2.1 · Explain: why token-based cost estimates matter more than word-based ones

**Prompt:** *"Your PM wants a rough cost estimate for a new feature before it's
built, based on a rough word count of expected prompts and responses. Walk me
through how you'd actually produce that estimate, and what could make it wrong."*

**Target:** 60–90 seconds spoken. Answer out loud before opening the rubric.

<details>
<summary>Rubric — score yourself</summary>

**Must hit (1 point each):**
- [ ] States that word count is not token count, and gives a rough conversion
      factor for English prose (~1.3 tokens/word)
- [ ] Notes that input and output tokens are typically priced differently, so
      they need to be estimated and calculated separately
- [ ] Points out that content type (code, JSON, non-English) tokenises
      differently and needs separate estimation if present
- [ ] Describes actually counting tokens on representative real samples rather
      than relying purely on a formula
- [ ] Multiplies per-request cost by expected request volume to get an
      aggregate (daily/monthly) figure, not just a per-call number

**Bonus — signals strength:**
- [ ] Flags that multi-turn conversations grow input token cost over time (Q2.24)
- [ ] Mentions caching as a potential cost reducer once the shape of the prompt is known
- [ ] Distinguishes a back-of-envelope estimate from a rate that should be validated once real traffic exists

**Red flags — deduct:**
- [ ] Uses word count directly as if it equals token count
- [ ] Gives a single blended per-token price without separating input/output
- [ ] Never connects the per-request number to volume — stops at "the cost per call is $X"

**Score: ___ / 5**

**Model answer:**
"Okay, so first thing — I wouldn't use word count directly, because tokens and
words aren't the same thing. English prose is roughly one-point-three tokens per
word, so I'd apply that as a starting multiplier, but if there's any code or JSON
in there I'd bump that up, because that stuff tokenises worse. Then I'd split it
into input and output separately, because those are usually priced differently —
output's often five times pricier per token. I'd estimate tokens for a typical
request on both sides, multiply by the per-token rate, and that gives me a
cost-per-request number. From there it's just per-request cost times expected
daily volume, times thirty for a monthly figure. But honestly, before I present
that to the PM as final, I'd actually run a handful of real examples through the
tokenizer, because estimates from a formula can be off by a lot, especially if the
content isn't plain English."
</details>

---

### E2.2 · Explain: managing a growing conversation without blowing the context window

**Prompt:** *"We have a customer support chatbot where some conversations run past
50 turns. What's your strategy for handling context as the conversation grows?"*

**Target:** 60–90 seconds spoken.

<details>
<summary>Rubric — score yourself</summary>

**Must hit:**
- [ ] States the API is stateless, so the app is responsible for what history
      gets resent each turn
- [ ] Names at least one concrete strategy: sliding window of recent turns,
      periodic summarisation, or retrieval of relevant past turns
- [ ] Notes the tradeoff of the chosen strategy (e.g. windowing can lose
      earlier context the user still references; summarisation costs an extra
      call and can lose detail)
- [ ] Connects unmanaged growth to both cost (resending more tokens every turn)
      and context-window risk (eventually hitting the hard limit)
- [ ] Says the strategy should probably be chosen based on the conversation's
      actual usage pattern, not applied uniformly without checking

**Bonus:**
- [ ] Mentions keeping a fixed system prompt/instructions separate from the
      rolling history so it isn't accidentally dropped by windowing
- [ ] Raises that summarisation quality itself needs evaluation, not just
      assumed to work
- [ ] Notes this interacts with prompt caching — a stable prefix strategy
      preserves cache hits better than constantly-changing windowed history

**Red flags:**
- [ ] Assumes the model "remembers" past turns without them being resent
- [ ] Proposes no bound at all — "just keep sending everything"
- [ ] Treats this as a purely technical problem with no mention of what the
      user actually needs preserved

**Score: ___ / 5**

**Model answer:**
"So the first thing to get straight is the API doesn't remember anything between
calls — every turn, we're the ones deciding what history to resend. Past a
certain length, I wouldn't just keep sending the whole thing, for two reasons:
cost, because you're paying for that history every single turn, and eventually
you'll just hit the context limit outright. So I'd do something like a sliding
window — keep the last N turns verbatim, and either drop or summarise anything
older than that. Summarisation's nicer because you don't lose the gist of
earlier context, but it costs an extra call and you need to actually check the
summary quality holds up. Which one I'd pick really depends on the conversation
pattern — support chats might genuinely need to reference something from turn 3
at turn 50, so I wouldn't guess, I'd look at real transcripts first."
</details>

---

### E2.3 · Explain: choosing temperature for a specific product

**Prompt:** *"We're building three features: a code-generation assistant, a
marketing-copy generator, and a data-extraction tool that pulls fields from
invoices. Walk me through what temperature you'd set for each and why."*

**Target:** 60–90 seconds spoken.

<details>
<summary>Rubric — score yourself</summary>

**Must hit:**
- [ ] Recommends low/near-zero temperature for data extraction — it's a
      correctness task, not a creative one
- [ ] Recommends low-to-moderate temperature for code generation, reasoning
      that correctness/consistency usually matters more than variety, though
      some allow slightly higher for brainstorming alternatives
- [ ] Recommends higher temperature for marketing copy, where variety and
      "interesting" phrasing has real value
- [ ] Explains the underlying principle: temperature should track how much
      value there is in output *variety* vs how much the task has one
      correct/best answer
- [ ] Doesn't treat temperature as a single global setting — ties it to the
      specific task, not the product as a whole

**Bonus:**
- [ ] Mentions structured output/schema enforcement as the real fix for
      extraction reliability, with temperature as a secondary lever
- [ ] Notes that even "creative" tasks often want some determinism at final
      review/approval stages
- [ ] Raises that this choice should be validated against actual output
      quality, not just picked from a rule of thumb

**Red flags:**
- [ ] Picks one temperature for all three features
- [ ] Says higher temperature always means "smarter" or "better" output
- [ ] Can't articulate why extraction specifically wants low temperature

**Score: ___ / 5**

**Model answer:**
"These three are pretty different asks, so I wouldn't use one setting across the
board. Invoice extraction — that's basically a correctness task, there's a right
answer for what the total or the invoice number is, so I'd run that at zero or
close to it, and honestly I'd lean on structured output enforcement more than
temperature for reliability there. Code generation, I'd probably keep low too,
maybe a little above zero — you want consistency more than creative variation in
most cases, though some people bump it slightly if they want a few alternative
approaches to choose from. Marketing copy is the one place I'd actually go
higher, because part of the value there is variety — you don't want the same
safe phrasing every time. The general rule I'm using is: how much does variety
actually help this task versus how much is there just one good answer."
</details>

---

### E2.4 · Explain: why the API being stateless matters architecturally

**Prompt:** *"A new engineer on your team is confused why you need a database
table for conversation history at all — 'doesn't the model just remember?' Explain
what's actually going on and why it shapes your architecture."*

**Target:** 60–90 seconds spoken.

<details>
<summary>Rubric — score yourself</summary>

**Must hit:**
- [ ] States plainly that the API has no memory between calls — each request is
      independent
- [ ] Explains that "conversation" is an illusion your application creates by
      resending history, not something the API tracks for you
- [ ] Connects this to why you need persistent storage (a database) for
      conversation history at all
- [ ] Notes the practical implication: growing history means growing cost and
      eventual context-limit risk (ties to E2.2)
- [ ] Distinguishes this from any provider-offered "conversation" or "thread"
      convenience feature, which — if it exists — is still built on the same
      stateless mechanism under the hood

**Bonus:**
- [ ] Mentions this is also why reproducing a bug requires the exact history
      sent, not just "what the user saw"
- [ ] Notes multi-tenant/session isolation depends on the app correctly
      separating whose history is whose — nothing about the API enforces that
- [ ] Ties statelessness to why prompt caching is valuable at all

**Red flags:**
- [ ] Implies the model has some persistent memory of past conversations
- [ ] Can't explain why a database is actually needed
- [ ] Describes this as a limitation to complain about rather than a design
      constraint to build around

**Score: ___ / 5**

**Model answer:**
"So the honest answer is no, the model doesn't remember anything — every single
call to the API is completely independent, it has no idea what happened five
minutes ago or five seconds ago unless we tell it. What feels like a
conversation is actually us, on the backend, pulling the full history out of the
database and resending it, every single turn, as part of the prompt. That's
exactly why we need that table — without it, there's nothing tying turn five back
to turn one. And it's not just a storage detail, it actually shapes cost and
limits too, because the more history we resend, the more we're paying for on
every single call, and eventually we'll hit the context window if a
conversation runs long enough without us managing it."
</details>

---

### E2.5 · Explain: when NOT to stream

**Prompt:** *"Streaming is usually presented as a pure UX win. Give me a case
where you'd deliberately choose not to stream a response, and explain your
reasoning."*

**Target:** 45–75 seconds spoken.

<details>
<summary>Rubric — score yourself</summary>

**Must hit:**
- [ ] Identifies structured output/JSON responses as a case where streaming
      raw partial output isn't directly usable (invalid mid-stream)
- [ ] Explains that you need the complete response before you can
      parse/validate/act on it in these cases
- [ ] Notes streaming can still be used under the hood (buffer until complete,
      then parse) even if the *user* doesn't see partial output
- [ ] States this is a case-by-case decision based on what happens to the
      output, not a blanket "streaming is always right/wrong" rule
- [ ] Mentions at least one other legitimate non-streaming case (e.g. a
      background batch job with no user watching in real time)

**Bonus:**
- [ ] Distinguishes "don't show the user partial output" from "don't use the
      streaming API at all" — these aren't the same decision
- [ ] Raises that some structured-output APIs do support incremental/streamed
      structured parsing, which changes the calculus
- [ ] Notes error handling gets more complex with streaming (partial output on
      a mid-stream failure) as an additional reason to avoid it for
      correctness-critical paths

**Red flags:**
- [ ] Says streaming should always be used regardless of use case
- [ ] Can't explain why partial JSON is a problem
- [ ] Confuses "not streaming to the user" with "the backend also can't use
      streaming internally"

**Score: ___ / 5**

**Model answer:**
"The clearest case is structured output — if I'm asking the model for JSON that
I need to parse and act on programmatically, streaming partial tokens to the
user doesn't help, because half a JSON object isn't valid JSON, there's nothing
useful to show or use until it's complete. So for that path I'd just wait for
the full response, validate it, then use it. That doesn't mean I can't use the
streaming API under the hood — I might still buffer it internally — I just
wouldn't expose partial output to the user or downstream logic. Same logic
applies to background jobs, like a batch pipeline with nobody watching in real
time — there's no perceived-latency benefit to chase there, so streaming just
adds complexity for no payoff."
</details>

---

### E2.6 · Explain: designing for prompt caching from the start

**Prompt:** *"You're architecting a new feature that will send a large,
mostly-static reference document alongside a short user query, at high volume.
Walk me through how you'd structure the prompt to actually benefit from caching,
and what could accidentally break it."*

**Target:** 60–90 seconds spoken.

<details>
<summary>Rubric — score yourself</summary>

**Must hit:**
- [ ] States that the static content should be placed first/consistently as a
      stable prefix, with the varying user content after it
- [ ] Explains caching requires an exact match on that prefix — any change
      invalidates it
- [ ] Names at least one realistic way this accidentally breaks (e.g. a
      timestamp, a per-request ID, or dynamic content slipped into the "static"
      block)
- [ ] Mentions monitoring cache hit rate as a way to catch this happening in
      production
- [ ] Connects the design decision to the actual payoff: cost reduction and
      lower TTFT at high call volume

**Bonus:**
- [ ] Discusses how frequently the "static" document actually changes, and
      what that means for cache effectiveness if it's updated often
- [ ] Notes multi-tenant considerations if the "static" content varies by
      customer (ties to Q2.30)
- [ ] Mentions cache entries aren't permanent — they typically expire, so very
      low-traffic periods can still cause cache misses even with a stable
      prefix

**Red flags:**
- [ ] Puts the varying user content before the static content
- [ ] Doesn't recognise any way the cache could accidentally break
- [ ] Treats caching as "set and forget" with no mention of monitoring

**Score: ___ / 5**

**Model answer:**
"The main thing is ordering — I'd put the static reference document first, as a
stable, unchanging block, and put the user's actual query after it, since
caching matches on that leading prefix. The trap I'd watch for is anyone
sneaking something dynamic into that supposedly-static block — even something
small like a timestamp or a request ID for logging purposes — because that
breaks the exact match on every single call and you silently lose all the
savings without anyone noticing, since the requests still succeed, they're just
not cached anymore. So I'd actually put a cache-hit-rate metric on a dashboard,
not just total cost, because a drop in hit rate is the signal that something
snuck into that prefix. Done right, at high volume this is a real cost and
latency win, not just a nice-to-have."
</details>

---

### E2.7 · Explain: batch vs synchronous decision framework

**Prompt:** *"Give me a framework for deciding, in general, whether a given LLM
workload should use the batch API or the synchronous API."*

**Target:** 60 seconds spoken.

<details>
<summary>Rubric — score yourself</summary>

**Must hit:**
- [ ] Names latency tolerance as the primary decision axis — is there a human
      or real-time system waiting on the response
- [ ] States batch trades latency guarantees for cost savings
- [ ] Notes volume matters — batch makes more sense as scale increases, since
      the discount compounds
- [ ] Gives at least one concrete example of each (batch: bulk classification/
      backfill; synchronous: user-facing chat)
- [ ] Explicitly states that a hard deadline disqualifies batch even if it's
      cheaper (ties to Q2.19), i.e. cost isn't the only variable

**Bonus:**
- [ ] Mentions hybrid approaches — e.g. synchronous for the real-time path,
      batch for reprocessing/backfill of the same data type
- [ ] Notes batch failure handling differs (per-item, asynchronous) from
      synchronous error handling
- [ ] Raises that batch is worth it even at moderate volume if the savings
      are large enough relative to engineering cost of implementing it

**Red flags:**
- [ ] Chooses purely on cost without mentioning latency requirements at all
- [ ] Says batch is always better because it's cheaper
- [ ] Can't give a concrete example of either use case

**Score: ___ / 5**

**Model answer:**
"The first question I'd ask is: does anything need this result in real time —
a user waiting on a screen, a downstream system blocking on it? If yes, that's
synchronous, full stop, no matter how much cheaper batch is, because batch can
take hours and doesn't guarantee a window. If there's genuinely no one waiting —
think reclassifying a backlog of old tickets overnight — then batch is usually
the better call, because you get a real discount and you're not paying for
latency you don't need. Volume matters too — at low volume the savings might not
be worth the extra complexity of handling an async job. So really it's latency
requirement first, then cost and volume second, not the other way around."
</details>

---

### E2.8 · Explain: designing rate-limit handling for a production service

**Prompt:** *"Design, out loud, how you'd handle LLM API rate limits in a
production service that has unpredictable traffic spikes."*

**Target:** 60–90 seconds spoken.

<details>
<summary>Rubric — score yourself</summary>

**Must hit:**
- [ ] Names backoff-with-jitter as the reactive mechanism for handling 429s
- [ ] Mentions respecting a `Retry-After` header if the provider sends one
- [ ] Proposes a proactive mechanism too — client-side rate limiting/request
      queuing to stay under the known quota rather than only reacting to 429s
- [ ] Notes the difference between a transient spike (worth queuing/retrying)
      and a sustained overload (may need shedding load or degrading gracefully)
- [ ] Mentions monitoring 429 rate as a leading indicator, not just fixing it
      after users notice

**Bonus:**
- [ ] Raises requesting a quota increase from the provider as a legitimate
      lever, not just an engineering workaround
- [ ] Distinguishes per-key/per-feature quota isolation as a design choice
      (ties to Q2.21)
- [ ] Mentions user-facing graceful degradation (e.g. a queued/"still working"
      state) rather than a hard failure during throttling

**Red flags:**
- [ ] Proposes immediate retry loops with no backoff
- [ ] Only has a reactive plan, no proactive quota management
- [ ] Treats every 429 the same regardless of whether it's a brief spike or
      sustained overload

**Score: ___ / 5**

**Model answer:**
"I'd do this in two layers. First, proactively — I don't want to be finding out
about the rate limit from a wave of 429s, so I'd put a client-side limiter or
queue in front of outbound calls that keeps us under the known quota by design.
Second, reactively, for whatever still slips through during a real spike — back
off, ideally respecting a Retry-After header if the provider gives one,
otherwise exponential backoff with some jitter so retries don't all pile up at
the same moment. I'd also treat a short spike differently from sustained
overload — if it's short, queuing and retrying is fine, but if we're
consistently near the ceiling, that's a signal to either request a higher quota
or start shedding lower-priority work. And I'd put 429 rate on a dashboard, not
just wait for someone to complain."
</details>

---

### E2.9 · Explain: structured output reliability beyond "just enable JSON mode"

**Prompt:** *"Your team enabled structured output/JSON mode and assumes the
parsing failures are now solved. Walk me through why that assumption is
incomplete, and what else you'd still build."*

**Target:** 45–75 seconds spoken.

<details>
<summary>Rubric — score yourself</summary>

**Must hit:**
- [ ] Acknowledges structured output substantially reduces, but doesn't
      guarantee zero, parsing failures
- [ ] Names truncation (hitting `max_tokens` mid-response) as a residual
      failure mode even with schema enforcement (ties to Q2.23)
- [ ] States you should still validate the parsed response against your
      expected schema/business rules, not just trust it parsed
- [ ] Proposes a retry-with-feedback path for genuine failures rather than
      assuming they won't happen
- [ ] Distinguishes "syntactically valid JSON" from "semantically correct for
      my use case" — schema enforcement guarantees the former, not the latter

**Bonus:**
- [ ] Mentions monitoring parse/validation failure rate in production as an
      ongoing signal, not a one-time check
- [ ] Notes edge cases like the model correctly following the schema but
      producing an empty or degenerate valid value (e.g. an empty array when a
      real result was expected)

**Red flags:**
- [ ] Claims structured output guarantees the content is correct, not just
      well-formed
- [ ] Has no fallback plan for a residual failure
- [ ] Doesn't distinguish syntactic validity from semantic correctness

**Score: ___ / 5**

**Model answer:**
"So it definitely helps a lot, but it's not a hard guarantee — for one,
truncation can still happen, if max_tokens is too low the response gets cut off
mid-structure and that's still invalid, schema enforcement or not. And even when
it does parse cleanly, that only tells you it's syntactically valid JSON
matching your schema, it doesn't tell you the content is actually right — you
could get a well-formed but empty or nonsensical result. So I'd still validate
the parsed output against whatever business rules actually matter, and I'd still
have a retry path with the error fed back to the model for the cases that do
fail, rather than assuming this is now a solved problem. I'd also just track the
failure rate over time, because if it starts climbing that tells you something
changed upstream."
</details>

---

### E2.10 · Explain: a cost/latency optimisation strategy end to end

**Prompt:** *"Your feature is in production and costs have grown 4x in three
months as usage scaled. Walk me through how you'd approach bringing that down,
in order."*

**Target:** 90 seconds spoken.

<details>
<summary>Rubric — score yourself</summary>

**Must hit:**
- [ ] Starts with measurement — break down cost by call type/feature/model
      before optimising blindly
- [ ] Identifies model tiering (cheaper model for easier subtasks) as a
      primary lever (ties to Q2.20/Q2.29)
- [ ] Identifies prompt caching as a lever if there's a stable, repeated prefix
      (ties to Q2.11/Q2.17)
- [ ] Considers reducing unnecessary token volume — trimming prompt bloat,
      capping output length appropriately, managing conversation history growth
      (ties to E2.2/Q2.24)
- [ ] States that any optimisation needs to be validated against quality — a
      cost win that silently degrades output isn't actually a win

**Bonus:**
- [ ] Mentions batch API as a lever for any non-real-time subset of the
      workload
- [ ] Frames this as ongoing monitoring, not a one-time fix — cost should be a
      tracked metric going forward
- [ ] Prioritises levers by expected impact vs implementation effort rather
      than doing them in an arbitrary order

**Red flags:**
- [ ] Jumps straight to "switch to a cheaper model everywhere" without
      measurement or quality consideration
- [ ] Proposes only one lever instead of a portfolio of options
- [ ] Never mentions verifying quality didn't regress after the changes

**Score: ___ / 5**

**Model answer:**
"First thing I'd do is actually break down where the cost is coming from —
which calls, which features, input versus output — because '4x in three months'
could be pure volume growth, which is a good problem, or it could be something
like unbounded conversation history quietly getting more expensive per call,
which is a real issue to fix. Once I know that, I'd look at model tiering first
— are we sending every single step to the most expensive model when some of them
are pretty mechanical and could run on something cheaper? Then I'd check if
there's a stable, repeated prefix anywhere that isn't cached yet, because that's
often a big win for not much engineering effort. And I'd look at whether we're
sending more tokens than we need to — bloated prompts, no cap on output length,
that kind of thing. Whatever I change, though, I'd validate against actual
output quality before calling it done, because a cheaper answer that's worse
isn't actually a win, it just moves the cost somewhere less visible."
</details>

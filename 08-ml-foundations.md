# 08 · ML Foundations

This file is deliberately conceptual — no backpropagation derivations, no
matrix calculus, no "implement attention from scratch." Entry-level AI
Engineer and FDE interviews rarely ask you to derive anything; they ask
whether you understand *what's actually happening* well enough to reason
about behavior, make a build-vs-buy call, or explain to a stakeholder why the
model just did something surprising. That's the bar this file is written to.

---

## Multiple choice

### Q8.1 · What attention actually does · [Recall]

In plain terms, what does the attention mechanism let a transformer do that
a simple fixed-window model (like averaging nearby word embeddings) can't?

- **A.** It lets the model process input faster than any other architecture
- **B.** It lets each token dynamically weigh and pull information from every other token in the context, based on learned relevance, rather than being limited to a fixed nearby window or a fixed averaging rule
- **C.** It eliminates the need for any training data
- **D.** It guarantees the model never hallucinates, since it can always "look back" at the source

<details>
<summary>Answer</summary>

**B**

**Why B:** Attention computes, for each token, a weighted combination of
*all* other tokens in context, where the weights are learned and depend on
content — "the cat that chased the mouse ran" can let "ran" attend strongly
back to "cat" even though several words separate them. A fixed window or
uniform averaging can't do this — it either misses long-range relationships
or treats all nearby tokens as equally relevant regardless of content.

**Why not A:** Attention isn't primarily a speed optimization — in fact,
naive self-attention is quadratic in sequence length, which is a real
scaling cost, not a speed advantage, addressed separately by architectural
and systems optimizations.

**Why not C:** Attention is a mechanism within a model architecture that
still needs to be trained on data — it has nothing to do with removing the
need for training data.

**Why not D:** Attention lets the model access context more effectively, but
it's a mechanism for weighting relevance, not a guarantee of factual
grounding — a model can attend perfectly well to the wrong thing, or attend
to context and still hallucinate past it, as covered in file 04's RAG
content.

**Interviewer's likely follow-up:** *"Why is quadratic scaling with sequence
length a practical problem?"* (Answer: cost and latency grow much faster
than linearly as context grows, which is a big part of why long-context
windows are expensive and why techniques like KV-caching, sparse attention,
or chunking exist — connects to file 02's cost/latency content.)
</details>

### Q8.2 · Transformer architecture, block-diagram level · [Recall]

At a block-diagram level, what are the two main sub-components repeated in
each layer of a standard transformer, and what does each roughly do?

- **A.** A convolution layer for local pattern detection, and a pooling layer for downsampling
- **B.** A self-attention sub-layer, which lets tokens exchange information based on relevance, and a feed-forward sub-layer, which processes each token's representation independently to add further transformation capacity
- **C.** An encoder and a decoder, which are always both present in every transformer regardless of task
- **D.** A tokenizer sub-layer and an embedding sub-layer, repeated at every depth

<details>
<summary>Answer</summary>

**B**

**Why B:** This is the standard block-diagram description — each transformer
layer alternates a self-attention sub-layer (tokens mixing information based
on learned relevance) with a position-wise feed-forward sub-layer (the same
small network applied independently to each token's representation),
typically with residual connections and normalization around each. This
holds across most decoder-only LLMs used today.

**Why not A:** Convolution and pooling are the defining components of CNN
architectures, not transformers — this describes a completely different
architecture family.

**Why not C:** Encoder-decoder is one transformer variant (used in original
translation-style transformers), but most modern LLMs (including the
decoder-only architecture used by most chat models) use only a decoder
stack — "always both present" overstates a specific historical variant as a
universal rule.

**Why not D:** Tokenization and embedding happen once, at the input stage,
not repeated at every layer — this option confuses input processing with
the repeated internal layer structure.

**Interviewer's likely follow-up:** *"What's the role of the residual
connections around these sub-layers?"* (Answer: they let gradients and
information flow more directly through many stacked layers during training,
which is a big part of why very deep transformers are trainable at all —
without derivation, just the practical "why it helps.")
</details>

### Q8.3 · Pretraining vs post-training vs RLHF · [Recall]

A candidate is asked to explain, at a conceptual level, the difference
between pretraining, post-training/fine-tuning, and RLHF. Which explanation
is most accurate?

- **A.** They're three names for the same process, just used interchangeably by different companies
- **B.** Pretraining teaches the model general language patterns and world knowledge from massive raw text (next-token prediction); post-training/fine-tuning adapts the pretrained model toward specific behaviors using smaller, curated datasets (e.g., instruction-following examples); RLHF is a specific post-training technique that uses human preference feedback to further shape the model's outputs toward what humans actually prefer, beyond what supervised examples alone capture
- **C.** Pretraining happens after RLHF, since RLHF requires a base model to compare against
- **D.** RLHF replaces pretraining entirely for modern models, since human feedback is more valuable than raw text

<details>
<summary>Answer</summary>

**B**

**Why B:** This captures the standard pipeline conceptually: massive
self-supervised pretraining builds general capability, then post-training
stages (instruction tuning, RLHF, and related techniques) shape that raw
capability into a model that follows instructions and produces outputs
humans actually prefer. RLHF specifically uses human preference comparisons
(often via a reward model) rather than just labeled correct/incorrect
examples.

**Why not A:** These are distinct, sequential stages with different data,
objectives, and techniques — treating them as interchangeable loses a
distinction interviewers specifically probe for.

**Why not C:** The ordering is backwards — RLHF and other post-training
techniques are applied *to* an already-pretrained base model; pretraining
has to happen first to produce something worth further shaping.

**Why not D:** RLHF doesn't replace pretraining — pretraining is what gives
the model its broad language ability and knowledge in the first place; RLHF
operates on top of that, shaping behavior, not building the foundational
capability from scratch.

**Interviewer's likely follow-up:** *"Why can't you skip straight to RLHF
on a randomly initialized model?"* (Answer: RLHF shapes behavior on top of
existing capability — a model with no pretrained language ability has
nothing coherent to shape; human preference feedback improves *which* of
the model's fluent outputs it prefers, it doesn't teach fluency itself.)
</details>

### Q8.4 · Why hallucination happens mechanistically · [Applied]

At a conceptual level, why do LLMs "hallucinate" — state incorrect facts
confidently — even when they've been trained on enormous amounts of correct
information?

- **A.** Hallucination happens because the model is deliberately trying to deceive the user
- **B.** The model is fundamentally a next-token predictor optimizing for plausible, fluent continuations based on learned patterns — it has no built-in mechanism that distinguishes "this specific claim is verified true" from "this is a fluent, plausible-sounding continuation," so a confident-sounding but false statement can be just as probable a continuation as a true one, especially for facts underrepresented or absent in training data
- **C.** Hallucination is purely a bug that will be fully eliminated by simply using a larger model
- **D.** Hallucination only happens when temperature is set above zero

<details>
<summary>Answer</summary>

**B**

**Why B:** This is the core mechanistic explanation — the model has no
internal "truth" flag separate from "plausible continuation." Fluency and
factual accuracy are correlated in training data (correct statements tend to
read naturally) but not identical, and for facts that are rare, ambiguous,
or absent in training data, the model still produces a fluent continuation
because that's literally what it's optimized to do — it just isn't
necessarily a true one.

**Why not A:** "Deliberately trying to deceive" implies intent and a model
of truth the system doesn't have — this anthropomorphizes a statistical
generation process in a way that misleads about the actual mechanism.

**Why not C:** Scaling up model size generally improves factual accuracy on
average but doesn't eliminate the underlying mechanism — a bigger model
still lacks a built-in ground-truth verification step; it can still
confidently generate a plausible but false continuation, especially for
knowledge outside or sparse in its training distribution.

**Why not D:** Hallucination happens even at temperature 0 (fully greedy
decoding) — temperature affects sampling randomness, not whether the model
has a mechanism to verify factual correctness, which is the actual root
cause.

**Interviewer's likely follow-up:** *"Given that mechanism, why does RAG
help reduce (but not eliminate) hallucination?"* (Answer: RAG gives the
model relevant retrieved text in-context to ground its continuation against,
which shifts the "most plausible continuation" toward something actually
supported by the provided text — but the model can still ignore, misread,
or fill gaps in that context with unsupported claims, which is exactly the
groundedness problem covered in files 04 and 06.)
</details>

### Q8.5 · Embeddings vs generative models · [Recall]

What's the fundamental difference in what an embedding model and a
generative (text-completion) model are trained to output?

- **A.** There's no real difference — any generative model can be used identically as an embedding model with no tradeoffs
- **B.** An embedding model is trained to output a fixed-size vector representing the semantic content of an input, useful for comparison/similarity (e.g., via cosine similarity); a generative model is trained to output a probability distribution over the next token, useful for producing new text one token at a time
- **C.** Embedding models only work on single words, while generative models work on full sentences
- **D.** Generative models produce vectors, and embedding models produce text

<details>
<summary>Answer</summary>

**B**

**Why B:** This is the core distinction that underlies why RAG pipelines
(file 04) use a separate embedding model for retrieval and a separate
generative model for producing the answer — embeddings are optimized so
that semantic similarity in the vector space reflects semantic similarity in
meaning, which is a very different training objective from next-token
prediction, even though both can be built on similar underlying
architectures.

**Why not A:** While it's technically possible to derive embeddings from a
generative model's internal representations, purpose-trained embedding
models are typically specifically optimized (often via contrastive
objectives) for the similarity-comparison task, and generative models used
naively for this don't perform equivalently — treating them as
interchangeable with no tradeoffs is inaccurate.

**Why not C:** Both types of models commonly operate on spans larger than a
single word — sentences, paragraphs, or documents — this option invents a
distinction that doesn't reflect how either is actually used.

**Why not D:** This reverses the actual outputs — generative models produce
text (token by token), embedding models produce vectors; this option has
the mapping backwards.

**Interviewer's likely follow-up:** *"Could you use one model for both
generation and embeddings in a single pipeline?"* (Answer: technically some
models can be adapted for both, but purpose-built separation is more common
in production because the training objectives that make a model good at one
task don't automatically make it good at the other.)
</details>

### Q8.6 · What fine-tuning actually changes · [Applied]

You fine-tune a base LLM on 5,000 examples of your company's internal support
tickets and their resolutions, hoping the model will "know" your product
better afterward. What does fine-tuning on this data actually change about
the model, and what does it *not* give the model?

- **A.** Fine-tuning gives the model perfect, always-up-to-date recall of every fact in the 5,000 examples, the same as looking them up directly
- **B.** Fine-tuning adjusts the model's weights to shift its behavior and style toward patterns present in the fine-tuning data (e.g., response format, tone, common resolution patterns) — it does not reliably give the model precise, retrievable factual recall of specific details the way a lookup does, and it doesn't automatically stay current as your product changes after training
- **C.** Fine-tuning has no effect at all unless the dataset is at least a million examples
- **D.** Fine-tuning replaces the model's general language ability entirely with only the fine-tuning domain's knowledge

<details>
<summary>Answer</summary>

**B**

**Why B:** Fine-tuning nudges weights toward patterns in the training
data — it's good at teaching *style*, *format*, and *general behavioral
tendencies* (how to phrase a resolution, what tone to use), but it's a poor
mechanism for reliable, precise factual recall of specific details, and
critically it's a static snapshot — once trained, it doesn't know about
product changes that happen afterward unless you retrain. This is exactly
why file 04's RAG-vs-fine-tuning framing generally favors RAG for
knowledge that needs to be current and precisely retrievable, and
fine-tuning for shifting behavior/style.

**Why not A:** This is the most common and costly misconception about
fine-tuning — it doesn't function like a database lookup, and expecting
"perfect recall" from fine-tuning routinely leads to disappointing,
hallucination-prone results in practice.

**Why not C:** Fine-tuning can have a meaningful effect on smaller datasets
too, especially for parameter-efficient methods (Q8.7) — the claim that
nothing happens below a million examples is an invented, overly rigid
threshold.

**Why not D:** Fine-tuning adjusts behavior on top of the model's existing
general capability — it doesn't wipe out general language ability, though
poorly done fine-tuning (too aggressive, too narrow a dataset) can degrade
general performance as a side effect, which is a real risk but different
from "replaces entirely."

**Interviewer's likely follow-up:** *"So when would fine-tuning actually be
the right call over RAG?"* (Answer: when you need to change *how* the model
behaves — format, tone, domain-specific reasoning style, following a
particular structured output convention consistently — rather than *what
facts* it has access to; the two are often complementary, not competing.)
</details>

### Q8.7 · LoRA and parameter-efficient tuning · [Applied]

Conceptually, what does LoRA (Low-Rank Adaptation) do differently from full
fine-tuning, and why does that matter practically?

- **A.** LoRA retrains the entire model's weights but uses a smaller learning rate
- **B.** LoRA freezes the original model's weights and trains a small number of additional low-rank matrices that get added to specific layers, adjusting behavior with far fewer trainable parameters than updating the whole model — this makes fine-tuning dramatically cheaper in compute/memory while still adapting behavior meaningfully
- **C.** LoRA is a data augmentation technique that generates more training examples, unrelated to model weights
- **D.** LoRA only works for image models, not language models

<details>
<summary>Answer</summary>

**B**

**Why B:** This is the core idea — instead of updating every weight in a
potentially huge model (expensive in both compute and memory, and requiring
storing a full new copy of the model per fine-tune), LoRA inserts small,
trainable low-rank matrices alongside frozen original weights. Far fewer
parameters need training and storing, which makes fine-tuning accessible on
much more modest hardware and makes it practical to maintain many
lightweight task-specific adapters instead of many full model copies.

**Why not A:** This describes standard full fine-tuning with a smaller
learning rate, not LoRA's actual structural difference — a smaller learning
rate doesn't reduce the number of parameters being updated or stored.

**Why not C:** LoRA is specifically a parameter-efficient *model
adaptation* technique, operating on the model's weights — it has nothing to
do with generating additional training data.

**Why not D:** LoRA is a general technique applicable across model
architectures including language models — it's widely used for LLM
fine-tuning specifically, not limited to image models.

**Interviewer's likely follow-up:** *"What's the tradeoff versus full
fine-tuning — is LoRA strictly better?"* (Answer: not strictly — LoRA is
usually cheaper and faster with often-comparable results for many tasks, but
full fine-tuning can sometimes reach higher performance ceilings for very
large distribution shifts from the base model's training, so it's a
cost/performance tradeoff, not a free upgrade.)
</details>

### Q8.8 · Distillation · [Recall]

What is model distillation, conceptually, and why would a team use it?

- **A.** Distillation removes duplicate training examples from a dataset before training
- **B.** Distillation trains a smaller "student" model to mimic the outputs/behavior of a larger "teacher" model, aiming to capture much of the teacher's capability in a cheaper, faster, smaller package suitable for latency- or cost-sensitive deployment
- **C.** Distillation is another name for quantization — they refer to the identical process
- **D.** Distillation only applies to reducing a dataset's size, not a model's size

<details>
<summary>Answer</summary>

**B**

**Why B:** This is the standard definition — a large, capable (often
expensive/slow) teacher model's outputs (or richer training signal derived
from it) are used to train a smaller student model, transferring much of
the teacher's useful behavior into a package that's cheaper and faster to
run at inference time. This is a common lever when a team needs to hit a
latency or cost budget that the largest available model can't meet.

**Why not A:** Deduplication is a data-cleaning step unrelated to training a
second, smaller model from a larger one's behavior.

**Why not C:** Distillation and quantization are both model-compression
techniques but operate differently — distillation trains a new, smaller
model on the teacher's behavior; quantization (Q8.9) reduces the numerical
precision of an existing model's weights. They're often used together but
aren't the same technique.

**Why not D:** Distillation is specifically about compressing a *model*
(transferring capability to a smaller architecture), not about reducing
dataset size — this option confuses two unrelated meanings of "smaller."

**Interviewer's likely follow-up:** *"What's typically lost in the process,
and how do you decide if it's an acceptable tradeoff?"* (Answer: some
capability/quality typically degrades relative to the teacher, especially on
less common or more complex tasks — the decision usually comes down to
whether the smaller model clears your task-specific quality bar via
evaluation, at the cost/latency point you actually need — ties back
directly to file 06.)
</details>

### Q8.9 · Quantization and its tradeoffs · [Applied]

You're deploying a model on infrastructure with limited memory and need to
reduce its footprint. A colleague suggests quantizing the model from
16-bit to 4-bit precision. What's actually happening, and what's the
tradeoff?

- **A.** Quantization removes entire layers from the model to make it smaller
- **B.** Quantization reduces the numerical precision used to represent the model's weights (and sometimes activations), shrinking memory footprint and often improving inference speed, at the cost of some accuracy/quality degradation — the severity of that degradation depends on how aggressively you quantize and which quantization technique is used
- **C.** Quantization has no effect on model quality whatsoever at any bit-width, since modern techniques are lossless
- **D.** Quantization only affects training time, not inference

<details>
<summary>Answer</summary>

**B**

**Why B:** Quantization represents weights with fewer bits (e.g., 16-bit
floats down to 8-bit or 4-bit integers/floats), which directly reduces
memory footprint and can speed up inference on hardware that benefits from
lower-precision arithmetic. This isn't free — representing values with less
precision loses some information, and depending on how aggressive the
quantization is and the specific method used, this can measurably degrade
output quality, which is why aggressive quantization needs to be validated
against your evaluation suite (file 06), not assumed safe.

**Why not A:** Removing layers is a structural pruning technique, distinct
from quantization, which changes numerical precision, not the model's
architecture/depth.

**Why not C:** While well-designed quantization schemes can be quite
close to lossless at moderate bit-widths (e.g., 8-bit), aggressive
quantization (e.g., 4-bit or lower) generally does introduce measurable
quality degradation — claiming it's always lossless overstates the
technique's guarantees.

**Why not D:** Quantization primarily affects inference-time memory and
speed characteristics (and can also be applied during/after training) —
framing it as training-time-only misses its main practical use case in
deployment.

**Interviewer's likely follow-up:** *"How would you decide if 4-bit
quantization is acceptable for your use case?"* (Answer: run your golden
eval set against the quantized model and compare scores to the full-precision
baseline — the same regression-suite discipline from file 06 — rather than
assuming a bit-width is safe without measuring it on your actual task.)
</details>

### Q8.10 · Overfitting and train/test split, why it still matters for LLM systems · [Design]

A colleague says "overfitting and train/test splits are a training-time
concern — since we're just calling a pretrained API model via prompting, not
training anything, none of that applies to us." Why is this wrong?

- **A.** They're completely right — train/test split concepts genuinely don't apply once you're just prompting an API model
- **B.** The underlying principle — don't evaluate a system on the same data you used to develop/tune it — still applies: if you iteratively tweak your prompt or few-shot examples by checking performance against the same small set of examples repeatedly, you can "overfit" your prompt to that specific set, producing a prompt that looks great on those examples but generalizes poorly to real, unseen inputs, exactly analogous to overfitting a trained model to its training set
- **C.** Overfitting can only happen to model weights, so prompt engineering is structurally immune to it by definition
- **D.** This concern only applies to fine-tuning, not to any other part of an LLM-based system

<details>
<summary>Answer</summary>

**B**

**Why B:** The core principle — generalization requires evaluating on data
you didn't use to make your decisions — transfers directly to prompt
engineering. If you iterate on prompt wording using the same 10 examples
repeatedly until they all pass, you've effectively "overfit" your prompt to
those 10 examples, and it may perform notably worse on the broader
real-world distribution it'll actually face — this is exactly why a proper
held-out golden set (file 06) matters even for pure prompting work with no
weight updates at all.

**Why not A:** This is the misconception the question is testing — the
concept generalizes beyond literal weight training to any iterative process
of tuning something against a fixed evaluation set, prompts included.

**Why not C:** Overfitting is a general phenomenon about optimizing/tuning
against a fixed sample and losing generalization — it's not mechanically
restricted to gradient-based weight updates; the same failure pattern shows
up whenever you repeatedly adjust something based on the same fixed
feedback set.

**Why not D:** The concern applies to *any* iterative process tuned against
a fixed sample — this includes prompt engineering, few-shot example
selection, and fine-tuning alike, not just the fine-tuning case
specifically.

**Interviewer's likely follow-up:** *"So practically, how do you avoid
overfitting your prompt during development?"* (Answer: keep a held-out
portion of your golden set that you don't look at while iterating, and only
check final performance against it occasionally — the same train/validation/
test discipline as traditional ML, applied to prompt development instead of
weight training.)
</details>

### Q8.11 · Tokens vs parameters, a common confusion · [Recall]

A candidate says "a bigger model always has a bigger context window, since
both come from having more parameters." Why is this conflation incorrect?

- **A.** It isn't incorrect — parameter count and context window size are always directly proportional in every model
- **B.** Parameter count (the model's total learned weights, related to capacity/knowledge) and context window (how many tokens of input the model can process at once, an architectural/design choice often tied to how attention and position encoding are implemented) are separate design dimensions — a model can be scaled up in parameters without necessarily increasing context length, and vice versa
- **C.** Context window size is fixed at exactly 4,096 tokens for all transformer models by mathematical necessity
- **D.** Parameter count only affects training speed, not model capability

<details>
<summary>Answer</summary>

**B**

**Why B:** These are genuinely independent axes of model design. Parameter
count relates to the model's overall representational capacity, learned
during pretraining. Context window length is a separate architectural
choice — how attention and position encoding are implemented and what the
model was trained/extended to handle — that model builders can extend
somewhat independently of parameter count through specific architectural and
training choices. Conflating the two is a common but incorrect
simplification.

**Why not A:** There's no such direct proportionality — model families
regularly ship variants with different context lengths at similar parameter
counts, and vice versa, demonstrating these aren't locked together.

**Why not C:** Context window sizes vary widely across models and have grown
substantially over time (from thousands to hundreds of thousands of tokens
in various modern models) — there's no fixed mathematical ceiling at 4,096;
that's just a value some earlier models happened to use.

**Why not D:** Parameter count is much more directly tied to the model's
learned capability/knowledge capacity than to training speed specifically —
this option significantly understates what parameter count actually
represents.

**Interviewer's likely follow-up:** *"Why doesn't a huge context window
alone make RAG unnecessary?"* (Answer: this connects to file 04's RAG-vs-
long-context framing — even with a large context window, stuffing
everything in is often more expensive, slower, and can suffer from
attention/recall degradation over very long contexts, so retrieval that
selects only the relevant material often still outperforms brute-force
long-context stuffing.)
</details>

### Q8.12 · Emergent behavior and scale, a calibrated view · [Design]

A colleague claims "if we just use a big enough model, reasoning ability
will automatically emerge with no other changes needed — capability is
purely a function of parameter count." How would a well-calibrated engineer
push back on this?

- **A.** Agree completely — parameter count alone has historically been the single reliable predictor of every capability improvement
- **B.** Push back by noting that training data quality/quantity, architecture choices, and post-training techniques (instruction tuning, RLHF) have all independently driven major capability gains alongside scale — scale is one important lever, not the only one, and some capabilities have come from better post-training or data curation rather than raw parameter growth alone
- **C.** Push back by claiming scale has never correlated with any capability improvements at all
- **D.** Agree, but only because parameter count is the sole variable that matters for cost, not capability

<details>
<summary>Answer</summary>

**B**

**Why B:** This is the calibrated, defensible position — scale (parameters,
data, compute) has clearly driven major capability jumps historically, but
it's not the only lever; substantial capability improvements have also come
from better training data curation, architectural refinements, and
post-training techniques like instruction tuning and RLHF that shape how
well a model's raw capability translates into useful, reliable behavior.
Treating scale as the sole variable oversimplifies a genuinely multi-factor
process.

**Why not A:** This overstates scale's role as the exclusive driver, ignoring
the substantial documented impact of data quality and post-training choices
on capability and reliability.

**Why not C:** This overcorrects in the opposite direction — scale
genuinely has correlated with meaningful capability improvements
historically; denying any relationship is just as miscalibrated as claiming
it's the sole factor.

**Why not D:** This conflates a claim about capability with an unrelated
claim about cost, and still incorrectly treats parameter count as the only
relevant variable — cost is driven by more than parameter count too
(inference infrastructure, context length, quantization, and more).

**Interviewer's likely follow-up:** *"Give a concrete example of a
capability gain that came primarily from post-training rather than raw
scale."* (Answer: a base pretrained model's raw ability to follow multi-step
instructions reliably or produce well-formatted structured output typically
improves dramatically after instruction-tuning/RLHF-style post-training,
even without changing the underlying parameter count — the base model
"knows" a lot but doesn't reliably *apply* it usefully until shaped by
post-training.)
</details>

### Q8.13 · Why smaller models sometimes outperform larger ones on a task · [Applied]

Your team fine-tunes a small, task-specific model for a narrow classification
task (routing support tickets into 5 categories) and finds it outperforms a
much larger general-purpose model prompted zero-shot on the same task. Why
is this a plausible, non-surprising outcome?

- **A.** It isn't plausible — a larger general-purpose model should always outperform a smaller model on any task
- **B.** A small model fine-tuned specifically on the narrow task distribution can specialize its limited capacity entirely toward that task's patterns, while a larger general-purpose model's broader capability doesn't automatically translate into optimal performance on one narrow task without task-specific adaptation — "more general capability" and "best at this one specific narrow task" aren't the same thing
- **C.** This can only happen if the larger model's API is broken
- **D.** Task-specific fine-tuning always requires more parameters than general-purpose pretraining, so this outcome is actually the small model secretly being larger

<details>
<summary>Answer</summary>

**B**

**Why B:** A small, task-specialized model can dedicate its entire capacity
to the narrow distribution of the actual task, effectively becoming very
good at exactly that one thing, whereas a large general-purpose model
spreads its representational capacity across an enormous range of
capabilities and hasn't been specifically adapted to this narrow
classification task — general capability and narrow-task optimality are
different properties, and this outcome is a well-documented, unsurprising
pattern in practice, not an anomaly.

**Why not A:** This assumes a strict capability hierarchy that doesn't hold
in practice — task-specific specialization is a well-known way smaller
models can beat larger general-purpose ones on narrow, well-defined tasks.

**Why not C:** Nothing in the scenario suggests an API malfunction — this
outcome is a normal, expected result of specialization versus generality,
not evidence of a technical failure.

**Why not D:** This is a nonsensical inference — model size and whether it's
been task-fine-tuned are independent facts, and there's no mechanism by
which fine-tuning secretly increases parameter count.

**Interviewer's likely follow-up:** *"So when would you choose the large
general-purpose model instead, even for a narrow-seeming task?"* (Answer:
when the task's distribution is likely to shift or expand over time, when
you don't have enough labeled data to fine-tune reliably, or when you need
the model's broader reasoning/world knowledge to handle edge cases outside
the narrow training distribution — the tradeoff is specialization/efficiency
versus flexibility/generalization.)
</details>

### Q8.14 · Which is NOT true about embeddings · [Recall]

Which of the following statements about embedding vectors is **NOT** true?

- **A.** Embeddings from the same model are typically compared using a similarity metric like cosine similarity
- **B.** Semantically similar inputs tend to produce embedding vectors that are close together in the vector space
- **C.** Embeddings produced by different, unrelated embedding models can generally be compared directly against each other in the same vector space with meaningful results
- **D.** Embedding dimensionality is a fixed property of a given embedding model, not something that varies per input

<details>
<summary>Answer</summary>

**C**

**Why C is the answer (this is a "NOT" question):** Embeddings from
different models are trained with different objectives, data, and internal
representations — their vector spaces aren't aligned or comparable to each
other. Mixing embeddings from two different models in the same similarity
search (e.g., embedding your documents with one model and your queries with
another) produces meaningless results; both sides of any comparison need to
come from the same embedding model.

**Why A is true:** Cosine similarity is the standard comparison metric for
embeddings, exactly because it captures directional (semantic) similarity
independent of vector magnitude — this is accurate and matches file 04's
content on why cosine over euclidean.

**Why B is true:** This is the entire premise embeddings are built on —
semantic proximity in meaning should correspond to proximity in vector
space, which is what makes similarity search useful in the first place.

**Why D is true:** A given embedding model produces vectors of a fixed,
consistent dimensionality (e.g., always 1536-dimensional) regardless of
input length or content — dimensionality is a property of the model
architecture, not the specific input.

**Interviewer's likely follow-up:** *"What happens in practice if you
accidentally mix embeddings from two different model versions in one vector
store?"* (Answer: retrieval quality silently degrades or becomes essentially
random, because similarity scores between incompatible vector spaces don't
reflect real semantic relationships — this is a real, easy-to-make
production mistake, especially when upgrading an embedding model without
re-embedding the existing corpus.)
</details>

### Q8.15 · Explaining "what is a transformer" to a non-technical stakeholder · [Applied]

A non-technical stakeholder asks you to explain, in one or two sentences,
what a transformer model actually does, without jargon. Which explanation
best balances accuracy and accessibility?

- **A.** "It's a neural network that processes text using matrix multiplication and softmax functions across multiple attention heads"
- **B.** "It reads all the words in what you type at once and figures out how much each word should influence its understanding of every other word, which lets it build a much richer sense of what you actually mean than reading word by word in isolation would"
- **C.** "It's basically a very large lookup table of pre-written responses"
- **D.** "It's magic — nobody really knows how it works, including the people who built it"

<details>
<summary>Answer</summary>

**B**

**Why B:** This captures the real mechanism (attention letting words
inform each other's interpretation) in plain, accurate language without
technical jargon like "matrix multiplication" or "softmax" — it's the kind
of explanation that helps a non-technical stakeholder build correct
intuition without needing any background, which is exactly the skill this
scenario is testing.

**Why not A:** This is accurate but violates the explicit "no jargon"
constraint — matrix multiplication, softmax, and attention heads are exactly
the kind of terms that would lose a non-technical audience, defeating the
purpose of the explanation.

**Why not C:** This is factually wrong and actively misleading — the model
isn't retrieving pre-written responses from a lookup table, it's generating
novel text token by token based on learned patterns; this explanation would
give the stakeholder an incorrect mental model.

**Why not D:** While transformer internals do have real interpretability
open questions, "nobody knows how it works, it's magic" is an
oversimplification that misrepresents a well-understood architecture (we
know its structure and training process in detail) as a total mystery —
it's evasive rather than genuinely explanatory.

**Interviewer's likely follow-up:** *"How would you extend that explanation
if the stakeholder then asked why it sometimes makes things up?"* (Answer:
connect back to Q8.4's hallucination mechanism in equally plain language —
it's predicting what sounds like a natural continuation, not checking a
fact database, so a fluent-sounding wrong answer is possible in a way it
wouldn't be for a lookup system.)
</details>

### Q8.16 · Base model vs instruction-tuned/chat model · [Recall]

What's the practical difference between a "base model" and an
"instruction-tuned" or "chat" version of the same underlying model, and why
does it matter which one you use in an application?

- **A.** There's no meaningful practical difference; they're marketing labels for the same behavior
- **B.** A base model is trained primarily to continue text plausibly (next-token prediction on raw data) and often doesn't reliably follow instructions or converse naturally; an instruction-tuned/chat model has undergone additional post-training specifically to follow instructions, hold conversations, and produce more directly useful responses — using a raw base model in a conversational application typically produces erratic, unhelpful completions rather than direct answers
- **C.** Base models are always smaller than instruction-tuned models
- **D.** Instruction-tuned models can never be used for anything except chat interfaces

<details>
<summary>Answer</summary>

**B**

**Why B:** This is the practical, application-relevant distinction — a raw
base model, given "What's the capital of France?", might continue with more
text in a similar style rather than directly answering, because it's just
predicting plausible continuations of the pattern it's seen, not
specifically trained to be a helpful respondent. Instruction-tuning/RLHF
(file 08's own Q8.3 pipeline) is what makes a model reliably behave like a
helpful assistant rather than a raw text-continuation engine — this is
exactly why almost all production applications use the instruction-tuned/
chat variant, not the base model.

**Why not A:** This is a real, practically significant difference in
behavior, not a naming formality — using the wrong variant produces
noticeably different (and usually much less useful) results.

**Why not C:** Model size is an independent property from whether a model
has undergone instruction-tuning — base and instruction-tuned versions of
the same underlying model are typically the same size, differing in
post-training, not parameter count.

**Why not D:** Instruction-tuned models are used far beyond chat
interfaces — structured output generation, tool use, summarization, and
most application-layer LLM use cases all rely on instruction-tuned models,
not literal chat UIs specifically.

**Interviewer's likely follow-up:** *"Are there any cases where you'd
actually prefer the raw base model?"* (Answer: rare, specialized cases like
research into raw model behavior, certain few-shot completion-style setups,
or as a base for further custom fine-tuning where you want to avoid
inheriting default assistant-style behavior — but virtually all standard
application use cases want the instruction-tuned variant.)
</details>

### Q8.17 · Why more training data doesn't always help · [Design]

A team wants to improve a model's performance on a specific reasoning task by
fine-tuning on a much larger dataset scraped broadly from the internet,
loosely related to the task. A more experienced colleague pushes back. What's
the strongest reason for that pushback?

- **A.** More data can never help under any circumstances, so this is always a bad idea
- **B.** Data quality and relevance to the specific target task/distribution often matter more than raw volume — a large but loosely related, noisy dataset can dilute or even degrade performance on the specific task compared to a smaller, well-curated, task-relevant dataset, especially for fine-tuning aimed at a narrow behavior
- **C.** Fine-tuning datasets must always be smaller than 1,000 examples by definition
- **D.** More data only matters for pretraining, never for fine-tuning of any kind

<details>
<summary>Answer</summary>

**B**

**Why B:** This mirrors the "garbage in, garbage out" principle applied
specifically to fine-tuning: a fine-tuning dataset's job is to shift the
model's behavior toward the target task's specific patterns, and a large
volume of loosely related, noisy data can actually dilute that signal or
introduce unwanted patterns, compared to a smaller set that's tightly
matched to the actual target distribution. This is a well-known practical
lesson distinguishing pretraining (where broad scale genuinely helps) from
task-specific fine-tuning (where relevance and quality often dominate).

**Why not A:** This overstates the case — more data absolutely can help
when it's relevant and well-curated; the actual concern is about
irrelevant, noisy volume specifically, not data quantity as a universal
negative.

**Why not C:** There's no fixed universal size threshold for fine-tuning
datasets — the right size depends heavily on the task, the base model, and
the fine-tuning method (e.g., LoRA can be effective with smaller sets than
full fine-tuning) — inventing a specific cutoff misrepresents how this
actually works.

**Why not D:** Data quality/relevance considerations apply throughout —
they matter for pretraining too, but the specific pushback in this scenario
is about fine-tuning dataset composition, where the relevance argument is
especially sharp because fine-tuning data is meant to closely target a
specific behavior.

**Interviewer's likely follow-up:** *"How would you actually validate
whether the larger dataset helped or hurt?"* (Answer: this is exactly what
file 06's evaluation discipline is for — run both fine-tuned variants
(small curated set vs. large loosely-related set) against the same golden
eval set for the target task and compare, rather than assuming more data is
better without measuring.)
</details>

### Q8.18 · Multimodal models, conceptual boundary · [Applied]

A stakeholder asks whether your text-based LLM assistant can "just look at"
a photo a customer uploaded to understand a product defect. What's the
correct conceptual framing of what's needed to make this work?

- **A.** Any LLM can process images natively without any architectural changes, since text and images are fundamentally the same kind of data to a neural network
- **B.** The model needs to be multimodal — specifically trained/architected to process image inputs (typically by converting image patches into a representation the model can attend over, alongside text tokens) — a purely text-trained model has no mechanism to interpret raw pixel data at all, so this requires using a model variant with that capability, not a configuration change to a text-only model
- **C.** Images can be converted to text via any embedding model and fed to a text-only LLM with identical results to true multimodal processing
- **D.** This is only possible by fine-tuning the text model on enough image captions

<details>
<summary>Answer</summary>

**B**

**Why B:** A text-only model has no learned representation for raw image
data — multimodal capability requires a model specifically trained/
architected to handle image inputs (often via a vision encoder that
converts image content into a representation the transformer can attend
over alongside text tokens). This is a real architectural distinction, not
just a prompting or configuration choice — you need the right model
variant, not a clever workaround on a text-only one.

**Why not A:** This is a significant oversimplification — a purely
text-trained model genuinely cannot process raw pixel data; multimodal
capability requires specific architecture and training designed for it, not
an assumption that all data types are equivalent to a neural network.

**Why not C:** A generic embedding model isn't the same as true multimodal
image understanding — converting an image through an unrelated embedding
process and feeding a vector to a text-only model doesn't give the model a
real, trained ability to interpret visual content the way a purpose-built
multimodal model does.

**Why not D:** Fine-tuning a fundamentally text-only architecture on image
captions doesn't give it a mechanism to process raw image pixels at all —
captions are still just text; you'd be training on descriptions of images,
not giving the model any actual visual processing capability.

**Interviewer's likely follow-up:** *"What would you tell the stakeholder
about the practical next step?"* (Answer: recommend using a multimodal-capable
model for this specific feature rather than trying to force the existing
text-only pipeline to handle images, and scope the change as using a
different model/capability, not a workaround.)
</details>

### Q8.19 · Knowledge cutoff, a practical implication · [Applied]

A customer-facing assistant confidently tells a user about a product feature
that was actually deprecated two months ago — information the model
couldn't have known from training alone. What's the most likely underlying
cause, and the most direct fix?

- **A.** The model is malfunctioning and needs to be replaced with a different provider
- **B.** The model's knowledge has a training cutoff and no built-in mechanism to know about changes after that date; without a retrieval step pulling in current information, it's relying on stale (or entirely absent) knowledge and generating a plausible-sounding but outdated answer — the fix is grounding time-sensitive answers in a current, retrievable source (RAG) rather than relying on the model's static parametric knowledge
- **C.** The model needs to be fine-tuned daily to stay current with every product change
- **D.** This can only be fixed by upgrading to a model with more parameters

<details>
<summary>Answer</summary>

**B**

**Why B:** This directly connects the knowledge-cutoff concept to the
RAG-vs-parametric-knowledge decision from file 04 — a model's training data
has a fixed cutoff, so anything that changed afterward isn't reliably known
unless it's supplied at inference time via retrieval from a current source.
For any information that changes over time (deprecations, pricing, policy),
grounding the answer in retrieved current data is the standard, direct fix —
relying on what the model "remembers" from training is exactly the failure
mode described.

**Why not A:** This isn't a malfunction — it's the expected behavior of a
model with a training cutoff and no access to current information; treating
it as a defect misdiagnoses a structural limitation as a bug.

**Why not C:** Daily fine-tuning to stay current is an impractically
expensive and slow way to solve what retrieval solves directly and cheaply —
this is precisely the kind of scenario file 04 uses to distinguish when RAG
beats fine-tuning: information that changes frequently is a poor fit for
baking into weights.

**Why not D:** Parameter count has no bearing on whether the model has
information from after its training cutoff — a larger model trained with
the same cutoff date has exactly the same staleness problem; this doesn't
address the actual cause at all.

**Interviewer's likely follow-up:** *"Why not just fine-tune on the new
information as it appears, since it's a smaller amount of new information
than a general refresh?"* (Answer: even incremental fine-tuning is slower,
costlier, and less precisely controllable for fast-changing facts than
retrieval — RAG gives you same-day updates by just changing what's in the
retrieval corpus, no retraining cycle required.)
</details>

### Q8.20 · Bringing it together — conceptual depth without the math · [Applied]

An interviewer says: "I'm not going to ask you to derive backpropagation or
explain the attention math. Instead — in your own words, what's the single
biggest conceptual thing people misunderstand about how these models work,
based on everything we've covered?" What's the strongest kind of answer to
give?

- **A.** List as many technical terms as possible to demonstrate vocabulary breadth
- **B.** Pick one genuinely load-bearing misconception (e.g., conflating fluency with factual correctness, or expecting fine-tuning to behave like a database) and explain it clearly with a concrete example and its practical consequence — demonstrating you can synthesize the material into an insight, not just recite definitions
- **C.** Say there are no common misconceptions, since anyone working with LLMs today already understands them well
- **D.** Redirect the question toward a topic you're more comfortable with, like tooling

<details>
<summary>Answer</summary>

**B**

**Why B:** This is exactly the kind of question this bank's whole design
philosophy is built around — the interviewer is explicitly testing
synthesis and articulation, not recall. Picking one real, well-understood
misconception (fluency-vs-correctness from Q8.4, or the fine-tuning-as-
database misconception from Q8.6) and explaining it with a concrete
consequence shows genuine conceptual grasp, which is precisely what a
"no math, explain it in your own words" question is probing for.

**Why not A:** Term-listing without synthesis is exactly the "pile of
correct keywords" failure mode called out in this bank's scoring guide —
it signals recognition, not understanding.

**Why not C:** This misconception genuinely persists even among practitioners
in the field — claiming otherwise reads as either unaware or evasive, and
either way fails to answer the actual question asked.

**Why not D:** Deflecting to a more comfortable topic instead of engaging
with the question directly is a red flag in almost any interview
context — it signals avoidance rather than confidence.

**Interviewer's likely follow-up:** *"Can you give me a real or hypothetical
example where that misconception caused an actual problem?"* (Answer: this
is exactly the kind of follow-up this bank trains for — having a concrete
example ready, not just the abstract concept, is what separates a
strong answer from a textbook-sounding one.)
</details>

---

## Explain prompts

### E8.1 · Explain: what attention does, to a non-technical audience

**Prompt:** *"Explain what 'attention' means in a transformer model to
someone with no ML background, without saying the word 'attention' more
than once."*

**Target:** 60–90 seconds spoken. Answer out loud before opening the rubric.

<details>
<summary>Rubric — score yourself</summary>

**Must hit (1 point each):**
- [ ] Conveys that each word/token can dynamically draw on information from every other word in the input
- [ ] Contrasts with a simpler alternative (fixed nearby window, or treating every word as equally relevant) to highlight what's different
- [ ] Avoids jargon (no "query/key/value," "softmax," "matrix multiplication")
- [ ] Uses a concrete example sentence to illustrate
- [ ] Stays within the "no ML background" framing throughout, not slipping into technical shorthand

**Bonus — signals strength:**
- [ ] Notes this relevance-weighting is learned from data, not hand-coded
- [ ] Connects it to why this helps with long-range relationships in text (e.g., pronouns referring back to something much earlier)

**Red flags — deduct:**
- [ ] Uses "attention" more than once (the prompt's actual constraint)
- [ ] Falls back on jargon-heavy technical explanation

**Score: ___ / 5**

**Model answer:**
So imagine the model's reading a sentence and, for every single word, it's
asking "which other words in this sentence actually matter for understanding
me right now?" And it's not just looking at the words right next to it —
it can reach all the way back to a word at the very start of a long sentence
if that word's the one that actually matters. So in something like "the cat
that chased the mouse eventually got tired," when the model's processing
"got tired," it can pull directly from "cat" way back at the start, even
though there's a bunch of other words in between. A simpler system might
just look at the last couple of words and miss that connection entirely.
And the model learned which connections matter from being trained on a huge
amount of text — nobody hand-programmed the rule, it picked up the pattern
itself.
</details>

### E8.2 · Explain: pretraining, post-training, and RLHF as a pipeline

**Prompt:** *"Walk me through, conceptually, how a raw pretrained model
becomes the chat assistant a user actually interacts with."*

**Target:** 90 seconds spoken.

<details>
<summary>Rubric — score yourself</summary>

**Must hit (1 point each):**
- [ ] Describes pretraining as learning general language patterns/knowledge from massive raw text via next-token prediction
- [ ] Notes a raw pretrained model doesn't reliably follow instructions or converse naturally by default
- [ ] Describes post-training/instruction-tuning as adapting behavior using smaller, curated instruction-style data
- [ ] Describes RLHF (or similar preference-based technique) as further shaping outputs toward what humans actually prefer
- [ ] States the stages happen in this sequence — pretraining first, post-training/RLHF after, not the reverse

**Bonus — signals strength:**
- [ ] Gives a concrete behavioral example of what changes after post-training (e.g., actually answering a direct question rather than continuing the pattern)
- [ ] Notes this pipeline is why "base model" and "chat model" are different practical tools (ties to Q8.16)

**Red flags — deduct:**
- [ ] Treats the three stages as interchangeable/synonymous
- [ ] Gets the sequence backwards

**Score: ___ / 5**

**Model answer:**
So it happens in stages. First, pretraining — the model reads a massive
amount of raw text and learns to predict the next word, over and over,
which is where it picks up general language ability and a huge amount of
world knowledge, just as a side effect of getting good at that prediction
task. But a model at that stage, if you ask it a direct question, might just
continue the text in a similar style instead of actually answering you —
it's not yet shaped to be a helpful assistant. That's what post-training
does — you fine-tune it on curated examples of instructions and good
responses, so it starts actually behaving like something that answers
questions. And then RLHF goes a layer further — instead of just correct/
incorrect examples, you're using human preference comparisons, like "which
of these two responses do people actually prefer," to nudge the model
toward outputs that are genuinely more helpful and pleasant to interact
with, not just technically instruction-following. Each stage builds on the
last — you can't skip pretraining and go straight to RLHF, there'd be
nothing coherent to shape yet.
</details>

### E8.3 · Explain: why hallucination isn't a solvable bug

**Prompt:** *"A stakeholder wants a guarantee that the model will 'never
make things up again' after we add RAG. Walk me through why you can't
promise that, and what you can promise instead."*

**Target:** 90 seconds spoken.

<details>
<summary>Rubric — score yourself</summary>

**Must hit (1 point each):**
- [ ] Explains the model has no built-in mechanism distinguishing "verified true" from "plausible continuation"
- [ ] States RAG reduces but doesn't eliminate hallucination — grounding helps but the model can still misread or fill gaps in context
- [ ] Avoids overpromising "never" — commits to a measurable, honest claim instead
- [ ] Proposes something concrete you can actually commit to (a measured groundedness rate, monitoring, fallback behavior for uncertain cases)
- [ ] Keeps the explanation accessible to a non-technical stakeholder, not overly technical

**Bonus — signals strength:**
- [ ] Notes this connects to why groundedness needs to be measured (file 06), not assumed
- [ ] Suggests a UX-level mitigation (citations, confidence framing) alongside the technical one

**Red flags — deduct:**
- [ ] Promises "never" or "100% accurate" to satisfy the stakeholder
- [ ] Gives an overly technical answer inappropriate for the audience

**Score: ___ / 5**

**Model answer:**
I can't promise "never," and honestly I don't think anyone building on
these models honestly can. The model doesn't have some internal switch that
says "this is verified true" versus "this just sounds right" — it's
generating what's statistically a plausible continuation, and most of the
time that lines up with being correct, but not always. RAG helps a lot,
because now it's got real, relevant source material in front of it to
ground its answer in, instead of just whatever it half-remembers from
training. But it can still misread that source material, or fill in a gap
the source doesn't actually cover with something that sounds
plausible. What I can commit to is something measurable — we track how
often the answer is actually grounded in what we retrieved, using an eval
process, and we can add things like citations so users can verify the
source themselves, and we can have the system say "I don't have enough
information" instead of guessing when the retrieved context is thin. That's
a real, honest commitment — a guarantee of zero errors isn't.
</details>

### E8.4 · Explain: RAG vs fine-tuning, the conceptual root of the decision

**Prompt:** *"Someone on your team wants to fine-tune the model on your
company's product docs instead of building RAG, because 'fine-tuning gives
the model the knowledge directly.' Walk me through the conceptual flaw in
that reasoning."*

**Target:** 90 seconds spoken.

<details>
<summary>Rubric — score yourself</summary>

**Must hit (1 point each):**
- [ ] States fine-tuning shifts weights/behavior/style, not reliable precise factual recall
- [ ] Notes fine-tuning is a static snapshot — doesn't stay current as docs change
- [ ] Contrasts with RAG's ability to ground answers in current, retrievable, precise source text
- [ ] Acknowledges fine-tuning still has a legitimate role (behavior/format/style), not dismissing it entirely
- [ ] Frames the real distinction as "what facts vs. how it behaves," not "RAG good, fine-tuning bad"

**Bonus — signals strength:**
- [ ] Mentions the two can be combined (fine-tune for format/behavior, RAG for facts)
- [ ] Gives a concrete failure example of docs-fine-tuning going stale

**Red flags — deduct:**
- [ ] Claims fine-tuning is simply always wrong / never useful
- [ ] Can't articulate why fine-tuning doesn't give reliable factual recall

**Score: ___ / 5**

**Model answer:**
The flaw is in treating fine-tuning like it's loading facts into a
database — it's not that. Fine-tuning nudges the model's weights toward
patterns in whatever data you trained it on, which is great for changing
*how* it behaves — tone, format, the way it structures an answer — but it's
not a reliable way to get precise, retrievable facts back out, the way
looking something up would be. And even if it worked well today, the second
those docs get updated, the fine-tuned model doesn't know that — you'd have
to retrain constantly just to stay current, which doesn't scale. RAG solves
the actual problem here directly — it retrieves the current version of the
doc at answer time and grounds the response in it, so it's always as fresh
as your source of truth. I wouldn't throw fine-tuning out entirely though —
if we also wanted the assistant to consistently follow our specific answer
format or tone, that's actually a reasonable fine-tuning use case. It's just
the wrong tool for "give it the facts."
</details>

### E8.5 · Explain: LoRA and why it matters practically, not just theoretically

**Prompt:** *"Explain why a startup with limited compute budget would care
about LoRA specifically, versus just doing full fine-tuning."*

**Target:** 60–90 seconds spoken.

<details>
<summary>Rubric — score yourself</summary>

**Must hit (1 point each):**
- [ ] States LoRA trains a small number of additional parameters while freezing the original model
- [ ] Contrasts with full fine-tuning updating/storing the entire model's weights
- [ ] Connects this to concrete practical benefits: lower compute/memory requirements, cheaper to train
- [ ] Notes the ability to maintain multiple lightweight task-specific adapters instead of many full model copies
- [ ] Acknowledges there's a tradeoff — not simply "strictly better" in all cases

**Bonus — signals strength:**
- [ ] Gives a concrete scenario (e.g., multiple customer-specific adapters) where this matters commercially
- [ ] Notes when full fine-tuning might still be worth the extra cost

**Red flags — deduct:**
- [ ] Can't explain what's actually different about LoRA's mechanism
- [ ] Claims LoRA is always strictly better with no tradeoff

**Score: ___ / 5**

**Model answer:**
For a startup specifically, it comes down to cost and flexibility. Full
fine-tuning means updating every weight in the model, which needs serious
compute and, worse, means storing a whole separate full-size copy of the
model for every variant you want. LoRA instead freezes the original model
and just trains a small set of additional parameters bolted onto specific
layers — way cheaper to train, and way cheaper to store, since you're
keeping tiny adapter files instead of full model copies. That matters
practically if, say, you want a slightly different adapted version per
customer or per use case — with LoRA that's actually affordable; with full
fine-tuning per customer, you'd be looking at storage and compute costs
that don't make sense for a startup. It's not automatically better in every
case — for a really large shift in behavior, full fine-tuning can sometimes
get further — but for most startup-scale customization needs, LoRA's the
much more practical starting point.
</details>

### E8.6 · Explain: quantization tradeoffs for a deployment decision

**Prompt:** *"We need to run inference on cheaper hardware. Someone suggests
quantizing the model to 4-bit. Walk me through how you'd evaluate whether
that's actually safe to do."*

**Target:** 90 seconds spoken.

<details>
<summary>Rubric — score yourself</summary>

**Must hit (1 point each):**
- [ ] Explains quantization reduces numerical precision of weights, shrinking memory/improving speed
- [ ] States this isn't free — can degrade output quality, more so at lower bit-widths
- [ ] Proposes actually measuring the impact rather than assuming it's safe
- [ ] Connects specifically to running the golden eval set against the quantized model and comparing to baseline
- [ ] Frames the decision as a cost/quality tradeoff to be validated, not a default-safe optimization

**Bonus — signals strength:**
- [ ] Notes different quantization methods/bit-widths have different degradation profiles
- [ ] Mentions checking specifically on task types most sensitive to precision loss, not just an aggregate score

**Red flags — deduct:**
- [ ] Assumes quantization is safe without proposing measurement
- [ ] Can't explain what quantization actually changes about the model

**Score: ___ / 5**

**Model answer:**
Quantization's basically representing the model's weights with less
numerical precision — think fewer decimal places, roughly — which
genuinely shrinks memory footprint and can speed things up, which is
exactly why it's tempting for cheaper hardware. But it's not free — you're
throwing away some information, and at something aggressive like 4-bit that
can measurably hurt output quality, depending on the task and the specific
method used. So I wouldn't just assume it's safe and ship it — I'd run our
actual golden eval set against the quantized version and compare the score
to the full-precision baseline, the same regression discipline we'd use for
any other change. If the score holds up within an acceptable margin, great,
ship it and enjoy the cost savings. If it doesn't, that tells us either to
back off to a less aggressive bit-width or accept the cost of full
precision for now. The point is: measure it, don't assume it.
</details>

### E8.7 · Explain: why "just add more data" isn't always the answer

**Prompt:** *"Your manager suggests scraping a much bigger, loosely related
dataset to improve a fine-tuned model's performance on a narrow task. How do
you push back, and what would you suggest instead?"*

**Target:** 90 seconds spoken.

<details>
<summary>Rubric — score yourself</summary>

**Must hit (1 point each):**
- [ ] States data relevance/quality often matters more than volume for narrow fine-tuning tasks
- [ ] Explains that loosely related, noisy data can dilute or degrade performance on the specific target task
- [ ] Distinguishes this from pretraining, where broader scale is more straightforwardly helpful
- [ ] Proposes an alternative: a smaller, well-curated, task-relevant dataset
- [ ] Proposes actually testing the hypothesis rather than just asserting it (tie to evaluation)

**Bonus — signals strength:**
- [ ] Gives a concrete mechanism for why noise dilutes signal in fine-tuning specifically
- [ ] Frames the pushback diplomatically, not dismissively, given it's a manager's suggestion

**Red flags — deduct:**
- [ ] Claims more data is never useful under any circumstances
- [ ] Can't propose a concrete alternative approach

**Score: ___ / 5**

**Model answer:**
I get the instinct — more data usually sounds like it should help — but for
fine-tuning a narrow task specifically, relevance tends to matter more than
raw volume. If we scrape a huge, loosely related dataset, we're not
necessarily teaching the model our specific task pattern more strongly,
we might actually be diluting it with noise that pulls behavior in
directions we don't want. That's different from pretraining, where broad
scale genuinely helps build general capability — this is a narrow,
task-specific fine-tune, which is a different regime. What I'd suggest
instead is going smaller but higher quality — a tightly curated set that
actually matches the specific task distribution we care about. And rather
than just debating this in the abstract, I'd want to actually test both —
fine-tune on the small curated set, fine-tune on the larger loose set, and
compare them against our eval set, so we're deciding based on a real
measurement instead of an assumption either way.
</details>

### E8.8 · Explain: the one thing you'd want a non-ML engineer teammate to understand

**Prompt:** *"You're pairing with a backend engineer who's never worked with
LLMs before, integrating a model into their service for the first time.
What's the one conceptual thing about how these models work that you'd make
sure they understood before they start writing code, and why that one?"*

**Target:** 90 seconds spoken.

<details>
<summary>Rubric — score yourself</summary>

**Must hit (1 point each):**
- [ ] Picks one specific, genuinely load-bearing concept (not a grab-bag of several)
- [ ] Explains *why* this specific concept matters for someone about to write integration code specifically
- [ ] Gives it in plain, non-jargon terms appropriate for someone new to LLMs
- [ ] Connects the concept to a concrete practical consequence for their code (e.g., a design decision it should influence)
- [ ] Shows clear prioritization/judgment in the choice, not just picking the first concept that comes to mind

**Bonus — signals strength:**
- [ ] Anticipates a mistake this teammate might make without this understanding, and names it
- [ ] Ties the choice back to something concrete from earlier in this file rather than a vague generality

**Red flags — deduct:**
- [ ] Tries to cover too many concepts instead of committing to one
- [ ] Picks something not actually consequential for integration code

**Score: ___ / 5**

**Model answer:**
Honestly, the one thing I'd make sure sinks in first is: this thing is
non-deterministic and doesn't have a "did I get this right" flag built in —
it's generating a plausible-sounding response, not looking up a guaranteed
correct answer. I'd pick that one specifically because it changes how you
write the integration code from the very first line — you can't write
`assert response == expected_string` the way you would for a normal API,
and you shouldn't wire its output straight into something consequential,
like a database write or a message send, without validating it first. If
they don't get that going in, the most likely mistake is treating this like
any other deterministic backend call — writing brittle exact-match tests
that'll flake constantly, or trusting the output enough to act on it
directly without a check in between. Once that clicks, most of the rest of
the integration decisions — validation, retries, how you test it — start
making a lot more sense on their own.
</details>

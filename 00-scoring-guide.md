# 00 · Scoring Guide

This bank is not a quiz. Interviewers for Applied AI Engineer, FDE, AI Solutions
Engineer, and AI-native SWE roles are not checking whether you recognise the
right answer in a list — they are checking whether you can **explain your
reasoning out loud, under mild pressure, to someone who will push back.**

That is a different skill from picking the right multiple-choice option. You
can pass every MCQ in this bank and still freeze the moment someone says
"okay, walk me through why." So grading here happens in three layers. Do not
skip layers 2 and 3 — they are where the actual interview skill gets built.

---

## Layer 1 — Rubric self-scoring

Every Explain-prompt (Format B) and Scenario (Format C) ships with a rubric
inside a collapsed `<details>` block: a short list of **must-hit concepts**,
a **bonus** list, and a **red flags** list.

**How to run it:**

1. Read the prompt.
2. Answer **out loud**, or type into a scratch file — not into the rubric.
   Do this *before* opening the `<details>` block. Opening it first defeats
   the entire exercise; you'll recognise the concepts instead of retrieving
   them, and recognition is not what happens in an interview.
3. Open the rubric. Go concept by concept and tick honestly — tick a box if
   you hit the *idea*, even if your wording was different. Don't tick a box
   because you technically said a related word.
4. Count red flags separately. A red flag can cost you the tick even if you
   technically hit a must-hit point elsewhere — a confidently wrong statement
   is worse than a vague correct one.
5. Score out of 5 (or the number of must-hit points, if different).

**What to do with the score:** Anything below 4/5 goes on your re-review
list — literally write the file + question number in `progress.md`. Come
back to it in week 4.

The point of this layer is **not the score**. It's noticing *which concepts
you consistently drop under pressure*. If you miss "what would change the
answer" three times across three different topics, that's a pattern, and
patterns are what actually lose interviews — not any single wrong answer.

---

## Layer 2 — Claude grading

Rubric self-scoring catches missing knowledge. It does not catch **bad
articulation** — rambling, circular explanations, or answers that are
technically correct but would make an interviewer's eyes glaze over. For
that, you need a second, less forgiving grader.

Copy-paste this into a fresh Claude conversation (paste your actual spoken
answer — transcribe it if you recorded yourself, don't clean it up first):

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

**Why both layers exist:** the rubric catches *missing knowledge* — did you
know the thing. Claude-as-grader catches *bad articulation* — could you
actually land it in the room. A rubric-passing answer can still fail an
interview if it's delivered as a disconnected list of buzzwords rather than
a reasoned explanation. Interviewers remember how you sounded, not just
whether your checklist was complete.

Use this layer selectively — not on all 86 explain-prompts, but definitely
on anything where your self-score was borderline (3/5) or where you *felt*
uncertain about how it came out, even if you ticked all the boxes.

---

## Layer 3 — Spoken practice

For the roughly 20 highest-value explain-prompts (agents, RAG failure
diagnosis, evaluation, the sponsorship conversation, and anything you scored
below 4/5 twice), **record yourself answering out loud.** Voice memo app is
enough.

Listen back and flag, specifically:

- **Filler** — "um," "like," "so basically," repeated as a crutch rather
  than natural pacing.
- **Circular explanation** — restating the question as if it were the
  answer, or explaining a term using the term.
- **Trailing off** — starting strong and losing the thread by the second
  half, especially on "what would change the answer" follow-ups.
- **Answering a different question than the one asked** — very common
  under pressure: you pivot to the thing you rehearsed instead of the thing
  they asked.

This layer is uncomfortable. Do it anyway — it is the closest rehearsal you
can get to the actual interview room, and it is the layer candidates skip
because reading the rubric *feels* like enough. It isn't.

---

## Logging progress

After each session, update `progress.md` with the file, date, your MCQ
score, your explain-prompt average, and a short note on which concepts to
re-review. See `README.md` for the full study plan this feeds into.

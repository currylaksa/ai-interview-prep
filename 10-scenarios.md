# 10 · Scenarios

These are FDE / Solutions Engineer case questions — the kind that get asked
as "walk me through what you'd do" rather than "what's the right answer."
There often isn't a single right answer; what's being evaluated is your
process under ambiguity: do you gather information before acting, do you
notice the political/human dimension alongside the technical one, do you
propose something concrete rather than an open-ended investigation.

**Practise these out loud.** Reading a good answer silently teaches you
nothing about how you'll actually sound improvising a response in real time.
Set a timer, talk through your 24-hour plan or your response, and only then
open the rubric.

---

### S1 · Pilot is failing on accuracy

**Setup:** You're three weeks into a six-week pilot with a logistics
company automating shipment-document data entry. Their ops lead messages
you: extraction accuracy is "way off" and their VP of Ops wants a go/no-go
call by Friday — three days away. You haven't seen a single failing
example yet, only the complaint.

**Your task:** What do you do in the next 24 hours? Talk through it.

<details>
<summary>Rubric — score yourself</summary>

**Must hit:**
- [ ] Gets actual failing examples before proposing any technical fix
- [ ] Distinguishes "accuracy is low" from "accuracy is low on the documents that matter most to them"
- [ ] Asks whether a numeric accuracy threshold was ever explicitly agreed
- [ ] Addresses the ops lead's position (they're the one catching heat internally) not just the technical problem
- [ ] Proposes a concrete, time-boxed plan that fits inside the three days, not an open-ended investigation

**Bonus:**
- [ ] Recognises the failure might be scoping (wrong document types included) rather than model performance
- [ ] Proposes getting a shared, agreed eval set going forward so "accuracy" stops being argued from anecdotes
- [ ] Explicitly plans what to say on the Friday call to the VP, not just to the ops lead

**Red flags:**
- [ ] Jumps to "let's try a different model" before seeing a single example
- [ ] Gets defensive about the pilot's data or the customer's expectations
- [ ] Has no plan for the Friday go/no-go conversation itself

**Score: ___ / 5**

**Commentary:** This tests whether a candidate's first instinct under
pressure is to gather evidence or to start problem-solving blind — a
surprisingly common and costly mistake, because "fixing" a problem you
haven't actually seen wastes the very time you don't have. Strong
candidates ask for the specific failing documents within the first
sentence of their answer, and immediately separate two very different
possible root causes: the model is actually bad, or the pilot's scope
quietly grew to include document types it was never built for. Weaker
candidates skip straight to solutions — "we should fine-tune" or "let's
add more examples" — without knowing what's actually broken. The Friday
call is also part of the test: a good FDE treats a hostile deadline as a
forcing function to produce something concrete, not a reason to panic or
over-promise a fix they can't deliver in three days.
</details>

### S2 · The metrics are fine but the champion is unhappy

**Setup:** Your eval dashboard shows the pilot hitting its agreed accuracy
target. Despite that, your internal champion — the person who sponsored
this deal — tells you privately she's "not feeling good about this" and is
worried about defending it internally. Nothing she's said points to a
specific technical failure.

**Your task:** How do you handle this conversation and the days after it?

<details>
<summary>Rubric — score yourself</summary>

**Must hit:**
- [ ] Takes the discomfort seriously even though the metrics look fine — doesn't dismiss it as "she's wrong, the numbers say we're good"
- [ ] Digs specifically into what's driving the feeling — a bad recent interaction, pressure from her own leadership, a gap between what the metric measures and what actually matters to her
- [ ] Recognises the metric might not be measuring the thing she actually cares about
- [ ] Protects her position — she's the one who has to defend this pilot upward
- [ ] Proposes a concrete next step, not just reassurance

**Bonus:**
- [ ] Suggests getting a specific example of a recent bad interaction, even one she personally noticed but didn't formally report
- [ ] Recognises this as a leading indicator worth acting on before it becomes a hard complaint
- [ ] Distinguishes "convince her the metric is right" from "find out what she actually needs"

**Red flags:**
- [ ] Leans on the dashboard number as if it settles the question
- [ ] Treats a vague, non-technical concern as unimportant because it isn't specific
- [ ] Has no plan beyond "reassure her"

**Score: ___ / 5**

**Commentary:** This is testing customer empathy specifically as distinct
from technical competence — a good metric doesn't mean a good pilot if the
metric isn't tracking what the human sponsor actually experiences or fears.
A gut-level "not feeling good" from your champion is real signal, usually
about something the eval set doesn't capture: an edge case that happened to
be visible to a stakeholder, a political risk she's absorbing that you
can't see in a dashboard, or simply not having enough concrete wins to
point to yet. Strong candidates treat this as an early warning worth real
investigation, not something to argue against with data. Weaker candidates
reflexively defend the metric, which reads as dismissive of the person
whose job is literally to stay comfortable enough to keep sponsoring this
internally.
</details>

### S3 · Scope creep mid-pilot

**Setup:** You're four weeks into an eight-week pilot scoped narrowly
around one workflow: summarizing incoming support tickets. Every week, the
customer's team has asked for "just one more thing" — first translation,
then sentiment tagging, then auto-drafting replies. Each ask sounded small
individually. You're now noticeably behind where you expected to be.

**Your task:** How do you address this, starting today?

<details>
<summary>Rubric — score yourself</summary>

**Must hit:**
- [ ] Names scope creep explicitly rather than continuing to silently absorb new asks
- [ ] Distinguishes which of the added asks are genuinely small versus which quietly expanded the pilot's real complexity
- [ ] Proposes re-anchoring to the original scope and success criteria, not just refusing all new requests
- [ ] Has a plan for the conversation itself — how to say "let's park this" without souring the relationship
- [ ] Offers a concrete path for the parked items (next phase, separate scoped follow-up) rather than a flat no

**Bonus:**
- [ ] Notes that saying yes to everything is itself a failure mode, not evidence of good customer service
- [ ] Proposes documenting scope explicitly going forward so future asks are visibly in/out of bounds
- [ ] Recognises this might be partly self-inflicted — not pushing back on request #1 made #2 and #3 easier to ask for

**Red flags:**
- [ ] Continues absorbing scope silently to avoid an uncomfortable conversation
- [ ] Refuses all new requests bluntly with no path forward for them
- [ ] Blames the customer for asking rather than owning that scope wasn't defended

**Score: ___ / 5**

**Commentary:** Scope creep is rarely one dramatic overreach — it's death
by a thousand reasonable-sounding small asks, and the skill being tested
here is noticing the pattern and interrupting it before the pilot's
original goal gets lost entirely. A strong candidate treats "just one more
thing" as a signal to actively re-anchor the conversation on the agreed
scope, not a routine request to fulfill. Equally important is *how* the
pushback happens — bluntly refusing everything damages the relationship
just as much as endless yes-ing damages the timeline. The best answers
propose a path for the good ideas (a phase two, a follow-up scoped
separately) so the customer doesn't feel shut down, just sequenced.
Candidates who blame the customer for "moving the goalposts" without
acknowledging their own role in not pushing back earlier come across as
defensive rather than reflective.
</details>

### S4 · Sales promised something engineering never scoped

**Setup:** You join a kickoff call and learn the sales team told the
customer the product could do real-time multi-language voice transcription
with speaker diarization — a capability that doesn't currently exist in
your product and wasn't scoped with engineering beforehand. The customer is
expecting it in the pilot.

**Your task:** Walk through how you handle this, in the room and after.

<details>
<summary>Rubric — score yourself</summary>

**Must hit:**
- [ ] Doesn't promise the capability on the spot to avoid an awkward moment
- [ ] Is honest with the customer about the current gap without publicly blaming sales in the room
- [ ] Separates what's genuinely feasible in the pilot timeframe from what isn't
- [ ] Plans a direct, private follow-up with sales about what was actually committed
- [ ] Proposes a concrete alternative or scoped-down version to keep the pilot moving

**Bonus:**
- [ ] Recognises this as a recurring organizational risk, not just a one-off mistake, and considers how to prevent it going forward (e.g., technical review before commitments)
- [ ] Manages the customer relationship carefully — this is a trust moment, not just a scoping problem
- [ ] Distinguishes "we can't do exactly that" from "here's what we can actually deliver that addresses your underlying need"

**Red flags:**
- [ ] Throws sales under the bus in front of the customer
- [ ] Promises to build the missing capability within the pilot timeline without checking feasibility
- [ ] Has no follow-up plan with sales to prevent recurrence

**Score: ___ / 5**

**Commentary:** This tests composure and judgment in a genuinely
uncomfortable moment where the honest answer risks embarrassing a
colleague and disappointing a customer simultaneously. The strongest
answers keep the customer-facing tone collaborative and forward-looking —
acknowledging the gap plainly, then immediately pivoting to what's
actually achievable — rather than either over-promising to save face or
publicly airing an internal miscommunication. The internal follow-up with
sales matters just as much as the external handling; without it, the same
mismatch recurs on the next deal. Candidates who treat this purely as a
one-time scoping fix, without considering the process gap that let an
unscoped promise reach a customer in the first place, are missing half the
lesson.
</details>

### S5 · Customer wants a chatbot when they need a workflow

**Setup:** A customer wants you to build a chatbot their claims processors
can "ask questions to" about policy coverage rules. As you dig in, it
becomes clear what they actually need is a deterministic, auditable
decision path — coverage determination is a compliance-sensitive process
where "the chatbot said so" isn't an acceptable audit trail, and a fixed
rules-plus-lookup workflow would serve them far better and more reliably
than free-form Q&A.

**Your task:** How do you handle this in the conversation?

<details>
<summary>Rubric — score yourself</summary>

**Must hit:**
- [ ] Doesn't just build what was literally asked for without raising the concern
- [ ] Identifies the specific reason a chatbot is the wrong fit — auditability, determinism, compliance risk
- [ ] Proposes the more appropriate alternative concretely, not just "that's not a good idea"
- [ ] Frames the pushback around the customer's actual risk/goals, not as a technical purity argument
- [ ] Leaves room for the customer to have context you don't — asks rather than just asserting

**Bonus:**
- [ ] Notes where an LLM could still add value within the better-fit architecture (e.g., drafting explanations of a rules-engine's decision, not making the decision)
- [ ] Recognises this connects to the general "when not to use an agent/chatbot" principle from file 05
- [ ] Anticipates the customer might have already committed to "chatbot" internally and plans for that resistance

**Red flags:**
- [ ] Builds the requested chatbot without raising the concern, to avoid conflict
- [ ] Refuses the request without offering a credible alternative
- [ ] Frames the pushback in purely technical terms with no connection to the customer's actual risk

**Score: ___ / 5**

**Commentary:** This scenario is really testing whether a candidate can
push back on a customer constructively — a core FDE skill that's easy to
get wrong in both directions. Simply building what was asked for, even
when it's a poor fit, optimizes for short-term customer comfort over their
actual long-term success and risk exposure, which erodes trust once the
compliance problem surfaces later. Flatly refusing without a credible
alternative is just as unhelpful. The strongest candidates translate the
technical objection into the language of the customer's actual stakes —
auditability, compliance risk — rather than an abstract architecture
preference, and they leave space for the possibility the customer already
knows something they don't (maybe "chatbot" is a political requirement
from someone senior, not a genuine technical ask) rather than assuming
they're simply wrong.
</details>

### S6 · Customer insists on fine-tuning when RAG fits better

**Setup:** A customer wants to fine-tune a model on their entire internal
wiki so the model "just knows" the answers, and is skeptical of RAG,
calling it "just a workaround." Their wiki changes weekly and has several
outdated pages that conflict with newer ones.

**Your task:** How do you navigate this technical disagreement?

<details>
<summary>Rubric — score yourself</summary>

**Must hit:**
- [ ] Explains, without jargon, why fine-tuning is a poor fit for frequently-changing, sometimes-conflicting source material
- [ ] Connects the recommendation to their specific situation (weekly changes, outdated pages) rather than a generic RAG-vs-fine-tuning lecture
- [ ] Takes the skepticism seriously rather than dismissing it as a misunderstanding
- [ ] Proposes RAG concretely, addressing what "just a workaround" gets wrong
- [ ] Leaves room for a combined approach if it genuinely fits (e.g., fine-tuning for tone/format, RAG for facts)

**Bonus:**
- [ ] Offers to prototype a small comparison rather than only arguing the point verbally
- [ ] Notes the outdated/conflicting pages problem would hurt a fine-tuned model *even more* than RAG, since RAG failures are at least traceable to a specific source
- [ ] Reads "just a workaround" as revealing a mental model worth correcting directly, not a random objection

**Red flags:**
- [ ] Gets condescending about the customer's technical understanding
- [ ] Caves and agrees to fine-tune purely to avoid the disagreement
- [ ] Can't explain the tradeoff in plain, non-jargon terms

**Score: ___ / 5**

**Commentary:** This tests whether a candidate can hold a well-founded
technical position under customer pushback without becoming either
condescending or a pushover — both common failure modes for junior FDEs
navigating their first real architecture disagreement with a customer. The
strongest answers ground the recommendation specifically in this
customer's stated situation (weekly changes, conflicting pages) rather than
reciting the general RAG-vs-fine-tuning tradeoff as an abstract lecture,
because specificity is what actually moves a skeptical stakeholder.
Equally telling is whether the candidate treats "just a workaround" as
worth directly addressing — it reveals the customer thinks RAG is a
lesser, hacky solution rather than the right engineering choice for this
exact situation, and that specific misconception is the actual thing
worth correcting, not just the technical recommendation itself.
</details>

### S7 · Discovery call, customer can't articulate the problem

**Setup:** You're on a discovery call with a mid-market logistics company.
When asked what problem they're hoping AI solves, the stakeholder says
"we just know we need to be doing something with AI, everyone else is."
There's no specific pain point on the table yet.

**Your task:** How do you run the rest of this call?

<details>
<summary>Rubric — score yourself</summary>

**Must hit:**
- [ ] Doesn't pitch a specific product/feature before understanding their actual operations
- [ ] Asks concrete, operational questions about their current process and pain points rather than abstract "what are your goals" questions
- [ ] Tries to surface a specific, high-friction workflow rather than accepting "AI" as the goal itself
- [ ] Manages the FOMO-driven framing directly and honestly rather than just riding along with it
- [ ] Ends the call with a concrete next step, not a vague "let's explore more"

**Bonus:**
- [ ] Asks about a recent specific incident or bad day, which often surfaces real pain faster than abstract questions
- [ ] Distinguishes urgency from importance — being asked to move fast doesn't mean the eventual project is well-defined
- [ ] Notes that "everyone else is doing it" is itself useful information about internal pressure, worth acknowledging rather than ignoring

**Red flags:**
- [ ] Pitches a specific solution before any real problem has surfaced
- [ ] Lets the call end without any concrete next step
- [ ] Never gets more specific than "AI," accepting vagueness as sufficient discovery

**Score: ___ / 5**

**Commentary:** This is one of the most common real FDE situations —
prospects driven by competitive anxiety rather than a clear internal pain
point — and it tests whether a candidate can resist the temptation to
pitch prematurely just to fill the silence. Good discovery in this
scenario means actively working to surface something concrete: recent
incidents, specific bottlenecked processes, a metric someone's being
measured on that AI could plausibly move. Candidates who let the
conversation stay at "we need to be doing something with AI" and pitch a
generic capability anyway are optimizing for a good-feeling call over an
actually useful one, which usually produces a pilot that solves nothing
real and stalls. The strongest answers leave with something specific to
investigate next, even if that specific thing is small.
</details>

### S8 · Discovery call, customer pitches a solution not a problem

**Setup:** A prospective customer opens a discovery call by saying "we want
a Slack bot that can answer any question about our company." When you ask
what questions people currently struggle to get answered, they're vague —
"just, you know, general stuff."

**Your task:** How do you steer this conversation?

<details>
<summary>Rubric — score yourself</summary>

**Must hit:**
- [ ] Recognises "a Slack bot that answers any question" as a solution, not a problem statement, and probes behind it
- [ ] Asks for specific, recent examples of questions people struggled to get answered
- [ ] Investigates where those answers currently live or who currently answers them, to understand the real gap
- [ ] Avoids simply agreeing to build the literal request without understanding whether it's the right scope
- [ ] Proposes narrowing to a specific, well-bounded first use case rather than "answers any question"

**Bonus:**
- [ ] Notes that "any question" scope is a red flag for an unbounded, hard-to-evaluate pilot
- [ ] Suggests looking at actual Slack search queries or help-channel history as a concrete discovery source
- [ ] Distinguishes between information that's genuinely hard to find versus information people are just too lazy to search for — different problems, different fixes

**Red flags:**
- [ ] Accepts "answers any question" as a workable, well-scoped pilot goal
- [ ] Doesn't push for concrete examples at all
- [ ] Pitches immediately without any further discovery

**Score: ___ / 5**

**Commentary:** Customers very often arrive with a solution already in
mind rather than a clearly diagnosed problem, and "a bot that can answer
anything" is a classic version of this — appealing-sounding, essentially
unscoped, and almost impossible to build a trustworthy eval for. This
scenario tests whether a candidate reflexively agrees to build the literal
ask or does the harder work of finding the actual bounded problem
underneath it. Concrete examples are the lever — vague answers like
"general stuff" are a sign to keep digging, not a stopping point. Strong
candidates use this moment to propose a specific, narrow starting scope
(one topic area, one team) rather than accepting an unbounded goal that
would be nearly impossible to evaluate as "working" or "not working" once
built.
</details>

### S9 · Demo breaks live in front of stakeholders

**Setup:** You're demoing your agent to a room that includes the
customer's CTO, whom you've never met before today. Midway through, the
agent calls the wrong tool, produces a nonsensical result, and the room
goes quiet.

**Your task:** What do you do in the next 60 seconds, and after the call?

<details>
<summary>Rubric — score yourself</summary>

**Must hit:**
- [ ] Doesn't panic or over-apologize in a way that amplifies the moment
- [ ] Acknowledges the failure plainly rather than pretending it didn't happen or awkwardly glossing over it
- [ ] Has a plan to keep the demo moving — a fallback example, a different path, or a brief explanation of what likely happened
- [ ] Follows up after the call to actually understand and address the root cause
- [ ] Manages the CTO's first impression deliberately, not just hoping it blows over

**Bonus:**
- [ ] Uses the failure itself as a moment to demonstrate maturity — e.g., briefly narrating that this is exactly the kind of failure mode their eval/guardrail process is designed to catch
- [ ] Has a pre-prepared backup demo path for exactly this situation, rather than improvising from zero
- [ ] Sends a specific, concrete follow-up (not just "sorry about that") addressing what happened

**Red flags:**
- [ ] Freezes or becomes visibly flustered in a way that derails the rest of the meeting
- [ ] Tries to hide or minimize that anything went wrong
- [ ] Has no follow-up plan after the call

**Score: ___ / 5**

**Commentary:** Live demo failures are close to inevitable in this line of
work, and this scenario is really testing composure and recovery, not
whether the candidate can prevent every possible failure. The instinct to
avoid is over-apologizing or freezing, both of which make the room focus
on the failure rather than move past it — a brief, calm acknowledgment
followed by a smooth pivot to a working example or a clear explanation is
far more reassuring to a skeptical CTO than pretending nothing happened.
The follow-up matters as much as the in-room recovery: a candidate who
just hopes the awkward moment fades misses the chance to turn a failure
into a credibility-building moment by actually diagnosing and reporting
back on what went wrong. Coming prepared with a backup path for exactly
this situation is a strong, experience-signaling detail.
</details>

### S10 · The model says something embarrassing live

**Setup:** During a live demo to a prospective client's leadership team,
your assistant confidently states a fabricated statistic about the
customer's own industry that's plainly wrong to everyone in the room. The
customer's Head of Compliance, sitting in, looks visibly uncomfortable.

**Your task:** How do you handle this moment and what follows?

<details>
<summary>Rubric — score yourself</summary>

**Must hit:**
- [ ] Names the hallucination directly rather than talking around it
- [ ] Doesn't get defensive about the model's capability
- [ ] Specifically addresses the Compliance stakeholder's likely concern (this is exactly the risk their function exists to catch)
- [ ] Connects the moment to what guardrails/groundedness measures actually exist or are planned (file 06/07/08 content) rather than a vague "it won't happen again"
- [ ] Has a concrete after-the-call follow-up specifically for Compliance

**Bonus:**
- [ ] Uses the moment honestly — this is a known, real limitation of LLMs, not a fixable bug, and says so rather than overpromising a full fix
- [ ] Distinguishes what mitigations exist today from what's aspirational, being careful not to overclaim in front of a compliance-sensitive audience
- [ ] Notes this is exactly why groundedness/citation features matter, if the product has them

**Red flags:**
- [ ] Promises hallucination will "never happen again"
- [ ] Ignores or downplays the Compliance stakeholder's reaction specifically
- [ ] Gets defensive or blames the demo environment/data

**Score: ___ / 5**

**Commentary:** This scenario specifically targets a candidate's
understanding that hallucination is a fundamental, unsolved property of
these models (file 08's content), not an embarrassing bug that better
engineering eliminates — and tests whether they can say that honestly to a
skeptical, compliance-minded audience without either downplaying the risk
or promising something they can't deliver. The Head of Compliance's
discomfort is the most important detail in the scenario: this stakeholder
specifically exists to catch exactly this kind of risk, and a strong
candidate directly engages with what would actually reassure that person —
concrete mitigations, honest framing of what's solved versus managed —
rather than a generic apology aimed at the room as a whole. Overpromising
"never again" is the single most damaging response here, because it will
almost certainly be tested and disproven later.
</details>

### S11 · Customer's data is messier than promised

**Setup:** The customer told you their document set was "clean, consistent
PDFs, all the same template." Once you get real access, it's a mix of
scanned images, inconsistent layouts across five years of template
changes, and a meaningful chunk of handwritten annotations. The pilot
timeline assumed the clean version.

**Your task:** What do you do?

<details>
<summary>Rubric — score yourself</summary>

**Must hit:**
- [ ] Assesses the actual scope of the mess quickly rather than reacting emotionally to the mismatch
- [ ] Communicates the gap to the customer directly and promptly, rather than quietly absorbing the extra work
- [ ] Distinguishes what's still achievable in the original timeline from what needs to be re-scoped
- [ ] Proposes a concrete adjusted plan (reduced scope, extended timeline, or phased approach) rather than an open-ended "this will take longer"
- [ ] Avoids blaming the customer for the mismatch in the conversation

**Bonus:**
- [ ] Distinguishes handling this as a technical problem (OCR, layout variance) from handling it as an expectations/relationship problem
- [ ] Proposes a smaller, well-scoped subset to prove value quickly (e.g., just the post-2022 template) while the harder cases get separately addressed
- [ ] Notes this as a lesson for scoping future pilots — validating data assumptions earlier, before commitments are made

**Red flags:**
- [ ] Silently absorbs the extra scope and works overtime rather than communicating the impact
- [ ] Blames the customer for misrepresenting their data
- [ ] Has no concrete adjusted plan, just a vague "we'll figure it out"

**Score: ___ / 5**

**Commentary:** Data being messier than represented is close to the single
most common real surprise in this line of work, and this scenario tests
whether a candidate's instinct is to communicate the mismatch honestly and
promptly or to quietly try to absorb it and hope the timeline still works
out. The latter reliably backfires — either quality suffers to hit the
original deadline, or the deadline slips anyway but later and with less
warning, which is a worse outcome for trust than raising it immediately.
Strong candidates move fast to actually understand the real scope of the
mess (is it 10% of documents or 60%?) and come back with a specific,
adjusted proposal rather than an open-ended complaint. Proposing a
narrower, achievable subset to demonstrate value while the harder cases
get a separate plan is a particularly strong move — it keeps momentum
without pretending the original scope is still realistic.
</details>

### S12 · The "clean" data reveals a structural problem

**Setup:** Partway through building a RAG pipeline over a customer's
"knowledge base," you discover that half the documents contradict the
other half — old and new policy versions coexist with no clear
versioning or deprecation marker, and neither the customer's team nor any
documentation tells you which is current.

**Your task:** How do you proceed?

<details>
<summary>Rubric — score yourself</summary>

**Must hit:**
- [ ] Surfaces this to the customer rather than silently picking one version arbitrarily
- [ ] Recognises this is a data-governance problem, not something retrieval or prompting can fix
- [ ] Explains clearly why this will directly cause bad or inconsistent answers if left unresolved
- [ ] Proposes a concrete path forward (e.g., customer designates an owner to triage/tag current-vs-deprecated documents)
- [ ] Avoids pretending the AI system can resolve an ambiguity the customer's own team hasn't resolved

**Bonus:**
- [ ] Suggests a stopgap (e.g., metadata filtering by date, or surfacing multiple conflicting answers with a caveat) while the underlying cleanup happens
- [ ] Frames this constructively as a finding that benefits the customer beyond just this project
- [ ] Notes this connects to file 04's "handling documents that update" content

**Red flags:**
- [ ] Picks a version arbitrarily without surfacing the issue
- [ ] Assumes better retrieval or a smarter model can resolve a genuine ground-truth conflict
- [ ] Treats this purely as a blocker with no proposed path forward

**Score: ___ / 5**

**Commentary:** This scenario tests whether a candidate correctly
diagnoses a data-governance problem as exactly that, rather than trying to
engineer around it — a conflict between two contradictory source-of-truth
documents isn't something retrieval tuning or a better model can resolve,
because there genuinely isn't a correct answer available in the data as it
stands. The instinct to avoid is quietly picking a version and moving on,
which produces a system that's confidently wrong roughly half the time on
this topic with no way for anyone to know when. Strong candidates
recognise this needs a human owner on the customer's side to actually
resolve, propose a reasonable stopgap in the meantime, and frame the
finding constructively — this is genuinely useful information for the
customer to have discovered, even though it's inconvenient timing.
</details>

### S13 · Procurement blocks deployment on a technicality

**Setup:** Your pilot performed well and the customer wants to move to
production, but their procurement team has frozen the deal over a single
line in your data processing agreement regarding sub-processor
notification timelines — a real but narrow legal point, not a
security-substance issue.

**Your task:** How do you move this forward?

<details>
<summary>Rubric — score yourself</summary>

**Must hit:**
- [ ] Doesn't try to personally negotiate or reinterpret the legal language without the right people involved
- [ ] Loops in the appropriate internal stakeholders (legal, deal desk) rather than trying to solve it solo
- [ ] Keeps the customer relationship warm and informed during what could be a slow process
- [ ] Distinguishes this narrow issue from the broader deal, so it doesn't stall everything unnecessarily
- [ ] Sets realistic expectations with the customer about timeline rather than overpromising a fast resolution

**Bonus:**
- [ ] Proactively asks the customer's procurement contact what specifically would satisfy them, rather than guessing
- [ ] Recognises this as a normal, expected part of enterprise deals rather than a crisis
- [ ] Keeps technical momentum going in parallel where possible (e.g., planning next steps) so the relationship doesn't go cold during the legal delay

**Red flags:**
- [ ] Tries to unilaterally promise contract changes without the authority to do so
- [ ] Lets the relationship go quiet and unmanaged during the delay
- [ ] Escalates internally with frustration rather than treating this as routine

**Score: ___ / 5**

**Commentary:** This tests whether a candidate understands the boundary of
their own role — a technical FDE getting into unilateral legal negotiation
over a data processing agreement is a scope overreach that can create real
liability, and the correct move is routing this to the right internal
owner while staying actively engaged with the customer relationship. What
separates a strong answer here is recognising this as a routine, expected
friction point in enterprise sales rather than treating it as an alarming
blocker — procurement and legal reviews commonly surface narrow technical
issues like this, and panicking or trying to personally push past it
usually makes things worse. Keeping the customer warm and informed during
what might be a multi-week legal back-and-forth, without letting the
relationship go cold, is the part candidates most often forget to mention.
</details>

### S14 · Security review flags data residency

**Setup:** A prospective bank's security review flags that your standard
SaaS deployment stores data in a cloud region outside Singapore, which
conflicts with their internal data residency policy. This is late in the
sales cycle and the deal team is anxious about losing momentum.

**Your task:** How do you respond?

<details>
<summary>Rubric — score yourself</summary>

**Must hit:**
- [ ] Takes the finding seriously as a real, valid technical requirement rather than something to argue around
- [ ] Understands and clearly explains the actual options (e.g., a Singapore-region deployment, VPC-hosted alternative) rather than a vague "we'll look into it"
- [ ] Is honest about what's currently possible versus what would require real engineering work
- [ ] Manages the internal deal-team anxiety without letting it pressure an overcommitment to the customer
- [ ] Proposes a concrete next step and realistic timeline

**Bonus:**
- [ ] Connects this to file 09's deployment-model content — offering VPC-hosted or region-specific SaaS as the realistic paths
- [ ] Distinguishes what's a quick configuration change (e.g., picking a different existing region) from what's a bigger commitment
- [ ] Keeps the conversation collaborative with the bank's security team rather than adversarial

**Red flags:**
- [ ] Downplays the requirement to try to keep the deal moving fast
- [ ] Promises a region-specific deployment without checking actual feasibility
- [ ] Lets internal sales pressure lead to overcommitting to the customer

**Score: ___ / 5**

**Commentary:** This scenario tests whether internal pressure (an anxious
deal team late in a sales cycle) distorts how a candidate handles a
legitimate customer requirement — the wrong move is letting that pressure
push toward downplaying a real data-residency issue or promising a
solution without verifying it's actually feasible. Strong candidates treat
the finding as valid and worth solving properly, and they know enough
about their own deployment options (existing regions, VPC-hosted
alternatives) to give the customer a genuinely useful, concrete answer
rather than a stalling non-answer. Equally important is protecting the
relationship with the bank's security team specifically — they're doing
their job well by catching this, and treating their finding as an obstacle
to route around rather than a legitimate requirement to solve is exactly
the kind of adversarial posture that damages trust with a security-
conscious buyer.
</details>

### S15 · "Why not just use ChatGPT?"

**Setup:** In a sales call, a technical stakeholder asks bluntly: "Honestly,
why would we pay you for this instead of just having our team use ChatGPT
directly?"

**Your task:** How do you answer, in the room, live?

<details>
<summary>Rubric — score yourself</summary>

**Must hit:**
- [ ] Doesn't get defensive or dismissive of the question — it's a fair one
- [ ] Articulates specific, concrete value beyond raw model access — integration with their systems, workflow automation, evaluation/reliability engineering, security/access controls, ongoing maintenance
- [ ] Grounds the answer in their specific situation rather than a generic pitch
- [ ] Is honest about where a general-purpose chat tool genuinely would be sufficient, if that's true for some of their needs
- [ ] Keeps the tone confident, not defensive or apologetic

**Bonus:**
- [ ] Distinguishes "raw model capability" from "a production system built around a model" clearly and concretely
- [ ] Uses a specific example relevant to their stated use case rather than an abstract list of features
- [ ] Turns the question into a useful moment to clarify what they actually need, rather than just defending the product

**Red flags:**
- [ ] Gets visibly defensive or treats the question as hostile
- [ ] Gives a vague, feature-list answer with no connection to their specific situation
- [ ] Claims there's no legitimate use case for using ChatGPT directly, which is usually untrue and reads as overselling

**Score: ___ / 5**

**Commentary:** This is one of the most common, and fairest, objections an
FDE hears, and it tests composure and clarity under a pointed challenge
more than technical knowledge — the answer is almost always some version
of "integration, reliability engineering, and workflow automation around
the model, not the model itself," but delivering that convincingly and
specifically, live, under a skeptical question, is a real skill. Weak
candidates either get defensive (a bad look) or retreat into generic
feature-speak that doesn't land because it isn't anchored in the
customer's actual situation. The strongest candidates use a concrete
example specific to the prospect's own stated workflow, and aren't afraid
to honestly acknowledge that for some simple, one-off tasks, their team
using ChatGPT directly might genuinely be fine — that honesty builds more
credibility than pretending every use case requires the product.
</details>

### S16 · A skeptical engineer challenges the value prop internally

**Setup:** Mid-pilot, one of the customer's own senior engineers — not
hostile, but clearly skeptical — says in a team meeting: "I could build
what you're selling us with a weekend and the API directly. What are we
actually paying for?"

**Your task:** How do you respond, in front of the rest of their team?

<details>
<summary>Rubric — score yourself</summary>

**Must hit:**
- [ ] Takes the engineer's technical competence seriously rather than dismissing the claim
- [ ] Distinguishes a weekend prototype from a production system — reliability, evaluation, security, maintenance, edge-case handling built over time
- [ ] Grounds the answer in specifics from this actual pilot (real integration work already done, real edge cases already handled) rather than abstractions
- [ ] Keeps the tone respectful and non-defensive, since this person likely has real influence on the team's technical opinion
- [ ] Doesn't overclaim — acknowledges what's genuinely true about a prototype being possible in a weekend

**Bonus:**
- [ ] Invites the engineer's expertise productively — e.g., asks what they'd specifically want to see to be convinced, turning skepticism into engagement
- [ ] Uses a concrete example from the actual pilot work already done (a specific edge case, a specific integration headache) as proof
- [ ] Recognises winning this person over matters more than winning the argument technically

**Red flags:**
- [ ] Gets defensive or dismissive of a technically credible internal skeptic
- [ ] Claims a weekend prototype is impossible, which is implausible and damages credibility
- [ ] Ignores the social dynamics — this person's opinion likely shapes the rest of the team's view

**Score: ___ / 5**

**Commentary:** This is a harder variant of the ChatGPT objection because
it comes from a credible internal technical peer in front of colleagues,
which raises the social stakes — losing this exchange doesn't just lose an
argument, it can shape how the whole team perceives the engagement going
forward. The honest, credible answer acknowledges that yes, a rough
prototype is genuinely buildable quickly, and then pivots to what actually
differentiates a production system: the accumulated edge-case handling,
evaluation rigor, and reliability work that a weekend prototype simply
hasn't done yet — ideally illustrated with something concrete from the
actual pilot already underway. Candidates who get defensive or try to
claim the prototype isn't realistically buildable lose credibility with a
technical audience that can tell the difference. The strongest answers
also read the room correctly — this is as much a relationship moment with
an influential skeptic as it is a technical question.
</details>

### S17 · Estimating effort with incomplete information

**Setup:** A prospective customer asks for a firm timeline to integrate
your product with their internal claims system before they'll approve
budget, but you don't yet know if that system has a documented API, and
their team hasn't been able to answer that question either.

**Your task:** How do you respond in the meeting?

<details>
<summary>Rubric — score yourself</summary>

**Must hit:**
- [ ] Doesn't refuse to give any useful answer at all
- [ ] Doesn't invent a falsely precise number to sound confident
- [ ] Gives a scenario-based range grounded in realistic best-case and harder-case assumptions
- [ ] Names the specific unknowns driving that range explicitly
- [ ] Proposes a concrete next step to narrow the estimate (discovery call, technical questionnaire, sandbox access request)

**Bonus:**
- [ ] Distinguishes a rough range that's genuinely useful for budget planning from a commitment they'd hold you to
- [ ] Explains clearly why overcommitting here is risky, if pushed
- [ ] Manages the customer's urgency (they want budget approval) without letting it force a bad estimate

**Red flags:**
- [ ] Gives a single precise number under pressure with no caveats
- [ ] Stonewalls with "we can't know until we know everything"
- [ ] Doesn't propose any concrete next step to actually resolve the uncertainty

**Score: ___ / 5**

**Commentary:** This scenario, alongside its counterpart in file 09,
directly tests a core FDE skill — being genuinely useful under real
uncertainty rather than either stonewalling or fabricating false
precision to satisfy pressure for a number. The instinct to avoid is
caving to the customer's need for a firm number by inventing one that
sounds confident but isn't actually grounded in anything; if the real
system turns out to be a mess of undocumented legacy endpoints, that
number will be badly wrong and the damage to credibility is worse than
giving an honest range upfront. The strongest answers give a genuinely
useful bounded estimate, explain exactly what's driving the width of the
range, and immediately propose a concrete way to narrow it — turning an
uncomfortable unknown into a next step rather than an impasse.
</details>

### S18 · Pressured into an unrealistic timeline

**Setup:** Your manager tells you the deal is at risk unless you commit to
a four-week delivery timeline for a scope you privately believe needs
eight. The customer is on the call in ten minutes.

**Your task:** What do you do before, and during, that call?

<details>
<summary>Rubric — score yourself</summary>

**Must hit:**
- [ ] Raises the concern with the manager directly before the call, even under time pressure, rather than silently going along
- [ ] Doesn't simply commit to four weeks on the call if it's genuinely not achievable
- [ ] Proposes a way to reconcile the pressure and the reality — e.g., a reduced scope that's actually achievable in four weeks, or a phased delivery
- [ ] Considers what happens if the commitment is made and missed — the cost of a broken promise versus a harder conversation now
- [ ] Handles the internal disagreement with the manager professionally, not as an outright refusal

**Bonus:**
- [ ] Comes with an alternative proposal ready, not just an objection
- [ ] Distinguishes between the manager's business-context pressure (which may be legitimate) and the technical reality (which the candidate is better positioned to assess)
- [ ] Notes that a missed public commitment usually damages the relationship more than an honest scoping conversation would have

**Red flags:**
- [ ] Silently agrees despite believing it's unrealistic
- [ ] Refuses outright with no proposed alternative
- [ ] Blindsides the manager by contradicting them live on the customer call without a prior heads-up

**Score: ___ / 5**

**Commentary:** This tests whether a candidate can disagree with a
stakeholder under real business pressure — a core "comfort with ambiguity
and disagreement" competency this bank's behavioural content targets
directly. The failure mode to avoid isn't just caving to unrealistic
pressure, it's also blindsiding a manager by contradicting them live in
front of a customer without warning, which damages an internal
relationship needlessly. The strongest answers get the disagreement out in
the open *before* the call, propose something concrete (a reduced scope
that's genuinely deliverable in four weeks, or a phased plan) rather than
just objecting, and reason explicitly about the actual cost of a broken
promise versus an uncomfortable but honest scoping conversation now — a
missed public commitment is usually far more damaging to the customer
relationship than a realistic timeline set upfront.
</details>

### S19 · Handing off to the customer's internal team

**Setup:** The pilot succeeded and it's time to hand ongoing ownership to
the customer's internal engineering team so your involvement can wind
down. Their team is competent generally but has zero prior experience
operating an LLM-based system in production.

**Your task:** How do you structure this handoff?

<details>
<summary>Rubric — score yourself</summary>

**Must hit:**
- [ ] Doesn't assume general engineering competence automatically transfers to operating this specific kind of system
- [ ] Identifies the specific gaps to close (eval practices, prompt change process, monitoring, cost/latency tradeoffs) rather than a vague "documentation handoff"
- [ ] Proposes a structured transition period with real knowledge transfer, not a single handoff document and a goodbye
- [ ] Sets up a way for them to reach you (or support) for a defined period after full handoff, rather than a hard cutoff
- [ ] Establishes what "ready to own this" concretely looks like before declaring the handoff complete

**Bonus:**
- [ ] Proposes shadowing/pairing on real operational tasks (a prompt change, an eval run) rather than only passive documentation
- [ ] Anticipates specific likely failure points for a team new to LLM systems (e.g., treating a prompt change like a normal code change with no eval)
- [ ] Builds in a checkpoint to revisit after some weeks of independent operation, not just a one-time handoff event

**Red flags:**
- [ ] Assumes a documentation dump is sufficient
- [ ] Has no defined support period or checkpoint after handoff
- [ ] Doesn't specifically address the LLM-specific operational gaps (eval, prompt regression, cost monitoring) versus generic engineering skills

**Score: ___ / 5**

**Commentary:** This tests whether a candidate understands that operating
an LLM-based system well requires specific, non-obvious skills (eval
discipline, prompt-change regression testing, cost/latency monitoring)
that don't automatically come with general engineering competence — a
skilled backend team can still make serious mistakes operating this kind
of system for the first time, like treating a prompt tweak the way they'd
treat a routine code change with no evaluation gate. The strongest answers
propose genuine, hands-on knowledge transfer — pairing on real tasks,
not just handing over a wiki page — and build in a defined support window
and a later checkpoint, rather than a hard cutoff the moment the pilot
officially ends. This scenario connects directly to file 06's evaluation
content: a team that doesn't understand why eval matters for this kind of
system is the most likely group to quietly regress quality after takeover.
</details>

### S20 · Handoff reveals an ownership conflict

**Setup:** During handoff planning, you learn the customer's engineering
team and their newer "AI/ML platform" team both believe they should own
this system going forward, and the disagreement hasn't been resolved
internally. Both have reached out to you separately, each somewhat
frustrated with the other.

**Your task:** How do you navigate this?

<details>
<summary>Rubric — score yourself</summary>

**Must hit:**
- [ ] Doesn't unilaterally pick a side or team to hand off to
- [ ] Recognises this as an internal customer organizational issue that needs their own resolution, not something for a vendor to settle
- [ ] Stays neutral and professional with both teams rather than getting pulled into the internal politics
- [ ] Surfaces the conflict to the customer's actual decision-maker/sponsor rather than letting it quietly stall the handoff
- [ ] Proposes the handoff proceed once ownership is clarified, with a concrete ask for that clarification

**Bonus:**
- [ ] Offers a neutral framing that might help resolve it (e.g., proposing both teams have a role, one operational and one platform-level) without prescribing the customer's internal structure
- [ ] Recognises the risk of quietly favoring whichever team is easier to work with, and avoids that
- [ ] Keeps the relationship warm with both teams regardless of how it resolves

**Red flags:**
- [ ] Picks a side based on personal rapport rather than deferring to the customer's own resolution
- [ ] Lets the handoff quietly stall without escalating
- [ ] Gets visibly caught in the middle of the internal conflict rather than staying professionally neutral

**Score: ___ / 5**

**Commentary:** This scenario tests political awareness and boundary-
setting — a vendor inserting themselves into a customer's internal
ownership dispute, even with good intentions, is overstepping in a way
that can damage relationships with whichever team feels overruled, and it
isn't actually the vendor's decision to make. The correct move is
recognising this needs resolution by the customer's own decision-maker,
raising it clearly rather than letting it quietly stall progress, and
staying deliberately neutral with both teams throughout. Candidates who
pick a side — even implicitly, by simply working more closely with
whichever team reached out first or seems easier to deal with — risk a
real relationship cost with the other team once the political dust
settles. The strongest answers also recognise this kind of ambiguity as a
completely normal part of enterprise engagements, not a crisis, and handle
it with calm, professional patience.
</details>

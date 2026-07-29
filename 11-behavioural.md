# 11 · Behavioural

Standard STAR-format prompts, but the rubrics here are tuned to what
Applied AI / FDE / AI Solutions Engineer interviews actually screen for at
entry level: **autonomy, comfort with ambiguity, customer empathy, and
shipping over perfecting.** A technically flawless answer that doesn't
signal these traits will land flatter than a rougher one that does.

**Practise these out loud, not silently.** A behavioural answer that reads
fine on paper often reveals filler, vagueness, or a missing "so what" the
moment you actually have to say it. Pick real stories from your own
background — SecureExam UTM, Daily Sparks Events, the Huawei internship,
your Claude Code workflow decisions — rather than inventing generic ones;
specificity is what makes an answer land.

---

### B11.1 · Why AI

**Prompt:** *"Why AI, specifically? What draws you to this field rather
than general software engineering?"*

**Target:** 60–90 seconds spoken. Answer out loud before opening the
rubric.

<details>
<summary>Rubric — score yourself</summary>

**Must hit:**
- [ ] Gives a genuine, specific reason rather than a generic "AI is the future" answer
- [ ] Connects to something concrete from their own background or experience
- [ ] Shows they understand what the work actually involves day to day, not just the hype
- [ ] Sounds like a real, considered answer rather than a rehearsed platitude

**Bonus:**
- [ ] Distinguishes their interest in *applying* AI (integration, product) from research-track interest, if relevant to the role
- [ ] Connects to a concrete moment or project that sparked the interest, not just an abstract statement

**Red flags:**
- [ ] Purely hype-driven language with no personal grounding
- [ ] Can't distinguish their answer from something anyone could say

**Score: ___ / 5**

**Model answer:**
Honestly, it's the Claude Code workflow stuff that really pulled me in — I
started using a dual-model setup, splitting thinking and worker models via
cc-switch, and that was the first time building with AI felt like real
engineering to me, not just prompting a chatbot. There were actual
tradeoffs to reason about — cost, latency, when a bigger model was worth
it. That's what got me hooked, not the general hype. I like that this
field sits right at the intersection of things I already care about —
security, systems thinking, and now this new layer of non-deterministic
behavior that makes you rethink how you build reliable systems in the
first place. It's not "AI is cool," it's that the actual engineering
problems here are genuinely interesting and new.
</details>

### B11.2 · Why this company

**Prompt:** *"Why us, specifically, and not one of the other companies
you're probably talking to?"*

**Target:** 60–90 seconds spoken.

<details>
<summary>Rubric — score yourself</summary>

**Must hit:**
- [ ] Shows specific knowledge of what this company actually does, not a generic compliment
- [ ] Connects the company's specific work to something concrete in their own background or interests
- [ ] Avoids implying every company would get the same answer
- [ ] Sounds genuinely researched, not templated

**Bonus:**
- [ ] References something specific and current about the company (a product decision, a market position, a technical approach) rather than boilerplate praise
- [ ] Connects to a real, personal reason beyond "good opportunity"

**Red flags:**
- [ ] Generic answer that could apply to any company in the space
- [ ] Shows no real research into what the company specifically does

**Score: ___ / 5**

**Model answer:**
[This answer needs to be filled in per-company before each interview —
don't try to memorise a generic version. Research: what they actually
build, who their customers are, what makes their technical approach
distinctive, and one specific thing from their engineering blog, product,
or public materials worth referencing directly. A strong version of this
answer names something concrete you learned about them in the last week,
not something you could have said before researching at all.]
</details>

### B11.3 · Why Singapore

**Prompt:** *"You'd be relocating from Malaysia for this. Why Singapore
specifically?"*

**Target:** 60 seconds spoken.

<details>
<summary>Rubric — score yourself</summary>

**Must hit:**
- [ ] Gives a genuine, specific reason beyond "better opportunities"
- [ ] Shows real familiarity with Singapore's market/ecosystem, not a vague gesture at proximity
- [ ] Answers directly and confidently, without sounding apologetic about being a foreign candidate
- [ ] Connects to something concrete about the AI/tech ecosystem specifically, given the role

**Bonus:**
- [ ] Mentions something specific about Singapore's tech/AI scene (density of companies, specific initiatives, market maturity) that shows real research
- [ ] Frames it as a deliberate choice, not a fallback

**Red flags:**
- [ ] Sounds apologetic or uncertain about wanting to relocate
- [ ] Gives an answer so vague it could apply to any country

**Score: ___ / 5**

**Model answer:**
Singapore's genuinely the most concentrated AI and tech ecosystem in this
part of the world — a huge number of regional HQs, a strong FDE and
solutions-engineering market specifically because so many companies use
Singapore as their APAC base, and a government that's actively pushing AI
adoption across industries, which means real demand for people who can
bridge technical and customer-facing work. It's close enough to home that
I'm not starting from zero culturally, but it's a genuinely bigger stage
for exactly the kind of work I want to do. This isn't a fallback choice
for me — it's the market that actually matches what I'm trying to build a
career in.
</details>

### B11.4 · Shipped under ambiguity

**Prompt:** *"Tell me about a time you had to ship something with unclear
or incomplete requirements."*

**Target:** 90 seconds spoken.

<details>
<summary>Rubric — score yourself</summary>

**Must hit:**
- [ ] Uses a specific, real example with concrete details (Situation, Task)
- [ ] Describes what they actually did to reduce or work within the ambiguity (Action)
- [ ] States a concrete outcome (Result)
- [ ] Shows genuine comfort operating without full information, not just tolerating it reluctantly

**Bonus:**
- [ ] Describes a specific technique used to manage the ambiguity (e.g., making an explicit assumption and validating it early, rather than freezing)
- [ ] Reflects on what they'd do differently, showing self-awareness

**Red flags:**
- [ ] Vague story with no concrete details
- [ ] Implies they just guessed with no reasoning process

**Score: ___ / 5**

**Model answer:**
The Daily Sparks Events rebuild is a good example — the stakeholder was a
non-technical business owner who could describe what she wanted in outcome
terms, "make it easy for people to book events," but not in terms of
actual requirements. So instead of waiting for a spec that was never
coming, I built a rough version of the booking flow in about two days,
showed it to her, and used her reaction to actually nail down the real
requirements — turned out half of what I assumed was needed, wasn't, and
there were two things she cared about a lot that I hadn't guessed. That
loop — build something small, get real reaction, adjust — was way faster
than trying to extract a complete spec up front from someone who didn't
have one to give. The site shipped on Next.js 15 with Sanity as the CMS,
and it's live and being used by a real client today.
</details>

### B11.5 · Disagreed with a stakeholder

**Prompt:** *"Tell me about a time you disagreed with a stakeholder or
teammate about a technical or product decision."*

**Target:** 90 seconds spoken.

<details>
<summary>Rubric — score yourself</summary>

**Must hit:**
- [ ] Describes a genuine disagreement, not a manufactured or trivial one
- [ ] Explains how they raised the disagreement — directly, respectfully, with reasoning
- [ ] States the actual outcome honestly, including if they didn't get their way
- [ ] Shows they can disagree without being combative or simply deferring

**Bonus:**
- [ ] Reflects on what they learned about the other person's perspective, not just defending their own position
- [ ] Shows they picked their battles — not every disagreement needs to be fought

**Red flags:**
- [ ] Story where they were simply right and the other person simply wrong, with no nuance
- [ ] Describes caving immediately or steamrolling the other person

**Score: ___ / 5**

**Model answer:**
During the SecureExam build, a teammate wanted to skip building the
Isolation Forest anomaly-detection microservice as a separate service and
just bolt anomaly checks directly into the main API, to save time before
our DIGITEX submission deadline. I disagreed — coupling it that tightly
meant any anomaly-detection failure could take down exam sessions
directly, which felt like an unacceptable risk for something meant to be
a security feature. I made the case with a specific failure scenario, not
just "I think this is bad architecture," and we ended up keeping it as a
separate microservice with a fallback if it's unavailable. It did cost us
some time we didn't love losing that close to the deadline, but it turned
out to matter — we hit an actual anomaly-service hiccup during testing,
and because it was isolated, the exam platform itself stayed up. I don't
think I'd have won that argument with a vaguer objection.
</details>

### B11.6 · A failure and what changed afterward

**Prompt:** *"Tell me about a time you failed at something. What changed
afterward?"*

**Target:** 90 seconds spoken.

<details>
<summary>Rubric — score yourself</summary>

**Must hit:**
- [ ] Describes a genuine failure, not a humble-brag disguised as one
- [ ] Takes real ownership rather than externalizing blame
- [ ] Describes a specific, concrete change in behavior or process afterward
- [ ] Shows the change actually stuck / was applied later, not just stated as an intention

**Bonus:**
- [ ] Shows emotional honesty about the failure without being self-flagellating
- [ ] The failure is substantive enough to be a real signal, not trivially small

**Red flags:**
- [ ] "My biggest weakness is I work too hard" style non-failure
- [ ] No concrete change described, just a vague "I learned to be more careful"

**Score: ___ / 5**

**Model answer:**
During the Huawei internship, I built one of the automation tools without
properly checking edge cases in the input data format, and it silently
produced wrong output for about a week before someone caught it downstream
— nothing catastrophic, but it was a real, embarrassing miss, and I felt
pretty bad that nobody noticed sooner, including me. What changed
afterward: I started treating "does this fail loudly or silently" as an
explicit design question for every tool I built after that, not an
afterthought — adding validation and logging specifically designed to
surface exactly the kind of quiet failure that had just happened. That
habit's stuck with me since — it's part of why I care about things like
eval and observability now, not just as abstract good practice, but
because I've personally felt what it costs when a system fails quietly
instead of loudly.
</details>

### B11.7 · Learned something fast

**Prompt:** *"Tell me about a time you had to learn something quickly to
get a job done."*

**Target:** 90 seconds spoken.

<details>
<summary>Rubric — score yourself</summary>

**Must hit:**
- [ ] Names a specific, concrete thing learned, not a vague "I learn fast" claim
- [ ] Describes the actual learning process — how, not just that it happened
- [ ] Connects the learning to a real deliverable or outcome under real time pressure
- [ ] Shows genuine intellectual engagement, not panic

**Bonus:**
- [ ] Describes a specific strategy for fast learning (targeted reading, building a minimal example, asking a specific expert) rather than vague immersion
- [ ] Shows awareness of what they still didn't fully understand, rather than overclaiming mastery

**Red flags:**
- [ ] Vague claim with no specific learning process described
- [ ] Overclaims deep expertise gained in an implausibly short time

**Score: ___ / 5**

**Model answer:**
When I started building the anomaly-detection piece of SecureExam, I'd
never actually implemented an Isolation Forest model before — I understood
the general idea of anomaly detection but not this specific technique. I
had about a week before we needed something working for a milestone. What
I did was skip the textbook-depth approach and go straight for a minimal
working implementation using an existing library, get it running on a toy
dataset, and only then go back and actually understand what the model was
doing under the hood — reading just enough to know why it was flagging
what it was flagging, since I needed to be able to explain and tune it,
not just call a function. That order — working example first, deep
understanding second — got me to something real fast, and I still had to
go deepen my understanding afterward once the pressure was off, which I
did.
</details>

### B11.8 · Least experienced person in the room

**Prompt:** *"Tell me about a time you were the least experienced person
in the room. How did you handle it?"*

**Target:** 90 seconds spoken.

<details>
<summary>Rubric — score yourself</summary>

**Must hit:**
- [ ] Describes a genuine situation with real seniority/experience gap, not a manufactured one
- [ ] Shows they contributed meaningfully despite the gap, rather than staying silent
- [ ] Shows they also knew when to listen and defer, not just push their own view
- [ ] Reflects genuine self-awareness about the dynamic, not false modesty or overconfidence

**Bonus:**
- [ ] Describes a specific moment where their outsider/fresh perspective actually added value precisely because they were less experienced
- [ ] Shows they built credibility deliberately (asking good questions, doing homework) rather than assuming it

**Red flags:**
- [ ] Claims they were secretly the smartest person in the room
- [ ] Describes staying completely silent and just absorbing, with no real contribution

**Score: ___ / 5**

**Model answer:**
At DIGITEX, presenting SecureExam meant standing in front of judges who'd
seen dozens of security projects and clearly had way more depth than me in
some areas — enterprise security architecture, specific compliance
frameworks I'd only read about. I didn't try to bluff depth I didn't have.
When a judge asked something outside what I'd actually built or verified,
I said so directly and explained what I *had* validated instead — the 25
mapped security controls, the actual RBAC implementation, the anomaly
detection working end to end. What I think landed well wasn't pretending
to be more senior than I was, it was being precise about exactly what I
knew cold versus what I hadn't gone deep on yet. Being the least
experienced person in that room meant leaning hard on being the person who
actually built the thing and knew every detail of *that*, rather than
trying to compete on breadth I didn't have yet.
</details>

### B11.9 · Took ownership beyond the defined role

**Prompt:** *"Tell me about a time you took ownership of something that
wasn't technically your responsibility."*

**Target:** 90 seconds spoken.

<details>
<summary>Rubric — score yourself</summary>

**Must hit:**
- [ ] Describes a specific gap or problem they noticed that wasn't formally theirs
- [ ] Explains why they chose to act rather than waiting for someone else to
- [ ] Describes concrete action taken, not just noticing the problem
- [ ] States a real outcome

**Bonus:**
- [ ] Shows judgment about scope — took it on without overstepping or excluding the actual owner
- [ ] Reflects on the tradeoff of spending time outside their defined lane

**Red flags:**
- [ ] Describes overstepping in a way that would have annoyed the actual owner
- [ ] Vague story with no specific problem or action

**Score: ___ / 5**

**Model answer:**
During the Daily Sparks Events project, deployment and hosting weren't
technically my responsibility — I was brought on for the frontend rebuild —
but partway through I noticed the client had no real deployment process at
all, just manual file uploads to an old host, which meant every update was
risky and slow. Nobody had asked me to fix that, but it was clearly going
to bite the client later, so I set up a proper Vercel deployment pipeline
connected to the repo, so pushes to main just deployed automatically. I
made sure to loop in the client about it rather than just silently
changing how things worked behind the scenes, since it wasn't my call
alone to make. It ended up saving real time on every subsequent update,
and the client specifically mentioned it as one of the more useful things
that came out of the engagement, even though it was outside what I was
technically hired to do.
</details>

### B11.10 · Shipped imperfect on purpose

**Prompt:** *"Tell me about a time you deliberately shipped something
imperfect rather than continuing to polish it."*

**Target:** 90 seconds spoken.

<details>
<summary>Rubric — score yourself</summary>

**Must hit:**
- [ ] Describes a specific, deliberate decision to ship at a certain point, not an accident of running out of time
- [ ] Explains the reasoning for why shipping then was the right call
- [ ] Names what was specifically left imperfect and why that was acceptable
- [ ] States a real outcome, including whether the imperfection actually mattered later

**Bonus:**
- [ ] Shows judgment about *which* imperfections were acceptable to ship with versus which weren't
- [ ] Reflects on the general principle, not just the one instance

**Red flags:**
- [ ] Describes shipping something broken with no real judgment involved, framed as a virtue
- [ ] Can't articulate what was actually imperfect or why it was an acceptable tradeoff

**Score: ___ / 5**

**Model answer:**
With the Daily Sparks Events site, I had a much more elaborate booking
calendar UI half-designed — animated transitions, a fancier date-picker
component — and I made a deliberate call to ship a plainer, functional
version instead of finishing the polished one, because the client's actual
priority was having a working, bookable site live before an upcoming event
season, not visual flourish. I was explicit about the tradeoff with the
client rather than just quietly cutting corners — told her exactly what
was simplified and why, and that we could revisit it later if it mattered
to her. It didn't end up mattering — she never asked for the fancier
version, because the plain one worked fine and got real bookings. That's
sort of the lesson I took from it broadly: the imperfection I chose to
ship was invisible to the actual outcome that mattered, and continuing to
polish it would have just been me optimizing for something the client
never actually needed.
</details>

### B11.11 · Advocated for the customer against internal pressure

**Prompt:** *"Tell me about a time you pushed back internally on behalf of
a user or customer's actual needs."*

**Target:** 90 seconds spoken.

<details>
<summary>Rubric — score yourself</summary>

**Must hit:**
- [ ] Describes a specific internal pressure that conflicted with the user/customer's actual interest
- [ ] Explains how they advocated for the user, concretely, not just in principle
- [ ] Shows they did this respectfully, not by simply overriding the internal stakeholder
- [ ] States a real outcome, even if imperfect

**Bonus:**
- [ ] Shows they grounded the advocacy in something concrete from the user's actual experience, not just their own opinion
- [ ] Reflects on the tension between internal pressure and user needs as an ongoing, recurring dynamic worth being alert to

**Red flags:**
- [ ] Describes simply agreeing with whatever the customer wanted with no real judgment
- [ ] Vague story with no specific internal-vs-customer conflict

**Score: ___ / 5**

**Model answer:**
For Daily Sparks Events, there was a point where I could have added a
feature the stakeholder mentioned offhand — a complex multi-step
registration flow — that would have looked impressive in a portfolio sense
but that I genuinely didn't think her actual users, mostly people quickly
booking a single event, would want to deal with. Rather than just building
what sounded technically interesting to build, I pushed back and asked her
specifically what problem she thought that flow would solve, and it turned
out the real underlying need was just capturing a couple of extra fields
for certain event types — solvable with a much simpler conditional form,
not a whole multi-step flow. I built the simpler version. It shipped
faster and, based on actual usage, nobody's ever asked for anything more
complex there. That's the pattern I try to watch for generally — the more
"impressive" build isn't always the one that actually serves the person
using it.
</details>

### B11.12 · Received tough feedback

**Prompt:** *"Tell me about a time you received critical feedback that was
hard to hear. What did you do with it?"*

**Target:** 90 seconds spoken.

<details>
<summary>Rubric — score yourself</summary>

**Must hit:**
- [ ] Describes genuinely difficult, specific feedback, not a softball
- [ ] Shows an honest initial reaction, not a fake "I loved hearing it" response
- [ ] Describes what they actually did with the feedback afterward, concretely
- [ ] Shows real change or growth, not just acknowledgment

**Bonus:**
- [ ] Shows they sought clarification or specifics rather than reacting defensively in the moment
- [ ] Distinguishes feedback they agreed with and acted on from feedback they considered and ultimately didn't fully agree with, if applicable, handled maturely

**Red flags:**
- [ ] Feedback described is trivially easy to hear, not actually hard
- [ ] No real behavioral change described afterward

**Score: ___ / 5**

**Model answer:**
A mentor reviewing an early version of the SecureExam RBAC design told me
directly that my role permission model was over-engineered — too many
fine-grained permission flags that nobody would actually configure
correctly in practice, more theoretically flexible than practically
usable. That stung a bit, because I'd put real thought into it and saw it
as one of the more sophisticated parts of the design. My first reaction
internally was mildly defensive, but I asked him to walk through
specifically which parts he thought were overkill rather than just
absorbing the criticism as "this is bad." Turned out he was right — I'd
designed for hypothetical flexibility nobody asked for. I simplified it
down to four clear, well-defined roles instead, which is what actually
shipped, and it made the whole system easier to reason about and audit
too — a direct security benefit I hadn't even been optimizing for when I
made the change.
</details>

### B11.13 · Prioritized under competing deadlines

**Prompt:** *"Tell me about a time you had to prioritize between two
competing deadlines or demands."*

**Target:** 90 seconds spoken.

<details>
<summary>Rubric — score yourself</summary>

**Must hit:**
- [ ] Describes a genuine, specific conflict between two real competing demands
- [ ] Explains the actual reasoning used to prioritize, not just "I just picked one"
- [ ] Describes how the deprioritized item was handled — communicated, delayed, or delegated, not silently dropped
- [ ] States a real outcome for both

**Bonus:**
- [ ] Shows a repeatable prioritization principle, not just a one-off judgment call
- [ ] Reflects honestly on any cost of the choice made

**Red flags:**
- [ ] Vague story with no real competing pressure
- [ ] Deprioritized item silently dropped with no communication to whoever was affected

**Score: ___ / 5**

**Model answer:**
Close to the DIGITEX submission deadline, I had the SecureExam anomaly
detection microservice still needing real tuning, and separately, the
demo presentation itself needed serious work — I hadn't rehearsed at all
yet. Both mattered, and I only had about two days. I prioritized the
tuning work first, because a broken core feature would undermine the
whole submission regardless of how good the presentation was, but I
capped that work at a hard time limit rather than letting it expand to
fill all the remaining time, specifically so I'd still have real time left
to rehearse. I also let my teammates know I was making that trade
explicitly, rather than just going quiet on presentation prep without
explanation. It worked out — the anomaly detection held up under judge
questions, and the presentation, while not as polished as I'd have liked
with more time, was solid enough. The main lesson was time-boxing the
technical work deliberately instead of letting "just one more fix" eat
the whole remaining runway.
</details>

### B11.14 · Worked with a difficult teammate or stakeholder

**Prompt:** *"Tell me about a time you had to work with someone difficult —
a teammate, a stakeholder, anyone."*

**Target:** 90 seconds spoken.

<details>
<summary>Rubric — score yourself</summary>

**Must hit:**
- [ ] Describes a genuine difficulty, specific and real, not a thinly-veiled complaint
- [ ] Avoids purely blaming the other person — shows some self-awareness about the dynamic
- [ ] Describes concrete actions taken to actually work through it
- [ ] States a real outcome, including if the relationship stayed imperfect

**Bonus:**
- [ ] Shows genuine empathy for the other person's perspective or constraints
- [ ] Reflects on what they'd do differently, if anything

**Red flags:**
- [ ] Pure complaint about the other person with no reflection on their own role
- [ ] Vague story with no concrete resolution attempt

**Score: ___ / 5**

**Model answer:**
On the SecureExam team, one teammate and I disagreed constantly early on
about how much documentation was "enough" — he wanted extremely thorough
docs before any code got merged, I wanted to move faster and document
after stabilizing. It got a bit tense for a couple of weeks, partly
because I wasn't great early on about explaining *why* I preferred moving
faster, I just pushed back on his requests. Once I actually asked him why
documentation mattered so much to him specifically, it turned out he'd
been burned badly on a previous project with undocumented code that
became unmaintainable — that context reframed the whole disagreement for
me. We landed on a middle ground: lightweight inline documentation at
merge time, fuller docs added once a component stabilized. It wasn't a
perfect resolution and we still had some friction later, but understanding
where he was actually coming from made the rest of the collaboration a lot
smoother than it started.
</details>

### B11.15 · Explained something technical to a non-technical audience

**Prompt:** *"Tell me about a time you had to explain something technical
to someone without a technical background."*

**Target:** 90 seconds spoken.

<details>
<summary>Rubric — score yourself</summary>

**Must hit:**
- [ ] Describes a real, specific instance, not a hypothetical
- [ ] Describes the actual technique used to make it accessible (analogy, avoiding jargon, focusing on outcomes)
- [ ] Shows they checked whether the explanation actually landed, not just delivered and moved on
- [ ] States a concrete outcome

**Bonus:**
- [ ] Shows awareness of what level of detail the audience actually needed versus what they left out deliberately
- [ ] Reflects on how they'd adjust the explanation for a different audience

**Red flags:**
- [ ] Explanation described is still full of jargon
- [ ] No real audience or outcome described

**Score: ___ / 5**

**Model answer:**
The Daily Sparks Events client asked why her site needed a "content
model" when she just wanted to "edit the words on the page" — Sanity's
structured content approach wasn't intuitive to her at all coming from
plain website editors she'd used before. Instead of explaining schemas and
structured content abstractly, I framed it around her actual daily
experience: "think of it like a form — you fill in the event name, date,
and description in specific boxes, and the website automatically puts
them in the right place everywhere they need to show up, instead of you
having to find and edit them separately on every page." I then walked her
through actually editing one real event live, rather than just describing
it. That landed — she picked it up in about ten minutes and has been
independently managing content ever since with basically no support
requests about it, which was really the actual measure of whether the
explanation worked.
</details>

### B11.16 · A mistake that affected others

**Prompt:** *"Tell me about a mistake you made that affected other
people. How did you handle it?"*

**Target:** 90 seconds spoken.

<details>
<summary>Rubric — score yourself</summary>

**Must hit:**
- [ ] Describes a real mistake with a real, specific impact on others
- [ ] Takes clear ownership rather than deflecting or minimizing
- [ ] Describes how they communicated about it to the affected people, not just fixed it silently
- [ ] Describes what changed afterward to reduce recurrence

**Bonus:**
- [ ] Shows they proactively surfaced the mistake rather than waiting to be caught
- [ ] Shows genuine accountability in tone, not just a checklist of the "right" things to say

**Red flags:**
- [ ] Downplays the actual impact
- [ ] No real communication to affected people described

**Score: ___ / 5**

**Model answer:**
On the Huawei internship automation tooling, I pushed a change to one of
the tools that changed an output file format slightly, without checking
whether anyone downstream depended on the old format — someone else's
process broke because of it, and they had to spend part of a day figuring
out why before tracing it back to my change. I proactively went to them
as soon as I realized, rather than waiting for it to be escalated,
explained exactly what I'd changed and why their process broke, and helped
fix it on the spot. Afterward, I started explicitly checking for and
documenting any consumers of a tool's output before changing its format,
which is a habit that's carried into how I think about API versioning and
breaking changes generally now — that mistake is honestly a big part of
why that topic isn't just abstract knowledge to me.
</details>

### B11.17 · Delivered under significant time pressure

**Prompt:** *"Tell me about a time you had to deliver something under
serious time pressure."*

**Target:** 90 seconds spoken.

<details>
<summary>Rubric — score yourself</summary>

**Must hit:**
- [ ] Describes a genuinely tight, real deadline with real stakes
- [ ] Describes the specific approach used to manage the pressure — not just "worked harder"
- [ ] Shows judgment about what got cut or simplified to hit the deadline, and why
- [ ] States a real outcome

**Bonus:**
- [ ] Shows they protected quality on the things that actually mattered while cutting scope elsewhere deliberately
- [ ] Reflects on the experience without glorifying overwork as the solution

**Red flags:**
- [ ] "I just worked really long hours" as the entire strategy
- [ ] Vague deadline with no real stakes described

**Score: ___ / 5**

**Model answer:**
The final week before the DIGITEX submission deadline, we still had a
meaningful list of things we wanted done, and clearly weren't going to get
to all of it. Rather than trying to grind through everything, I sat down
and explicitly ranked what actually affected the judging criteria versus
what was just "nice to have" polish, and we cut the second category
deliberately rather than letting it silently eat time that the first
category needed. I did work long hours that week, I won't pretend
otherwise, but the actual thing that got us there wasn't the hours, it was
being ruthless about scope early enough that the hours we did put in went
toward things that mattered. We submitted on time with the core features
solid, and a couple of things on our nice-to-have list just never
happened — and looking back, nobody who evaluated the project seemed to
notice or care that they were missing.
</details>

### B11.18 · Changed your mind based on new information

**Prompt:** *"Tell me about a time you changed your mind about something
significant based on new information or feedback."*

**Target:** 90 seconds spoken.

<details>
<summary>Rubric — score yourself</summary>

**Must hit:**
- [ ] Describes a genuine, specific position they held and later changed
- [ ] Explains what specifically caused the change — new information, a specific event, feedback
- [ ] Shows intellectual honesty rather than treating the initial position as embarrassing to admit
- [ ] Describes what they actually did differently afterward

**Bonus:**
- [ ] Shows this wasn't a trivial or low-stakes change of mind
- [ ] Reflects on what made them receptive to changing their mind in that instance

**Red flags:**
- [ ] Describes a change of mind that cost them nothing and required no real reconsideration
- [ ] Frames the original position as obviously wrong in hindsight, undercutting that it was a genuine belief at the time

**Score: ___ / 5**

**Model answer:**
Early on, I was fairly convinced the right approach for SecureExam's
anomaly detection was a rules-based system — explicit thresholds for
things like unusual mouse movement patterns or tab-switching frequency —
because it felt more explainable and controllable than a learned model.
Once I actually started testing against real exam session data, though,
the rules kept needing constant manual tuning and still missed patterns
that were obviously anomalous to a human looking at the data but didn't
trip any single rule. That pushed me to actually try the Isolation Forest
approach instead, which I'd been avoiding partly because it felt like a
bigger unknown to implement. It performed meaningfully better and, with
some work on making its outputs interpretable for invigilators, didn't
end up sacrificing as much explainability as I'd assumed it would. What
changed my mind wasn't someone telling me I was wrong, it was just seeing
the rules-based approach actually fail against real data — which is
probably why it stuck.
</details>

---

## The sponsorship conversation

As a Malaysian candidate requiring Employment Pass sponsorship, "do you
need visa/work pass sponsorship?" is a question you will almost certainly
get, sometimes early in the process. How you handle it matters — not
because the honest answer is a problem, but because *how* you say it signals
confidence (or the lack of it) the same way any other answer does.

**The core principle: answer directly, without apologizing, and don't let
the question shrink you.** Needing sponsorship is a logistics fact, not a
character flaw or a mark against your candidacy. Companies that hire in
Singapore sponsor Employment Passes constantly — this is routine, not
exotic. Treating it as something to apologize for or hedge around signals
that *you* think it's a liability, which invites the interviewer to wonder
the same thing.

**How to answer the direct question:**

> "Yes, I'd need Employment Pass sponsorship — I'm a Malaysian citizen, so
> I don't need a separate visa to work in Singapore in the way some other
> nationalities would, but the company would need to sponsor the EP itself.
> Happy to answer anything else about that process if it's useful."

Notice what this does: states the fact plainly, adds the one piece of
context that actually reduces perceived friction (Malaysians don't need
a separate entry visa, and Singapore's EP process for degree-holders in
tech roles is well-established and routine), and immediately hands the
conversation back rather than lingering on it.

**What not to do:**
- Don't preface it with "I'm sorry, but..." or "I know this might be an
  issue, but..." — you're narrating a problem into existence that the
  interviewer may not have been thinking of as one.
- Don't over-explain the entire EP process unprompted — answer what's
  asked, offer more only if they want it.
- Don't downplay or hide it if asked directly — getting caught seeming
  evasive about a logistics fact is worse than the fact itself.

**Preempting it — leading with what makes the hire worth the paperwork.**
If you sense the sponsorship question is coming, or you're in a context
where addressing it proactively makes sense (a cover note, an early
screening call), the strongest move is leading with your value, not the
logistics — make the sponsorship a footnote to an already-compelling case,
not the headline.

A rough shape for this, adapted to the actual role and company:

> "I built and shipped [specific project — SecureExam's Zero-Trust
> architecture, or the Daily Sparks Events production rebuild], which is
> exactly the kind of [security-minded / customer-facing / full-stack]
> work this role needs. I'm a Malaysian citizen requiring Employment Pass
> sponsorship, which is a standard, well-established process — happy to
> share more detail if useful, but wanted to lead with why I think I'm a
> strong fit for what you're building."

The point isn't to hide the sponsorship need — it's clearly stated — it's
to make sure it arrives *after* the interviewer already has a reason to
want to solve that logistics problem for you, rather than arriving as the
first thing they learn about you.

**If pushed on cost or process specifics** ("Employment Passes can be
expensive/slow, why should we bother"): this is where leaning on concrete
value again is the right move, not defensiveness. Something like: "The EP
process for a role like this is genuinely routine — it's not an unusual
ask for a Singapore tech employer — and I think the specific value I'd
bring [name it concretely] is worth that standard paperwork. Happy to
answer any specific questions about the process itself." Confidence here
comes from treating the question as a fair, ordinary business question,
not an accusation to defend against.

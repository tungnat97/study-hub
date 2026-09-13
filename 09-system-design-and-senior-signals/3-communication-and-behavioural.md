[← back to the field index](README.md)

# System Design · Part 3 — Communication, Practice and Senior Signals

Nodes `SD15`–`SD17`.

---

## SD15 · Communicating like a senior

`Advanced` · Requires: `SD01`, `SD07`, `SD10`, `SD13`, `SD14`, `O14`, `O17`, `O18`, `M34`, `F27` · Unlocks: `SD17`

### Preface

Two candidates can produce the same diagram and receive different verdicts. The difference is how the
design was arrived at and how it was explained.

What is actually being assessed: can you drive an ambiguous conversation, make decisions with reasons,
state trade-offs without being asked, change your mind when given new information, and stay
comfortable when challenged.

### Details

#### 1. The loop

**Theory.** A repeatable structure for the whole round: **clarify → estimate → simple design → walk
the flow → identify the bottleneck → address it → state the trade-off → repeat**.

**Example.** Out loud, that sounds like: "Let me make sure I understand the scope… At those numbers we
are looking at roughly X… Here is the simplest design that meets it… Let me walk a write through it…
The bottleneck at this scale is the read path… I would address it with a precomputed timeline… which
means the timeline can lag a few seconds, and I would measure that lag and alert if it exceeds one
second."

**Advanced.** The last clause is the senior part: **naming what you would measure to know whether the
decision was right**. "I would add a cache" is a junior answer; "I would add a cache, expect a 90% hit
rate, and alert if it falls below 70% because that would mean the key design is wrong" is a senior
one. Attach a metric to every significant decision (`O14`).

#### 2. Trade-offs, unprompted

**Theory.** Every choice costs something. Saying the cost before being asked demonstrates that you
chose rather than defaulted.

**Example.** The phrasing to practise: "I am choosing X over Y. X gives us A, at the cost of B. I
would revisit it if C happened." For instance: "Precomputing the timeline gives us constant-time
reads, at the cost of write amplification and eventual consistency. I would revisit it if the
follower distribution changed, or if write volume became the constraint."

**Advanced.** Also say what you would **not** build: "I would not shard yet — at these numbers one
Postgres handles it, and sharding costs us cross-shard queries and an irreversible key decision. The
signal to shard would be sustained write saturation after we have exhausted indexing, caching and
read replicas" (`DB25`). Restraint with a stated trigger is one of the clearest seniority signals
there is.

#### 3. Handling pushback

**Theory.** Interviewers push on your design — sometimes because it is wrong, sometimes to see how you
respond. Both need the same behaviour: engage with the substance, and update if the argument is good.

**Example.** Good: "That is a fair point — if writes are that heavy, precomputation costs more than it
saves, and I would switch to fan-out on read for those accounts." Also good: "I think it still holds,
because the read ratio is 100:1, so we are trading 200 writes for 20,000 reads. But if the ratio were
closer to even, I would agree with you." Bad: defending a position after the counter-argument has
landed, or abandoning a correct design at the first question.

**Advanced.** Being able to **change your mind visibly and cheaply** is a strong signal, because it is
what makes someone good to work with. So is saying "I do not know" and following it with how you would
find out: "I have not operated Cassandra at that scale; I would want to benchmark the write path
before committing, and here is the specific thing I would measure."

#### 4. What gets downgraded

**Theory.** Interviewers name the same anti-patterns consistently, and they are easy to avoid once you
know them.

**Example.** The list:
- Jumping to microservices, Kafka or a mesh with no requirement demanding it (`M01`, `SD03`).
- Buzzword stacking without justification.
- Ignoring the requirements that were agreed five minutes earlier.
- No numbers anywhere (`SD02`).
- Designing for 1000x the stated scale.
- Silence — designing in your head instead of out loud.
- Defensiveness when challenged.
- Never mentioning failure, cost, operations or the people who will run it.

**Advanced.** The positive counterparts are worth stating deliberately during the round: mention
**who operates this** and what they would be paged for (`O16`); mention **cost** where it is
significant (`O18`); mention **migration** because the system probably already exists (`SD14`); and
mention what you would build **first** if you had two weeks, which shows you can sequence work rather
than only architect it.

### Interview questions

- "What would you do differently if you had 10x the traffic? One tenth of the team?"
- "Why not use Kafka here?"
- "I disagree with your storage choice." (the response is the assessment)
- "What would you build first?"

---

## SD16 · Practice problem set

`Advanced` · Requires: `SD12` · Unlocks: —

### Preface

Design ability comes from doing it out loud, under time pressure, repeatedly. Reading about designs
produces recognition; talking through them produces fluency.

Do these at 45 minutes each, standing at a whiteboard or a blank document, **speaking aloud**, ideally
recorded so you can hear where you rambled or went silent.

### Details

#### 1. The problems

**Theory.** Fifteen problems covering the recurring shapes. The first five are the most commonly asked.

**Example.**
1. **URL shortener with analytics** — id generation, cache-heavy reads, asynchronous analytics.
2. **Rate limiter as a service** — distributed counters, accuracy versus cost (`SD11`).
3. **News feed / timeline** — fan-out hybrid, the celebrity problem (`SD09`).
4. **Chat (WhatsApp / Slack)** — connections, ordering, offline delivery, presence (`SD12`).
5. **Ticket booking (Ticketmaster)** — atomic claims, reservations, flash-sale load (`SD10`).
6. **Ride-hailing matching (Uber)** — geospatial index, high-volume location writes, trip state.
7. **File sync (Dropbox)** — chunking, deduplication, conflict resolution, metadata.
8. **Payments / wallet ledger** — double-entry, idempotency, reconciliation (`DB38`, `SD10`).
9. **Notification service** — multi-channel, preferences, deduplication, digesting (`SD12`).
10. **Metrics and alerting pipeline** — ingest, stream aggregation, columnar storage (`SD12`).
11. **Job scheduler at scale** — due-time indexing, exactly-one execution, fairness (`Q21`).
12. **E-commerce checkout with inventory** — reservations, saga across payment (`M14`, `SD10`).
13. **Multi-tenant SaaS API platform** — tenancy model, isolation, quotas (`M29`, `SD11`).
14. **Live leaderboard** — sorted sets, sharding, approximate ranks.
15. **Collaborative document editing** — and knowing where to stop and say "this needs CRDTs or
    operational transformation, and here is why I would use an existing library" (`M12`).

**Advanced.** Number 15 is worth including precisely because the correct senior answer includes
recognising that you would not build the core algorithm yourself. Knowing the boundary of what to
implement versus adopt is judgement, not a gap.

#### 2. What to produce for each

**Theory.** Write the same artefacts every time, so the structure becomes automatic under pressure.

**Example.** For each problem, produce: the agreed **requirements** (functional and non-functional with
numbers); the **estimates**; a **diagram**; the **data model** with key indexes; the **bottleneck**
and how you address it; a **failure story** for each major component; and **one trade-off** you would
flag to a staff engineer. Half a page each, and the practice is in speaking it, not writing it.

**Advanced.** After each, note what you **fumbled** — where you went quiet, where you had no number,
where you could not answer a follow-up. That list is your study plan, and it is far more useful than
re-reading material you already know. Two or three iterations of this loop close most gaps.

#### 3. How to practise alone

**Theory.** You will not always have a partner. Self-practice works if you enforce the constraints.

**Example.** Set a 45-minute timer. Record audio. Speak every thought aloud, including the clarifying
questions you would ask and the answers you will assume. Play it back and listen for: did you scope
first; did you give numbers; did you go silent; did you state trade-offs; did you cover failure; did
you finish. The playback is uncomfortable and it is where the improvement comes from.

**Advanced.** Then do the same problem again a week later with a different constraint — "now it must
be multi-region", "now the team is three people", "now writes exceed reads" — which forces you to
adapt the design rather than replay a memorised one. Adaptation is what the interview actually tests
(`SD15`).

#### 4. Depth to prepare per problem

**Theory.** Interviewers go deep on one or two areas. For each problem, know which areas are likely and
prepare one level below the obvious answer.

**Example.** For the feed: the celebrity problem (`SD09`), and how the timeline is rebuilt when the
ranking changes. For booking: the atomic claim and the expiry race (`SD10`). For chat: ordering and
offline delivery. For the rate limiter: accuracy versus cost and the Redis failure mode (`SD11`). For
payments: the unknown outcome and reconciliation (`SD10`).

**Advanced.** The follow-up that catches most people is operational: "how would you know this is
broken?" and "what would you be paged for?" (`O16`). Prepare a monitoring answer for each design —
the three metrics you would alert on and why — because it is asked often and almost nobody has
prepared it.

### Interview questions

- All fifteen above, timed and out loud.
- For each: "how would you know this was broken?"
- For each: "what would you build first, and what would you leave out?"

---

## SD17 · Senior behavioural signals

`Advanced` · Requires: `SD15` · Unlocks: —

### Preface

The behavioural round is not a formality, and for senior roles it is frequently where offers are
decided. The technical rounds establish that you can build things; this one establishes whether you
can be trusted with scope, ambiguity and other people.

Prepare it with the same seriousness as system design: specific stories, told concisely, with numbers
and with what you would do differently.

### Details

#### 1. The stories to have ready

**Theory.** Ten stories cover almost every question asked. Prepare each in **STAR** form — Situation,
Task, Action, Result — at about 90 seconds, with a real number in the result.

**Example.**
1. A system you designed end to end, and its trade-offs.
2. A production incident you led — detection, mitigation, root cause, prevention.
3. A technical disagreement you **lost** gracefully, and one you won with evidence.
4. Something you shipped that was wrong, what it cost, and what you changed afterwards.
5. Mentoring or levelling up a teammate, with a concrete outcome.
6. Pushing back on scope or a deadline, with data.
7. A migration or refactor you sequenced safely (`SD14`).
8. Working across teams to land something you did not control.
9. A time you chose the boring, simple solution over the interesting one.
10. An ambiguous, under-specified task and how you made it tractable.

**Advanced.** Stories 3, 4 and 9 are the ones that most distinguish senior candidates, and the ones
people prepare least. Being able to describe a decision you got wrong, without defensiveness and with
what you learned, reads as confidence. Being unable to name one reads as either inexperience or lack
of self-awareness.

#### 2. What "senior" sounds like

**Theory.** The difference between a mid-level and a senior answer is rarely the scale of the work; it
is what the story is **about**.

**Example.** The markers interviewers listen for:
- You talk about **why** and about trade-offs, not only about what you built.
- You mention **users, cost and operations**, not only code.
- You say "we" for the team's work and "I" for your own actions — and you are specific about which.
- You **quantify**: "reduced p99 from 2s to 200ms", "cut the bill by 40%", "the team went from weekly
  to daily deploys".
- You describe **influencing without authority** — convincing rather than mandating.
- You show judgement about what **not** to build.
- You mention what you would do **differently**.

**Advanced.** The subtlest and strongest marker is **taking responsibility for outcomes rather than
tasks**: "the project was at risk because the requirements were unclear, so I spent two days writing
them down and getting agreement" is a senior story even though nothing was built. Owning the outcome,
including the parts outside your job description, is what the level actually means.

#### 3. Answering well

**Theory.** Keep answers to about 90 seconds, lead with the point, and let the interviewer ask for
depth.

**Example.** The failure modes: rambling for six minutes so the interviewer loses the thread; giving a
generic answer with no specifics; describing the team's work without saying what **you** did;
blaming other people or previous employers; and having no result — a story that ends without saying
what happened is not a story.

**Advanced.** Prepare the **numbers** in advance, because you will not recall them under pressure:
team size, traffic, data volume, latency before and after, deployment frequency, how long the
migration took. Specific numbers make a story credible; vague ones make it sound rehearsed or
invented.

#### 4. Your questions for them

**Theory.** The questions you ask are assessed. They reveal what you care about and whether you are
evaluating them as seriously as they are evaluating you.

**Example.** Questions that work:
- "What does the on-call rotation look like, and how many pages a week is typical?" (`O16`)
- "How do decisions like a framework or database choice actually get made here?"
- "What is the biggest piece of technical debt a new senior engineer would meet?"
- "How do you measure a senior engineer's impact in the first six months?"
- "What broke most recently, and what changed afterwards?"
- "What would make you regret hiring someone into this role?"

**Advanced.** The last two are particularly good: "what broke recently and what changed" tells you
whether they have a functioning learning process (`O16`), and the answer to "what would make you
regret it" is usually unusually honest and tells you the real expectations. Asking questions that
could produce an uncomfortable answer signals that you are genuinely evaluating the role, which is how
senior candidates behave.

### Interview questions

- "Tell me about a time you disagreed with a technical decision."
- "What is the biggest mistake you have made in production?"
- "Tell me about something you chose not to build."
- "What questions do you have for us?"

# How Aura Works

*And what we mean to build.*

---

## About this document

Aura's design currently lives in seven places that disagree with each other. This
puts it in one place.

Five markers appear throughout:

> **▸ In the code.** Verified against the running implementation, with a line
> reference.

> **▸ Not yet built.** Intended by the team. Not implemented.

> **▸ Proposed.** Our suggestion. No team decision yet.

> **▸ Open.** Undecided.

> **▸ Decided.** Was open; a human called it. The call carries their name and where
> they made it.

Previous attempts failed by writing intentions as descriptions. The markers are the
fix.

The scorer referenced throughout is
[`scorer/verifications/aura.py`](https://github.com/Meta-Node/BrightID-Aura-Node/blob/master/scorer/verifications/aura.py)
on `master` of the Aura node fork — 477 lines, the only scoring implementation we
found in either org. `master` stops at October 2024; the `dev` branch is active
(scorer last touched July 2025, repo through May 2026, differing from `master` only
by a `modified` timestamp on impacts). Which branch production runs is unverified —
every mechanism cited below is identical on both.

---

## 1. What Aura is

Aura provides expert evaluations to applications — answers to questions no
algorithm can settle. Is this account a real, distinct person? Is this restaurant
good? The answers come from participants who have proven they answer accurately.

Participants answer questions honestly. Every layer above decides who to listen to,
and how much. A participant has one job: be accurate and honest. Get it right and
the people above you listen more; get it wrong and they listen less.

**Weighting belongs to the listener, never the speaker.** An answer doesn't change
because someone decided to trust its author less. Any number of parties can weigh
the same body of answers differently.

**New information can always change the answer.** Recomputation and revocation keep
the present answer accurate rather than preserving past ones, and every incentive
points at being accurate and honest — including updating when you learn you were
wrong. Attacks will sometimes succeed; each one that breaks teaches the network what
it looks like, and that attack doesn't work twice.

---

## 2. Evaluations

An evaluation is the only kind of fact in Aura. Scores, levels, and published
answers are derived from evaluations by arithmetic.

An evaluation is one participant's answer to a domain's question about a subject,
with a confidence in that answer.

In the BrightID domain, the question is "is this identifier this person's only
account?", the answer is positive or negative, and confidence is one of four values.

> **▸ In the code.** Confidence is an integer 1–4, shown as Low / Medium / High /
> Very High (`aura:packages/domain/src/labels.ts`). One answer and one confidence per
> evaluator, per subject, per tier. Neither field is validated server-side —
> `confidence` is an unconstrained number in `web_services/foxx/brightid/schemas.js`.

> **▸ Not yet built.** A domain may ask several questions, with a confidence per
> answer. BrightID asks one.

### Confidence does two jobs

Confidence scales an evaluation's weight: a very-high-confidence answer counts four
times a low-confidence one.

Confidence is also a gate. A team can require that a level be supported by
evaluations at or above a given confidence, from evaluators at or above a given
level. Without the gate, one powerful evaluator's offhand opinion could confer a
level, and many weak opinions could add up to a confidence nobody holds.

### Teams set what a rating means

A team tells its participants what its ratings mean. If a three means you have
direct knowledge, then giving threes casually makes you unreliable, and the people
evaluating you will score you accordingly. Another team may draw the line elsewhere.

The conditions for giving a four are often subjective and unconfirmable, so they're
a shared expectation rather than a rule the protocol enforces. Confidence is only
comparable between evaluators who share a calibration; the team supplies it.

### Negative answers count for more

A negative evaluation carries four times the weight of an equivalent positive one.
Negative information is how a compromised participant is stopped quickly, so it
should propagate harder than praise.

> **▸ In the code.** `FLAGGING_MULTIPLIER = 4` (`aura.py:17`), applied in all four
> tiers (lines 117, 202, 287, 377). The 4× appears in no document — the Levels doc's
> formula gives sign as ±1.

> **▸ Not yet built.** The multiplier should be configurable. Four has felt roughly
> right for years, but there will be domains where it isn't.

> **▸ Decided — energy flows only along positive evaluations, and that is correct.**
> Adam, on PR #4. An earlier draft of this document had negatives influencing energy
> flow as future work; the argument against it is decisive. A negative evaluation acts
> by **withdrawing** flow, not by reversing it. When Alice evaluates Bob negatively,
> Bob loses the energy Alice had been sending him and his ability to influence other
> managers shrinks — the intended result. Energy that flowed *negatively* would hand
> Bob more to spend, growing the influence of the manager the network just judged
> badly; and if he could choose the sign he passes on, he would simply flip it.
>
> The 4× above belongs to score accumulation on the edges. It never touches energy.
> **Two engines, and the boundary between them is deliberate** — see Section 5.

### An evaluation is a standing claim

An evaluation states a position you currently hold, not something you did once. You
remain responsible for it while it stands.

An evaluation carries the weight its author has now, not the weight they had when
they gave it. Someone with no standing can evaluate freely; their answers affect
nothing. The day someone above them vouches for them, everything they've said goes
live at once, including answers given months earlier.

---

## 3. Who evaluates, and what gets evaluated

### Evaluators must be accountable

An evaluation is only worth something if manufacturing evaluators is hard. An
evaluator is either:

- **A verified-unique human.** The existing BrightID guarantee.
- **An agent backed by a human or an organization.** Agents are copyable, so an
  agent can't be verified unique. Backing supplies what a counterparty needs:
  continuity, and an identified party who answers for it. A thousand agents achieve
  nothing when each must anchor to a scarce identity with standing to lose.

> **▸ Proposed.** The agent model — backing, lineage, what a backer risks — gets its
> own proposal document. The principle stated here is the part we're confident in:
> agents participate as first-class evaluators and subjects, and accountability
> reaches them through the identities that back them.

### Subjects need an identifier, not an identity

A subject is whatever the domain asks about. In the BrightID domain that's a person,
because the question is about people. In a restaurant domain it's a restaurant.

Joe's Pizza isn't a verified-unique human. It needs a stable identifier, so two
evaluators can be sure they mean the same restaurant. Nothing more.

> **▸ In the code.** Today all subjects are identities. Evaluations are edges in the
> `connections` collection between `users/` documents, so a subject is necessarily a
> user. That is the BrightID domain's shape, not a design commitment.

> **▸ Open.** How identifiers for non-person subjects are created and deduplicated.
> The direction: a task for the domain's own experts — check whether the thing is
> already listed before adding it — with incentives arranged accordingly.

---

## 4. Domains and tiers

A **domain** is a question space: what subjects can be evaluated, and what is asked
about them.

Questions are append-only. A domain can add questions and turn them on and off, but
never reset them, so an answer always refers to a question that still exists.

### A domain is a stack of tiers

Each **tier** says who is evaluated, what is asked about them, and who evaluates.

| Tier | Subjects | The question | Evaluated by |
|---|---|---|---|
| Subject | BrightIDs | is this a person's only account? | Players |
| Player | Players | does this player answer accurately? | Trainers |
| Trainer | Trainers | does this trainer back accurate players? | Managers |
| Manager | Managers | does this manager evaluate well? | Managers |

Each tier judges how well the tier below judges. A trainer looks for accurate
players and takes responsibility for the ones they vouch for; if those players prove
accurate, the trainer accrues standing, and if not, the trainer loses it.

The bottom tier's subjects are the domain's real subjects. Every tier above takes
the tier below's evaluators as its subjects — a trainer is a subject in the trainer
tier and an evaluator in the player tier.

The top tier evaluates itself. That's what Section 5 is about.

### Why three

Three evaluator tiers is not a default that happened to stick. It is the partition
that lets someone contribute while knowing only their own layer:

1. **Players** need to know how to answer the questions asked about subjects.
2. **Trainers** need to know who qualifies as a player on a team.
3. **Managers** need to understand how energy flows.

Nothing above the manager is left to learn, and collapsing any two tiers forces one
participant to hold two of those jobs at once. That is what the tier system buys —
an open system that keeps taking new participants, because the amount you must
understand to be useful stays small.

Trainers also carry a power the table above doesn't show: **an evaluation names the
team or teams it counts for.** A trainer is therefore deciding which players land on
which teams, and can hold new players on a starter team until they have built a
record.

> **▸ Decided — three evaluator tiers, fixed.** Adam, on PR #4 (the rationale above is
> his, recorded here for the first time); Philip agrees. Tier count is not the domain
> author's to choose. A domain may simplify what its participants *see* — a narrow use
> case can present as flat — but the three tiers still run underneath. This retires
> the flat-domain question in Section 5: there is no one-tier domain, only a one-tier
> presentation.

### Anyone can participate immediately

There is no gate on doing the work. Anybody can answer questions as a player, or
start evaluating players or trainers.

What you can't do is matter before someone above you says you should. Until a
participant in the tier above evaluates you, your standing is zero and your answers
affect nothing. They're still recorded, and the moment you receive standing they all
become live.

The Levels doc names this **level 0** — a provisional role. Anyone can hold one.

> **▸ In the code.** The scorer computes a score in a tier only for participants
> evaluated in that tier. One concrete unlock rule is shipped: you become eligible
> for Trainer evaluations after making your third evaluation.

> **▸ Open.** Unlock rules in general. They'll differ by domain; the constraint is
> firm — gate influence, not participation.

---

## 5. How scores are computed

Two calculations run in Aura. **Energy** handles the top tier, which evaluates
itself. **Accumulation** handles every tier below, where evaluation flows strictly
downward.

The top tier needs different treatment for two reasons. Managers evaluate managers,
so trust can loop — I vouch for you, you vouch for someone who vouches for me — and
left alone that inflates without bound. And someone has to be the origin: with no
tier above to confer weight, the top tier must be seeded.

### Manager energy

Energy exists only among managers. It's the input to manager scores, and it never
leaves the manager tier — everything below is computed from scores by arithmetic.

Energy starts at the team's owners and passes along positive manager-to-manager
evaluations in proportion to confidence. A manager passes on everything they hold at
each hop and retains nothing; what they have after a hop is only what flowed to
them. The owners hold all the trust at the start and none of it by the second hop
unless the people they empowered vouch back. Seeding is an origin, not a privilege.

When the hops finish, the settled energy is spent as weight. A manager's **score**
is the sum, over the manager evaluations they received, of confidence × the
evaluator's settled energy × sign. Holding energy gives you influence over other
managers' scores; your own score comes from who evaluated you, weighted by *their*
energy.

```
scale  = Σ confidence over the holder's positive manager evaluations
energy = holder.energy × confidence / scale          # per outgoing edge, each hop

manager score = Σ  confidence × evaluator.energy × (positive ? 1 : −4)
```

> **▸ In the code.** `STARTING_ENERGY = 1000000` split evenly across two hardcoded
> `TEAM_OWNERS`; `HOPS = 4` (`aura.py:11–17`). The destination collection is emptied
> each hop — no retention, so a manager's final energy is the last hop's inflow
> alone. Manager scores at `aura.py:97–133`: the weight is the evaluator's energy,
> looked up from the settled `energy` collection.

> **▸ Open.** How initial energy is allocated between owners. The even split is an
> implementation default, not a decision.

> **▸ Decided — hops are log(N).** Adam, on PR #4: the SybilRank result, corroborated
> by years of running it. Neither the code's fixed 4 nor the Levels doc's "2 or 4
> depending on the size of the team" is the rule — both are that rule frozen at one
> scale. `HOPS = 4` should become the computed value. *Still to pin down: the base,
> and N over which population — every participant in the domain, or the managers the
> iteration actually runs over.*

### Energy is conserved

Energy is a closed system: a fixed quantity redistributed, never created. Whenever
leakage has crept in before, we've regretted it — so the design goal is to make leaking
impossible, with guardrails that are themselves elegant.

Today it leaks. A manager who holds energy but evaluates no other managers has no
outgoing edge, so their share vanishes at the next hop. The cost isn't that
manager's standing — their score comes from their evaluators, not their own energy.
The cost is systemic: the total pool shrinks every hop, so every manager's score
shrinks against level thresholds that are fixed absolute numbers. Levels get harder
to reach for reasons unrelated to anyone's behaviour.

> **▸ Decided — block the dead ends in the code.** Adam, on PR #4: "I agree we can
> block routing energy to dead ends." Energy is not routed to a manager with no
> outgoing edge; it redistributes as though that edge weren't there. Chosen over
> renormalizing the pool each hop, and it settles a question that had previously been
> treated as socially solved — nag the manager, or stop sending them energy. A result
> we know we don't want should be made impossible rather than discouraged.

> **▸ Open.** The mechanics: exactly how the redistribution is computed, and what
> happens on the last hop, where by construction every node is a dead end.

> **▸ Decided — there is no flat domain.** See Section 4: three evaluator tiers are
> fixed, so a domain that looks flat is a presentation over the same stack rather than
> a different shape. The question dissolves instead of being answered — there is no
> lone evaluator tier needing a source of weight, because there is no lone tier.

### Accumulation below

A manager's score weights their evaluations of trainers; trainer scores weight
player evaluations; player scores weight subject evaluations. Energy stays at the
top. Scores are what travel down.

Each evaluation contributes:

```
impact = confidence × max(evaluator score, 0) × (positive ? 1 : −4)
score  = Σ impacts
```

An evaluator with a negative score contributes nothing — not a reduced amount,
nothing — across their entire body of work at once. A negative score doesn't say a
particular answer was wrong. It says that in totality this person should not be
listened to, so nothing they've said is counted.

This is what lets a large attack be stopped by a single evaluation change. The
protection against collateral damage isn't a cap on vouching — it's having more
than one person above you. If your only backer is
discredited, that's a fact about how you built your standing.

> **▸ In the code.** The `max(…, 0)` guard at `aura.py:201, 286, 376` (trainer,
> player, subject). The manager tier needs no guard — energy can't go negative.
> Added October 2024; commit message: *"fix bug where an evaluator with a negative
> score can have an impact."*

> **▸ Open.** The Levels doc's formula caps how much any single evaluator can
> contribute. No cap exists in the code. Dropped once level requirements existed, or
> never built — nobody currently knows.

### Scores are derived, never stored as facts

A score is always recomputed from the evaluations currently standing and the
configuration currently in force. Nothing is grandfathered. Copy a domain's
configuration and participants exactly and its scores come out identical; change
anything and they change with it.

This settles what to do when a parameter changes: recompute everything under the
current parameters. A score is a present claim, not a historical record.

### The pipeline today

An evaluation enters through the node's API and lands within seconds as an edge in
the `connections` collection — one per (evaluator, subject, tier), newest replacing
oldest. The scorer runs in batches, roughly every twenty minutes (Adam,
2026-07-20): it reads the graph, computes energy, then scores, then levels, and
writes results to the `verifications` collection. Applications consume from there —
per user, `domains → categories → { score, level, impacts }`, one category per
tier.

---

## 6. Levels

A score is a number whose magnitude means nothing outside its own team. A **level**
is a claim that means something.

A level requires a minimum score **and** minimum evaluation requirements — how many
evaluations, at what confidence, from evaluators at what level. Requirements combine
with AND and OR.

> **▸ In the code.** Subject level 3 (`aura.py:387–390`):
> ```
> score >= 100000000 and (
>     count(impacts[* filter CURRENT.level >= 2 and CURRENT.confidence >= 3]) >= 1 or
>     count(impacts[* filter CURRENT.level >= 2 and CURRENT.confidence >= 2]) >= 2
> )
> ```
> The full shipped ladders: manager 1000 / 200000 and trainer 500000 / 1M — score
> only (`aura.py:122–126, 207–211`); player 1M / 2M / 3M and subject 10M / 50M /
> 100M / 150M, with requirements on the upper rungs. Subject reaches level 4, which
> demands level-3 evaluators.

The shipped ladders belong to the implicit single team (Section 13) — which is why
absolute thresholds can coexist with "score magnitude means nothing outside its own
team." When real teams exist, each defines its own.

The requirements are why levels are worth having. A score can be reached by
accumulation; a level can't be reached without a specified quality of evidence from
evaluators who have themselves earned standing.

**Level −1** exists for a negative score. Requirements don't apply to it; the score
alone decides. **Level 0** is the provisional state — a role taken but not yet
vouched for.

One bootstrap rule: team owners — the Levels doc calls them captains; same role —
are exempt from minimum evaluation requirements and reach their manager levels on
score alone. Without the exemption a new team
couldn't start — its founders can't satisfy requirements that need evaluators who
don't exist yet.

> **▸ Decided — design choice.** Adam, on PR #4: every role is open to every
> participant, and evaluations from others are what graduate someone out of
> provisional. Manager and trainer levels carrying no evaluation requirements is the
> open-participation rule working, not a gap in the implementation.

> **▸ Open.** Current thresholds and requirements are starting points, not the
> destination. Level definitions need to stay flexible enough to experiment with
> gates nobody has thought of yet.

---

## 7. Teams

A team is an independent scoring body. It has its own owners and its own standards,
and computes its own scores from the evaluations it receives.

Multiple teams give the system resilience. If a team is captured or becomes
unreliable, consumers stop weighting it and the rest keep working.

Teams differ by their standards, not their rosters. Two teams can share every
trainer and still answer differently, because their level definitions differ: a
low-confidence evaluation clears one team's bar and not the other's, with no
decision by the evaluator.

Standards run in both directions. A team says what it takes to **receive** a level —
score and evaluation requirements (Section 6) — and what it means to **give** a
rating: if a three means direct knowledge, giving threes casually makes you
unreliable there.

### Membership

The sources give different answers, and the gaps need a team conversation rather
than a quiet ruling here.

- The Definitions doc: a player joins a team on receiving a level from it, and the
  domain caps how many teams a player can join.
- The Teams page: a manager or trainer adds a participant they evaluate to any of
  the teams they themselves belong to.
- The Features page: players belong to any team whose energy reaches them, even
  unknowingly.

> **▸ Decided — the cap is outbound, at trainer and above.** Adam, on PR #4: "This is
> exactly right." Players never choose or join — a trainer on a team evaluates you
> and your evaluations start counting there. The choice and the cap sit at trainer
> and above, on the **outbound** side: standing flows in uncapped from any number of
> teams, and you pick a handful to pass down through. Teams you don't pick get
> nothing from you. The point of the limit is scarcity — good evaluators become
> something teams compete for, which pushes teams to send real resources downstream
> and keeps them from converging on the same people.

> **▸ Open — the number.** Adam recalls five from discussions with Philip, and accepts
> the consequence: five teams per trainer, and a player with several trainers reaching
> more than five that way. Five is the working convention; nobody has a derivation for
> it, and the historical inbound player cap may have had a reason we have lost.

### Team owners

A team starts with one or more owners, who become its first managers and the origin
of its energy. Owners add and remove owners by a two-thirds majority. Owners define
the team's levels. Creating a team costs a fee.

> **▸ Not yet built.** All of Section 7. The backend has two hardcoded owner keys
> acting as a single implicit team — no teams collection, no membership, no per-team
> scoring.

---

## 8. The league, and the mix

An application asking Aura a question needs one answer, not a table of team scores.
Two separate things produce that answer, and an earlier draft of this document ran
them together — describing a league that ingested team outputs, weighted them, and
published a result. That is not the design.

### The mix is a computation, not an institution

Teams publish their own scores. The weights over those teams come from **crowd
wisdom** — the people who ultimately rely on Aura's answers. A consumer takes the
weights and the scores and combines them, and every consumer doing so arrives at the
same result independently. Nothing sits in the middle ingesting anything, and there
is no privileged copy of the answer.

This is the same rule as everywhere else in Aura: the output is derived, so anyone
holding the inputs can recompute it (Section 5). Choosing the mix is the one input
the system collects from outside itself.

> **▸ Decided.** Adam, on PR #4: "Crowd wisdom determines the weights of different
> teams in a mix… Teams publish their own scores. Consumers take the crowd wisdom
> weights and the team scores and combine them. Every consumer using these would
> arrive at the same combined scores for evaluations independently." See also
> *[Decentralizing BrightID with Collective
> Intelligence](https://paragraph.com/@adamstallard/decentralizing-brightid-with-collective-intelligence)*.

### The league is the organization, not the calculator

Adam's position is that crowd wisdom supersedes the notion of a league. Philip's is
that it supersedes the league's **arithmetic** and not the league itself: something
still has to make this one cohesive platform rather than a protocol with participants
scattered around it. Somebody convenes the teams, runs whatever crowd wisdom is
collected through, markets one product, and answers for it when it is wrong. That is
the league, and it survives the mix moving out of its hands.

The **league contract** in Section 9 is the concrete form of it, and Adam specified
it in the same review: one contract per domain that takes team registrations and
fees, holds the funds applications put in, and distributes them by crowd wisdom. It
computes nothing about trust. So the sharp version of the league is **the money and
membership rail** — not the calculator, and not nothing.

How the league governs itself is deliberately out of scope. Today it's a few people
making decisions; later it may have many participants, or defer to a market —
possibly a prediction-market layer where timestamped answers are graded as reality
reveals itself and payouts follow the results.

There is one league today and network effects favor there staying one, so the fork
exists as an outlet rather than a rival: participants and teams are independently
addressable and never owned by the league. A league that is only an organization is
already thin in the way that requires.

> **▸ Open — for the team.** Whether these two positions actually conflict, or are one
> picture described from opposite ends. What is clear either way: no entity computes
> the mix.

> **▸ Open — what crowd wisdom is, mechanically. The largest gap in this document.**
> It sets every weight in the system, it is the only input gathered from outside Aura,
> and per Section 9 **it is also what moves the money** — the league contract
> distributes application funds to teams by applying it. Nothing anywhere describes how
> it is collected, who counts as the crowd, or why it resists capture. A weighting
> scheme can be provisional; a payout rule can't. Adam — is Updraft the intended
> mechanism here?

> **▸ Open — levels or scores as the published unit.** Adam says teams publish scores.
> Ali argues for levels (PR #4): a level is not a score-range label, it is a score
> **plus** the evaluation requirements of Section 6, so a level is something that has
> already cleared the team's evidence bar where a raw score has not — which is the same
> reason Section 6 gives for levels being worth having at all. Both positions are on
> the record and neither has seen the other. Undecided, and it decides the format every
> team must publish. The older sources split the same way: the Definitions doc has
> leagues weighting team **levels**; the Teams page and the collective-intelligence
> article both mix team **scores**.

> **▸ Not yet built.** None of this exists in the code. No weights are collected, no
> combination is computed, and there is one implicit team to mix (Section 13).

---

## 9. Money

Applications that want answers fund the domain that produces them. The more
resources in a domain, the more participants direct their time and attention to its
questions, because payouts follow. Show me the incentive and I'll tell you the
outcome: people put funds in the top because they want answers, and participants
answer accurately and honestly because proving accuracy is what gets rewarded.

Rewards reach participants through their teams — scores provide a natural scale for
dividing a team's share — and applications can direct attention where they need it,
like paying teams to cover a region.

### The league contract

Money moves through a contract, one per domain. Adam's description, on PR #4:

- It accepts **registration from teams**, including a fee, to enter a domain.
- It accepts **funds from applications** into that domain.
- It **applies crowd wisdom results to distribute those funds to teams.**

Team owners withdraw their team's funds and allocate them to participants according
to each participant's score within their tier.

Note what this is and isn't. The contract holds money and gates membership; it does
not compute anything about trust. Scores stay with teams and the mix stays with
consumers (Section 8) — but the crowd wisdom that weights the mix is also what moves
the money, which is the whole reason that mechanism has to be pinned down.

> **▸ Not yet built.** The contract does not exist.

> **▸ Open.** Whether team owners' allocation by score is enforced by the contract or
> is a convention they follow, and what stops an owner allocating otherwise.

> **▸ Proposed.** The rest of the mechanics — fee structure, payout paths, the fee
> that creates a team (Section 7) — get their own proposal document, on their own
> timeline. Ali's call, and it stands: economics does not gate the core spec. The
> League Contract sketch and our economics draft both exist and disagree in places.

---

## 10. Time

An evaluation is a standing claim, so it keeps counting until its author changes it.
That's usually right and sometimes wrong.

A restaurant review from five years ago wasn't incorrect; the restaurant has changed
hands twice since. The reviewer has no new information and no reason to revisit
their answer, so nothing about the evaluation itself will ever mark it as stale.

> **▸ Decided — decay is an option a domain can turn on.** Adam, on PR #4: "Could be
> useful in some domains. Let's add it as an option." Not a default and not universal:
> a domain that asks "is this a unique human" wants a rate near zero, a domain asking
> about restaurants wants a fast one. Decay appears in no existing Aura document — it
> arrived via the protocol draft and the rationale is ours.

> **▸ Open.** What the rate attaches to. The argument above says decay should track
> how fast the *subject* changes, which points at the question rather than the domain
> — a domain could hold both a slow question and a fast one.

---

## 11. Starting a new domain

A new domain shouldn't start empty. We have the data, the connections, and the
structure of who listens to whom; a new domain should be able to start from them.

What transfers cleanly: **level definitions** (editable), and **subjects** when the
new domain asks about the same things. What never transfers: **scores** — they're
recomputed from what you brought and how you configured it (Section 5).

Evaluations and participants are where forking gets dangerous. An evaluation answers
a specific question; a trainer said a player was good *at evaluating BrightIDs*, and
carrying that standing into an insurance domain converts it into an opinion the
trainer never gave.

The same argument runs the other way at the top of the stack. "This manager evaluates
well" is a claim about judgment itself, and judgment travels better than domain
knowledge does.

> **▸ Decided — what a fork carries.** Adam, on PR #4, and now in the Definitions doc:
> *"Team owners can copy a team into a different domain, taking manager and trainer
> sets and manager evaluations as a starting point."*
>
> - **Managers and trainers copy wholesale**, with manager evaluations. Management
>   skill transfers across domains readily enough that carrying it isn't the category
>   error described above.
> - **Players don't.** Their trainer decides which of them come and at what starting
>   level — everyone at level 1, or hand-picked mappings. This is the case where the
>   judgment really was about one domain's question, and Adam's reason is the same one
>   this document gave: player skill in one domain may or may not transfer to another.
>
> This supersedes the older Definitions language, which had forks copying subjects,
> teams, participants and evaluations wholesale.

> **▸ Proposed — two bounds on the default copy.** Copying by default is right for the
> cold start, and this document's earlier draft was wrong to demand opt-in everywhere.
> What still needs protecting is narrower: a copy should never become a *claim* about
> the person copied.
>
> - **You can disown a domain.** A copied participant can remove themselves and their
>   standing from a fork at any time, without needing the forking team's cooperation.
> - **A domain can't advertise you.** That a manager's evaluations were copied into a
>   domain is a fact about the copy, not a statement that they back it. Copied standing
>   is not endorsement and can't be presented as such.
>
> Room for automation either way: standing offers, tranches, a team joining a new
> domain as a unit and bringing its structure with it.

> **▸ Decided — a fork is a snapshot. No live link to the source.** Ali and Adam, both
> on PR #4 — Adam: "It shouldn't stay linked to its source. It needs to be allowed to
> fully diverge." Domain isolation exists so that failure stays contained, and a live
> link would make a fork's integrity depend on its parent staying healthy —
> reintroducing exactly what isolation removes. The consent story is cleaner too:
> opting into a snapshot is a bounded decision, opting into an ongoing link is
> open-ended and harder to understand what you agreed to. If linking's benefit ever
> proves out in practice, a participant can re-fork by hand; there is no need to build
> automatic propagation speculatively.

---

## 12. Privacy

The rule Aura already lives by: **no information should be shared with anyone that
doesn't already know it.** Verification comes from people who already know the
person being verified — that's what makes it privacy-preserving rather than
privacy-invading.

For domains, at least four things separate: the **trust structure** (who evaluates
whom, and their standing), the **question set**, the **answers**, and the
**interpretation** that turns answers into levels. A business may want the structure
open and the rest closed — asking questions and getting answers without publishing
either.

An opaque thing is trusted through whoever stands behind it. That's one mechanism
serving two cases: an agent is trusted because accountable identities back it, and a
private domain is trusted the same way. A domain that's completely dark — no visibility, no credible
backers — isn't trustable.

> **▸ Open.** Nearly all of this. Which layers can be private, what a private domain
> must publish, and whether its outputs can be relied on by third parties.

---

## 13. What is not built

The design above is largely not implemented. The honest state:

| | Status |
|---|---|
| Evaluations, confidence, signed scores, 4× negatives | Built |
| Level requirements (AND/OR over count, confidence, evaluator level) | Built, for subject and player levels only |
| Manager energy, seeded and bounded | Built — even split, leaks at dead ends; hops hardcoded to 4 where the rule is log(N) |
| Teams | Not built. Two hardcoded owner keys act as one implicit team |
| Multiple domains | Not built. Evaluations carry a `domain` field; the scorer ignores it and pools everything under `"BrightID"` |
| The mix — crowd-wisdom weights, combined consumer-side | Not built. No weights are collected and no combination exists |
| The league contract — registrations, fees, fund distribution | Not built |
| Decay | Not built |
| Non-person subjects | Not built. Subjects are `users/` documents |
| Per-evaluator cap | Not built, though the Levels document's formula has one |
| Configurable negative multiplier | Not built |
| Input validation on confidence and evaluation | Not built |

`master` stops at October 2024; `dev` carries dashboard-integration and
signed-verification work through May 2026. The scoring algorithm itself is
unchanged since October 2024 apart from a timestamp field. Most recent work has
gone into the app and SDK.

---

## 14. Open questions

**For Adam — the scoring math**

1. Dead-end routing is settled (Section 5); the mechanics aren't. How the
   redistribution is computed, and what happens on the last hop, where by
   construction every node is a dead end.
2. `log(N)` hops: which base, and N over which population — every participant in the
   domain, or the managers the iteration actually runs over.
3. How initial energy is allocated between team owners.
4. The per-evaluator cap: dropped once level requirements existed, or never built?

**For the team — design**

5. **What crowd wisdom is, mechanically** (Sections 8 and 9). It sets every weight in
   the system, it is the only input from outside Aura, and it moves the money.
   Largest undefined thing in this document by a distance.
6. **Levels or scores as the published unit** (Section 8) — decides the format every
   team must publish. Adam and Ali are on record on opposite sides.
7. Whether "the league" and "crowd wisdom supersedes the league" are in conflict or
   are one picture from two ends (Sections 8 and 9).
8. Whether the two bounds on a default fork copy — disown, and no advertising —
   are accepted (Section 11).
9. Whether allocation by score inside a team is contract-enforced or a convention
   (Section 9).
10. The outbound number (Section 7). Five is convention, not derivation.
11. What a decay rate attaches to — the domain or the question (Section 10).
12. Role unlock rules beyond the shipped third-evaluation rule. The general rule is
    settled (Section 6); per-domain rules aren't.
13. Identifiers for non-person subjects (Section 3). Ali working.
14. The privacy model, essentially all of it (Section 12). Ali working.
15. Which branch production runs — `master` or `dev`. The route tables are
    identical; the tell is whether live verification responses carry `modified`
    inside impacts. One valid BrightID settles it.

**Closed since the first draft**, with whose call it was:

| | Called by | Where |
|---|---|---|
| Energy flows only along positive evaluations; negatives withdraw flow | Adam | §2 |
| Three evaluator tiers, fixed — flat domains dissolve | Adam, Philip | §4, §5 |
| Hops are log(N) | Adam | §5 |
| Block dead-end routing in the code | Adam | §5 |
| Role unlock is a design choice, not an artifact | Adam | §6 |
| The team cap is outbound, at trainer and above | Adam | §7 |
| Decay exists, as a per-domain option | Adam | §10 |
| A fork carries managers and trainers by default; players come via their trainer | Adam | §11 |
| A fork is a snapshot; no live link | Ali, Adam | §11 |

This list is not complete and isn't meant to be. Found a new one? Add it.

---

## How to engage

This document lives in aura-decisions. Changes are PRs against it.

- Something marked **▸ In the code** is wrong? Cite the line that proves it — those
  claims are meant to be falsifiable in minutes.
- Disagree with a **▸ Proposed**, or have your own? Say so on the PR, or write
  yours with the same marker. Don't write intentions as descriptions — that's how
  the previous attempts died.
- Can answer an open question? Answer it and put your name on it.

A decision is real only when a human states it — here, in an issue, or in the
workroom. Nothing in this document, markers included, is a decision until then.

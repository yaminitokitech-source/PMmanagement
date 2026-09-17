# Monolith → Microservices on AWS: Manager's Playbook

Migration + optimization project notes. Three parts:

1. The generic phase-by-phase plan
2. A worked example (travel booking monolith) — how a manager would actually run it
3. Interview questions to test whether someone really ran one of these

---

# Part 1 — The Generic Plan

## Phase 0 — Baseline and business case (3–4 weeks)

Before any code moves, get agreement on *why*.

- Pick 1–2 measurable goals: release speed, cost per transaction, scale ceiling, or team autonomy. "Microservices because monolith bad" will kill you in month six when someone asks what you bought.
- Capture the before-picture: current AWS cost (Cost Explorer), p95 latency, deploy frequency, incident count, infra footprint. This is what you'll report against for the optimization half.
- Inventory everything: modules, DB tables and their joins, batch jobs, cron, file drops, third-party integrations, scheduled reports. The forgotten nightly job is what breaks cutover.
- Get a written sponsor and a budget line for the parallel-run period.

## Phase 1 — Decomposition design (4–6 weeks)

- Slice by business domain (bounded contexts), not by technical layer. A "DAO service" is a mistake.
- Build a dependency map and a *sequence* of extraction. Order matters more than the map.
- Choose the first service on: low coupling, owns its own data, real business pain, low blast radius. Never start with the crown jewel.
- Decide upfront which modules will **not** be extracted. Some parts should stay as a modular monolith. This decision saves months.
- Data is the hard part, not the code. For each service, answer: who owns these tables, what joins break, what transaction spans boundaries.

## Phase 2 — AWS foundation (parallel with Phase 1)

- Account structure (dev / test / prod), VPC design, IAM baseline, Organizations + guardrails.
- Runtime choice: ECS Fargate vs EKS. Pick on team skill, not fashion. Fargate if you don't have a platform team; EKS if you do and need portability.
- Non-negotiables on day one: Terraform/IaC, ECR, per-service CI/CD pipeline, centralised logging, distributed tracing (OpenTelemetry/X-Ray), secrets in Secrets Manager.
- Edge and plumbing: API Gateway or ALB, service discovery, SQS/SNS/EventBridge for async.
- If observability comes later, you will be debugging blind exactly when you need it most.

## Phase 3 — Strangler-fig extraction (the long stretch)

- Put a facade/gateway in front of the monolith. Route one capability at a time to the new service, keep the monolith as fallback.
- Data separation pattern per service: shared DB → read-only views → own database, using DMS/CDC to sync during transition. Avoid long-lived dual writes.
- Use the **outbox pattern** for reliable events, and **sagas** instead of distributed transactions.
- Contract testing (Pact or similar) between services — this replaces the safety net you lose when you break up the build.
- Accept eventual consistency and get business sign-off on where it's acceptable. This is a business conversation, not a technical one.

## Phase 4 — Optimization

Treat this as a separate workstream with its own targets, not a leftover.

- Right-size and autoscale per service; Graviton and Fargate Spot for suitable workloads; Savings Plans once the shape is stable.
- Caching (ElastiCache), read replicas, async offloading of anything the user doesn't wait for.
- Tag every service for cost attribution so each team sees its own bill. This changes behaviour faster than any mandate.
- Set per-service SLOs and alert on them.

## Phase 5 — Cutover and decommission

- Per service: shadow traffic → canary → full switch → rollback plan documented and tested.
- Then actually delete the old code and turn off the old infra. Put this in the definition of done, with a date.

## Major risks to track as manager

| Risk | What it looks like | Mitigation |
|---|---|---|
| Parallel-run cost | Running both systems for 12–18 months | Budget 25–40% overlap cost explicitly upfront |
| Never-finished migration | 60% extracted, monolith still live in year three | Hard decommission dates per slice; sequence tied to sponsor reviews |
| Distributed monolith | Services that must deploy together | Enforce independent deployability as an exit criterion |
| Feature freeze pressure | Business sees no new features | Ship an early visible win; run feature work in parallel on the monolith |
| Skills gap | Team new to containers, IaC, async patterns | Training budget, first service as a guided pilot, hire one platform lead |
| Conway's law | Org chart doesn't match service map | Reorg teams to own services end to end before extraction, not after |

## Metrics to report monthly

- The DORA four: deploy frequency, lead time, change failure rate, MTTR
- Cost per transaction
- p95 latency per domain
- % of monolith decommissioned

## Realistic timeline shape

For a mid-size monolith: 3–4 months of foundation and first service, then 12–18 months of extraction, with optimization running as a continuous track from month six.

---

# Part 2 — Worked Example: Travel Booking Monolith

## The scenario

**"BookingCore"** — a Spring Boot monolith, ~800k lines, one big Oracle schema, 40 engineers, deploys once every three weeks on a Saturday night. Handles search, pricing, booking, payment, ticketing, cancellations, loyalty, notifications, and B2B agent APIs. Peak load around holiday sales. Two of the original architects have left.

**Business pain that got funding:** a fare-rule change takes six weeks to ship, and every Black Friday we over-provision the whole monolith because search spikes 20x while booking only doubles.

That last sentence is the entire business case. Search and booking have completely different scaling profiles but share one deployment unit. That's where the money is.

## Target service map

Roughly 8–10 services, not 40:

- **Search & Availability** — read-heavy, no state, spikes hardest
- **Pricing & Fare Rules** — changes most often, own release cadence
- **Booking/PNR** — the crown jewel, owns the reservation record
- **Payment** — compliance boundary, worth isolating for PCI scope alone
- **Ticketing & Fulfilment** — GDS/supplier integration, async by nature
- **Customer Profile & Loyalty** — stable, low change
- **Notifications** — easy, fully async
- **Agent/B2B API** — different consumers, different SLAs

**Staying in the monolith for now:** cancellations, refunds, reporting. Say this out loud in the kickoff so nobody assumes everything gets extracted.

## Extraction order and the reasoning

**1. Notifications** (month 3–4)
Nobody's afraid of it. Real purpose: prove the pipeline — ECR, Terraform, CI/CD, tracing, on-call. If this takes four months instead of one, that's a finding, and better to find it here.

**2. Search & Availability** (month 4–8)
The money slice. Read-only, no writes to untangle, and it's what's driving the over-provisioning. When search scales independently and the Black Friday bill drops, the sponsor stops asking whether this project is worth it. *Always spend your first real win on the person who controls the budget.*

**3. Pricing & Fare Rules** (month 7–11)
The speed slice. Six weeks to six days on a fare change is the story the business actually cares about.

**4. Payment** (month 10–14)
Isolating it shrinks PCI audit scope, which has its own budget owner who will co-fund it.

**5. Ticketing & Fulfilment** (month 12–17)
Already async in spirit — supplier calls, retries, queues.

**6. Booking/PNR** (month 15–22)
Last, deliberately. By now the team has done this five times and the surrounding services are stable. Attempting PNR first is how these projects die.

Search and Pricing overlap on purpose — two squads working in parallel once the foundation is proven.

## Team structure

Reorganise **before** extracting, not after.

- Three squads of 6–8, each owning services end to end including on-call
- One 4-person platform squad owning Terraform, pipelines, observability and the gateway
- One architect across all of them, part-time on each
- A fourth group stays on the monolith doing feature work

That last point is what managers skip and regret. The business will not accept an 18-month feature freeze, and if you don't plan for parallel feature work, it'll happen anyway — chaotically.

## Data separation: the actual hard part

Staged plan for PNR, over months not a weekend:

1. Booking service reads from **read-only views** on the Oracle schema; writes still go to the monolith
2. Booking service **owns writes**, publishes events via the **outbox table**; monolith consumes them
3. **DMS/CDC** keeps the legacy tables in sync for anything still reading them
4. Cut the legacy readers one by one, then drop the sync

**The ugly question:** cancellation spans Booking, Payment and Ticketing. That becomes a **saga with compensating actions**, and the business has to sign off that a refund may show as "processing" for 30 seconds. Take that to the ops director in writing rather than letting an engineer decide it in a sprint.

## First 90 days, concretely

| Weeks | What |
|---|---|
| 1–3 | Baseline metrics, full inventory, sponsor signed, budget for parallel run agreed |
| 3–6 | Domain modelling workshops with ops, revenue management and agent support — not just engineers |
| 4–10 | Platform squad builds the foundation: accounts, VPC, ECS Fargate, Terraform, pipelines, OpenTelemetry, gateway |
| 6–9 | Team reorg announced and settled |
| 8–12 | Notifications extracted, deployed, on-call rota live |
| 12 | First steering review: show the pipeline working end to end |

If week 12 slips badly, better to reset scope then than at month twelve.

## What to watch for as manager

- Engineers wanting to start with PNR because it's the interesting problem. Say no, explain why.
- Search service still calling back into the monolith for a config lookup. That's a distributed monolith forming, and it's easiest to kill early.
- Decommission dates quietly slipping. Make "old code deleted, old infra off" part of done, with a named owner and a date on the plan.
- The 18-month mark, when enthusiasm dies. That's why the money win lands at month 8 and the speed win at month 11.

---

# Part 3 — Interview Questions

The trick isn't asking hard questions — it's asking questions where a rehearsed answer and a lived answer sound completely different.

## On the business case

**"What would have happened if you'd done nothing?"**
- Weak: "We'd be stuck on legacy tech."
- Strong: a specific cost or revenue number, and an admission that some parts genuinely didn't need migrating.

**"Who signed the cheque, and what did you promise them?"**
If they can't name the sponsor and the one metric that sponsor cares about, this was an engineering project with no business owner.

**"What was your baseline, and what is it now?"**
Anyone who actually ran this remembers their before-numbers. Vague "we improved deployment speed" means they never measured it.

## On decomposition

**"How many services did you end up with, and how many did you originally plan?"**
The gap is the interesting part. Nobody gets this right first time.

**"What did you decide *not* to extract, and why?"**
Best question in the set. Someone who says "everything" either didn't finish or didn't think. Real answers name a module that stayed in the monolith on purpose.

**"Which service did you start with, and would you pick it again?"**
Listen for whether they picked for learning value or for the interesting problem.

**"Tell me about a service boundary you got wrong."**
If they say none, they're either lying or the project didn't run long enough to find out. Good answer: which two services ended up chatty, and whether they merged them back.

## On data — spend the most time here

**"Walk me through how you split one specific table that two services needed."**
The single best separator of real experience from reading. Want to hear views, CDC, dual-write windows, and what the cutover looked like. Hand-waving here means they managed the project from a slide deck.

**"Where did you have a transaction spanning what became two services, and what did you do about it?"**
Want to hear saga or outbox — and more importantly, that they took the eventual-consistency question to the *business* for sign-off rather than deciding it in a sprint.

**"How long did you run both systems in parallel, and what did that cost?"**
If they don't have a number, they didn't budget for it, which means it came out of someone's hide mid-project.

## On delivery and reality

**"What was the business shipping while you migrated?"**
"We froze features" is a red flag unless it was a few weeks. Real answer: a team stayed on the monolith, and there was pain merging.

**"What did you actually decommission, and when?"**
The question most people fail. Plenty of migrations reach 70% and stall with the monolith still running. Want dates and a specific "we turned off the old EC2 fleet in March."

**"Did you reorganise the teams? Before or after?"**
After usually means they fought Conway's law and lost for a while.

**"Tell me about the worst incident this caused in production."**
Everyone breaks something. Listening for whether they had tracing in place to diagnose it, or spent six hours guessing across five services.

## On the AWS and cost side

**"ECS or EKS, and what was the argument against your choice?"**
Doesn't matter which they picked. Matters that they can argue the other side, which proves it was a decision rather than a default.

**"Your bill went up before it went down. How did you explain that?"**
Tests whether they saw it coming and warned the sponsor, or got ambushed.

**"Can you tell me what one service costs to run today?"**
Only possible if they tagged for cost attribution. Reveals a lot about operational maturity.

## The closers

- **"What would you do differently?"** — if the answer is polished and flattering, push again.
- **"What's still not done?"** — every one of these projects has a leftover. Someone honest names it immediately.
- **"If your best engineer told you the whole thing was a mistake at month nine, what would you have done?"**

## The general tell

People who **ran** it talk about *sequence and tradeoffs*.
People who **observed** it talk about *architecture and tools*.

If every answer is about technology and none about ordering, budget, or the conversation they had with the ops director, they weren't in the chair.

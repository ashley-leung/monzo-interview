# Behavioral interview
## "Tell me about a time where you led a complex project"

### Situation
eBay partnered with Breitling, a luxury Swiss watch manufacturer to enable sellers to attach a blockchain-verified Digital Product Passport to their listings, allowing sellers to transfer ownership digitally to buyers after purchase.

The problem was complex because I was integrating across:
- 6 systems with internal teams, external teams and third-party provider (Wallet team, Portfolio team, Arianee, Federated GraphQL, VAS Events, Kafka)
- No blockchain infra existed internally
- High ambiguity — unclear partner timelines, evolving requirements, and no prior blueprint inside eBay for Node.js.

### Task
I owned end-to-end delivery of the purchase-to-fulfilment flow, specifically:

- Ensuring that when a watch was purchased, the digital passport transferred correctly from seller to buyer
- Designing a system that was correct, reliable, and observable

Success meant:
- Zero incorrect ownership transfers
- No data loss across asynchronous systems
- Production readiness with clear observability and runbooks

### Action
**First**, I distilled complexity into a simple model by defining the rule of:

“Only transfer ownership when two signals are true: purchase complete + physical authentication complete.”

This helped align stakeholders and guided architecture decisions.

**Second**, I lead a key architectural decision under ambiguity

We needed to choose the event source for triggering transfers:

- Option A: Payment event (faster and simpler because of existing webhook infrastructure with built-in retries and event tracking)
- Option B: Program-specific purchase event via Kafka (slower and more complex because I had to build NodeJS Kafka consumer from scratch whilst eBay’s ecosystem is in Java)

Chose Option B: Optimized for correctness and simplicity of business logic over speed and operational ease.

Why correctness mattered more:

- Option A is a finance signal (“payment succeeded”), not the right semantic boundary
- Option B is an event that is already enriched with the needed properties
- Using Option A would mean building enrichment logic inside my service — creating a “shadow enrichment layer” that duplicates existing work

Why simplicity of business logic mattered more:

- Avoiding shadow enrichment keeps the service thin and focused on DPP transfer logic only
- Option A would bloat the service with OMS integration code to reconstruct what the existing enrichment service already computes

The trade off was:
Speed:
- 3-4 weeks building Kafka consumer (connection management, group IDs, offset commits, retry semantics)
- Delayed production launch vs. using existing HTTP webhook

Operability (initially):
- Built from scratch: consumer lag metrics, dead-letter handling, dashboards/alerts, operational runbooks
- No existing battle-tested NodeJS Kafka framework at eBay (Java/Spring was standard)

**Thirdly**, I drove cross-team alignment with:

- Product, to align scope and timelines
- Internal Web3 teams, to align event contracts (wallet, portfoolio)
- External marketplace teams, to align Federated GraphQL schema
- 3rd party (Arianee), to implement blockchain integration

To keep alignment:

I defined clear domain boundaries (what our service owns vs others)
Documented decisions and trade-offs transparently
Created 13+ runbooks so other teams could operate and reuse the system

### Result
Even though the partnership was later paused for commercial reasons, the system delivered strong technical outcomes:

- 100% of eligible transactions processed with zero data loss
- Architecture became the template for the next partnership, reducing their ramp-up from weeks to days
- Consumer lag reduced from 60,000 to 100
- Shared knowledge through company wide Engineering excellance talks about kafka

### Reflection
I realised I treated this as a guaranteed partnership rather than a platform investment with risk.

Next time, I would:
- Surface options earlier to leadership:
- “MVP in 2–3 months vs full system in 6 months”
- Push for a thin experiment first to validate user value before building full infrastructure

## What they are looking for:
- What was it, distill complexity
- Stakeholders, alignment, communication
- Tracking (data) and success
- Ownership of complex work
- Led delivery of work with
    - Ambiguity
    - Multiple stakeholders
    - Meaningful risk
    - Can articulate their decisions, not just the process.

Strong signal
“I realised we were over-committing, so I reset expectations with product.”

--- 
## "Tell me about a time where there was an unforeseen obstacle"
### Situation
eBay launched a digital collectibles platform where users could collect NFT trading cards and redeem them for physical rewards from premium brand partners like Lord of the Rings and Harry Potter. Users had to own specific NFT cards to be eligible for physical reward redemption. Our blockchain indexer tracked NFT ownership to determine eligibility.

### Task
My goal was to ensure 100% data completeness and reliability in production.

However, during QA with another team, we hit an unforeseen issue:

- A user bought an NFT, but it didn’t appear in their portfolio
- It eventually showed up ~1 hour later, which was unacceptable for a real-time product

This exposed a hidden race condition during deployments, creating a 5-second blind spot where events were silently missed.

### Action
**Firstly**, I diagnosed the issue under pressure

I traced the issue to a race condition during deployments. When the indexer restarted, there was a 5-second blind spot between the old instance shutting down its WebSocket connection and the new instance reconnecting. That 5-second window happened to be when the Portfolio team were testing the NFT purchase event and were missed. What's worse is that, our gap-filling logic was reactive — it only triggered when the next event arrived, which took nearly an hour during low blockchain activity. So the missing NFT were invisible for an hour but then it eventually showed up on the Portfolio.

**Secondly**, I communicated early and reset expectations

I immediately told stakeholders (Portfolio + Product) that we have a data correctness issue during deployments where users can temporarily lose visibility of assets

I was explicit about:
- What went wrong (missed events during restart)
- User impact (incorrect ownership view)
- Short-term reality (not safe to launch campaigns yet)

This helped pause downstream assumptions.

**Thirdly**, I made a decision under time pressure

We had two options:

- Fast fix: Use a third-party API (2 days, but no control or debuggability)
- Robust fix: Build proactive gap-filling (2 weeks, but guarantees correctness)

I chose the slower, more robust approach because:
- This system determined real-world rewards and brand trust
- A third-party dependency would make failures unexplainable and unfixable

The tradeoff was:
Time-to-market:
2 weeks implementation vs 2 days with third-party API

Simplicity and operational ease:
Took on significant operational burden: monitoring gap-fill duration, RPC rate limits, startup latency
Added complexity: chunked processing, checkpointing, error recovery, concurrent safety
Needed runbooks for: startup taking >30 seconds, RPC provider outages, rate limit exhaustion
third-party API would have eliminated all of this

**Forthly**, I changed the system design to use proactive gap-filling at startup, instead of reactive:

On restart, the system:
- Checks the last processed blockchain block
- Fetches the current block
- Processes any missing range in chunks before going live

This eliminated the blind spot entirely.

## Result
- Eliminated all data loss during deployments
- Reduced NFT visibility delays from ~1 hour → near real-time
- Ensured 100% correctness for reward eligibility
- Unblocked campaign launches with confidence

## Reflection
Rare Edge Cases Become Common at Scale

2% per deployment × 100 deployments = 2 incidents

Lesson: "Rare" doesn't mean "won't happen" — it means "will happen eventually."

This is applied to future work:
- Design for the 100th deployment, not the 1st
- Automated recovery for "rare" events you can't eliminate

### What they are looking for
- About reaction and management of this, recover and adapt, reflect and learn
- Could be an expansion on q1 and same project
- What was it and what did you do?
- Adaptation to it
- Managing expectations
- How could you anticipate it?
- Decision-making under pressure which handles:
    - Missed assumptions
    - Production incidents
    - Delivery risk
    - Communicates clearly and early.

Strong signal
“Here’s what went wrong, here’s what we told stakeholders, and here’s what changed.”

---

## Tell us about a time where you had to use persuasion

### Situation 
In the Digital Product Passport system, it had multiple complex integrations with APIs, databases, and external services.

The tech lead proposed we prioritise heavy on end-to-end (E2E) test coverage to ensure reliability.

However, I felt this approach would:

- Slow down development significantly
- Be brittle and hard to maintain

So there was a clear disagreement on testing strategy, with real delivery impact.

### Task
My goal was to align the team on a testing strategy that:

- Balanced confidence vs delivery speed
- Scaled with system complexity
- Didn’t create long-term maintenance overhead

Importantly, I wasn’t the tech lead — so this required influencing, not dictating.

### Action
**First** Start with understanding, not arguing

Instead of pushing my idea immediately and shooting down others, I listen and ask:

“Why"
"What do you worry about most, where E2E tests would catch?”

This surfaced that their main concern was:
- Cross-service integration failures in production

That helped me understand their position wasn’t about preference — it was about risk mitigation.

**Secondly** Reframed the problem around shared goals

I aligned us on the actual goal:

“We both want high confidence in production, but also fast iteration speed.”

This shifted the conversation from “which approach is better” → “what combination best achieves both”.

**Thirdly** Proposed an alternative with clear trade-offs

I suggested a testing pyramid approach:

- Lots of fast unit tests
- Some Integration tests
- Few E2E tests for critical paths

I explicitly compared both approaches:

- E2E-heavy:
   - ✅ Have high confidence in full flows
   - ❌ Have the tradeoff of Slow, flaky, expensive to maintain
- Pyramid:
   - ✅ Have faster feedback, easier debugging, scalable
   - ❌ Have the tradeoff of choosing the right E2E coverage

**Fourthly** Used data and examples to persuade

Instead of opinion, I backed it with:

- Examples of flaky E2E pipelines slowing teams down
- Industry best practices (testing pyramid widely adopted)
- Our own system complexity — many external dependencies would make E2E brittle

I also suggested a compromise of:

“Let’s find the most important flow and keep E2E there, not everywhere.”

**Fifthly** Managed the decision collaboratively

Rather than “winning” the argument, I:

- Wrote down both approaches and trade-offs
- Asked the tech lead to challenge my assumptions
- Focused on what best serves the product, not individual preferences

This helped us reach alignment without friction.

## Result
- We adopted a balanced testing pyramid strategy
- Reduced test execution time significantly, improving developer velocity
- Avoided flaky E2E pipelines while still covering critical flows
- The approach scaled well as the system grew

## Reflection
What worked well was:
- Listening first — understanding the root concern (risk, not preference)
- Framing around shared goals rather than competing ideas
- Using data and trade-offs, not opinions
- Keeping ego out of the decision

One thing I’d do even better:

- Run a small proof of concept earlier (e.g. compare test runtimes or flakiness) to make the trade-off even more tangible

### What they are looking for
- Brought people to your way of thinkings. 
- Find root cause and then finding out what matters to those objecting
- Persuade about what?
- Diverse perspectives, did they help to shape it
- Tactics to convince, data and empathy are key, e.g. POC, demo, market research, presentation, spreadsheet, writing, informal chat, literature review, etc.) 
- Stakeholder management beyond updates.
- Proactively manages:
    - Conflicting priorities
    - Pushback
    - Tradeoffs
    - Influences direction, not just execution.

Strong signal
“Product wanted X, but we couldn’t safely deliver it in time, so I proposed Y.”

---

## Tell us about a time where you had to coach and mentor

### Situation
While working on the Digital Product Passport system at eBay, I introduced a Kafka-based architecture where our business unit uses Node.JS but wider eBay organisation used Java.

In the Web3 group, most engineers:
- Had no experience with Kafka or even Kafka in Node.js
- Were starting to build event-driven systems independently
- Risked inconsistent implementations and reliability issues

I realised the problem would repeat across multiple squads unless addressed.

### Task
My goal was to raise the engineering standard across the org, not just solve my team’s problem.

Specifically:
- Enable other engineers to safely adopt Kafka
- Prevent common pitfalls (data loss, poor observability, misconfigured consumers)
- Create a repeatable, scalable approach to learning, not ad-hoc support

### Action
I used a combination of approaches, depending on how people learn:

Structured talk (broad audience):
- Delivered an engineering excellence session to the Web3 org on:
    - Kafka fundamentals
    - Scalability and failure modes
    - Real production lessons (e.g. silent throttling, consumer lag)
- Practical example (hands-on learners):
    - Built a reference repository showing:
    - How to implement a Kafka producer and consumer correctly
    - Patterns for retries, offset management, and observability
    - “What good looks like” in production
- 1:1 support (targeted coaching):
    - Helped engineers applying it in their own services, adapting guidance to their specific use cases

### Result
- 4 teams have successfully adopted Kafka using the patterns I introduced
- Reduced onboarding time from weeks → days for new Kafka integrations
- Prevented common production issues by standardising observability and error handling
- Received inbound engagement — including engineers from the US reaching out for guidance — showing the approach scaled beyond my immediate team

### What we are looking for
- proactive is better, multiple methods best adapt to the persons needs
- Led to it, approach
- Results and learnings
- Mentorship as leverage
- Mentorship is:
    - Intentional
    - Repeatable
    - Outcome-driven
    - Goes beyond onboarding.

Weak L40 signal
Mentorship = helping someone get up to speed once.

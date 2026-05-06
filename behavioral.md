# Behavioral interview
## "Tell me about a time where you led a complex project"

### Situation
I led the delivery of a Digital Product Passport system for eBay in partnership with Breitling, enabling blockchain-verified ownership certificates for luxury watches.

The problem was complex because I was integrating across:
- 6 systems with internal teams, external teams and third-party provider (Wallet team, Portfolio team, Arianee, Federated GraphQL, VAS Events, Kafka)
- No blockchain infra existed internally

There was also had high ambiguity — unclear partner timelines, evolving requirements, and no prior blueprint inside eBay for Node.js.

### Task
I owned end-to-end delivery of the purchase-to-fulfilment flow, specifically:

- Ensuring that when a watch was purchased and authenticated, ownership of the digital passport transferred correctly from seller → buyer
- Designing a system that was correct, reliable, and observable, given this was tied to trust and high-value goods
- Aligning multiple stakeholders across product, Web3, marketplace infrastructure, and external partners

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

- Option A: Payment event (faster, simpler, but wrong abstraction)
- Option B: Program-specific purchase event via Kafka (slower, more complex, but correct)

I chose Option B, even though it delayed us by ~3–4 weeks.

My reasoning:

- Payment success ≠ order completion (risk of incorrect transfers)
- Kafka stream already contained enriched, filtered data
- Avoided rebuilding complex logic (“shadow enrichment”) in our service

I explicitly reset expectations with product:
We are trading short-term delivery for long-term system integrity.

**Thirdly**, I drove cross-team alignment with:

- Product → scope and timelines
- Internal Web3 teams → event contracts (wallet, portfoolio)
- External marketplace teams → Federated GraphQL
- 3rd party (Arianee) → blockchain integration

To keep alignment:

I defined clear domain boundaries (what our service owns vs others)
Documented decisions and trade-offs transparently
Created 13+ runbooks so other teams could operate and reuse the system

### Result
Even though the partnership was later paused for commercial reasons, the system delivered strong technical outcomes:

- 100% of eligible transactions processed with zero data loss
- Consumer lag reduced to <30 seconds under load
- On-call incidents dropped from ~3/week → <1/month
- Architecture became the template for the next partnership, reducing their ramp-up from weeks to days

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
I was responsible for the production reliability of a blockchain indexer at eBay, which tracked NFT ownership for a digital collectibles platform tied to major brand campaigns like The Lord of the Rings and Harry Potter.

This data directly determined whether users could redeem physical rewards, so correctness was critical — even small gaps meant users either couldn’t claim rewards or could claim incorrectly.

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

I immediately told stakeholders (Portfolio + Product):

“We have a data correctness issue during deployments — users can temporarily lose visibility of assets. It self-recovers, but the delay is unacceptable for reward eligibility.”

I was explicit about:
- What went wrong (missed events during restart)
- User impact (incorrect ownership view)
- Short-term reality (not safe to launch campaigns yet)

This helped pause downstream assumptions and avoided a bad user experience.

**Thirdly**, I made a decision under time pressure

We had two options:

- Fast fix: Use a third-party API (2 days, but no control or debuggability)
- Robust fix: Build proactive gap-filling (2 weeks, but guarantees correctness)

I chose the slower, more robust approach.

Reasoning:

This system determined real-world rewards and brand trust
A third-party dependency would make failures unexplainable and unfixable
99% accuracy wasn’t acceptable — we needed 100% correctness

**Forthly**, I adapted the system design to replace reactive recovery with proactive gap-filling at startup:

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

Applied to future work:

- Calculate expected failure rates over time, not per event
- Design for the 100th deployment, not the 1st
- Automated recovery for "rare" events you can't eliminate
- Monitor for patterns across deployments, not just individual failures

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

Q3. Persuasion
- Brought people to your way of thinkings. 
- Find root cause and then finding out what matters to those objecting
- Persuade about what?
- Diverse perspectives, did they help to shape it
- Tactics to convince, data and empathy are key, e.g. POC, demo, market research, presentation, spreadsheet, writing, informal chat, literature review, etc.) 

Stakeholder management beyond updates.
Proactively manages:
- Conflicting priorities
- Pushback
- Tradeoffs
- Influences direction, not just execution.
Strong signal
“Product wanted X, but we couldn’t safely deliver it in time, so I proposed Y.”

Q4. Coaching and mentoring, 
- proactive is better, multiple methods best adapt to the persons needs
- Led to it, approach
- Results and learnings

Mentorship as leverage
Mentorship is:
- Intentional
- Repeatable
- Outcome-driven
- Goes beyond onboarding.
Weak L40 signal
Mentorship = helping someone get up to speed once.

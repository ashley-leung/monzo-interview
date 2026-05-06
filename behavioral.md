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
First, I distilled complexity into a simple model by defining the rule of:

“Only transfer ownership when two signals are true: purchase complete + physical authentication complete.”

This helped align stakeholders and guided architecture decisions.

---

Second, I lead a key architectural decision under ambiguity

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

---

Thirdly, I drove cross-team alignment with

Product → scope and timelines
Internal Web3 teams → event contracts (wallet, portfoolio)
External marketplace teams → Federated GraphQL
3rd party (Arianee) → blockchain integration

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


```
What they are looking for:
- What was it, distill complexity
- Stakeholders, alignment, communication
- Tracking (data) and success

Ownership of complex work.
Led delivery of work with:
- Ambiguity
- Multiple stakeholders
- Meaningful risk
- Can articulate their decisions, not just the process.
Strong signal
“I realised we were over-committing, so I reset expectations with product.”
```
Q2. Unforeseen obstacle
- About reaction and management of this, recover and adapt, reflect and learn
- Could be an expansion on q1 and same project
- What was it and what did you do?
- Adaptation to it
- Managing expectations
- How could you anticipate it?

Decision-making under pressure.
Handles:
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

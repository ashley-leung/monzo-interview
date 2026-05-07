# Behavioral interview
## "Tell me about a time where you led a complex project"

#### Situation
eBay partnered with Breitling, a luxury Swiss watch manufacturer to enable sellers to attach a blockchain-verified Digital Product Passport to their listings, allowing sellers to transfer ownership digitally to buyers after purchase.


#### Task
I owned end-to-end delivery of the purchase-to-fulfilment flow, specifically:

- Ensuring that when a watch was purchased, the digital passport transferred correctly from seller to buyer
- Designing a system that was correct, reliable, and observable

#### Action
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

#### Result
Even though the partnership was later paused for commercial reasons, the system delivered strong technical outcomes:

- 100% of eligible transactions processed with zero data loss
- Architecture became the template for the next partnership, reducing their ramp-up from weeks to days
- Consumer lag reduced from 60,000 to 100
- Shared knowledge through company wide Engineering excellance talks about kafka

#### Reflection
I realised I treated this as a guaranteed partnership rather than a platform investment with risk.

Next time, I would:
- Push for a thin experiment first to validate user value before building full infrastructure
- Surface options earlier to leadership:
- “MVP in 2–3 months vs full system in 6 months”


### What made this project complex rather than just difficult?

The problem was complex because I was integrating across:
- 6 systems with internal teams, external teams and third-party provider (Wallet team, Portfolio team, Arianee, Federated GraphQL, VAS Events, Kafka)
- No blockchain infra existed internally
- High ambiguity — unclear partner timelines, evolving requirements, and no prior blueprint inside eBay for Node.js.

### How did you break the work down?
The biggest challenge was that this was effectively a greenfield blockchain platform inside eBay, with multiple external and internal dependencies and a hard launch deadline tied to the Breitling partnership announcement.

So I broke the work down into four major streams:
- Core transaction lifecycle — defining how a DPP moves from seller to buyer safely
- Platform integrations — Kafka purchase events, AG authentication events, wallet services, GraphQL federation
- Blockchain orchestration — Arianee integration, custodial wallets, NFT transfer flow
- Operational readiness — observability, runbooks, deployment tooling, replay/backfill capability

I intentionally prioritised the event flow and idempotency first because those were the highest-risk areas. If duplicate transfers occurred or events were lost, we could create ownership inconsistencies on-chain which would be very difficult to recover from.

Once the core lifecycle was stable, I focused on the read path through GraphQL federation and then production hardening.

### What dependencies existed?
Internally, we depended on:
- Purchase events service via Kafka
- Authenticity Guarantee systems for physical authentication events
- GraphQL Federation for listing enrichment
- Wallet and Portfolio teams for buyer-facing NFT visibility

The complexity was that these systems all operated asynchronously and were owned by different teams with different priorities and release cadences.

### How did you prioritise what mattered?
- Steel thread
- Listing flow, Purchase flow working first
- Edge cases such as return flow

### What parts were most ambiguous?
- We wanted to display a badge on listing that says "Has DPP" and to do that, we had to become a provider for eBay's Federated GraphQL, which meant that I had to design the GraphQL schema to enrich a listing with additional data.
- At the beginning, too much additional data such as color and material of watch, when the DPP was minted
- In the end, I choose a simple property of "HasDigitalProductPassport" to keep it simple

### How did you know your approach was the right one?
- Single responsibility principle
- Knowing that option A would bloat the service with OMS integration code to reconstruct what the existing enrichment service already computes
- Keeping the service thin and focused on DPP transfer logic only

### What constraints were you operating under?
- Time pressure and commercial uncertainty
- Partnership timelines and leadership expected us to be ready, so development had to move quickly and show result. At the same time, we didn't know if Breitling would reliably generate transfer links in production which meant we were investing in platform-level capabilities under real uncertainty about whether the partner would actually launch on the original timeline

### What were the unknowns at the start?
- No signed agreement between Breitling and eBay, operating under uncertainty 
- eBay has a distributed architecture with many services but no clear documentation of which services does what, lots of investigation into which service is needed

### What was your role specifically?
- The architecture had 2 parts, listing flow and purchase flow. Starting with the listing flow, we have a Typescript NestJS service that receives a HTTP webhook (BES) if a new Breitling listing was created on the eBay marketplace. Then, we would create a DB record for that listing for further processing in the purchase flow. We then send an email to the seller giving them instructions to try and find their watch's DPP. If they found it, then they would upload it to our RemixJS UI, part of the Web3 subdomain within eBay. We transfer the DPP custodial ownership to us whilst it's being listed on eBay and complete the listing flow by showing a badge on that listing, giving buyers trust to say that this listing has an "Official Digital Passport".
- For the purchase flow, I have a Kafka consumer within the same Typescript NestJS service that subscribes to checkout success events and filter those events to see if the same Brietling watch listing that was stored in the DB from earlier have been purchased. Together with the Kafka purchased event and the HTTP webhook (BES) authentic event from eBay's physical authenticator, known as the AG centers, the transfer of the DPP from seller to buyer occurs. Finally, we notify the Portfolio service via HTTP so that buyers can view and manage their DPPs through their eBay account.
- My role was end-to-end ownership of the purchase-to-fulfillment flow, which was specifically:
    - Architecting the event-driven DPP transfer flow (Kafka consumer + HTTP Webhook BES event handling)
    - Building Kafka consumer infrastructure with LZ4 compression and offset management
    - Building the Arianee blockchain service integration with RS256 JWT authentication
    - Designing and implementing the Transfer entity schema in eBay's mongo DB equivalent
    - Implementing GraphQL Federation subgraph provider to enrich listing data with DPP badges
    - Writing 13+ runbook documentation pages for production support and incident response

### Who were the stakeholders?
There were several stakeholder groups:
- Product cared about launching in time for the Breitling partnership announcement and minimising buyer friction.
- The Authenticity Guarantee team cared about fraud prevention and ensuring blockchain ownership only transferred after physical verification.
- The Federated GraphQL teams cared about federation performance and maintaining low-latency listing enrichment.
- The Wallet and Portfolio teams cared about buyer experience after transfer.
- Infrastructure teams cared about Kafka throughput, quota management, and operational reliability.
- Arianee, as the external blockchain vendor, cared about API integration correctness and wallet lifecycle management.

### Who disagreed and why?
One major disagreement was around when ownership transfer should happen.

Product initially preferred transferring the DPP immediately after purchase because it created a more seamless buyer experience.

I disagreed with that approach because the physical watch still needed to go through eBay’s Authenticity Guarantee process. If we transferred the NFT before authentication and the item later failed verification, we’d end up with inconsistent physical and digital ownership.

I proposed splitting the flow into two stages:
- purchase acknowledgment via Kafka
- fulfillment only after AG authentication

That added architectural complexity and asynchronous coordination, but significantly reduced fraud risk and avoided difficult rollback scenarios on-chain.

### How did you track progress?
I tracked progress at two levels: delivery milestones and business.

Success meant:
- Zero incorrect ownership transfers
- No data loss across asynchronous systems
- Production readiness with clear observability and runbooks

For delivery, I broke the launch into integration milestones:
- Kafka purchase ingestion
- AG authentication handling
- Arianee wallet integration
- NFT transfer orchestration
- GraphQL federation
- production readiness

For business, the number of listings that has a digital product passport uploaded

### What could have gone wrong?
There were several meaningful failure modes.

The highest-risk scenario was inconsistent ownership state between the physical watch and the blockchain NFT.

For example:
- duplicate NFT transfers
- transferring ownership before authentication
- lost Kafka events
- replaying events incorrectly

Any of those could create disputes over ownership that are much harder to unwind on-chain than in traditional systems.

--- 
## "Tell me about a time where there was an unforeseen obstacle"
#### Situation
eBay launched a digital collectibles platform where users could collect NFT trading cards and redeem them for physical rewards from premium brand partners like Lord of the Rings and Harry Potter. Users had to own specific NFT cards to be eligible for physical reward redemption. Our blockchain indexer tracked NFT ownership to determine eligibility.

#### Task
My task was to ensure 100% data completeness and reliability in production.

But, during QA testing with the Portfolio team, a bug occurred where:

- A user bought an NFT, but it didn’t appear in their portfolio
- It eventually showed up ~1 hour later

#### Action
**Firstly**, I traced the issue to a race condition during deployments. When the indexer restarted, there was a 5-second blind spot between the old instance shutting down its WebSocket connection and the new instance reconnecting. That 5-second window happened to be when the Portfolio team were testing the NFT purchase event and were missed. What's worse is that, our gap-filling logic was reactive — it only triggered when the next event arrived, which took nearly an hour during low blockchain activity. So the missing NFT were invisible for an hour but then it eventually showed up on the Portfolio.

**Secondly**, I immediately told stakeholders (Portfolio + Product) that we have a data correctness issue during deployments where users can temporarily lose visibility of assets

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

#### Result
- Eliminated all data loss during deployments
- Reduced NFT visibility delays from ~1 hour → near real-time
- Ensured 100% correctness for reward eligibility
- Unblocked campaign launches with confidence

#### Reflection
Rare Edge Cases Become Common at Scale

2% per deployment × 100 deployments = 2 incidents

Lesson: "Rare" doesn't mean "won't happen" — it means "will happen eventually."

This is applied to future work:
- Design for the 100th deployment, not the 1st
- Automated recovery for "rare" events you can't eliminate

### What made this difficult to diagnose?
The issue was difficult because the system appeared healthy most of the time.

Deployments completed successfully, the WebSocket reconnected correctly, and eventually the missing NFTs would appear after reconciliation triggered.

So there was no obvious permanent failure.

The real issue only appeared if three conditions aligned:
- an event occurred during deployment restart
- blockchain activity was low afterwards
- someone checked portfolio state before reconciliation triggered

That made it intermittent and timing-dependent, which is why it initially looked like a possible third-party discrepancy rather than a systemic flaw in our own architecture.

### How did you react once you realised it was your system?
Initially there was uncertainty about whether third party API or our indexer was wrong.

Once I validated directly against blockchain events, it became clear our system had dropped data.

At that point I focused less on defending the implementation and more on understanding the blast radius and recovery strategy.

My immediate priorities became:
- quantify how widespread the issue was
- determine whether data loss was recoverable
- prevent future missed events
- communicate the risk clearly to stakeholders

I think one important lesson was being willing to challenge our own assumptions quickly instead of assuming the external provider was incorrect.

### What did you communicate to stakeholders?
I communicated the issue in layers depending on the audience.

For engineering teams, I explained the exact race condition and reconciliation gap.

For product stakeholders, I framed it in terms of customer impact:
- some users could temporarily appear ineligible for rewards
- ownership data would eventually self-heal
- we had no evidence of false positives or permanent corruption

I also communicated uncertainty clearly. At that stage we didn’t yet know the full blast radius, so I avoided overpromising until we completed historical analysis.

Once I confirmed we could backfill missing events safely from blockchain history, expectations became much easier to manage.

### How did you determine the blast radius?
After identifying the restart race condition, I wanted to understand whether this was a one-off or systemic.

So I analysed historical deployments and looked for blockchain events occurring near restart windows.

---

## Tell us about a time where you had to use persuasion

#### Situation 
In the Digital Product Passport system, it had multiple complex integrations with APIs, databases, and external services.

The tech lead proposed we prioritise heavy on end-to-end (E2E) test coverage to ensure reliability.

However, I felt this approach would:

- Slow down development significantly
- Be brittle and hard to maintain

So there was a clear disagreement on testing strategy, with real delivery impact.

#### Task
My goal was to align the team on a testing strategy that:

- Balanced confidence vs delivery speed
- Scaled with system complexity
- Didn’t create long-term maintenance overhead

Importantly, I wasn’t the tech lead — so this required influencing, not dictating.

#### Action
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

**Thirdly** Proposed my option with clear trade-offs

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

#### Result
- We adopted a balanced testing pyramid strategy
- Reduced test execution time significantly, improving developer velocity
- Avoided flaky E2E pipelines while still covering critical flows
- The approach scaled well as the system grew

#### Reflection
What worked well was:
- Listening first — understanding the root concern (risk, not preference)
- Framing around shared goals rather than competing ideas
- Using data and trade-offs, not opinions
- Keeping ego out of the decision

One thing I’d do even better:

- Run a small proof of concept earlier (e.g. compare test runtimes or flakiness) to make the trade-off even more tangible

---

## Tell us about a time where you had to coach and mentor

#### Situation
While working on the Digital Product Passport system at eBay, I introduced a Kafka-based architecture where our business unit uses Node.JS but wider eBay organisation used Java.

In the Web3 group, most engineers:
- Had no experience with Kafka or even Kafka in Node.js
- Were starting to build event-driven systems independently
- Risked inconsistent implementations and reliability issues

I realised the problem would repeat across multiple squads unless addressed.

#### Task
My goal was to raise the engineering standard across the org, not just solve my team’s problem.

Specifically:
- Enable other engineers to safely adopt Kafka
- Prevent common issues (data loss, poor observability, misconfigured consumers)
- Create a repeatable, scalable approach to learning, not ad-hoc support

#### Action
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

#### Result
- 4 teams have successfully adopted Kafka using the patterns I introduced
- Reduced onboarding time from weeks → days for new Kafka integrations
- Prevented common production issues by standardising observability and error handling
- Received inbound engagement — including engineers from the US reaching out for guidance — showing the approach scaled beyond my immediate team

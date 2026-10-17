# Growth playbooks

> **Audience:** Founders, team.
> **Purpose:** Concise, actionable playbooks for acquiring, activating, and retaining event agencies. One playbook = one focused motion. No theory, no filler.

---

## Principles

1. **The pitch board is the product demo.** Every board sent to a client is a passive acquisition channel.
2. **Concierge first.** Empty catalog = empty value. We import the first venues for the agency.
3. **One city, one ICP.** Small–mid agencies (5–50 people). Fast decision, high word-of-mouth density.
4. **Measure approvals, not signups.** North-star metric: pitch boards approved per team per month.
5. **No paid acquisition until 10 paying agencies.** Direct + community only.

---

## Playbook 1 — First 10 paying agencies (Cold start)

**Goal:** 10 paying agencies in one target city (Naples / Austin / Nashville).

**Motion:** Direct outreach + white-glove onboarding.

| Step | Action                                                                                                                                          | Owner         | Success signal                    |
| ---- | ----------------------------------------------------------------------------------------------------------------------------------------------- | ------------- | --------------------------------- |
| 1    | Build list of 30 agencies (LinkedIn + local directories + MPI/ILEA chapters)                                                                    | Founder       | 30 qualified contacts             |
| 2    | Personalized outreach: "I built a tool that turns your venue files into a client pitch board in 5 minutes — want to try it on your next brief?" | Founder       | 8–12 replies                      |
| 3    | 15-min call → offer free import of first 20 venues                                                                                              | Founder       | 6–8 accept                        |
| 4    | Concierge import + first pitch board generated for a real brief                                                                                 | Founder + eng | Agency sends board to real client |
| 5    | Follow up after first client interaction; convert to paid                                                                                       | Founder       | 10 paying                         |

**Talking points that work:**

- Time to first pitch board (minutes, not hours)
- Client can approve the same day
- Immutable record of what was agreed
- Knowledge stays when a planner leaves

**Do not:**

- Pitch the catalog as the main value
- Ask for a long trial without a real brief
- Promise extraction accuracy before measuring it

---

## Playbook 2 — Activation (Day 0 → first approval)

**Goal:** Agency reaches first client approval within 14 days of signup.

**Sequence:**

| Day | Trigger                    | Action                                                                        |
| --- | -------------------------- | ----------------------------------------------------------------------------- |
| 0   | Signup                     | Concierge or self-serve: import ≥ 5 venues (or use master-catalog backdrop)   |
| 1   | First venue ready          | In-app prompt: "Generate your first pitch board"                              |
| 3   | No pitch yet               | Email + in-app: "Pick a real brief this week and send a board"                |
| 7   | Pitch sent, no client open | Nudge: "Client hasn't opened the board yet — want a short reminder template?" |
| 14  | Still no approval          | Personal check-in: "What blocked the approval?"                               |

**Activation metric:** % of new agencies that generate ≥ 1 pitch board and receive ≥ 1 client open within 7 days.

**Kill criterion:** If < 40% of agencies reach first pitch in 14 days → improve onboarding / empty-state UX before scaling acquisition.

---

## Playbook 3 — Organic / community seeding (10 → 50 agencies)

**Goal:** Sustainable inbound without paid ads.

**Channels (ranked by expected leverage for this ICP):**

1. **Industry communities**
   - MPI, PCMA, ILEA local chapters
   - EventProfs Slack / Discord
   - LinkedIn groups for event planners  
     Action: share real before/after (hours saved, client reaction to board). No hard sell.

2. **Case-study content**
   - "How [Agency] closed a brief in 4 hours instead of 2 days"
   - Publish on BizBash / Skift Meetings / EventMB after first 3–5 testimonials
   - Format: numbers + screenshot of board (anonymised if needed)

3. **Product Hunt**
   - Only after 10 paying customers + real testimonials
   - Lead with the client-side experience (the board itself)

4. **Referral**
   - 1 free month for every paying referral that converts
   - Make it visible in the agency dashboard after first approval

**Content rule:** Always show the pitch board. Never lead with "AI extraction" or "venue library".

---

## Playbook 4 — Viral loop (built-in)

**Mechanism:** Every free-tier pitch board carries a small "Powered by Shortlisty" footer.

**Loop:**
Agency A sends board → Client sees Shortlisty → Client asks their other agencies "what tool is this?" → Agency B hears about it from a client.

**Amplifiers:**

- Make the board visually premium (agency is proud to send it)
- Keep free tier useful enough that agencies actually send boards
- Track: % of new signups that mention "saw it on a pitch" or "client asked about it"

**Do not remove the footer on free tier.** It is the primary viral surface.

---

## Playbook 5 — Retention & expansion

**Retention drivers:**

- Catalog becomes the team's shared knowledge base (switching cost rises)
- Junior planners rely on it to answer questions without asking seniors
- Approved snapshots are used in post-event discussions

**Tactics:**

| Trigger                      | Action                                                                                  |
| ---------------------------- | --------------------------------------------------------------------------------------- |
| 30 days, low usage           | Personal email: "Which brief are you working on this month? Happy to help load venues." |
| First approval               | Celebrate + ask for short testimonial                                                   |
| Team size grows              | Prompt upgrade to Business ($300) for white-label                                       |
| Storage / branding limit hit | Natural upgrade moment                                                                  |

**Churn risk signals:**

- No pitch board generated in 30 days
- Only one user active in a multi-person agency
- Catalog stays under 10 venues

**Response:** Concierge re-engagement (offer to import more venues or generate a board for an upcoming brief).

---

## Playbook 6 — Pricing & packaging experiments

Current:

- Pro $150/mo — full catalog + pitch boards + unlimited team
- Business $300/mo — white-label + priority + API

**Experiments (run after 20+ paying):**

1. Annual prepay discount (2 months free)
2. "Founding agency" lifetime discount for first 25 (creates urgency + advocates)
3. Usage-based add-on only if AI costs become material (prefer not to)

**Rule:** One approved deal must clearly pay for multiple months. Keep pricing simple.

---

## Measurement dashboard (minimum)

| Metric                                 | Cadence   | Owner   |
| -------------------------------------- | --------- | ------- |
| Paying agencies                        | Weekly    | Founder |
| Pitch boards created                   | Weekly    | Product |
| Pitch boards approved                  | Weekly    | Product |
| Activation rate (first pitch ≤ 14d)    | Bi-weekly | Growth  |
| Referral / "saw it on a board" signups | Monthly   | Growth  |
| Logo retention (logo-level)            | Monthly   | Founder |

North star remains: **approved pitch boards per team per month**.

---

## Playbook 7 — Build in public / open-core (parallel developer channel)

**Goal:** Build a developer community around the open-source document intelligence layer, creating a second acquisition surface that runs in parallel with the agency-direct motion and compounds over time.

**Positioning:** The IQKV platform is already open source. The promise is an open-core system for document intelligence — extracting and reconciling information across PDFs, scans, tables, and emails. This is hard infrastructure that event agencies benefit from, but its audience is broader: any team that drowns in unstructured documents.

**Reference projects:** [Papermark](https://github.com/mfts/papermark) (open-core line, doc sharing → analytics), [Postiz](https://github.com/gitroomhq/postiz-app) (founder-led social content cadence), [Kestra](https://github.com/kestra-io/kestra) (community flywheel around pipeline/workflow infra), [Plausible](https://github.com/plausible/analytics) (open-core SaaS, privacy-first alternative positioning), [Documenso](https://github.com/documenso/documenso) (open-source doc signing, community-led trust building), [Cal.com](https://github.com/calcom/cal.com) (aggressive open-core with hosted tier, rapid GitHub star growth), [Dify / LangGenius](https://github.com/langgenius/dify) (open-source LLM app platform, developer community around AI pipelines), [Fleetbase](https://github.com/fleetbase/fleetbase) (open-core logistics and supply-chain platform, modular extension ecosystem), [Paymenter](https://github.com/paymenter/paymenter) (open-source billing and client portal for hosting providers, self-hosted SaaS alternative).

**Motion:** Ship visibly → narrate publicly → let the repo pull inbound.

| Step | Action                                                                                            | Owner   | Success signal                             |
| ---- | ------------------------------------------------------------------------------------------------- | ------- | ------------------------------------------ |
| 1    | Publish IQKV with a clear README: what it solves, what's open, where the proprietary line will be | Founder | Repo is linkable and readable              |
| 2    | First build-in-public post: "Why we open-sourced our document intelligence layer" (X + LinkedIn)  | Founder | Post circulates; first GitHub stars        |
| 3    | Weekly content: one real extraction problem solved, one lesson learned, one before/after          | Founder | Consistent impressions; community DMs      |
| 4    | Reddit seeding: r/selfhosted, r/MachineLearning, r/devops — answer real questions, link naturally | Founder | Inbound repo traffic from Reddit           |
| 5    | Hacker News Show HN when repo has ≥ 100 stars and a working demo                                  | Founder | Front page or top comments; spike in stars |
| 6    | Developer signups convert to Shortlisty waitlist or paid when they have the event agency use case | Founder | Cross-channel signups mentioning IQKV      |

**Content rule:** Always lead with the problem (reconciling conflicting data across 40 venue PDFs). Never lead with "AI" or model names. Show the output, not the architecture.

**Channels (ranked by expected leverage):**

1. **GitHub** — the repo itself is the primary surface; README quality and issue responsiveness drive stars and forks
2. **X (Twitter)** — build-in-public content; short loops, screenshots, raw numbers
3. **LinkedIn** — same content, slightly longer form; reaches event industry adjacent audience
4. **Hacker News** — Show HN for milestone releases; high signal-to-noise for infrastructure tools
5. **Reddit** — r/selfhosted, r/devops, r/MachineLearning; answer first, link second
6. **Facebook / industry media** — lower priority; revisit after traction on the channels above

**Open-core line:** Not yet drawn. This is a required decision before the first public announcement. Key question: what stays open (extraction engine? reconciliation logic?) and what stays proprietary (multi-source aggregation, tenant isolation, master catalog merge, pitch board output)?

**What this playbook is not:**

- A replacement for Playbooks 1–3. Agency acquisition stays direct and manual.
- A paid acquisition channel. No spend until organic traction is proven.
- A community management commitment. One founder, one cadence — sustainable > comprehensive.

**Metrics:**

| Metric                                   | Cadence | Owner   |
| ---------------------------------------- | ------- | ------- |
| GitHub stars                             | Weekly  | Founder |
| Inbound signups mentioning IQKV / GitHub | Monthly | Founder |
| Reddit / HN referral traffic             | Monthly | Founder |
| Developer → agency conversion (if any)   | Monthly | Founder |

**Kill criterion:** If GitHub stars stall under 50 after 3 months of consistent content and the inbound is zero — pause the channel, reassess the open-core line, not the content cadence.

---

## Open questions (to resolve)

- Which city is the true cold-start market? (density of agencies + willingness to try new tools)
- How strong is the client-side viral effect in practice? (measure after 50 boards sent)
- Is white-label the main upgrade driver, or is it priority processing / API?
- Should we offer a free "viewer" seat for junior coordinators, or keep unlimited team on Pro?
- What is the open-core line for IQKV? (what stays open, what stays proprietary — must be resolved before first public announcement)
- Does the developer audience convert to paying agencies, or is it an entirely separate funnel? (measure after first 6 months of build-in-public)

---

**Docs:** [What is Shortlisty?](../README.md) · [Market structure](market.md) · [Business proposal](digital-sales-room-for-events/proposal.md) · [Sales messaging](sales-messaging.md) · [Vision](../roadmap/vision.md)

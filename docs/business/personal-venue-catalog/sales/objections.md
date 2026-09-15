# Objections

> [!NOTE]
> Personal Venue Catalog segment reference. Not the primary direction — see [Digital Sales Room](../../../digital-sales-room-for-events/README.md).

> **Audience:** Founders, team.
> **Purpose:** Common objections heard in sales conversations and how to handle them. Each entry has the underlying concern, the response, and the follow-up that moves the conversation forward.

---

## How to use these

An objection is almost never what it sounds like on the surface. The response column addresses the real concern underneath the stated one. Read the underlying concern first — if you misread it, the response will land wrong.

Never rebut directly. Acknowledge, reframe, then advance.

---

## Objections by category

### On AI accuracy

---

**"How accurate is the extraction?"**

Underlying concern: _I will trust my client's event to data the AI got wrong and look unpredished._

Response: Accuracy varies by document quality — clean text-based PDFs extract at high confidence; scanned or design-heavy decks extract with lower confidence on some fields. Shortlisty shows you a confidence score on every field and cites the exact page it came from. You can verify anything with one click and override it permanently. We are not asking you to trust the AI blindly — we are showing you exactly what it knows and how sure it is.

Follow-up: "Would it help to run one of your actual venue decks right now so you can see what the confidence scores look like on a real document?"

---

**"What happens when it gets something wrong?"**

Underlying concern: _Errors will silently propagate and I won't catch them._

Response: The system surfaces uncertainty rather than hiding it. Low-confidence fields are flagged visually. When two documents about the same venue give different values, Shortlisty shows the conflict and asks you to resolve it — it does not silently pick one. Every override you make is stored as a permanent correction and applied going forward.

Follow-up: "The confidence model means the system tells you where to look, not where to trust blindly. Want to see a conflict resolution example?"

---

**"Our venue decks are messy — scanned PDFs, design-heavy layouts, tables inside images."**

Underlying concern: _My documents are harder than the ones in your demo._

Response: That is exactly the use case we built for. Clean PDFs are easy — any tool handles them. The value is in messy, real-world documents. Scanned PDFs go through OCR. Design-heavy layouts with multi-column text are handled by the layout-aware parser. Complex tables are reconstructed row by row. Confidence scores will be lower on difficult documents, but the extraction still runs and the results are still searchable.

Follow-up: "Send me one of your hardest venue decks. We'll run it and show you the output before you commit to anything."

---

### On switching cost and adoption

---

**"We already have everything in Google Drive / Dropbox / SharePoint."**

Underlying concern: _Moving files is effort I don't have time for, and I'll lose what I have._

Response: You don't move anything. Your existing storage stays exactly where it is. You upload to Shortlisty the venue files that matter — PDFs, floor plans, spec sheets — and it reads them and builds them into a structured venue portfolio your whole team can search. Most teams start with their top 20 or 30 venues and expand from there. We can do that first import for you as part of onboarding. The key difference: Drive stores files; Shortlisty manages a portfolio. You can't ask Drive "which of our venues has kosher catering and a freight entrance" and get an answer in five seconds.

Follow-up: "How many venues do you actively work with? We can have those in your portfolio within a day."

---

**"We already have Aisle Planner / Planning Pod / HoneyBook. Why another tool?"**

Underlying concern: _I already pay for and live inside an all-in-one event platform. Adding another SaaS feels like duplicate effort, another login, another place data can get out of sync — I've been burned by tool sprawl before._

Response: Totally fair question — we hear it from every agency that's already invested in one of those tools. The short answer: Shortlisty does not replace Aisle Planner or Planning Pod. It plugs into them and solves the one thing none of them do well — the 1–3 hours before the venue is confirmed, when you're hunting through Drive folders and old emails for venue specs, copy-pasting data into Canva or Qwilr, and chasing client feedback in WhatsApp. Those tools are brilliant at contracts, invoicing, timelines, BEOs, seating, and day-of project management. Shortlisty is brilliant at: reading all your venue decks automatically, giving you a searchable knowledge base of every venue you've ever worked with, turning 4 venues into a branded client link in 60 seconds, and getting a frictionless sign-off with a visible history. Then, when the client clicks Approve, we sync the approved venue snapshot — contact, capacity, catering policy, floor plans, client preferences, budget line items, timeline anchors — straight into Aisle Planner / Planning Pod / HoneyBook with one click. No retyping. No data drift. You use the all-in-one for everything downstream, exactly as you do today. Shortlisty just makes the upstream venue-selection loop 10x faster and 10x more pleasant.

Follow-up: "Walk me through what your current venue-selection-to-Approved flow looks like end to end. I want to see exactly where the friction is right now and whether Shortlisty takes it out without touching the parts that already work for you."

---

**"My team won't adopt a new tool."**

Underlying concern: _I've bought software before that nobody used._

Response: The adoption question is real and worth taking seriously. Shortlisty's answer is the search result. The moment a junior planner answers a client question from the library without asking a senior colleague, they use it again. The value is immediate and individual — it does not require the whole team to adopt it simultaneously for the first person to get value.

Follow-up: "Who on your team spends the most time hunting for venue details? Start with them. If they find it useful in a week, the rest of the team follows."

---

**"We're too small to need this."**

Underlying concern: _The problem isn't bad enough to justify paying for a solution._

Response: The smaller the team, the harder the knowledge concentration problem. A five-person agency where one planner holds all the venue knowledge is more exposed than a twenty-person agency with some redundancy built in. If that person is unavailable when a client calls, the agency looks unprepared. Beyond the personnel risk: small agencies often have a tighter, more curated venue portfolio than large ones — and that portfolio is a real competitive asset. Keeping it in a shared Drive folder with no structure means you're underusing it every day. Shortlisty Intelligence is a $49/month way to turn that portfolio into something your whole team can search and build on.

Follow-up: "Has there ever been a time when a client asked something and nobody could find the answer quickly?"

---

**"We don't have time to set it up."**

Underlying concern: _The onboarding cost will exceed the value, at least in the short term._

Response: Setup is: create an account, create a venue, upload a PDF. That's it. The free tier gets you to ten venues with no credit card. The concierge onboarding offer means we can do the first batch of imports for you — you send us the files, we handle the upload. First value in under an hour.

Follow-up: "What would 'set up' look like for you? Let's figure out if there's a smaller starting point."

---

### On data and security

---

**"I'm not comfortable uploading client-sensitive venue documents to a third-party platform."**

Underlying concern: _My clients trust me with confidential information. If it leaks, that's my relationship on the line._

Response: Venue data is isolated per account — no other customer can see your library. Documents are encrypted in transit and at rest. Access is role-based: you control exactly who on your team can see what. For enterprise customers, we offer Azure OpenAI processing which keeps documents within a defined data region rather than passing them through the standard OpenAI API.

Follow-up: "What's your current policy for storing venue PDFs? Most teams have them in Google Drive or email, which have far weaker isolation guarantees than Shortlisty."

---

**"We sign NDAs with our clients. What happens if venue information leaks from your platform?"**

Underlying concern: _An NDA creates a legal obligation — a data breach is not just embarrassing, it is potentially a breach of contract with real consequences._

Response: The concern is legitimate and we take it seriously. Venue files are encrypted at rest and in transit, processed in isolated tenant schemas — no cross-tenant access is architecturally possible — and extracted data never leaves your account. AI extraction calls go to OpenAI's API under their enterprise data processing terms, which explicitly exclude customer data from model training. For agencies with explicit NDA obligations, the Enterprise tier includes Azure OpenAI processing with a signed Data Processing Agreement and defined data residency, so you can demonstrate to a client exactly where their information is processed and confirm it is not used outside your account.

Follow-up: "What does your NDA actually require? Is it data residency, a signed DPA, or just the ability to say data is not shared or used for training? Let's match the right tier to your legal obligations."

---

**"What does the AI do with our documents?"**

Underlying concern: _Our documents are being used to train a model we don't control._

Response: Documents are sent to OpenAI's API for extraction. Under OpenAI's data processing terms for API customers, data submitted via the API is not used to train their models by default. We do not store the raw API payloads beyond processing. The extracted structured data lives in your account only. Enterprise customers can opt for Azure OpenAI processing for explicit data residency guarantees.

Follow-up: "I can send you our data processing summary if you want to share it with your legal or compliance team."

---

**"What if AI platforms like this get replaced by something bigger — Claude, GPT — tomorrow? Why build a workflow around a tool that might disappear?"**

Underlying concern: _I've seen tools get wiped out overnight by a new AI release. I don't want to depend on something that fragile._

Response: The AI is the extraction engine — it reads venue PDFs and pulls out structured fields. The product is the structured knowledge base, the shared team library, the pitch board, and the approval record. Those things have value regardless of which model does the extraction. If a better model ships tomorrow, we swap in the better model and your venue library gets more accurate — nothing breaks, nothing disappears. The risk is not "Claude replaces event management platforms." The risk is building your workflow on tools that are only useful if AI stays exactly as it is today. Shortlisty is designed the other way: AI improves, and the product gets better with it.

Follow-up: "What's the actual workflow you'd lose if you had to stop using Shortlisty tomorrow? That's the question to stress-test — is the value in the AI, or in the structured venue knowledge and the client approval record you've built up?"

---

**"We need a personal touch with clients, especially for high-value events. AI makes things feel impersonal."**

Underlying concern: _My clients — especially wedding clients — chose me because they trust me as a person, not because I have the best software. I don't want a tool that makes me feel like a machine._

Response: Nothing in Shortlisty sends anything to your client automatically. Every message the client receives is written and sent by you. What the AI does is invisible to the client — it reads your PDFs and structures the data so you have everything at your fingertips when you sit down to write that personal note or pick up the phone. The pitch board the client receives is branded with your agency, written in your voice, and represents your judgment about which venues fit their event. The personal touch is not just present — it is sharper, because you spent ten minutes on the brief instead of forty-five digging through files.

Follow-up: "Think about your last high-value brief. How much time did you spend finding and assembling the venue information versus actually thinking about what was right for that client? Which part do you want more time for?"

---

**"We have a GDPR / data residency requirement."**

Underlying concern: _I need to be able to demonstrate to our DPO or legal team where data lives._

Response: The platform is built with multi-tenant isolation and data residency hooks from day one — this was a design requirement, not an afterthought. Enterprise tier includes Azure OpenAI processing (EU data region available), explicit DPA contracts, and right-to-erasure support. For free and Pro tiers, data is processed via standard OpenAI API under their EU SCCs.

Follow-up: "What's the specific requirement — data residency, a signed DPA, or something else? Let's work out whether Pro or Enterprise is the right fit."

---

### On pricing and value

---

**"$49 a month is too expensive for what this does."**

Underlying concern: _I don't believe the time saving is worth the monthly cost._

Response: One saved senior planner hour a month more than pays for it. If the platform saves one hour of venue research per week across a team of three planners, the annual ROI is multiples of the subscription cost. The question is not whether $49 is a lot — it is whether the time saving is real. That is why there is a free tier with ten venues. Prove it to yourself before paying.

Follow-up: "What would make $49 feel obviously worth it? Let's make sure you hit that threshold in your first two weeks."

---

**"What happens if I need more than 500 venues?"**

Underlying concern: _I might outgrow Pro and face a large price jump._

Response: Pro covers 500 venues per workspace. For agencies managing more than that, Enterprise pricing is custom and scales with actual usage. We have not yet had a customer hit the Pro ceiling — most agencies with active venue portfolios work with 100–300 venues at any given time. If you are approaching 500, contact us and we will work out the right arrangement.

---

**"Can I try it before committing?"**

Underlying concern: _I've been burned by software that looked good in a demo._

Response: Yes, unconditionally. Free tier supports ten venues, no credit card, no time limit. Upload real documents, run real searches, share it with a colleague. If it does not deliver in the first week, cancel and nothing has been lost. The concierge offer means we can get you to ten real venues in the library within a day — so the trial starts with actual content, not an empty state.

Follow-up: "Want to start the trial right now? I can walk you through importing your first three venues while we're on this call."

---

### On the product and roadmap

---

**"We need [feature X] and it's not there yet."**

Underlying concern: _The product is not complete enough for my workflow._

Response: Depends entirely on what X is. If it is on the near-term roadmap (geo-spatial search, floor plan analysis, bulk import), be honest about timing without committing to dates. If it is not planned, say so and ask whether it is a hard blocker or a nice-to-have. A hard blocker means this is the wrong time for them — do not oversell.

Follow-up: "Tell me more about how you'd use it. Sometimes what sounds like a missing feature is actually already handled in a different way."

---

**"Is this just another AI hype product that will be irrelevant in two years?"**

Underlying concern: _I'm tired of adopting tools that disappear or pivot._

Response: The underlying problem — venue knowledge is scattered and hard to find — is not going away. AI makes the extraction faster and more accurate, but the knowledge base, the shared library, and the search experience are the product. Those would be valuable even if AI extraction improved tenfold or got ten times cheaper. The AI is the extraction engine, not the product itself.

Follow-up: "What would make you confident this is a long-term tool worth embedding in your workflow?"

---

**Docs:** [What is Shortlisty?](../../README.md) · [Business Proposal](../proposal.md) · [Competitive Landscape](../comparison.md) · [Pitch](pitch.md) · [Battlecards](battlecards.md)

# Objections

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

Response: Accuracy varies by document quality — clean text-based PDFs extract at high confidence; scanned or design-heavy decks extract with lower confidence on some fields. BENE shows you a confidence score on every field and cites the exact page it came from. You can verify anything with one click and override it permanently. We are not asking you to trust the AI blindly — we are showing you exactly what it knows and how sure it is.

Follow-up: "Would it help to run one of your actual venue decks right now so you can see what the confidence scores look like on a real document?"

---

**"What happens when it gets something wrong?"**

Underlying concern: _Errors will silently propagate and I won't catch them._

Response: The system surfaces uncertainty rather than hiding it. Low-confidence fields are flagged visually. When two documents about the same venue give different values, BENE shows the conflict and asks you to resolve it — it does not silently pick one. Every override you make is stored as a permanent correction and applied going forward.

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

Response: You don't move anything. Your existing storage stays exactly where it is. You upload to BENE the venue files that matter — PDFs, floor plans, spec sheets — and it reads them and builds them into a structured venue portfolio your whole team can search. Most teams start with their top 20 or 30 venues and expand from there. We can do that first import for you as part of onboarding. The key difference: Drive stores files; BENE manages a portfolio. You can't ask Drive "which of our venues has kosher catering and a freight entrance" and get an answer in five seconds.

Follow-up: "How many venues do you actively work with? We can have those in your portfolio within a day."

---

**"My team won't adopt a new tool."**

Underlying concern: _I've bought software before that nobody used._

Response: The adoption question is real and worth taking seriously. BENE's answer is the search result. The moment a junior planner answers a client question from the library without asking a senior colleague, they use it again. The value is immediate and individual — it does not require the whole team to adopt it simultaneously for the first person to get value.

Follow-up: "Who on your team spends the most time hunting for venue details? Start with them. If they find it useful in a week, the rest of the team follows."

---

**"We're too small to need this."**

Underlying concern: _The problem isn't bad enough to justify paying for a solution._

Response: The smaller the team, the harder the knowledge concentration problem. A five-person agency where one planner holds all the venue knowledge is more exposed than a twenty-person agency with some redundancy built in. If that person is unavailable when a client calls, the agency looks unprepared. Beyond the personnel risk: small agencies often have a tighter, more curated venue portfolio than large ones — and that portfolio is a real competitive asset. Keeping it in a shared Drive folder with no structure means you're underusing it every day. BENE Intelligence is a $49/month way to turn that portfolio into something your whole team can search and build on.

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

Follow-up: "What's your current policy for storing venue PDFs? Most teams have them in Google Drive or email, which have far weaker isolation guarantees than BENE."

---

**"What does the AI do with our documents?"**

Underlying concern: _Our documents are being used to train a model we don't control._

Response: Documents are sent to OpenAI's API for extraction. Under OpenAI's data processing terms for API customers, data submitted via the API is not used to train their models by default. We do not store the raw API payloads beyond processing. The extracted structured data lives in your account only. Enterprise customers can opt for Azure OpenAI processing for explicit data residency guarantees.

Follow-up: "I can send you our data processing summary if you want to share it with your legal or compliance team."

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

**Docs:** [What is BENE?](../../README.md) · [Business Proposal](../proposal.md) · [Competitive Landscape](../comparison.md) · [Pitch](pitch.md) · [Battlecards](battlecards.md)

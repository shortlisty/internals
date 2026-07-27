# What is IQ BENE?

_What this is, why it exists, and whether it makes sense as a business — no jargon._

**Docs:** [What is IQ BENE?](what-is-vip.md) · [Business Overview](business-overview.md) · [Competitive Landscape](intelligence-and-competitive-landscape.md) · [Architecture](architecture.md)

---

## The problem

Think of someone whose job is to find the perfect place for a birthday party, a company conference, or a wedding. They do this every day for many different clients. Over time they visit lots of venues and collect piles of files — PDFs that venues email them, floor plans, photos from site visits, spec sheets with room sizes, catering rules, what you can and cannot do.

These files end up everywhere — email, Google Drive, someone's desktop, someone's head.

When a client calls and asks "can you find me a venue for 150 people, kosher catering, downtown?" the planner knows they have seen the right place somewhere. They just cannot find it. They spend 45 minutes digging, send an answer they are not sure about, and the client wonders why it took so long.

All the knowledge exists. It is just buried.

---

## What IQ BENE does

IQ BENE is like giving your entire collection of venue files a brain. It reads them and turns them into searchable profiles.

You drop in a PDF, a floor plan, a photo set. IQ BENE pulls out the details that matter — how many people fit, what catering is available, whether there is a freight elevator, what is and is not allowed, who to call — and saves them as a structured profile. Your whole team can see it. Anyone can search it.

When the next client asks, you type the question and get the answer in seconds.

---

## Why nothing else does this

You might wonder — surely someone has built this already? Here is why they have not.

Big platforms like Cvent know thousands of venues — but only what venues choose to post publicly. They do not know what is in your files.

File tools like Dropbox and Google Drive store your files but cannot read them. They cannot answer "which of my venues allows open flame." They just hold folders.

Spreadsheets and Notion work for a while, but someone has to type everything in manually. That never stays current.

IQ BENE is the missing piece — it takes documents you already have, understands what is in them, and makes that knowledge searchable and shared across your whole team.

---

## Who it is for

**Small event planning agencies** — teams of 5 to 50 people managing dozens or hundreds of venues. The knowledge exists but it is scattered across individuals and inboxes. When a senior planner leaves, their knowledge leaves with them.

**Corporate event teams** — companies that run recurring events and repeat the same venue research every time because nothing was saved properly last time.

**Solo planners** — one person who has built years of venue knowledge and is tired of carrying it all in their head.

---

## How it works

No training needed. No complicated setup. Four steps:

1. Add a venue — just a name and address to start
2. Upload whatever files you have — a PDF, some photos, a floor plan
3. IQ BENE reads them and fills in the details automatically
4. Your whole team can now search across everything

If something is wrong, fix it with one click.

---

## How we make money

**Free to start** — up to 10 venues, no credit card. Most people understand the product the moment they see a 40-page PDF turn into a structured profile in 30 seconds.

**$99 a month for professional teams** — up to 500 venues, all file types, unlimited team members. One saved client conversation pays for months of the subscription.

**Custom pricing for larger agencies** — unlimited everything, white-label, API, priority support.

**In the future**, venues will pay to be visible to planners inside the platform — turning it into a two-sided marketplace where planners find venues for free and venues compete to be found.

---

## Is this a real business

Short answer: yes. Here is why.

The event management software market is around $18 billion globally and growing at 15% a year. There are over 3,400 event agencies in the US alone. Agencies are the fastest-growing buyer segment.

More to the point — the specific gap IQ BENE fills is unoccupied. Every tool either helps planners find new venues they do not know yet, or helps venues run their own operations. Nobody helps planners organize and search the venues they already know and trust. That gap is real and nobody is filling it.

Vertical SaaS products that solve one specific workflow pain for professionals and become embedded in that workflow consistently reach $500K to $2M in annual revenue with a small team. The numbers work at 100 paying agencies. They work better at 500.

---

## Why this is the right background to build it

Building IQ BENE is hard because reading messy documents and pulling out reliable structured data is hard. Most people trying to build this would be learning that skill from scratch while also learning the event industry. That is not the case here.

The hardest engineering challenge in IQ BENE is extracting structured data from messy, inconsistent documents and reconciling conflicts when multiple sources disagree. That is exactly what hotel content ETL pipelines do — normalize unstructured property content from hundreds of suppliers into a consistent schema. It is also exactly what a Product Information Manager does for ecommerce. Different domain, same problem.

Most founders building something like this would be learning both the domain and the engineering at the same time. That is not the case here.

---

## The one risk worth watching

AI extraction accuracy on real venue PDFs. Venue decks vary a lot — some are clean, some are scanned, some are design-heavy with text embedded in images. Before making any accuracy promises to customers, run 50 real venue documents through the pipeline and measure how often the key fields come out right. That benchmark needs to happen in the first weeks of building, not after the first customers sign up.

---

## One sentence

IQ BENE turns the pile of venue files your team has collected over the years into a searchable, shared knowledge base — so anyone on your team can find any venue detail in seconds, not 45 minutes.

---

_Want to see it in action? A 2-minute demo shows more than any document can._

---

**Docs:** [What is IQ BENE?](what-is-vip.md) · [Business Overview](business-overview.md) · [Competitive Landscape](intelligence-and-competitive-landscape.md) · [Architecture](architecture.md)

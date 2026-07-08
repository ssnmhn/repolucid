# Technical Due Diligence Memo: Umami (Sample)

**A Repolucid Buy-Side sample memo — a demonstration of the deliverable, performed on a well-known open-source project.**

---

**Read this first.** Umami is **not for sale** and has no connection to Repolucid. We chose it for this sample precisely because its code is public, widely respected, and excellent — which means you, the reader, can verify every single claim in this memo yourself by looking at the same public repository we did. Nothing here is a criticism of the Umami project; it is one of the best-run open-source codebases we have examined, and where we note risks, they are risks *for a hypothetical buyer in a hypothetical deal*, not flaws in the project. If the Umami maintainers would like anything here changed or removed, we will comply the same day.

**Scope disclaimer:** This is a technology assessment only. It is not legal, financial, or investment advice, and it does not evaluate Umami as a business — only what a buyer would learn from reading its code.

**What we examined:** The public repository at github.com/umami-software/umami, cloned on July 7, 2026, at version 3.2.0 (released June 24, 2026). Every file path, count, and number below was checked against that clone, not recalled from memory.

---

## 1. The one-page verdict

Imagine a business built on this codebase is listed for $150,000 on an acquisition marketplace, and you — a smart operator with no engineering background — are the buyer. Here is what the code itself tells you.

### Green lights

1. **The technology choices are mainstream and modern**, which means the pool of developers who can maintain this for you is about as large as it gets in software.
2. **The project's own license is MIT**, the most buyer-friendly license in open source — no strings attached to owning, changing, or commercializing the code.
3. **The code is unusually tidy and consistent**, with an automated style-enforcement tool wired in so every contributor's code looks the same.
4. **The project ships regularly** — six releases in the eight months before our review — which signals a living product, not abandonware.
5. **Privacy is designed in, not bolted on**: no cookies, no raw visitor IP addresses stored — which materially lowers your exposure to privacy regulation (GDPR and similar).
6. **It is cheap to run**: one small server and one database will serve a small customer base for roughly the cost of two streaming subscriptions a month.
7. **The growth path is already built**: when traffic outgrows the simple setup, the code already supports heavier-duty infrastructure — you switch it on rather than rebuild.

### Yellow flags

1. **Founder dependence is extreme**: one author wrote 59% of all 6,514 commits, and the top three authors — who appear to be from the same family and company — wrote 82%; if that team walked away from your hypothetical deal, deep knowledge walks with them.
2. **Automated test coverage is thin**: we count 23 test files guarding 881 source-code files, so most changes are verified by human care rather than by machine.
3. **One dependency (the component that identifies visitors' browsers) carries an AGPL license** — harmless while the product stays open source, but a genuine legal constraint if a buyer ever wanted to take modifications private.
4. **The code contains very few explanatory comments** — experienced engineers will read it fine, but a newcomer gets no guided tour.

### Red flags

1. **For this specific hypothetical: the software itself is free to everyone, including your future competitors** — so a buyer here would be paying for customers, brand, and operations, never for exclusive code, and any deal priced as if the code were proprietary would be mispriced.

**Bottom line:** 7 green, 4 yellow, 1 red. As a codebase, this is the good end of what we see. As a hypothetical acquisition, the price would need to reflect an open-source reality and a heavy dependence on its founding team.

---

## 2. What you would actually own

Umami is a **web analytics** product — it tells website owners how many people visited, from where, and what they did. It is a privacy-respecting alternative to Google Analytics: website owners paste one small snippet of code onto their site, and Umami turns the resulting stream of visits into charts.

Here is the stack — the set of technologies it's built from — in plain terms:

- **TypeScript** (the programming language, 881 files, about 58,000 lines): think of TypeScript as English written with a ruthless proofreader who refuses to let a sentence leave the building with a broken reference in it. It catches a whole class of mistakes before customers ever see them. It is also currently the most employable skill in web development — good news for hiring.
- **Next.js** (the framework, version 16): if building a web app from scratch is building a restaurant in a bare warehouse, Next.js is leasing a fully fitted commercial kitchen — the plumbing, wiring, and fire code are handled, and your team just cooks. It's made by a well-funded company (Vercel) and used by thousands of businesses, so it is not going away.
- **PostgreSQL** (the database): the filing room where every visit is recorded. Postgres is the boring, beloved, 30-year-old workhorse of databases — "boring" being the highest compliment in infrastructure.
- **Prisma** (the database toolkit): a professional translator standing between the application and the filing room, so the app can ask for "last week's visitors" in one consistent way regardless of the filing system behind it.
- **The tracker** (`src/tracker/index.js`): the courier slip. It's a deliberately tiny script — 419 lines — that customers attach to their websites; it reports each visit back to your server. Its small size matters commercially: it doesn't slow customers' sites down, which is a selling point in this market.
- **Optional heavy machinery**: the code can also plug into **ClickHouse** (a warehouse built purely for counting things at enormous scale), **Redis** (a short-term memory for frequently needed answers), and **Kafka** (a conveyor belt for very high event volumes). All three are switched off by default and activate via configuration (`src/lib/clickhouse.ts`, `src/lib/redis.ts`, `src/lib/kafka.ts` each check for their setting before turning on). This is the pre-built growth path from Green Light #7.

The product also ships translated into **52 languages** (52 translation files in `public/intl/messages/`) — a real asset if your customer base is international.

---

## 3. Architecture in plain English: how a page view becomes a chart

Follow one visitor through the system.

A person opens a customer's website. The customer pasted Umami's courier-slip script into their site, so as the page loads, that script quietly wakes up, notes the page address, the visitor's browser and screen size, and mails a small package of that information to your server — specifically to an address called `/api/send` (the code handling it lives at `src/app/api/send/route.ts`).

Your server opens the package and does four sensible things, in order:

1. **Checks the paperwork.** Is the package shaped correctly? Does it name a real website in your system? Malformed packages are rejected at the door.
2. **Turns away robots.** A large share of web traffic is automated crawlers; a filter discards them so your customers' numbers reflect humans.
3. **Issues an anonymous ID.** Instead of storing who the visitor is, the server blends the visitor's network address and browser signature with a secret ingredient that changes every month, producing an anonymous label like "visitor #a83f…". Same person returns tomorrow, same label — so you can count *unique* visitors — but nobody can run the recipe backwards to identify them, and next month the labels reset. This is the privacy-by-design we flagged as a green light, and you can see it in the code (`getSalt` in `src/lib/crypto.ts`).
4. **Files one row.** The visit becomes a single record — timestamp, page, country, browser — in the events table. The database's filing layout is defined in `prisma/schema.prisma` (18 record types: users, websites, sessions, events, teams, reports, and so on).

Later, the customer opens their dashboard. The dashboard never rummages through the filing room itself; it asks a dedicated **query layer** (`src/queries/`, 87 files) questions like "group last month's visits by country." The query layer counts the rows and hands back totals, and the browser draws them as charts.

```
 Visitor's browser              Your server                    Your database
 ─────────────────              ───────────                    ─────────────
 loads customer's site
   └─ tracker script ────────▶  /api/send
      ("courier slip")            ├─ 1. check paperwork
                                  ├─ 2. turn away robots
                                  ├─ 3. issue anonymous ID
                                  └─ 4. file one row ────────▶  events table
                                                                     │
 Customer's dashboard ◀────────  query layer (counts rows) ◀─────────┘
      (charts)
```

That's the whole machine. Its simplicity is a feature: fewer moving parts means fewer 3 a.m. phone calls.

---

## 4. Code health

We read the code the way a building inspector walks a property. Findings, with the exact locations so any engineer can verify:

- **Small, single-purpose parts.** The core utility folder `src/lib/` holds 56 files, each doing one job with an honest filename: `password.ts` handles passwords, `auth.ts` handles login checks, `crypto.ts` handles the anonymization math. When every tool hangs on a labeled hook, repairs are cheap.
- **A genuinely elegant pattern for the two-database design.** Every statistic is written twice — once for the simple database, once for the heavy-duty one — behind a single switch. Look at `src/queries/sql/getWebsiteStats.ts`: one function, two interchangeable implementations, chosen automatically by configuration. This is what "the growth path is pre-built" looks like at the code level, and it's disciplined work to maintain both paths for 87 query files.
- **The server's address book is self-explanatory.** There are 113 API endpoints (an endpoint is one address the app answers at, like `/api/send`) under `src/app/api/`, and the folder structure mirrors the addresses exactly — an engineer can find any endpoint's code by reading its URL.
- **Style is enforced by a robot, not by nagging.** The file `biome.json` configures an automated formatter/linter, so all 400+ contributors' code comes out looking like one person wrote it. Consistency is what makes a codebase cheap to hand to a new hire.
- **Process is written down.** `CONTRIBUTING.md` documents a two-branch discipline (a stable branch for releases, a development branch for work in progress) — small evidence of an operation run deliberately.
- **Database changes are versioned like legal amendments.** The folder `prisma/migrations/` contains 20 numbered, named change-scripts (`01_init` through `20_add_heatmap`), plus 12 more for the heavy-duty database in `db/clickhouse/migrations/`. You could reconstruct the database's entire history from these — exactly what you want to inherit.
- **The weak spot: sparse narration.** Across all 56 files in `src/lib/` we count roughly 31 explanatory comment lines total. The code is clean enough to read without them — and the few comments that exist are high-value, explaining *why* security decisions were made (see the top of `src/lib/auth.ts`) — but a new engineer gets no tour guide. Expect a slower first month for any hire.
- **Documentation for users is good; documentation for developers is thin.** The `README.md` (136 lines) gets you from zero to running quickly and links to polished hosted docs; there's even a short testing guide at `src/test/README.md`. What's missing is an architecture overview — the document you're reading section 3 of is the one they never wrote.

**Verdict: healthy.** Consistent, well-organized, professionally maintained. Its readability relies on convention rather than explanation — fine while conventions are upheld, a risk if stewardship lapses.

---

## 5. Founder dependence (the "bus factor")

Engineers call it the bus factor: how many people could be hit by a bus before nobody understands the code? We measure it from the project's commit history — the permanent record of who changed what, when.

The numbers from the full history (6,514 commits since the first one on July 17, 2020, by 425 total contributors):

| Author | Commits | Share |
|---|---|---|
| Mike Cao (founder) | 3,844 | 59.0% |
| Francis Cao | 1,073 | 16.5% |
| Brian Cao | 436 | 6.7% |
| **Top three combined** | **5,353** | **82.2%** |
| Other 422 contributors | 1,161 | 17.8% |

The pattern is not softening: in the 90 days before our review, the top two authors wrote 324 of 405 commits — 80%.

**What this means for a buyer.** The 425 contributors are real and speak well of the project's openness, but the deep, load-bearing knowledge lives in roughly three heads — apparently one family's — operating as Umami Software, Inc. For the open-source project this is normal and even healthy: focused stewardship is why the code is so consistent. For a *buyer*, it defines your single biggest technical risk: in any deal built on a codebase with this profile, the transition package — handover period, documentation sprint, availability for questions — is not a nice-to-have; it is a large fraction of what you are actually buying. Price it accordingly, and get it in writing.

---

## 6. Dependencies and licensing

A **dependency** is a ready-made component the product is built from — bought-in parts rather than parts machined in-house. Umami declares 66 of them for the running product and another 39 used only during construction (`package.json`).

**The project's own license is MIT.** In two sentences: MIT is the most permissive mainstream license — whoever holds the code may use, modify, sell, or keep private anything built on it, with essentially no obligations beyond keeping a copyright notice. The flip side is that this freedom belongs to *everyone*, not just you: the same code is free to any competitor, which is the root of our one red flag.

We then checked the license of every one of those 105 declared dependencies against the official package registry. What matters here is spotting **copyleft** licenses — the "share-alike" family (GPL, AGPL) that can require anyone who builds on the component to open-source their own work too. Findings:

- **102 of 105 are permissively licensed** (MIT, Apache-2.0, BSD, ISC, and similar) — no obligations that concern a buyer.
- **One real copyleft item: `ua-parser-js` version 2.0.10** — the component that identifies which browser and device a visitor used (used by the server in `src/lib/detect.ts`). Version 1 of this component was MIT; version 2 was re-licensed **AGPL-3.0**, the strongest share-alike license, whose obligations trigger even when users only interact with your software over the network. Because Umami itself is open source, the obligation is comfortably met today. But a buyer who dreamed of taking a modified version proprietary would need to either license this component commercially (the vendor sells such licenses), replace it, or abandon the dream. This is precisely the kind of two-line detail in a lock-file that changes a legal posture — and why dependency license scans belong in every acquisition.
- **Two footnotes, neither a problem:** `jszip` is dual-licensed "MIT OR GPL-3.0" (you simply choose MIT), and one build-time-only tool (`rollup-plugin-dts`, LGPL) never ships inside the product.

The presence of automated dependency-update commits in the history (53 from the `dependabot` bot) and current versions of the major components (React 19, Next.js 16, Prisma 7) tell us bought-in parts are kept fresh rather than left to rust.

---

## 7. Security and data posture (hygiene level)

To be explicit about scope: we assess *hygiene* — whether the practices visible in the code are the ones professionals use — not whether specific holes exist. Hole-hunting is a penetration tester's job, and publishing such findings about a live open-source project would be irresponsible. Nothing in this section is a vulnerability report.

The hygiene is good:

- **Passwords are stored the right way.** `src/lib/password.ts` uses bcrypt — the industry-standard method that stores an irreversible fingerprint of each password rather than the password itself, so even a stolen database doesn't yield working passwords.
- **Logins use signed tokens, and changing a password locks out old tokens.** The logic in `src/lib/auth.ts` deliberately invalidates a user's existing sessions when their password changes — a detail sloppier codebases skip, and one the authors took care to explain in comments.
- **The front door checks IDs.** The public data-collection endpoint (`src/app/api/send/route.ts`) validates the exact shape of every incoming package before touching it, filters out bots, honors an IP blocklist, and even defends against an obscure spreadsheet-injection trick in exported data. This is defense-in-depth thinking on the single most exposed door in the product.
- **Secrets stay out of the code.** Passwords and keys are supplied through the server's environment at runtime — the standard practice — not written into files an acquirer might accidentally publish. One onboarding note for a buyer's first-day checklist: set the application's dedicated secret value (`APP_SECRET`) explicitly in production, as the sample configuration reminds you to.
- **Data minimization is structural.** As covered in section 3, visitor identities are never stored — no cookies, no raw network addresses in the database, anonymous IDs that reset monthly. For a buyer, this isn't just ethics; it's less regulated data to breach, disclose, or delete on request.
- **The software stays current.** Six releases in eight months (v3.0.0 in November 2025 through v3.2.0 in June 2026) and automated dependency updates mean security fixes in bought-in parts flow through routinely.

---

## 8. What it costs to run

The default deployment is refreshingly humble: **one application server plus one PostgreSQL database.** The repository ships a one-command Docker setup (`docker-compose.yml` — Docker being shipping containers for software: the app and its environment packed into one box that runs identically anywhere), plus recipes for several hosting platforms (`Dockerfile`, `netlify.toml`, `app.json`, a `podman/` folder).

Rough monthly math for a small customer base — say, dozens of customer websites totaling up to a few million page views a month:

- Small cloud server (2 CPU / 4 GB memory): **$12–25**
- Managed PostgreSQL database (recommended over self-managing, for the backups alone): **$15–30**
- Domain, backups, incidentals: **$5–10**

**Call it $30–65 a month.** The tracker script is tiny, so bandwidth is negligible. There is no per-seat licensing, no vendor fee — the software is free.

**What scaling pressure looks like:** the events table grows with every page view across every customer site, forever. Into the low tens of millions of events per month, Postgres with these indexes will cope; past that, dashboards slow first (the counting queries, not the collection). The escape hatch is the one already built in — activating ClickHouse, the counting warehouse, roughly doubles-to-triples infrastructure cost but raises the ceiling by orders of magnitude. The buyer's question is not *can it scale* (it can) but *where on that curve is the business today* — question 6 below.

---

## 9. Tests and maintainability

An automated test is a self-checking exam the code runs on itself — the smoke detectors of software. They're what let a new owner's developer change something on Tuesday and know by Tuesday afternoon whether they broke something.

The honest count: **17 unit test files** inside `src/` (concentrated on the security- and logic-critical utilities — `src/lib/auth.test.ts`, `src/lib/crypto`-adjacent code, the permissions rules in `src/permissions/`) plus **6 end-to-end test files** in `tests/e2e/` that drive the product like a real user (login, creating websites, team management). Total: **23 test files guarding 881 source files.**

Three fair readings of that number:

1. **The tests that exist are well-aimed.** Login, permissions, and the anonymization math are exactly what you'd protect first with a small test budget.
2. **The safety net has large gaps.** The dashboard, the charts, most of the 113 API endpoints, and the 87 query files are essentially untested by machines. The continuous-integration setup (`.github/workflows/ci.yml` — an automatic gatekeeper that runs on every change) does run the tests and a full build, so the code always *compiles*; TypeScript's proofreading catches another whole class of errors. But "it builds" is not "it's correct."
3. **The practical consequence for a buyer:** today, the thing preventing regressions is the founding team's deep familiarity — the same concentration risk from section 5, now wearing a different hat. A new owner's engineer will change things more slowly and more nervously than the current team does. Budget for your first hire to *add tests before adding features*; it is the single best investment in making this codebase safely operable by strangers.

---

## 10. The 15 questions to ask the seller

The questions this analysis would arm you with in a real deal of this shape. Plain English; screenshot freely.

1. **What exactly am I buying?** Since the code is free to the world, walk me through the assets: customer contracts, brand, domains, infrastructure, private modifications, data.
2. **How far has your production system drifted from the public code?** Show me every custom patch, and tell me who understands each one.
3. **Do you have signed IP assignments** from every person — employee or contractor — who ever wrote code for your private modifications?
4. **What's the handover package?** How many hours of your time, over how many months, are included after closing — and is that in the contract?
5. **Walk me through your monthly infrastructure bill,** line by line, for the last six months.
6. **Are you running the simple setup or the heavy-duty one** (ClickHouse/Redis/Kafka)? If simple: how close is the traffic to needing the upgrade, and who would perform it?
7. **How do you take upstream updates?** When a new open-source release ships, what's your process, how long does it take, and has an upgrade ever broken production?
8. **How big is the database, and how fast is it growing?** What share of all events comes from your single largest customer?
9. **Do any of your private modifications have tests?** If not, how does anyone know an update didn't break them?
10. **What's your position on the AGPL-licensed browser-detection component?** Do you hold a commercial license for it, or rely on open-source compliance — and has counsel reviewed that?
11. **Show me the last incident.** When did the service last go down, how did you find out, and what did you change afterward?
12. **Where do the secrets live** — application keys, database credentials — and exactly which humans and systems can read them today?
13. **When did you last actually restore a backup,** not just take one? Walk me through it.
14. **What does support really look like?** Tickets per week, the three most common complaints, and how many are about performance.
15. **What's your relationship with the upstream maintainers,** and what's the plan if the open-source project ever changed direction, slowed down, or re-licensed?

---

## 11. Methodology and limits

**What we did:** cloned the full public repository (all 6,514 commits of history), read the code, counted what could be counted (files, tests, endpoints, database migrations, contributors, releases), and checked the license of all 105 declared dependencies against the official package registry. Every claim above cites a file path or a number reproducible from the repository — that is the standard we hold every memo to, and we invite you (or any engineer you trust) to check our work. If a claim can't be verified, we don't write it.

**What a public sample cannot see — and a full engagement covers:**

- **The seller's private code and infrastructure.** Real deals run on private repositories with custom patches; we review those under NDA, with the same evidence-cited method.
- **Real operating data.** Actual traffic volumes, database sizes, infrastructure bills, error rates, and uptime history — the difference between "this should be cheap to run" and "here is what it costs."
- **Seller interviews.** An hour of structured technical questions (built from a list like section 10) reveals more than any document; we join the call and translate in real time.
- **Deal-specific risk weighting.** Whether thin tests or founder dependence *matters* depends on your plans — flip it, hold it, or grow it — so a full memo weights findings against your intent.
- **Professional security testing**, by a specialist firm, if warranted; we scope it, we don't perform it.

**Turnaround** for a full buy-side engagement on a deal this size is typically 3–5 business days from access.

The purpose of this document is simple: before you wire six figures for a software business, someone who reads code should read yours — and then explain it in your language, with receipts.

---

*Prepared by Repolucid. Sample memo on a public codebase; Umami is not for sale, is not affiliated with Repolucid, and its maintainers may request changes or removal at any time — we will comply the same day. Technology assessment only; not legal, financial, or investment advice.*

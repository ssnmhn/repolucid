# The Repolucid Legibility Rubric (v1)

*How legible is your codebase — to a new engineer on day one, and to the AI coding agents your
team already uses? 25 checks, 1 point each. Score honestly.*

## A. Orientation (5)

1. README's first screen states what the project is, who it's for, and what it is **not**.
2. A newcomer can locate the entry point(s) from the docs alone, without grepping.
3. Directory layout is documented (or self-evident: top-level dirs named by domain, not by type).
4. The install-and-run quickstart works on a clean machine, exactly as written.
5. Domain terms are defined once (glossary or consistent naming) — no unexplained internal jargon.

## B. Architecture (5)

6. An ARCHITECTURE.md (or equivalent) exists and matches the current code.
7. The 3–7 load-bearing abstractions each have a written, single-sentence responsibility.
8. The lifecycle of the core operation is written down end-to-end (request path, pipeline, loop).
9. Invariants are documented: "X must never…", ordering rules, threading model, encoding rules.
10. The weird parts have decision records — *why not* the obvious approach.

## C. Contribution path (5)

11. CONTRIBUTING.md exists and its environment setup works today.
12. Tests run with one documented command; the suite's layout is explained.
13. Style/lint/format rules are stated **and enforced by tooling**, not tribal knowledge.
14. PR conventions (branching, commit format, review expectations) are written down.
15. There's a first-contribution trail: labeled starter issues or an onboarding doc.

## D. Agent context (5)

16. AGENTS.md or CLAUDE.md exists at the repo root.
17. It states build/test/lint commands in machine-usable form.
18. It states the sharp edges: generated files, do-not-touch zones, slow or flaky test areas.
19. It states repo-specific conventions an agent would otherwise violate (error handling,
    import style, API-stability rules, changelog discipline).
20. It's current: every path it references exists; updated alongside significant changes.

## E. Examples & tests as documentation (5)

21. Runnable examples cover each primary use case.
22. Examples run in CI, so they cannot rot.
23. Tests read as specifications: descriptive names, clear arrange-act-assert.
24. Edge-case behavior is documented where users will hit it (limits, encodings, concurrency).
25. The public API has reference docs on ≥90% of exported symbols.

## Bands

| Score | Band | What it means |
|---|---|---|
| 20–25 | **Legible** | New engineers ship in days; agents work near their ceiling. |
| 15–19 | **Workable** | Onboarding relies on a few overloaded seniors; agents need babysitting. |
| 10–14 | **Tribal** | The real docs live in three people's heads. Bus factor is the roadmap. |
| 0–9 | **Opaque** | Every newcomer re-derives the architecture. Agents actively mislead. |

---
*Rubric by [Repolucid](https://ssnmhn.github.io/repolucid/) — we read your entire codebase and
write the docs it deserves, in 48 hours. Free 3-finding mini-audit for any repo.*

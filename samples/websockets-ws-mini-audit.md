# Mini-audit: websockets/ws

*An unsolicited sample by Repolucid — we read codebases and write the docs they deserve. Produced with admiration for this project; offered freely to its maintainers.*

**Commit audited:** 8df8265 · **Date:** 2026-07-06 · **Legibility score: 13/25**

## Scorecard

| Category | Score | Justification |
|---|---|---|
| Orientation | 4/5 | A 548-line README with TOC and FAQ, a 22-line `index.js` manifest, and 13 flat, self-describing modules in `lib/` — newcomers orient fast. |
| Architecture | 2/5 | Superb JSDoc and inline comments, but the design — two state machines, a send queue, backpressure — is written down nowhere as a whole. |
| Contribution path | 2/5 | Setup is genuinely `npm install && npm test`, and SECURITY.md is exemplary — but there is no CONTRIBUTING.md, no PR template, no written conventions. |
| Agent context | 1/5 | No AGENTS.md/CLAUDE.md/.cursorrules. The Node 10 syntax floor and other agent-breaking constraints live only in `package.json` and CI config. |
| Examples & tests as docs | 4/5 | 10,263 lines of mocha tests (188 cases in `websocket.test.js` alone), two runnable example apps, and an instructive heartbeat FAQ. |

## Three findings

**The architecture is excellent — and exists only in the maintainers' heads and scattered comments.**
The entire wire pipeline is two state machines: `Receiver` (`lib/receiver.js`), a `Writable` running a seven-state parser (`GET_INFO`…`DEFER_EVENT`, lines 17–23), and `Sender` (`lib/sender.js`), whose queue serializes frames whenever deflate or Blob reads make a send asynchronous (lines 22–24, 536–554). Backpressure is precise: `socketOnData` pauses the socket when `receiver.write()` returns false (`lib/websocket.js:1373–1377`) and resumes on the receiver's `'drain'` (lines 1189–1193). The repo's only design diagram is ASCII art of the closing handshake, hidden inside a JSDoc comment at `lib/websocket.js:285–295`.
Cost: every contributor re-derives this data flow from scratch, and a coding agent will edit invariants it cannot see.

**Hard-won operational invariants are documented only at the point of fix, never as a system.**
The zlib `Limiter` is *intentionally* a module-global capping concurrency at 10, because Node's shared thread pool fragments memory catastrophically otherwise (`lib/permessage-deflate.js:17–24`, citing nodejs/node#8871 and ws#1202). The defaults `maxBufferedChunks: 1024 * 1024`, `maxFragments: 128 * 1024`, `maxPayload: 100 MiB` (`lib/websocket-server.js:72–74`) are the remedy for the May 2026 memory-exhaustion DoS (`SECURITY.md:44–45`). The 8 KiB random mask pool must stay lazily initialized because server frames are never masked (`lib/sender.js:93–98`). Each rationale appears exactly once, in a comment, at its fix site.
Cost: a well-meaning "simplification" PR — human or agent — that touches any of these reintroduces a documented CVE class.

**The contribution path is all convention, no document.**
There is no CONTRIBUTING.md anywhere in the tree and no PR template (`.github/` holds only FUNDING.yml, an issue template, and CI). The Node.js 10 support floor is visible only in `package.json:32` and the 27+-job CI matrix (`.github/workflows/ci.yml:16–29`, Node 10→26 across three OSes plus x86 Windows). Nothing states that lint runs on a single matrix cell (`ci.yml:58–61`), that tests should also pass without native addons (`WS_NO_BUFFER_UTIL`, `lib/buffer-util.js:115`), or that the Autobahn conformance suite (`test/autobahn.js`) exists and is not part of `npm test`.
Cost: newcomers ship syntax that fails on Node 10, and nobody outside the core team knows to run the suite that backs the project's headline claim.

## What a Docs Sprint would deliver

- **AGENTS.md** at root: Node 10 syntax floor; `bufferutil`/`utf-8-validate` must remain optional; error codes in `doc/ws.md` are public API; never alter `Limiter` concurrency or the receiver limit defaults without reading the linked CVE history; run `npm test`, `npm run lint`, and the `WS_NO_BUFFER_UTIL=1` variant before proposing changes.
- **ARCHITECTURE.md** (~4 pages): the Receiver/Sender state machines, backpressure chain, masking invariants, close-handshake sequence (promoting the ASCII diagram from `lib/websocket.js`), and a "security archaeology" table mapping each SECURITY.md entry to the code that now guards it.
- **CONTRIBUTING.md quickstart**: three-command setup, what CI actually checks per matrix cell, how to run integration and Autobahn suites, and PR conventions distilled from recently merged PRs.
- **A `lib/` module map** in the README: one line per file, so the 13-module layout is navigable in thirty seconds.
- **A test-authoring guide**: where a new case belongs in the 10k-line suite, the `test/duplex-pair.js` fixture pattern, and how coverage gating via nyc/Coveralls works.

### Excerpt: opening of the ARCHITECTURE.md we'd write

ws is, at its core, two state machines joined by a raw socket. Everything else — handshake negotiation, extension parsing, the streams wrapper — exists to construct those machines correctly and tear them down safely.

Inbound, `Receiver` (`lib/receiver.js`) is a Node.js `Writable` that consumes TCP chunks through a seven-state parser (`GET_INFO` through `DEFER_EVENT`), enforcing RFC 6455 framing — masking direction, RSV bits, control-frame limits, UTF-8 validity — before anything reaches user code. Backpressure is real, not decorative: when `receiver.write()` returns false, the socket pauses (`socketOnData`, `lib/websocket.js`) and resumes only on the receiver's `'drain'`.

Outbound, `Sender` (`lib/sender.js`) frames messages and serializes them through an internal queue whenever compression or a Blob read makes a send asynchronous, so frames can never interleave on the wire.

Between them sits one deliberately global object: the zlib `Limiter` (`lib/permessage-deflate.js`), capping concurrent deflate calls at 10 because Node's shared thread pool fragments memory catastrophically without it. That constant is a load-bearing wall; treat it as such.

---

*Maintainers: everything above is yours to use, no strings. If you want the full pack, it's free for this repo — we're building our portfolio.*

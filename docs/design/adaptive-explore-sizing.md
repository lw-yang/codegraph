# Design + status: adaptive `codegraph_explore` sizing (sibling skeletonization)

**Status:** Implemented & validated, **default-on**, on branch
`feat/adaptive-explore-sizing` (commit `d6d059f`, 2026-05-29). Escape hatch:
`CODEGRAPH_ADAPTIVE_EXPLORE=0`.
**Motivation:** make `codegraph_explore` size its output to the *answer* rather
than always filling the budget cap — so a "sibling-heavy" flow (many
interchangeable implementations of one interface) stops costing *more* than
plain grep/read, without starving "diffuse" flows that genuinely need broad
source.

---

## TL;DR

`codegraph_explore` returned full source for **every** relevant file up to its
char budget. On a question whose answer spans many *same-shaped* classes — e.g.
"how does OkHttp process a request through its interceptor chain?", which touches
~14 `class … : Interceptor` implementations — that meant ~28 KB of mostly
**redundant full bodies**. Because those bodies ride in the context window for
the rest of the session, the WITH-CodeGraph arm cost *more* than the WITHOUT arm
(which answers the well-named interceptor question in ~10 cheap greps). OkHttp
was the benchmark's cost outlier (−3% — i.e. *costlier* than native search).

Fix: when a file is **both (a) off the synthesized flow spine and (b) a
polymorphic sibling**, render it as a **skeleton** (class + member *signatures*,
bodies elided) instead of full source — keeping the on-spine exemplar and the
mechanism in full.

- **OkHttp:** explore `28.5k → 16.6k` chars; headless A/B median **$0.413 ON vs
  $0.462 shipped vs ~$0.57 without-CodeGraph** → flips OkHttp from −3% costlier
  to **~28% cheaper than native**, with **reads NOT raised** (median 1 vs 3).
- **Excalidraw / Tokio / Django / VS Code / Gin:** explore output is
  **byte-identical** with the flag on/off (0 skeletons) → **provably zero
  regression**. Their flows have no off-spine ≥3-implementer sibling group.

---

## The problem in one picture

`handleExplore` gathers relevant files, sorts by relevance, and fills up to
`maxOutputChars` (the "whole-small-file rule" dumps any relevant file ≤220 lines
in full). The budget is a **target**, not a ceiling:

```
OkHttp explore (shipped):  RealCall (full) + RealInterceptorChain (full)
                         + CallServerInterceptor (full, 8.7k)
                         + Bridge/Connect/Cache/… (full, ~4-5k each)   ← all ~same shape
                         = ~28k, most of it redundant interceptor bodies
```

The agent only needs the **mechanism** (`RealInterceptorChain.proceed` iterating
the chain) + the **contract** every interceptor implements + maybe one concrete
example. The other five full bodies are padding — but only *because they're
interchangeable*. On a diffuse question (Excalidraw's render pipeline:
`mutateElement → … → renderStaticScene`), the off-spine files are **distinct
steps**, and their bodies do real work — eliding them just makes the agent
reconstruct them from signatures (more reasoning, net costlier; see "Dead ends").

So the whole game is: **tell "interchangeable sibling" apart from "distinct
step," cheaply.**

## The two-condition gate

A file is skeletonized iff **both** hold (and `CODEGRAPH_ADAPTIVE_EXPLORE != 0`):

1. **Off the flow spine.** `buildFlowFromNamedSymbols` now returns its path node
   set (`pathNodeIds`) in addition to the rendered Flow text. A file with any
   symbol on that traced chain is "on-spine" and always kept full — that's the
   mechanism + the exemplar the agent is actually tracing through. (Gated on a
   spine existing at all; if there's no spine, nothing skeletonizes.)

2. **A polymorphic sibling.** The file's class `implements`/`extends` a supertype
   that has **≥ 3 implementers** (`MIN_SIBLINGS`). This is the signal that the
   class is one of many *interchangeable* implementations rather than a unique
   step. Computed from real `implements`/`extends` edges (see "Why this signal"),
   cached per-supertype so it stays a handful of edge lookups.

`RealInterceptorChain` *also* implements `Interceptor`, but its `proceed` is
**on the spine** → kept full (condition 1 fails). `RealCall` is off-spine but
implements nothing with ≥3 impls → kept full (condition 2 fails). The other
interceptors are off-spine **and** ≥3-impl siblings → skeletonized. Exactly right.

## Why "shared supertype with ≥3 implementers" is the signal

The thing that makes OkHttp's interceptors interchangeable is precisely that
they're **N implementations of one interface**, invoked polymorphically. That is
a *structural* property the graph records as `implements`/`extends` edges:

```
14 classes ──implements──▶ Interceptor      (BridgeInterceptor, CacheInterceptor,
                                              CallServerInterceptor, … )
```

Excalidraw's `renderStaticScene`, `Scene`, `Collab` share **no** common
supertype — the ≥3-implementer query returns nothing for them. So the signal
cleanly separates the two repos, and (validated below) leaves every non-sibling
flow untouched.

The `≥ 3` threshold matters: 1:1 "service interface → single impl" pairs (the
common Spring/Java shape) are **not** siblings and stay full. Only genuine
many-impl families (interceptor chains, strategy/visitor families, codec
registries) trip the gate.

## Skeleton rendering

For a skeletonized file we emit the class + member **signature lines** (not
bodies). Because a symbol node's `startLine` can point at a decorator/annotation
(`@Throws`, `@Override`, `@objc`), we scan forward up to 4 lines for the line
that actually *names* the symbol, so the skeleton shows the real signature:

```
#### …/CallServerInterceptor.kt — CallServerInterceptor, intercept, … · skeleton (signatures only; Read for a full body)
```kotlin
30  object CallServerInterceptor : Interceptor {
32  override fun intercept(chain: Interceptor.Chain): Response {
194 private fun shouldIgnoreAndWaitForRealResponse(code: Int): Boolean =
```
```

The header still lists the file's symbols and says `Read for a full body`, so the
agent can pull one specific implementation if it truly needs it.

## Validation

Headless `claude -p`, Opus 4.8, median of 3, WITH-CodeGraph adaptive **on vs off**
(isolates the flag). Probe sizes from `scripts/agent-eval/probe-explore.mjs`.

| Repo | explore OFF→ON | skeletons | A/B cost (ON vs shipped) | reads |
|---|---|---|---|---|
| **OkHttp** | 28.5k → **16.6k** | 6 | **$0.413 vs $0.462** (~28% < native's $0.57) | flat (1 vs 3) |
| Excalidraw | 28.6k → 28.6k | 0 | byte-identical → neutral | — |
| Tokio | identical | 0 | neutral | — |
| Django | identical | 0 | neutral | — |
| VS Code | identical | 0 | neutral | — |
| Gin | identical | 0 | neutral | — |

The decisive check (the open risk of skeletonization) **passed**: skeletonizing
the off-spine interceptors did **not** push the agent to Read them back — reads
stayed flat (lower, if anything). And the 5 non-sibling repos are byte-identical
with the flag toggled, so default-on carries no regression for them.

## Dead ends (don't re-attempt these)

1. **Demote/rank low-value files** (e.g. broaden `isLowValuePath` to drop
   `*-testing-support/` fixtures). Improves *content quality* but **not size** —
   explore refills the freed budget with other full bodies (28,478 → 28,424).
   Ranking ≠ shrinking; you must *skeletonize* to shrink.
2. **Gate on entry-node membership.** A precise symbol-bag explore query *names*
   every chain participant, so they're all "entry nodes" — no separation, nothing
   skeletonizes.
3. **Rely on interface-impl synthesizer edges** (`synthesizedBy:'interface-impl'`)
   for the sibling signal. They were **not** created for OkHttp's `Interceptor`
   (a Kotlin `fun interface`), so the signal must come from the real
   `implements`/`extends` edges, not synth edges.
4. **A plain "core-floor" gate** (keep first N full, skeletonize the rest) —
   skeletonized Excalidraw's *distinct* steps → **+17% cost regression**. The
   sibling condition is what makes it safe.

## Code

- `src/mcp/tools.ts`
  - `adaptiveExploreEnabled()` — the flag (default on).
  - `buildFlowFromNamedSymbols()` — now returns `{ text, pathNodeIds }`.
  - `handleExplore()` — `isPolymorphicSibling()` helper (supertype ≥3-impl
    detection, cached) + the skeleton branch in the source-section loop.

## Frontier / future work

- **No regression test yet** for the skeletonization (a fixture with ≥3 interface
  impls + a flow spine asserting off-spine siblings skeletonize, distinct steps
  stay full, `=0` disables). Recommended before/with merge.
- **Non-interface sibling families** (Go `HandlerFunc` slices, function-pointer
  registries) aren't caught — they have no `implements`/`extends` edge. Gin's
  middleware chain, for instance, doesn't trip the gate (its handlers are funcs,
  not interface impls).
- **Exemplar selection** when *no* interceptor is on the spine: today all siblings
  skeletonize and the agent leans on the interface contract; showing one as a
  forced exemplar might read slightly better (untested).

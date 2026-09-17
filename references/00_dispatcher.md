# Chomsky Vase First Scaffolding — Technical Operational Dispatcher

**Framework Author**: William Fitzpatrick (*Writer Science*)  
**Source Lecture**: [How to Write a Great Sentence | Writing Tips from an English Professor](https://www.youtube.com/watch?v=vFsQgIQFtwM)  
**Parent Collection**: [Master Collection](../../kirby-fitzpatrick-writers-collection/SKILL.md) | [Global Help](../../../kirby-help/SKILL.md)  

---

## 1. Cognitive Foundation: The Vase Is Built Before the Flowers Are Chosen

**The lecture's craft claim.** Every sentence is a *container*. Fitzpatrick's account of how a strong sentence gets made puts the container first: the writer settles the shape — the structural frame, the slots, the hinge between subject and predicate, the place where the clause will land — and only then decides what content occupies it. The vase is chosen and built; the flowers are put in afterward. Writers who begin with the flowers (a good idea, an interesting fact, a striking image) end up with a container that was shaped *by accident*, one that fits exactly those flowers and nothing else. The structural frame is reusable across a hundred contents. The flowers are not.

**The Chomsky layer beneath it.** Fitzpatrick's container is Chomsky's *deep structure*. A generative grammar separates the abstract structural frame from its surface realizations: one deep structure, many well-formed surface sentences, all sharing the frame that licenses them. And the frame is *independent of meaning* — the famous sentence demonstrates it precisely: **"Colorless green ideas sleep furiously"** is structurally immaculate and semantically void. Form can exist with no content. Content can exist with no form. Those are two different failures, and confusing them is expensive.

**Why this is a software engineering problem, not a prose problem.** Translated into code, the vase is:

- **type contracts** — the shapes the pipeline's stages are allowed to exchange,
- **error envelopes** — the closed taxonomy of ways the operation can fail, and what each failure carries,
- **lifecycle guards** — who acquires, who releases, who cancels, what the deadline is, what happens on abandon,
- **the boundary** — where untrusted input is admitted or rejected, once.

and the flowers are **domain logic**: the pricing rule, the ledger reconciliation, the retry-worthy classification, the parsing of a response body.

Two failure modes follow directly, and they are not equally priced:

| Failure | In prose | In code | Detection |
|---|---|---|---|
| **Form without content** | A perfectly shaped sentence carrying nothing | An `interface` with bare bodies, a `TODO`-riddled scaffold, a `Result` type nothing implements | *Compile time*, type checker, stub marker. **Loud and early.** |
| **Content without form** | A heap of striking images with no grammatical frame | A linear happy-path function whose container is emergent: implicit lifecycle, unspecified failure, unpinned concurrency | *Production.* **Silent and late.** |

> **The asymmetry that decides the practice: an empty vase compiles; a heap of flowers does not.**

Payload-first code doesn't fail at build time, because there is nothing to fail against. It fails eight months later in an incident review, and the incident is unattributable — the failure is distributed across every call site that never agreed on an envelope.

**Why models default to flowers-first.** Token-level generation is a *surface-structure engine*: it emits the most probable continuation of the prose surface it was handed. A request that says "write a client that fetches user records" supplies a flowers-shaped prompt, and the highest-probability completion is the happy path. The error envelope is the *least* interpolated part of any program — every system has its own taxonomy — so it is also the part most likely to be emitted last, generically, and wrong (`catch (e) { log(e); return null; }`). Vase-first scaffolding is the countermeasure: force the non-interpolable, system-specific structure into existence *while there is still nothing to hide it behind*.

**The load-bearing definitions.**

1. **Vase** — the compile-time structural envelope: types, error taxonomy, lifecycle pairing, deadline budget, boundary validation, concurrency schedule. Reusable. Survives every payload swap.
2. **Flowers** — the domain payload: the rule, the transform, the classification, the business decision. Disposable, iterated, and frequently rewritten.
3. **Flower-bed** — one payload site: exactly one function body, one handler, one stage. One bed per edit.
4. **Water** — the ambient services the vase must route explicitly rather than let the payload reach for: clock, RNG, transport, cancellation token, retry policy, logger. Every un-routed service is a hidden parameter that no test can vary.
5. **Rim** — the boundary where the vase meets the world: validation, authentication, size limits, admission control. Water is poured at the rim, once.
6. **Crack** — a silent invariant breach: the vase held shape while the payload went around it (a `null` returned under a `Result` signature; a `context.Context` never checked; a `panic!` inside a lock).

### Before vs After, Visualized

```text
BEFORE — FLOWERS-FIRST  (the container is a residue of one happy path)

 func FetchUser(ctx, id) (*User, error) {
     resp, _ := http.Get(url(id))          ← no client, no timeout, no retry policy
     body, _ := io.ReadAll(resp.Body)      ← Body never closed: lifecycle absent
     u := parse(body)                      ← parses into map[string]any: no contract
     return u, nil                         ← error slot exists, envelope does not
 }
        │
        ├── callers each invent their own failure handling   (N boxes, N taxonomies)
        ├── 503 / malformed body / deadline → all indistinguishable
        └── concurrency, cancellation, idempotency: unstated, therefore accidental

   The vase is whatever shape the first flower happened to be.
   Nothing can be reviewed except the flower. Nothing can be reused. Nothing fails early.

 ────────────────────────────────────────────────────────────────────────────

AFTER — VASE-FIRST  (the container is decided, then filled bed by bed)

 ┌── VASE ─────────────────────────────────────────────────────────────────┐
 │  types        UserId, User, RawBody                       ← no map[string]any
 │  envelope      FetchError = NotFound | Timeout | Transport{retryable}
 │                            | Decode{offset} | Refused{status}
 │  boundary      Validate(UserId) at the rim; reject once, carry the reason
 │  lifecycle     Client owns Transport; Body closed by scope guard (defer/RAII)
 │  deadline      Deadline budget in ctx/param, passed down, never re-invented
 │  schedule      Bounded concurrency, backpressure, idempotency key on retry
 │  observability stage/error tags emitted at the frame, not in the payload
 └─────────────────────────────────────────────────────────────────────────┘
        │  compiles with every body unimplemented — this is the gate
        ▼
 ┌── FLOWERS (one bed at a time, vase unchanged) ──────────────────────────┐
 │  FetchUser body   → decode + map Decode→Decode{offset}, Timeout→Timeout │
 │  Caching layer    → fills the rim/boundary bed; contract untouched      │
 │  New provider     → new vase instance; same types, same envelope        │
 └─────────────────────────────────────────────────────────────────────────┘

   The vase is what a reviewer can audit before any flower exists.
   Flower swaps are additive; vase edits are their own decision.
```

### Detection Signals of Flowers-First Code

Recognize it before reviewing it:

| Signal | What it looks like |
|---|---|
| **Error handling at call sites** | Every caller has its own `if err != nil` / `try/except` / `catch` adaptation. The taxonomy lives in the callers, which means there is no taxonomy. |
| **No closed failure vocabulary** | Failures are strings, `error` with no variants, or a single `Error` class. 503, timeout, decode failure, and cancellation are indistinguishable. |
| **Ambient reaches** | The body touches `time.Now()`, `rand`, `os.Getenv`, a module-level singleton, or the network directly. Untestable as written; the world is a hidden parameter. |
| **Emergent lifetime** | Resource acquisition and release are not paired in one lexical unit. `Body`/lock/file/transaction ownership is inferable only by reading to the end. |
| **Unpinned concurrency** | Fan-out count comes from input size; queues are unbounded; ordering, atomicity, and replay are unmentioned. |
| **Boundary sprinkled** | Validation appears in three places at three depths, so no single point admits or rejects an input. |
| **Payload-shaped types** | `map[string]any`, `interface{}`, `dict`, `Record<string, unknown>` where a contract belongs — the vase is the union of every flower ever passed. |

---

## 2. Core Transformation Protocols

### Protocol 1 — Name the vase before you open the file
Write the types, the error variants, and the two or three signatures first, as a skeleton whose every body is `todo!()` / `raise NotImplementedError` / `panic("unimplemented")`. If you cannot name the envelope in one screenful, you do not yet know the problem — you know one flower.

### Protocol 2 — Enumerate the flower-beds, then stop
List the payload sites by name before writing any of them (`fetch`, `decode`, `classify-retryable`, `map-to-domain`). Each is one bed. Beds are filled separately and reviewed separately; two beds in one edit means two decisions in one commit.

### Protocol 3 — Build the error envelope before the success path
The success path is interpolable; the envelope is not, and it is the part that determines every caller's control flow. Decide first: is this a **closed sum type** (`Result<T, E>` with named variants), a **typed exception hierarchy**, or a **status-plus-detail struct**? Then write it, with each variant's payload — `Timeout{deadline}`, `Decode{offset, cause}`, `Refused{status, retry_after}`. A variant with no payload is a variant that will be re-diagnosed in production.

### Protocol 4 — Make illegal states unrepresentable, so the flowers cannot deform the vase
If "connected but not authenticated" is impossible, there is no `connected: bool, token: Option<T>` — there are two states. Sum types are the vase's walls: they mean the payload physically cannot leak an unmodelled state, and the type checker enforces them for free, forever.

### Protocol 5 — Own the lifecycle before the body
Pair every acquisition with its release in one lexical unit — `defer`, `with`, RAII, `try/finally`, scope guard — and route the ambient services explicitly (Protocol 2's *water*). No payload function may construct its own clock, transport, or RNG; those arrive as parameters or injected ports. Cost: one extra parameter per body. Return: the body becomes executable against a fake world.

### Protocol 6 — Fence concurrency with a schedule, not a hope
Decide and write down, in the vase: **bounded** fan-out, **bounded** queues with explicit backpressure behaviour, the atomicity boundary (what is one transaction), the ordering guarantee (per-key, global, none), the idempotency key, and the cancellation propagation path. If the answer is "whatever the runtime does," it is not a vase — it is a shape that will be discovered by the incident.

### Protocol 7 — Compile the vase empty; that is the acceptance gate
The skeleton must type-check, lint, and pass the project's build with every body unimplemented. This is the vase-first gate, and it is cheap: it proves the envelope is complete and self-consistent *before* a single flower can hide a gap. Flowers-first has no equivalent gate — hence its failures are late.

### Protocol 8 — Fill one bed per edit, vase unchanged
Each payload change should be additive inside an existing frame. If filling a bed requires changing a type, an error variant, or a guard, the edit is two edits; split it.

### Protocol 9 — The flower may not bend the vase
Contract changes — new error variant, widened type, new lifecycle rule, changed ordering guarantee — are their own change, with their own rationale and blast-radius. Never smuggle a contract change through a payload diff; that is how one flower's convenience becomes N callers' liability.

### Protocol 10 — Validate at the rim, once
Every untrusted input is admitted or rejected at the boundary, with the rejection carrying a structured reason. Interior payloads may then assume well-formedness instead of re-checking it. Scattered validation is the signature of a vase with no rim.

### Protocol 11 — The vase is observable; the flowers are not
Stage tags, error-variant metrics, retry counts, deadline-remaining, trace context: emitted at the frame, once, so every bed inherits them. Instrumentation placed inside payloads dies whenever a payload is rewritten.

### Protocol 12 — Retire the scaffolding deliberately
Stubs, feature flags, shadow paths, and placeholder variants are *temporary vases*. Name them, track them, delete them on a stated trigger. Scaffolding that outlives its build becomes a second truth.

### Protocol 13 — STOP Signals
Halt and re-scaffold when you catch yourself doing any of these:

- Adding a second `catch`/`if err != nil` per call site to compensate for an ambiguous envelope.
- Writing `map[string]any` / `interface{}` / `dict` "for now" in a type position.
- Putting `time.Sleep`, an inline HTTP call, or a bare goroutine spawn inside domain logic.
- Reaching for input length to size a fan-out or a buffer.
- Making a payload edit that touches a shared type "in passing".
- Discovering on the fifth flower that the vase needed a different rim.

### Conversion Table: Anti-Pattern → Clean Replacement

| Anti-pattern (flowers-first) | Clean replacement (vase-first) |
|---|---|
| `func Handle(x any) (any, error)` | Named request/response types in the frame; payload maps between them. |
| `error` reused for 6 distinct causes | Closed error enum with per-variant payload (`Timeout{deadline}`, `Decode{offset}`). |
| `try { … } catch { return null }` | Exhaustive envelope handling: every variant has a defined caller behaviour, checked by the compiler. |
| `map[string]any` / `dict` / `Record<string, unknown>` "temporarily" | A contract type at P1; ad-hoc shapes stay inside one bed, never across a boundary. |
| `http.Get(url)` inline | Transport injected as a port; deadline, retry policy, and pool configured in the vase. |
| `resp.Body` closed far below (or never) | Scope guard pairs acquisition and release in the same lexical unit. |
| `len(items)` fan-out with `errgroup`/`Promise.all` | Bounded worker pool + bounded queue + declared backpressure policy. |
| Retry wrapped around a whole request | Idempotency key in the envelope; retry only variants marked `retryable`, with a deadline budget. |
| Validation inside each payload step | Single rim validator; interior assumes well-formed input. |
| `panic!` / uncaught throw inside a lock or critical section | Critical section returns an envelope value; the guard releases on every exit path. |
| Metrics/log lines copied into each handler | Stage + variant tags emitted once at the frame; payloads inherit them. |
| Contract tweak bundled into a payload PR | Separate change with rationale, migration note, and blast-radius list. |

---

## 3. Engineering Application Scenarios

### Scenario A — Code Reviews: audit the vase before reading a single flower

Establish review order explicitly, and reject the PR on vase grounds first:

1. **Envelope audit.** List the failure variants the change can produce, and check each has a defined caller. Any new failure introduced without a variant is a blocker regardless of passing tests.
2. **Lifecycle audit.** For every acquired resource, find its release in the same lexical unit. `Body`/lock/tx/file/conn with no paired release is a blocker.
3. **Boundary audit.** Confirm untrusted input is admitted or rejected once, with a structured reason. Interior re-validation is a smell, not a defence.
4. **Schedule audit.** For any concurrency: find the bound, the ordering guarantee, the atomicity boundary, the idempotency key. "It's fine because the queue drains" is not an answer.
5. **Payload audit — last.** Only now read the domain logic for correctness. Correct flowers in a cracked vase are a future incident, not a review pass.

Reviewer evidence block to paste into the review:

```text
## Vase review — pipeline/reconcile

Envelope   variants: NotFound | Timeout{deadline} | Refused{status} | Decode{offset} | Conflict{key}
           new this PR: Conflict{key}  → callers updated? 3 of 3  ✅
Lifecycle  tx released via defer in all 2 paths; body closed by scope guard   ✅
Boundary   rim validate() on tenant_id + batch_seq, single site               ✅
Schedule   bounded pool 8; queue 256 with backpressure→Refused; per-key order ✅
           idempotency key (tenant_id, batch_seq) present in envelope        ✅
Payload    reconciliation rule reads correctly; no ambient clock reached      ✅
Verdict    vase closed; flowers approved
```

Blocking comments to write verbatim when the vase is open:

- `Blocker: this error is distinguishable only from a log string. Add a variant with the payload, or the caller cannot act on it.`
- `Blocker: acquisition and release are 90 lines apart with two early returns between them. Pair them in one scope.`
- `Blocker: fan-out is sized by input length. Bound it and state the backpressure behaviour, or this is an unbounded-memory change.`

### Scenario B — PR Descriptions: publish the vase diff and the flowers diff

A vase-first PR description has two visibly separate sections, because a reviewer's risk assessment differs for each. The vase section is where the *reasoning and blast radius* go; the flowers section is where the *behaviour* goes.

Template:

```markdown
## Vase (structure — decide first, review first)

**Contracts**    `FetchError` gains `Conflict{key}`; `User` unchanged.
**Envelope**     Conflict is non-retryable; mapped to HTTP 409 at the rim.
**Lifecycle**    Transport now owned by `Client`; body closed by scope guard.
**Schedule**     Bounded pool 8, queue 256, backpressure surfaces as Refused.
**Deadline**     Single budget from the rim; no stage re-invents a timeout.

Blast radius: 3 callers of `FetchUser`; 1 telemetry dashboard keyed on `FetchError`.
Vase-only commit: `a1b2c3d` — compiles with every body unimplemented.

## Flowers (payload — additive, vase frozen)

- `mapResponse` now returns `Conflict` on 409 instead of `Refused`.
- `classifyRetryable` excludes `Conflict`. Behaviour change: duplicate writes stop.

## Verification

- `cargo test --locked` → 412 passed (3 new envelope fixtures).
- `replay --trace prod-2026-03-11` → 0 unclassified failures (was 214 as Refused).
```

Rules for the description:

- **Vase before flowers, always, in that order.** The reader's risk model is built by the envelope, not the rule.
- **State the blast radius, not the line count.** "3 callers, 1 dashboard" is a decision input; "+180/−42" is not.
- **Name the vase-only commit hash.** It proves the container existed before the payload and can be reviewed in isolation.
- **Declare behaviour changes explicitly** ("duplicate writes stop") rather than describing the code delta.

### Scenario C — Architecture RFCs and ADRs: the envelope section is normative, the payload section is illustrative

The vase-first RFC inverts the usual emphasis. The structural envelope is *normative* (implementations must conform). Domain algorithm details are *illustrative* (any implementation meeting the envelope is acceptable). This is what makes one RFC survive three provider swaps.

```text
RFC-019  Multi-provider user fetch

Status        Proposed
Context       Each service fetches users with its own ad-hoc HTTP call: no
              deadline, no envelope, indistinguishable failure modes. 4 modules,
              7 call sites, 3 retry policies, none bounded.

Decision (normative — the vase)
  D1  Envelope   FetchError = NotFound | Timeout{deadline} | Transport{retryable}
                            | Decode{offset} | Refused{status,retry_after}
                            | Conflict{key}
     Rationale    Callers must act differently per variant; Retry-After must
                  survive the boundary or the schedule is a guess.
     Rejected     Single Error{code,string} — forces string-matching upstream.
  D2  Lifecycle  One Client owns a connection pool; every Body closed by scope
                  guard; deadline budget originates at the rim, never extended.
  D3  Schedule   Bounded pool (8/proc, tunable), bounded queue (256),
                  backpressure surfaced as Refused — never as silent drop.
  D4  Boundary   Validation once, at the rim; interior assumes well-formed input.
  D5  Change     Contract changes require a new RFC revision; payload changes do
                  not. Envelope versioning: additive variants only.

Illustrative (non-normative — the flowers)
  I1  Retry policy: exponential, jittered, capped at the deadline; Transport and
      Refused{429} only. Implementation is not fixed by this RFC.
  I2  Caching: SWR at the client rim; not visible to the envelope.

Consequences  ~2 weeks of scaffolding before first payload port; then provider
              swaps become payload-only. 7 call sites collapse to 1 envelope.
Migration     1. Land vase with unimplemented bodies + envelope fixtures (gate).
              2. Port call sites one per PR. 3. Delete legacy call paths.
Rejected      Adapter-per-service with shared envelope — 4 pools, 4 schedules.
Blocking      412 existing contract fixtures must pass against the vase skeleton.
```

The RFC rule: **if a paragraph describes an algorithm, it belongs in the illustrative section; if it describes a type, an error, a lifetime, or a bound, it is normative and must be reviewable without any payload existing.**

---

## 4. Verification Checklist

- [ ] **Envelope closed and compile-checked.** Every failure the code can produce has a named variant with a payload, and every call site handles it exhaustively — no string matching, no `catch`-all, no `nil`/`null` returned under a `Result` signature.
- [ ] **Scaffold compiles empty.** The vase type-checks, lints, and builds with every payload body unimplemented; the vase-only commit exists and is reviewable in isolation.
- [ ] **Lifecycle paired and ambient services routed.** Each acquisition has its release in the same lexical unit; no payload constructs its own clock, transport, or RNG — all arrive as parameters or injected ports.
- [ ] **Concurrency has a written schedule.** Fan-out bound, queue bound, backpressure behaviour, ordering guarantee, atomicity boundary, idempotency key, and deadline propagation are all stated — never "whatever the runtime does."
- [ ] **One bed per edit, no vase bending.** Payload changes are additive; any type, variant, or guard change is a separate change with its own rationale and blast-radius list; rim validation happens once and no scaffolding stub outlives its stated deletion trigger.
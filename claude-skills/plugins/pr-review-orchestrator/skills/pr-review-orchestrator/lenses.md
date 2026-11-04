# The lens roster

Seven core reviewers plus three verification stages. Adapt the specifics to the stack (the right column is
stack-agnostic). Not every PR needs all of them — the Step 0 triage kills any lens with no files in its
class before an agent is launched; a backend-only PR skips frontend entirely.

## Grouping (the token lever)

Lenses that read the same material share one agent. Every fresh agent otherwise re-reads the full diff and
the same hot files — grouping cuts a large-PR campaign from ~10 agents to ~4 while keeping each headspace
intact:

- **A — correctness**: backend/business-logic + architecture/plan + forward-compat (all read the core code
  and the plan docs).
- **B — attack surface**: security + devops/ops (+ frontend/a11y only if triage shows UI files).
- **C — proof**: tests & integration + empirical execution (they share the suite and scratch-DB setup).
- **D — verdict**: adversarial verification + ship-gate (ship-gate reads exactly what adversarial just
  verified).

Group only lenses that share reading material, and cap at two or three — "general correctness + security +
tests" in one brief produces shallow all three. A–C are independent: fire them concurrently. D fires only
after A–C are posted.

## Core lenses

### 1. Backend / business logic (group A)
Claims the domain rules are right.
- Trace each user-visible flow end to end against its spec (requirements doc, issue, legacy code, ticket).
  Name the spec source in every finding — "diverges from <issue #4 / legacy login()>".
- Concurrency: every mutating path locks and re-checks; double-submit races; idempotency.
- State machines: illegal transitions reachable? Partial-failure leaves consistent state?
- Data integrity: cascade/orphan behavior, unique constraints vs application-level checks, soft-delete rules.
- Parity checks: if porting/replacing behavior, verify against the *actual source* (the legacy function, the
  old API's responses), not its documentation.

### 2. Architecture & plan coherence (group A)
Claims the change fits the design and the design docs tell the truth.
- Does the implementation match the architecture docs / ADRs / plan (and where the plan is now stale —
  doc-updates-you-forgot is a recurring finding class).
- Layering rules actually enforced (by lint/AST guards, not convention)?
- Abstraction altitude: logic in the right layer, no shortcuts around the seams.
- Duplicated sources of truth (config constants in two places, doc-mandated files that don't exist).

### 3. Tests & integration (group C)
Claims the tests prove what they claim.
- Assertion strength vs smoke (status-200-only, tautologies, tests re-deriving expectations from the code
  under test).
- **Mutation spot-checks**: delete/invert a behavior, run the suite. Survivors are the findings.
- Fix-pinning: do previously-fixed bugs have regression tests?
- Integration with the existing skeleton: new code vs established seams; does anything regress a prior fix?
- Coverage honesty: which tests silently skip in CI; DB/service dependencies handled fail-not-skip.

### 4. Security (group B)
- AuthN/AuthZ per endpoint (IDOR, object-level checks re-derived server-side, never trusting client state).
- Sessions/tokens: fixation, rotation on privilege change, expiry semantics on long-lived connections.
- Input handling end to end: where untrusted input is rendered/executed/stored (follow it across layers).
- Resource control: unauthenticated AND authenticated DoS (expensive endpoints without rate limits/caps —
  check every layer that could throttle).
- Secrets: where they live, where they leak (logs, error messages, audit trails, client bundles).
- Outbound calls (SSRF shapes, timeouts, failure handling); SQL/DDL executed against shared resources.
- New dependencies: pinning, trust.

### 5. Frontend / UI / a11y (group B — only if triage shows UI files)
- The established baseline survived the change (focus management, live regions, semantics, keyboard paths).
- New interactive surface: keyboard-operable, state exposed programmatically (not color/title-only),
  announced changes, contrast computed from actual tokens.
- DOM replacement (fragments/swaps): focus destruction, silent reorders.
- XSS surface: all user data via textContent/equivalent; autoescape never bypassed.

### 6. DevOps / operations (group B)
- Deploy path: what happens on merge + first deploy (build steps, migrations on shared resources, env
  assumptions)? Is any safety rule enforced mechanically, or only written in prose?
- Day-2: logs/metrics for the new feature (can on-call distinguish failure shapes?), rollback story,
  scheduled jobs under failure, upgrade with live state.
- Config: per-env defaults, env-var validation (fail-open knobs?), secrets handling in CI/deploy.
- Capacity: connection/thread/memory arithmetic vs expected load.

### 7. Forward compatibility (group A)
Claims the next 3 planned changes don't force rework of this one.
- Read the roadmap/next phases; interrogate each seam this PR creates against what's coming.
- **If the next phase is already open/stacked, verify the seams against its ACTUAL branch** — what the
  successor really did is the strongest forward-compat evidence there is (did it extend your mechanism by
  data changes, or restructure it? did it mount your shipped contracts without touching your files?).
- Vocabulary extensibility (event types, enums, view registries): adding the next item is data, not surgery.
- Scale cliffs: what breaks at 10× data / 10× users (paginate-vs-fanout, per-item work).
- Migration/upgrade chains: does the next step stack cleanly on this one?

## Verification stages (groups C–D)

### 8. Empirical execution (group C)
Run the thing. Clone/setup in a scratch environment, drive the user flows, capture verbatim messages,
timings, and response headers; reproduce failure paths deliberately. Findings classes reading can't reach:
boot failures, wrong default configs, broken deploy sequences, "works in tests, dead in prod shape".
Also: confirm/refute earlier findings with evidence — a confirmed finding with verbatim output is unarguable.

### 9. Adversarial verification (group D)
Assume every finding is wrong; try to refute each (counter-evidence, alternate explanations, unreachable
scenarios). Verdicts: CONFIRMED / REFUTED / DOWNGRADED (+ what survives). Mark execution- and
mutation-proven items as ground truth and don't re-run them — spend skepticism on read-verified ones.
Output: verdict table + corrections to append + fix-packaging guidance (which findings must land as one
change to actually close). Expect: findings narrowed, strengthened, occasionally refuted; refuting half a
finding is as valuable as confirming it.

### 10. Ship-gate (group D)
One question: merge now? Read the whole fixed/current state, verify fix interactions and collateral, sort
all findings into blocks-merge / before-deploy / ride-along, and rule: MERGE / MERGE-WITH-N / DO-NOT-MERGE.
Key judgment: "deploy-gate findings don't block a merge" holds **unless the PR's own merge performs the
unsafe act** (a deploy step or migration that runs on merge) — then the gate must be mechanical in this PR.
Name the residual blind spots honestly (the never-booted boundary, the stubbed runtime, the trusted corpus).

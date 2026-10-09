---
name: ak-test
description: Create meaningful behavior coverage, audit weak tests, and optimize CI by measured blast radius. Use for test creation, failing tests, redundant or tautological tests, expensive CI, or ak:test create/audit/optimize/optimize --ultra requests.
---

# AK Test

Optimize confidence per unit of maintenance and CI cost. Treat green CI as evidence only for the behavior actually exercised. Never equate test count or line coverage with correctness.

## Dispatch

Interpret `ak:test` as a logical command, not a universally registered slash command. Accept `ak-test`, `$ak-test`, `/ak-test`, natural language, and a configured `/ak:test` alias.

- `create [scope]`: inspect the project and implement only missing meaningful coverage.
- `audit [scope]`: report findings without modifying files. With explicit repair/cleanup authorization, implement evidence-backed repairs/removals.
- `optimize [scope]`: measure, implement bounded CI/test improvements, then compare results.
- `optimize --ultra [scope]`: generate five independent proposals and have a separate verifier choose; follow [ultra.md](references/ultra.md).
- No mode: default to read-only `audit`. Reject unknown flags with supported usage; never infer permission to weaken checks.

Follow repository instructions and existing toolchain. Keep this skill stack-neutral. Read [integration.md](references/integration.md) when installing or adapting invocation.

## Non-negotiable integrity rules

1. Never skip, disable, quarantine, comment out, rename out of discovery, mark expected-failure, add early returns to, or remove a failing test simply to make CI green. Require explicit user authorization for any new skip/quarantine; record test ID, reason, owner, issue, expiry, and remaining protection. Existing skips must be reported, not silently approved.
2. Never weaken assertions, widen tolerances, swallow exceptions, blindly bless snapshots, lower thresholds, use pass-with-no-tests, add continue-on-error, mask exit codes, or filter failures out to satisfy CI. Change expectations only when an independently established contract intentionally changed; document old/new behavior and preserve unaffected contracts.
3. Investigate every failure observed. An unrelated or pre-existing failure is still an unresolved failure; report it with evidence. Do not claim success or block unrelated authorized work unnecessarily.
4. Never delete tests from grep results, age, low runtime, mock usage, or coverage percentage alone. A trivial-looking test may guard a critical boundary or historical regression.
5. Do not invent execution, benchmark data, savings, flaky-test diagnoses, approvals, coverage, or independent verification. Label unrun commands and estimates.
6. Preserve user changes. Use isolated temporary worktrees for deliberate faults or base-revision comparisons. Never mutate production data or globally reset a working tree.

## Scout before changing tests

Inspect repository instructions, manifests, test discovery/configuration, CI workflows, changed files, contracts, callers, and existing nearby tests. Identify real test commands from project evidence, not assumed framework defaults. Record revision and working-tree state.

For a diff-based scope, inspect staged and unstaged changes and relevant untracked source files; for a PR use the merge-base against its target and both sides of renames/deletions. Do not assume HEAD~1 is the PR base. If base, history, dependencies, or scope are unavailable, state the limitation and broaden verification.

Write a compact decision table before implementation:

| Changed behavior / contract source | Plausible failure | Existing test IDs and evidence | Lowest reliable layer | Decision and reason |
| --- | --- | --- | --- | --- |

Answer: What behavior changed? What can break? Which existing tests already prove it? What is the lowest-cost layer that reliably observes the risk? Is another test needed?

Choose unit tests for pure rules, integration tests for persistence/transactions/serialization/wiring, contract tests for service boundaries, and E2E for essential user journeys. Do not replace real boundary coverage with mocks just because it is faster. A justified no-new-test outcome is valid.

## Create

1. Derive expected outcomes from requirements, public contracts, bug reproductions, or domain invariants independently of the implementation. If ambiguous, resolve the specific contract before encoding it.
2. Reuse or strengthen existing tests where possible. Add the smallest set covering distinct risks; include relevant boundaries, invalid input, authorization, failure propagation, retries/idempotency, or concurrency only where the changed behavior warrants them.
3. Exercise the real system under test. Mock external nondeterminism at a boundary, not the behavior under test. Assert observable results, state, events, or required interactions. Assert asynchronous work is awaited and failures propagate.
4. For regressions, show the test fails on the faulty revision and passes after the fix when feasible. Otherwise introduce one realistic, temporary targeted fault in isolation and show the assertion detects it. Avoid unrelated import/setup failures as proof. If infeasible, state the missing red-phase evidence.
5. Run the focused tests, then affected integration/dependent lanes. Stop expanding once justified confidence and repository gates are met. Report uncovered risks.

## Handle failures

Capture the exact command, revision, test ID, error, environment, and relevant logs. Reproduce narrowly. Classify with evidence: product regression, incorrect expectation, fixture/mock defect, nondeterminism, infrastructure, or unresolved.

Compare the base revision in isolation when useful; do not label a failure pre-existing without reproduction or reliable historical evidence. For suspected flakes, inspect time, ordering, randomness, shared state, races, and external dependencies. A successful retry does not erase the original failure. Bound diagnostics; do not retry until green.

Fix the cause, preserve valid assertions, rerun the failing test and affected neighbors. If blocked, report the failure and required next step; never hide it behind a passing subset.

## Audit

Read [audit.md](references/audit.md). Inspect implementation, fixtures, mocks, assertions, discovery, and available history together. Search is triage only.

Report each finding as: severity; file:line/test ID; protected behavior; evidence; plausible escaped bug; recommended keep/strengthen/merge/replace/remove action; confidence. Prioritize false-green checks and missing high-impact behavior above cosmetic duplication.

For authorized removal, record a deletion ledger: removed test, reason, surviving test IDs or proof of absent behavior, unique risks retained, and verification commands. Similar tests at different boundaries are not automatically duplicates. Replace the unique protection before deleting. Run surviving impacted tests and verify discovery counts to detect accidental collection loss.

## Optimize

Read [ci-lanes.md](references/ci-lanes.md). Baseline first: collect comparable recent runs, preferably at least five when available. Record sample count, revision, runner type, wall time, aggregate runner minutes, setup/install/test/upload time, cache state, retries, and slowest jobs. Use median and p95 only with sample-size caveats. If no measurements exist, add instrumentation and present a provisional plan; do not claim savings.

Prefer low-risk changes supported by measurements: remove proven redundancy, reuse expensive fixtures without state leakage, cache dependencies safely, balance shards by historical durations, cancel superseded PR runs, and select affected lanes using a verified dependency map. Parallelism can reduce latency while increasing billed minutes; evaluate both.

Use critical smoke checks on every code change, affected lanes per PR/push, and a blocking full suite for stable release candidates. Add periodic full-suite reconciliation to detect selector misses. Keep full verification until the selector is validated in shadow mode. Unknown impact means full suite.

Do not change branch protection or organization settings without authorization. Do not claim a workflow gate is enforced until the relevant settings are verified. Keep workflow edits reviewable and provide a rollback to full-suite execution.

## Final report

Return concise evidence, not private reasoning:

- Scope, changed behavior, and decision table summary.
- Files/tests added, strengthened, removed, or left unchanged and why.
- Exact commands, pass/fail/not-run outcomes, and unresolved failures.
- Selected/excluded lanes, dependency evidence, and full-suite fallbacks.
- Before/after measurements with sample size; separate estimates from observations.
- Residual risks, deletion ledger if applicable, and rollback for CI changes.

Never say “all tests pass” when only a subset ran. Match the user's language for explanations; keep code and technical identifiers unchanged.

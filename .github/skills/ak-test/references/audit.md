# Test value audit

## Classify using behavior, not syntax

| Smell | Investigate | Correct action |
| --- | --- | --- |
| Literal/self equality | Does any SUT output reach the assertion? | Remove only if no protection; otherwise replace with real outcome |
| Expected value computed by SUT | Could the same defect corrupt actual and expected? | Use independently derived examples or invariant |
| Mock echo | Is the test calling a mocked method and asserting its configured return? | Exercise real caller and observable behavior |
| Overmocked integration | Can transaction, SQL, serialization, or wiring fail unnoticed? | Retain/add a real boundary test |
| Happy-path status only | Could wrong payload/state still pass? | Assert meaningful contract and side effects |
| Broad exception catch | Can the assertion fail inside a swallowed exception? | Use precise exception assertions; preserve exit failure |
| Unawaited async | Can test finish before assertion? | Await completion and exercise rejection path |
| Snapshot churn | Was an intentional contract change established? | Inspect diff; retain meaningful assertions |
| Duplicate | Same risk, inputs, layer, oracle, and failure sensitivity? | Merge only after proving surviving coverage |
| Outdated | Is behavior actually retired, or only moved? | Link change/contract before removing |
| Skip/xfail/only | Does discovery/reporting conceal failures or tests? | Report; repair cause; require approval for new exclusions |
| Flake | Is nondeterminism reproduced and understood? | Isolate cause, not retry-until-green |

## Counterexamples

Weak: stub `repository.save` to return `42`, call that stub directly, assert `42`.
Useful: invoke the real service, assert persisted business state via the actual database when transaction behavior matters.

Weak: `expected = calculate_total(items); assert calculate_total(items) == expected`.
Useful: derive a total from documented pricing/rounding rules; assert explicit boundary values through the public function.

Do not discard property/metamorphic tests merely because they compare two computations. Commutativity, conservation, round-trip, or monotonicity can be independent domain invariants. Ask which realistic defect they detect.

A constant assertion can be meaningful if the constant is the observable result of actual production behavior. A mocked interaction can be meaningful if invocation, ordering, or absence of side effects is the contract. Preserve authorization denials and historical regressions even when simple.

## Removal evidence

For each candidate, identify what mutation or realistic regression its replacement would detect. Run a targeted counterexample where practical. Preserve complementary integration and unit tests. If unique value is uncertain, retain the test and explain the uncertainty.

Audit is read-only by default. Cleanup permission is not permission to silence failing tests or introduce skips.

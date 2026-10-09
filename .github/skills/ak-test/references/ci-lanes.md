# CI selection and cost protocol

## Model lanes from the real dependency graph

Record for each lane: ID, owned paths, upstream dependencies, test IDs/command, boundary covered, runner, estimated duration, always-run condition, and full-run condition. Use repository-native affected tooling if available and verify its handling of implicit dependencies. Do not infer independence from directory layout alone.

Example design, not a ready-to-run workflow:

| Lane | Typical scope | Triggers |
| --- | --- | --- |
| smoke | startup and critical invariants | every code/config change |
| domain | pure business rules | changed module plus reverse dependencies |
| persistence | migrations, queries, transactions | schema/ORM/database consumers |
| contract | API/messages/serialization | producer and every known consumer |
| journey | critical end-to-end behavior | affected cross-boundary user flows |
| full | all required checks | uncertainty, release candidate, reconciliation |

## Fail-safe selection algorithm

1. Resolve a trusted base and tested head for the event. On PRs use target merge-base and validate the actual merge candidate where required. On pushes use the event range; handle first push/all-zero base. Include both old and new paths for renames, deleted files, and relevant untracked files for local use.
2. Build changed-path set with NUL-safe Git output. Missing history/base, truncated change lists, parser errors, or invalid map => full suite (or fail the selector, never green empty output).
3. Check global triggers first: CI/selector/test configuration, build scripts, shared fixtures, root manifests/lockfiles, toolchains, shared auth/security, schema/contracts, generated-code inputs, and unknown files. Default to full suite unless a verified map proves all affected consumers.
4. Traverse reverse/transitive dependencies, including runtime wiring, generated artifacts, shared configuration, data schemas, and implicit resource dependencies.
5. Select union of affected lanes plus critical smoke. Include changed tests and their owning lanes. Explicitly allow docs-only exclusion only for known non-executable docs with no generated code/build effect.
6. Emit selected lane IDs, reasons, revision IDs, mapping version, and full-suite flag as machine-readable output. Reject unknown lane IDs; never execute commands derived from untrusted filenames.
7. Confirm selected jobs actually execute and collect expected tests. Unexpected zero discovery is failure. Log intentionally excluded lanes with the rule and evidence.

Lane selection is not test disabling: exclusions require a validated no-impact policy. New ad hoc exclusions for failing tests require user authorization. If no verified policy exists, keep running full coverage while introducing the policy in shadow mode.

## GitHub Actions implementation requirements

Adapt actual workflows, never paste a generic passing placeholder. Preserve least-privilege permissions, action pinning conventions, fork safety, and secrets isolation. Do not execute untrusted PR code in a privileged pull_request_target context.

Use a stable aggregate required check that always runs and depends on selection and all lane jobs. It must fail on selection failure/cancellation, any selected lane failure/cancellation/unexpected skip, or missing results. An intentionally unselected job may be skipped only when selection succeeded and evidence marks it unselected. Avoid workflow-level path filters that leave required checks pending. Handle merge_group when a merge queue is used.

Run the full suite against the exact stable release candidate SHA before promotion. A full run after publication is too late. Add scheduled/manual full runs to reconcile selector decisions; retain current full-suite gates during rollout. Detect missed failures in omitted lanes and revert selection to full until repaired. Do not let an empty matrix or success from an unrelated job satisfy the gate.

Cache dependencies with keys covering OS, runtime, lockfiles and relevant config. Never reuse stale passing test results as evidence for changed code. Avoid secrets in caches/artifacts. Cancel superseded PR work only; preserve required release verification.

## Validation and rollback

Verify representative diffs: leaf change, shared library consumer, migration, auth, root lockfile, renamed/deleted file, test-only change, docs-only change, unmapped path, shallow history, failed selector, cancelled/failed/skipped selected job, empty discovery, release event. Use controlled failures to verify the aggregate gate itself fails.

Shadow-run selected and full suites on representative commits before enabling exclusions. Retain logs for reconciliation. If timing data is sparse, report a limited baseline; never convert predicted savings into measured results.

Compare equivalent runner/cache conditions and account for retries, sharding setup duplication, billed-minute rounding and actual runner prices when calculating monetary costs. Without billing data report runner minutes, not invented dollars. Track p50/p95 wall time and total runner minutes separately. A smaller critical path is not automatically cheaper.

Rollback: force full=true or restore the prior workflow while retaining integrity fixes. Record exact rollback patch/commit and trigger criteria (missed regression, selector uncertainty, unexplained collection loss).

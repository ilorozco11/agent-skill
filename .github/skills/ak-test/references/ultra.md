# Ultra: independent proposals and verification

Use only for `optimize --ultra`. Do not execute five competing CI rewrites.

1. Capture a shared immutable evidence packet: revision/diff, repository instructions, behavior/risk map, test inventory, workflow/config, timing samples, constraints and available budget. Redact secrets. If execution cost is material and no budget exists, agree a bounded budget first.
2. Launch five independent proposal agents when the host permits subagents. Give each only the evidence packet and the same goal: propose the strongest measured optimization while preserving detection. Do not give other proposals or a preferred conclusion. They must not modify shared files. Each returns bottleneck evidence, approach, expected latency AND runner-minute effect, coverage proof, dependencies, experiment and rollback.
3. Require five meaningfully different approaches. If convergent, ask for alternatives and record convergence honestly; never fabricate independence.
4. Send raw evidence and anonymized proposals to a separate verifier that did not author a proposal. Give it no preferred winner. First reject anything that weakens assertions, bypasses failures, cannot map impact, or depends on invented measurements. Score eligible options on protection (40%), measured cost/latency benefit (25%), maintainability (20%), and rollout/reversibility (15%). Label scores as judgments. It may reject all proposals or require more evidence.
5. Implement only the selected bounded plan. Run the baseline comparison, selector/gate checks, and affected/full validation required by the rollout. Verifier reviews actual diff and results before accepting. Publish short candidate comparison, selected rationale, measurements, uncertainty, and rollback.

If independent agents/verifier are unavailable, explicitly report `ultra degraded: independent verification unavailable`. Offer five sequential alternatives with a separately labeled self-review, but do not claim five independent agents or verifier approval. Continue safe baseline collection; do not silently present ordinary optimize as fully verified ultra.

# Grounding commit — Day 1

**Asker:** Kidane Gebremedhin Gidey
**Date:** 2026-05-05.

> **[FILL AFTER EVENING CALL — pointer to the actual edit made to a Week 10 or 11 portfolio artifact, with one paragraph explaining what changed and why.]**

---

**Edit:** `[path/to/artifact]` — `[lines or section heading]` — `[git commit hash, once committed]`.

**One-paragraph explanation:**

> Before the explainer, the [memo paragraph / model card section / probe / agent code] said [old language] which silently assumed [unstated mechanism assumption — e.g. that the provider's fast-mode latency advantage would persist if the model were ported to a self-hosted serving stack]. After understanding [the load-bearing mechanism the explainer named], I can now defend [the specific quantitative claim] independent of provider behavior, because [precise reason — e.g. the latency budget now decomposes into prefill cost, decode cost, and a per-call speculative-decoding speedup that is bounded by per-token target entropy on this workload]. The edit makes that decomposition visible in the artifact instead of implicit in my head.

---

## Candidate edits worth considering for this question's grounding commit

(For triage during/after the evening call — pick exactly one.)

- **Decision memo, latency / cost-per-task line.** Replace the un-decomposed latency figure with a prefill+decode breakdown plus a footnote naming whether the provider's fast mode is part of the budget, and what the budget becomes if the fast mode is dropped.
- **Observability spec.** Add the per-token signals the explainer identified as load-bearing — at minimum prefill_tokens vs decode_tokens per call, plus (where the API exposes it) a speculative-decoding acceptance-rate counter. Without these the question "is speculative decoding actually firing on this call" cannot be answered from production traces.
- **Model card / serving notes.** Add a paragraph on which generation steps in the pipeline are templated (high acceptance rate, large speedup) vs open-ended (low acceptance rate, marginal or negative speedup), so a future reviewer can see why total per-task latency does not scale linearly with output token count.

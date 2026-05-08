# Morning call summary — Day 4

**Pair:** Kidane Gebremedhin Gidey + Nurye Nigus.
**Date:** 2026-05-09.

Initial drafts on both sides were too coarse to merit a day of research. Kidane's first attempt was a single-clause "what's the right confidence interval for my dev accuracy" — sharpened in the call into a tighter framing on the train-vs-held-out gap as both a small-sample question (8.3-point gap at N_dev = 50 is barely above binomial sampling noise) *and* an adaptive-data-analysis question (the dev set has been touched ~27 times during development for checkpoint and seed selection), and a punchline that asks whether to hold out a fresh test partition now or wait for the V0.2 expansion to bring labelled pairs that can become a proper test set. Nurye pushed back on the original framing for being too generic about "evaluation reliability" and not anchored to a specific reportable claim — the rewrite explicitly anchors to the 86.0% headline in `MODEL_CARD.md`, the 94.3% train-side number that is currently footnoted, and the ~27 dev-access count that is the empirical hook the explainer can reason about.

Nurye's first draft asked "why are my two p-values different" — sharpened into a sub-question structure that names the three estimators in play (`two_proportion_z_test`, `paired_bootstrap_ci`, `exact_mcnemar`), the load-bearing observation that `paired_task_count = 2` while the unpaired test is computing on the full counts in each arm, and a punchline that demands a written rule for *which* test applies to *which* eval setup, not just an explanation of why the current artifacts disagree. Kidane pushed for a specific deliverable — a decision-tree-style rule for any future ablation — which became the explicit final-section ask, so the answer is forced to produce a reusable artifact rather than just a one-time interpretation of the current `ablation_results.json` numbers.

Both questions are final for the day.

— Confirmed by Nurye Nigus.

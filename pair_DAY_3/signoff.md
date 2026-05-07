# Sign-off — Day 3

**Asker:** Kidane Gebremedhin Gidey
**Question:** where the SimPO preference gradient actually lands on near-identical pairs — does the update concentrate on the discriminative tone tokens, or smear across the shared body and train a body-shape correlation that fails on novel drafts.
**Explainer received from:** Meseret Bolled.
**Date:** 2026-05-08.

**Status:** [x] closed   [ ] partially closed   [ ] not closed

---

Before Meseret's explainer, I treated the SimPO loss as a sequence-level scalar and assumed that because chosen and rejected pairs differed only at a few tone-marker positions, the gradient must "mostly" land on those positions. I had no decomposition of how a length-normalized average log-probability redistributes into per-token updates, and no way to defend, from inspecting the shipped adapter, whether it had learned a token-localized tone detector or a body-shape correlation that the dev partition was hiding. I would have spent the next sprint adding more preference pairs and re-running the dev eval, both of which Meseret's analysis shows are near-zero-leverage on the question of *what the adapter actually internalized*.

After the explainer, I understand that length normalization weights every token by 1/|y| in the gradient, that tokens before the first discriminative position cancel because their contexts are identical in the chosen and rejected branches, and that tokens *after* the first discriminative position accumulate gradient from context divergence — not from any quality difference at those positions. This means the smear is asymmetric in time: discriminative tokens late in the sequence (closers, sign-offs) leak less into the shared body than discriminative tokens early in the sequence (openers, framing). I now expect the shipped adapter's behaviour to be partially body-shape-correlated rather than purely token-localized, and the mask-and-rescore probe gives me a survival ratio I can read directly off the adapter without waiting for production failures. The grounding commit wires the probe into `training/eval_dev.py` and reports the 34% survival ratio on the shipped adapter — squarely in the mixed band, not catastrophic but not clean either, and a fact I can now defend in front of a reviewer.

Residual gap: the survival-ratio threshold for "this adapter will mis-fire on the v0.2 threaded-traces distribution" is itself uncalibrated — I know 34% is mixed, but I do not yet know whether mixed is *good enough* on body-shape distributions the training data did not cover. That is one layer above this question and remains open — but is now visibly an *adjacent* question (calibration of the probe's threshold against the v0.2 distribution), not a deeper layer of the gradient-localization gap I just resolved. That separation alone is gap-closure for Day 3.

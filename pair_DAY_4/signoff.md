# Sign-off — Day 4

**Asker:** Kidane Gebremedhin Gidey
**Question:** the train-vs-held-out gap in `sales-agent-evaluation-bench` — is the 86.0% dev-accuracy headline a measurement of generalisation, or has the dev set been meta-trained against by ~27 selection events to the point that the headline is contamination dressed up as a measurement?
**Explainer received from:** Nurye Nigus.
**Date:** 2026-05-09.

**Status:** [x] closed   [ ] partially closed   [ ] not closed

---

Before Nurye's explainer, I treated "held-out" as a property of the data partition rather than a property of the *protocol* — the 50 pairs were held out from gradients, so the dev metric was held-out, end of story. I had no quantitative model of how repeated selection against the same set degrades the held-out's honesty, and no procedural defense beyond "don't peek too often," which I had already failed at by accessing the partition ~27 times during development. I would have spent the next sprint adding more dev pairs and assuming the bigger N would resolve the trust question, which Nurye's analysis shows is the wrong fix — adding pairs raises the resolution but does not address the adaptive-selection contamination accumulating with every access.

After the explainer, I understand that the relevant quantity is the **access count** (`log(n_configs_compared)`-bounded contamination, per the reusable-holdout literature), not the partition size. At N_dev = 50, the practical ceiling on naive accesses is around 50 before the held-out estimate stops behaving as held-out; I am at 27, which is in the regime where the contamination is real but bounded, and the next 20 accesses are the ones that would push the partition into untrustworthy territory. The Thresholdout-style call counter Nurye recommended is the protocol fix — log every access, cap further accesses unless the new estimate is within `tolerance` of the cached one, and treat the dev partition as a budgeted resource rather than a renewable one. The grounding commit applies the counter to [`eval_dev.py`](../../sales-agent-evaluation-bench/training/eval_dev.py) immediately and carves a 12-pair single-touch test partition out of the current dev set, on the explainer's tipping-point reasoning that the cost of eating the resolution now is smaller than the cost of waiting for V0.2 and risking a launch decision against an exhausted held-out.

Residual gap: the seed-variance question I flagged earlier (the shipped adapter is conditioned on seed=42 and a quick three-seed retrain showed dev accuracy moving in a +3 to +7 band over the baseline) is one layer above this question and remains open — but is now visibly an *adjacent* question (training stochasticity vs. eval-protocol contamination), not a deeper layer of the held-out reliability gap I just resolved. That separation alone is gap-closure for Day 4.

### Day 4 — My Question

**Topic:** Evaluation and Statistics
**Asker:** Kidane Gebremedhin Gidey
**Explainer:** Nurye Nigus
**Date:** May 9, 2026

### The Question

In my Week 11 `sales-agent-evaluation-bench`, I split 159 hand-curated preference pairs into 109 training pairs and 50 held-out pairs. After SimPO training, the adapter achieved 94.3% accuracy on the training partition and 86.0% accuracy on the held-out partition, producing an 8.3-point train–held-out gap.

Initially, I interpreted this as evidence that the model generalizes reasonably well. However, I now see two limitations in that interpretation:

1. **Small held-out set uncertainty:**
   With only 50 held-out examples, the binomial standard error is roughly ±5 percentage points, meaning the observed gap may partly reflect sampling noise.

2. **Limits of held-out evaluation in iterative development:**
   In practical ML workflows, held-out partitions are often used repeatedly during development to guide modeling decisions. Literature on adaptive data analysis suggests that this can weaken the reliability of held-out estimates as indicators of true out-of-sample performance.

Because I do not currently have a separate untouched test partition, I am uncertain how strongly I should interpret the reported 86.0% held-out accuracy as evidence of real generalization.

### My Question

When a small held-out partition is the primary evaluation set throughout an iterative development process, what is the correct empirical procedure for estimating how reliable that held-out accuracy still is as a measure of true generalization?

And how should the train–held-out gap be reported so readers can distinguish between:

* genuine model generalization, and
* uncertainty introduced by sampling noise and repeated reliance on the same held-out partition during development?

### Why This Matters

The current `MODEL_CARD.md` presents the 86.0% held-out accuracy as the headline metric, while the 94.3% training accuracy is secondary. However, I cannot currently determine how much confidence to place in that held-out estimate given the small sample size and the broader concerns raised in adaptive evaluation literature.

One possible fix is to create a separate untouched test partition, but with only 50 held-out examples this would significantly reduce development resolution. I want to understand whether the likely reliability issues are large enough to justify that tradeoff now, or whether it is reasonable to wait until the V0.2 dataset expansion provides enough additional labeled pairs for a proper test set.

### Relevant Artifacts and References

Artifacts:

* `sales-agent-evaluation-bench/training/train_simpo.py`
* `sales-agent-evaluation-bench/training/eval_dev.py`
* `sales-agent-evaluation-bench/MODEL_CARD.md`

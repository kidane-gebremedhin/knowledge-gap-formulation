# Sources — Day 3 explainer

*Two canonical sources cited in `explainer.md`, plus the supporting catalogue of LLM-judge biases, the formal frame for reward hacking, and the audit pattern used.*

## Canonical sources (load-bearing citations)

1. **Direct Preference Optimization.** Rafailov, Sharma, Mitchell, Ermon, Manning, Finn, *Direct Preference Optimization: Your Language Model is Secretly a Reward Model*, NeurIPS 2023.
   https://arxiv.org/abs/2305.18290
   *Why load-bearing:* §4 derives DPO's implicit-reward gradient and shows that the update is fully determined by the (chosen, rejected) labels and the reference policy — no representation of *what* the labels mean. This is the mechanism by which any systematic structure in the labels (including a position-biased judge's skew) gets faithfully absorbed into the policy. Without this paper's derivation, the "DPO is a label-faithful transducer" claim in the explainer would be hand-waved.

2. **Scaling Laws for Reward Model Overoptimization.** Gao, Schulman, Hilton, *Scaling Laws for Reward Model Overoptimization*, ICML 2023.
   https://arxiv.org/abs/2210.10760
   *Why load-bearing:* Figure 1 and §3 establish the proxy-vs-gold reward decomposition that the dual-eval audit in the explainer applies to Meseret's setup. The paper's empirical finding — proxy reward keeps rising while gold reward plateaus or drops with optimization pressure — is the framing that turns "is the +0.19 lift real?" from a yes/no question into a measurable gap (`Δ_fixed − Δ_rotation`).

## Supporting sources

- **Wang et al., 2023 — *Large Language Models are not Fair Evaluators*.** https://arxiv.org/abs/2305.17926. The position-bias mechanism: autoregressive judges inflate scores for first-listed criteria. Cited because Meseret's Day-1 research already established this bias exists in her tone-compliance judge; this paper is the canonical mechanism-level reference.
- **Zheng et al., 2023 — *Judging LLM-as-a-Judge with MT-Bench and Chatbot Arena*.** https://arxiv.org/abs/2306.05685. §5 catalogues position bias, length bias, and self-preference bias as the three load-bearing LLM-judge artifacts, with mitigation patterns (multi-pass swapping, calibration). Cited because the explainer recommends a length-bias check alongside the position-bias audit, and this paper is the standard reference for the joint set.
- **Skalse, Howe, Krasheninnikov, Krueger, 2022 — *Defining and Characterizing Reward Hacking*.** https://arxiv.org/abs/2209.13085. Provides the formal definition that the best-of-N diagnostic in the explainer is operationalizing: a proxy is hackable iff a policy can improve on it without improving on the true objective. Cited because this is the framing that turns "the model has learned the bias" from a metaphor into a testable claim.

## Tool / pattern used

**Criterion-rotation audit on a held-out preference-pair sample.** A Latin square over the 5 tone markers, applied as a wrapper around the existing tone-compliance judge call. Sketched usage:

```python
import itertools, random, statistics
from your_judge_module import score_tone_with_order

ORDERS = list(itertools.permutations(
    ["direct", "grounded", "honest", "professional", "non_condescending"]
))
random.shuffle(ORDERS)
ORDERS = ORDERS[:5]   # Latin-square subset; full set of 120 is overkill

def rotation_fair_label(draft, ctx):
    scores = [score_tone_with_order(draft, ctx, order) for order in ORDERS]
    return statistics.median(scores) >= PASS_THRESHOLD

audit = [
    (pair, rotation_fair_label(pair.draft, pair.ctx), pair.original_label)
    for pair in random.sample(preference_pairs, 40)
]
agreement = sum(rf == orig for _, rf, orig in audit) / len(audit)
```

The output `agreement` is the headline number for Experiment 1 in the explainer. The same `score_tone_with_order` wrapper is reused for the dual-eval (Experiment 2) by averaging across orderings instead of taking the median.

## Follow-on pointer

- **Pan, Bhatia, Steinhardt, 2024 — *Feedback Loops With Language Models Drive In-Context Reward Hacking*.** https://arxiv.org/abs/2402.06627. Becomes relevant if Meseret's tenacious-bench moves toward online preference collection (the trained agent's own outputs feeding back as new preference pairs). The paper shows that even small label-bias contamination compounds across feedback iterations, which is the worst-case version of the propagation mechanism explained here.

# Tweet thread — Day 3

*Compressed from `explainer.md`. 6 tweets. Each stands alone for a reader who never clicks through.*

---

**1/6**

A teammate asked: "I trained a SimPO critic on preference pairs where chosen and rejected differ in only a handful of tone tokens. The dev eval looks great. But I have no idea whether the adapter is actually scoring the tone tokens — or scoring a body-shape correlation that's about to break."

The answer hinges on a per-token decomposition most explanations skip.

---

**2/6**

SimPO's loss is the length-normalized average log-prob of the sequence:

`R(y) = (1/|y|) · Σ_t log π(y_t | x, y_{<t})`

Each token contributes exactly 1/|y| of the gradient. Length normalization does **not** concentrate the update on any one position — it spreads it uniformly across all N tokens.

So how does the discriminative tone token win?

---

**3/6**

On near-identical pairs (77 shared tokens, 3 discriminative), the per-token chosen-minus-rejected push splits three ways:

• **Before** the first discriminative position — contexts are identical, log-probs match, contributions cancel.
• **At** the discriminative position — large asymmetry, large signal.
• **After** the discriminative position — contexts have diverged, contributions accumulate smear unrelated to quality.

The smear is asymmetric in time.

---

**4/6**

Practical implication: if your tone markers live in the closer (late in the sequence), most of the gradient comes from cancellation + signal. Clean.

If they live in the opener (early), most of the sequence follows the divergence point and gradient piles up on the shared body — training a body-shape correlation, not a tone detector.

Position of discriminative tokens predicts how token-localized your critic ends up.

---

**5/6**

Cheap probe — `mask_and_rescore`. Compute the SimPO reward gap on a held-out near-identical pair. Mask the discriminative phrases (your scoring evaluator already knows them). Recompute the gap.

`survival_ratio = |gap_masked| / |gap_intact|`

• <20% → token-localized critic. Discriminative tokens carry the signal.
• 20-50% → mixed. Body shape contributes.
• >50% → sequence-diffuse. The body shape *is* the signal. Will mis-fire on novel body shapes.

Forward-pass-only. Runs in seconds on a held-out slice.

---

**6/6**

Ran it on my shipped adapter: 34%. Mixed band. The deterministic anti-marker filter is doing more of the safety lift than the SimPO training alone — a more honest claim than "SimPO training improved tone compliance," and one I can defend.

Full writeup: [link to blog post]

Sources:
• SimPO (Meng et al., 2024) https://arxiv.org/abs/2405.14734
• DPO (Rafailov et al., 2023) https://arxiv.org/abs/2305.18290

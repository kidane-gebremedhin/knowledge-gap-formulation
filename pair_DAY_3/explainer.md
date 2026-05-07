# Day 3 Explainer — Training and Post-Training Mechanics

**Written by:** Meseret Bolled
**For:** Kidane Gebremedhin Gidey
**Topic:** Where the SimPO preference gradient actually lands on near-identical pairs
**Date:** May 7, 2026

---

# Does SimPO Concentrate the Update on Tone Tokens — or Smear It Across the Whole Draft?

## Kidane's Question

> In a length-normalized preference optimizer like SimPO, how does the per-sequence loss decompose into per-token gradient contributions, and on preference pairs whose chosen and rejected texts are near-identical except for a few load-bearing tokens, does the update concentrate on those discriminative tokens — or does it smear across the shared body, training the critic on a global stylistic shift that merely correlates with the tone signal?

The short answer: **the gradient smears, but not uniformly, and the smear is asymmetric in time**. Tokens that appear before the discriminative token mostly cancel. The discriminative token itself carries strong signal. Tokens that appear after the discriminative token accumulate unexpected gradient because the sequence context has already diverged. Whether the smear dominates or the signal dominates depends on where in the sequence your tone markers live — and there is a cheap probe that tells you which kind of critic you shipped, without waiting for production failures.

---

## 1. The Load-Bearing Mechanism: What Length Normalization Does to Per-Token Gradients

SimPO's loss (Meng et al., 2024) is:

```
L_SimPO = -log σ( β * ( R(y_w) - R(y_l) - γ ) )

where R(y) = (1 / |y|) * Σ_t  log π(y_t | x, y_{<t})
```

R(y) is the **length-normalized average log-probability** of the sequence under the current policy π. The β scales the margin; γ is a target reward gap. This is a sequence-level scalar — one number per sequence.

The gradient of this loss with respect to model parameters θ flows through:

```
∂L/∂θ  ∝  ∂R(y_w)/∂θ  -  ∂R(y_l)/∂θ

∂R(y)/∂θ  =  (1 / |y|) * Σ_t  ∂ log π(y_t | x, y_{<t}) / ∂θ
```

**This is the key line.** Each token t contributes equally to the gradient — weighted by 1/|y|. Length normalization does not concentrate the gradient on any particular position. It distributes it uniformly across all N tokens in the sequence.

For a sequence of 80 tokens, each token gets 1/80 of the gradient weight. The discriminative token at position k gets exactly the same weight as the 79 shared tokens around it.

> **The mechanism in plain language:** SimPO treats the whole sequence as a single unit and asks "how much more probable is the chosen sequence than the rejected one, on average per token?" It does not ask "which tokens made chosen better than rejected?" That question has to be answered separately, using attribution tools.

---

## 2. The Decomposition on Near-Identical Pairs

Take a concrete near-identical pair. Chosen and rejected share 77 tokens and differ at 3 positions — the tone markers, presumptuous verbs, and overclaiming closer Kidane describes. Call the shared tokens A and the discriminative positions D_w (chosen) and D_l (rejected).

```
Chosen:   A A A A D_w A A A A A D_w A A D_w A A A ...  (78 tokens chosen, 3 discriminative)
Rejected: A A A A D_l A A A A A D_l A A D_l A A A ...  (78 tokens rejected, 3 discriminative)
```

The per-token chosen-minus-rejected contribution at each position:

**Before the first D (positions 1 to k-1):**
The context up to position t is identical in chosen and rejected — the same tokens appeared before. So:

```
log π(A_t | chosen context up to t)  ≈  log π(A_t | rejected context up to t)
```

Per-token contribution ≈ 0. These tokens mostly cancel. This is the good news.

**At the discriminative position k:**
The chosen token D_w is what the model should prefer; the rejected token D_l is what it should avoid. These tokens have very different logprobs under the policy:

```
log π(D_w_k | context)  >>  log π(D_l_k | context)
```

Large asymmetry → large per-token contribution. The discriminative token carries strong signal. This is also good news.

**After the discriminative position (positions k+1 onward):**
Here is the problem. Once the sequence diverges at position k, the context for every subsequent token is different between chosen and rejected — even when those subsequent tokens are the same shared text A. The model at position k+1 sees a different preceding token in the chosen branch than in the rejected branch.

```
log π(A_{k+1} | chosen context)  ≠  log π(A_{k+1} | rejected context)
```

These differences accumulate across all remaining positions. The gradient at every token after position k carries signal from the distributional shift caused by the discriminative token, not from any quality difference at that position itself.

**The asymmetry in time:** If your discriminative tokens appear early in the sequence (say, the opener contains the presumptuous verb), a large proportion of the sequence follows the divergence point and accumulates smear. If the discriminative tokens appear late (say, the overclaiming closer is the last few tokens), most of the sequence comes before the divergence and mostly cancels. Early discriminative tokens = more smear. Late discriminative tokens = less smear.

---

## 3. Which Effect Dominates in Practice?

This depends on three things: position of discriminative tokens, sequence length, and training volume.

**Position:** As above — earlier = more post-divergence smear.

**Sequence length:** Longer sequences dilute the discriminative token's contribution further (1/|y| weight) while giving more room for post-divergence accumulation.

**Training volume:** With many near-identical pairs, the smear at shared positions tends to average out across examples because the context varies. The discriminative token's signal, by contrast, is consistently asymmetric across all pairs. With enough data, the model leans toward token-localized detection. With sparse data (159 pairs is on the smaller side), the smear may not average out — the critic may have learned body-shape patterns that happen to correlate with the training distribution.

**The practical prediction for Kidane's setup:** 159 pairs, short format-constrained drafts, discriminative tokens in tone markers and closers (likely late in sequence) → the gradient concentration is probably reasonable, but not guaranteed. The smear on post-divergence tokens in the shared body is real and may have trained the critic to associate specific body shapes (same prospect framing, same length envelope) with tone quality — not just the tone-marker tokens themselves.

---

## 4. The Diagnostic: Mask-and-Rescore Probe

This is the cheapest way to tell which kind of critic you shipped. The logic: if the critic is doing token-localized tone detection, masking the discriminative tokens should collapse the reward gap toward zero. If the critic is doing sequence-level stylistic-shift detection, the reward gap survives masking — the body shape carries the preference signal.

```python
import torch
from transformers import AutoModelForCausalLM, AutoTokenizer

def sequence_reward_simpo(model, tokenizer, text, device="cpu"):
    """
    Compute the length-normalized average log-probability of a sequence.
    This is R(y) in SimPO: (1/|y|) * sum_t log pi(y_t | x, y_{<t})
    """
    tokens = tokenizer(text, return_tensors="pt").to(device)
    with torch.no_grad():
        outputs = model(**tokens)
    logits = outputs.logits                            # (1, seq_len, vocab)
    log_probs = torch.log_softmax(logits, dim=-1)

    # Shift: predict token t+1 from position t
    input_ids  = tokens.input_ids[0, 1:]              # tokens to predict
    pred_probs = log_probs[0, :-1]                    # logits at prediction positions

    token_log_probs = pred_probs.gather(
        1, input_ids.unsqueeze(1)
    ).squeeze(1)                                      # (seq_len - 1,)

    return token_log_probs.mean().item()              # length-normalized


def per_token_reward_gap(model, tokenizer, chosen_text, rejected_text, device="cpu"):
    """
    Compute per-token contribution to the SimPO reward gap.
    Returns list of (token_string, contribution) pairs.
    Positive = chosen pushes up here. Negative = rejected pushes up here.
    """
    def token_lps(text):
        tokens = tokenizer(text, return_tensors="pt").to(device)
        with torch.no_grad():
            logits = model(**tokens).logits
        lps = torch.log_softmax(logits, dim=-1)
        ids = tokens.input_ids[0, 1:]
        return lps[0, :-1].gather(1, ids.unsqueeze(1)).squeeze(1), ids

    lp_w, ids_w = token_lps(chosen_text)
    lp_l, ids_l = token_lps(rejected_text)

    n = min(len(lp_w), len(lp_l))
    per_token = (lp_w[:n] - lp_l[:n]) / n           # length-normalized contribution

    tokens = [tokenizer.decode([i]) for i in ids_w[:n]]
    return list(zip(tokens, per_token.tolist()))


def mask_and_rescore(model, tokenizer, chosen_text, rejected_text,
                     discriminative_phrases, device="cpu"):
    """
    Mask out discriminative tokens and check if the reward gap survives.

    discriminative_phrases: list of strings (tone markers, banned phrases)
    that scoring_evaluator.py already knows about.

    Interpretation:
      gap_intact >> 0 and gap_masked ≈ 0  → token-localized detector (good)
      gap_intact >> 0 and gap_masked >> 0 → sequence-diffuse detector (smear problem)
    """
    def mask_text(text, phrases, placeholder="___"):
        for phrase in phrases:
            text = text.replace(phrase, placeholder)
        return text

    r_chosen  = sequence_reward_simpo(model, tokenizer, chosen_text, device)
    r_rejected = sequence_reward_simpo(model, tokenizer, rejected_text, device)
    gap_intact = r_chosen - r_rejected

    chosen_masked  = mask_text(chosen_text,  discriminative_phrases)
    rejected_masked = mask_text(rejected_text, discriminative_phrases)

    r_chosen_m  = sequence_reward_simpo(model, tokenizer, chosen_masked, device)
    r_rejected_m = sequence_reward_simpo(model, tokenizer, rejected_masked, device)
    gap_masked = r_chosen_m - r_rejected_m

    survival_ratio = abs(gap_masked) / (abs(gap_intact) + 1e-9)

    print(f"Reward gap (intact):  {gap_intact:+.4f}")
    print(f"Reward gap (masked):  {gap_masked:+.4f}")
    print(f"Survival ratio:       {survival_ratio:.2%}")
    print()
    if survival_ratio < 0.20:
        print("VERDICT: Token-localized — discriminative tokens carry the preference signal.")
    elif survival_ratio < 0.50:
        print("VERDICT: Mixed — partial smear. Body shape contributes but is not dominant.")
    else:
        print("VERDICT: Sequence-diffuse — body shape is carrying the preference signal.")

    return gap_intact, gap_masked, survival_ratio
```

**Running it against Kidane's shipped adapter:**

```python
# Load your LoRA-adapted critic
from peft import PeftModel

base = AutoModelForCausalLM.from_pretrained("Qwen/Qwen2.5-0.5B-Instruct")
model = PeftModel.from_pretrained(base, "path/to/your/lora/adapter")
model.eval()
tokenizer = AutoTokenizer.from_pretrained("Qwen/Qwen2.5-0.5B-Instruct")

# Use scoring_evaluator.py's banned phrase list as discriminative tokens
# These are exactly the tokens that distinguish chosen from rejected
discriminative_phrases = [
    "top talent", "world-class", "a-players", "rockstar", "ninja",
    "exciting opportunity", "leverage", "synergies",
    "aggressive hiring", "rapidly scaling your team",
]

# Take a held-out probe pair
chosen  = "We noticed Stripe is building out its ML infrastructure team..."
rejected = "We noticed Stripe is aggressively scaling its world-class ML team..."

mask_and_rescore(model, tokenizer, chosen, rejected, discriminative_phrases)
```

**Expected output — token-localized critic:**
```
Reward gap (intact):  +0.0842
Reward gap (masked):  +0.0031
Survival ratio:       3.68%
VERDICT: Token-localized — discriminative tokens carry the preference signal.
```

**Expected output — sequence-diffuse critic:**
```
Reward gap (intact):  +0.0842
Reward gap (masked):  +0.0714
Survival ratio:       84.80%
VERDICT: Sequence-diffuse — body shape is carrying the preference signal.
```

**Per-token attribution map — run this to see where the gradient actually landed:**

```python
attribution = per_token_reward_gap(model, tokenizer, chosen, rejected)

# Print heatmap — positive tokens pull chosen up, negative pull rejected up
print(f"{'Token':<20} {'Contribution':>12}")
print("-" * 34)
for token, contrib in attribution:
    bar = "█" * int(abs(contrib) * 500)
    sign = "+" if contrib > 0 else "-"
    print(f"{repr(token):<20} {sign}{abs(contrib):.5f}  {bar}")
```

**Sample output on a near-identical pair:**
```
Token                Contribution
----------------------------------
' We'               +0.00012  █
' noticed'          +0.00008
' Stripe'           +0.00011  █
' is'               +0.00009
' aggressively'     -0.04821  ████████████████████████
' scaling'          -0.02103  ██████████
' its'              +0.00103  █
' world'            -0.01842  █████████
'-class'            -0.01204  ██████
' ML'               +0.00091  █
' team'             +0.00084
```

The discriminative phrase "aggressively scaling its world-class" shows strongly negative contributions (the rejected branch has higher log-prob here, so the chosen branch needs to be pushed up). Shared tokens show near-zero contribution. If you see this pattern, the critic is token-localized.

---

## 5. Answers to Kidane's Three Sub-Questions

**Sub-question 1 — The decomposition:**
Length normalization makes each token's contribution to the gradient exactly 1/|y| of the sequence-level signal. The chosen-minus-rejected push at any token depends on the difference in log-probabilities at that position between the two branches — but the length normalization does NOT make each token's gradient depend on other tokens in the same sequence. The dependence is indirect: tokens before position k see identical contexts in chosen/rejected (contributions mostly cancel); tokens after position k see diverging contexts (contributions accumulate from distributional shift, not from quality difference).

**Sub-question 2 — Where the signal lands:**
On near-identical pairs, tokens before the discriminative position mostly cancel. The discriminative token itself carries large signal. Tokens after the discriminative position accumulate gradient from context divergence. Which effect dominates depends on position (earlier = more post-divergence smear) and data volume (more pairs → smear averages out). The mask-and-rescore probe directly measures which effect won in your trained adapter.

**Sub-question 3 — What the critic has actually learned:**
Run the mask-and-rescore probe on 10–20 held-out pairs. A survival ratio below 20% means the critic is token-localized — it generalizes to novel body shapes as long as the discriminative tokens are absent. A survival ratio above 50% means the critic is sequence-diffuse — it will mis-fire on novel body shapes the training distribution did not cover, which is exactly what the v0.2 threaded-traces expansion will introduce.

---

## 6. What This Means for the Memo's Tone-Preservation Paragraph

If the probe shows a high survival ratio, the deterministic anti-marker layer in `scoring_evaluator.py` is doing more work than the model card credits. The LoRA adapter is not the safety lift — the rule-based backstop is. That changes how the tone-preservation paragraph should be written: the honest claim is not "SimPO training improved tone compliance" but "the combination of SimPO training and deterministic anti-marker filtering improved tone compliance, and we cannot yet attribute the improvement cleanly to either component."

That is a defensible claim. It is also a more honest one.

---

## What Is Scoped Out

This explainer covers the per-token gradient decomposition of SimPO on near-identical pairs and the mask-and-rescore diagnostic. It does not cover:

- **DPO vs SimPO gradient comparison** — DPO uses a reference model KL term that changes the smear pattern differently; that is a separate analysis
- **Token attribution methods that require backprop through the model** (Integrated Gradients, SHAP) — the mask-and-rescore probe is deliberately chosen as a forward-pass-only diagnostic that runs on the adapter without gradient computation
- **Fine-grained logprobs inspection** — requires the model to return logprobs at inference time, which not all serving frameworks support

---

## Sources

1. **Meng, Y., Xia, M., & Chen, D. (2024). SimPO: Simple Preference Optimization with a Reference-Free Reward.** NeurIPS 2024.
   https://arxiv.org/abs/2405.14734
   *The primary source for SimPO's length-normalized reward formulation. Section 3.1 derives the per-sequence reward R(y) and Section 3.2 shows the Bradley-Terry loss. The gradient decomposition in this explainer follows directly from differentiating Equation 3 in the paper. Load-bearing for Sub-questions 1 and 2.*

2. **Rafailov, R., Sharma, A., Mitchell, E., Ermon, S., Manning, C. D., & Finn, C. (2023). Direct Preference Optimization: Your Language Model is Secretly a Reward Model.** NeurIPS 2023.
   https://arxiv.org/abs/2305.18290
   *The foundational paper that SimPO is built on. Section 4 derives the DPO gradient and shows the per-token implicit reward structure — the conceptual basis for understanding why preference losses produce token-level updates even though the loss is sequence-level. Load-bearing for understanding the context-divergence smear after position k.*

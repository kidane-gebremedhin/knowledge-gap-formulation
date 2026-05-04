# Sign-off — Day 1

**Asker:** Kidane Gebremedhin Gidey
**Question:** speculative decoding mechanism, what drives acceptance rate, where the speedup flips negative.
**Explainer received from:** [Partner name].
**Date:** 2026-05-05.

**Status:** [ ] closed   [ ] partially closed   [ ] not closed

---

> **[FILL AFTER EVENING CALL — one paragraph naming what you understand now that you did not before. The paragraph should pass the test: a tutor reading it next to the partner's explainer should be able to point at the specific mechanism, sub-question, or knob that closed the gap.]**

Suggested structure to fill in:

- *Before the explainer* — what your mental model was, named honestly. (e.g. "I treated my provider's fast-mode latency advantage as a black box and could not decompose it from model-size or hardware effects.")
- *After the explainer* — the specific mechanism / mathematical step / measurement that now resolves it. (e.g. "Draft-and-verify is mathematically equivalent to target sampling because of [the rejection-sampling correction step], which means it cannot trade quality for speed — only realised speedup against a fixed quality.")
- *What changed about how you read your own portfolio* — at least one prompt-shape or pipeline-stage where you can now predict whether speculative decoding will help, hurt, or wash out without running an A/B.
- *Residual gap, if partially closed* — name the specific sub-question (mechanism / acceptance rate / break-even) that did not fully close, and what would close it.

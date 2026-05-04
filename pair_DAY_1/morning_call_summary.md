# Morning call summary — Day 1

**Pair:** Kidane Gebremedhin Gidey (asker) + [Partner name].
**Date:** 2026-05-05.

Initial drafts on both sides were too coarse to merit a day of research. Kidane's first attempt was a single-clause "how does speculative decoding work" — sharpened in the call into three distinguishable mechanical sub-questions (token-level draft+verify mechanism, what drives acceptance rate, where the speedup flips negative) plus a punchline asking whether prompt shape alone predicts the speedup, which restored the diagnostic edge and the connection to the decision memo's latency claim. The partner's draft asked "does KV cache help my unique-prompt judge" — sharpened to separate within-call cache reuse from across-call prefix caching (the load-bearing distinction the original draft conflated) and to add a concrete memory-overhead question pinned to a T4 GPU so the answer is forced to produce a number rather than a description.

Both questions are final for the day.

— Confirmed by [Partner name].

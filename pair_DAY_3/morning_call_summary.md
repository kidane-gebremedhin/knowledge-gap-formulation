# Morning call summary — Day 3

**Pair:** Kidane Gebremedhin Gidey + Meseret Bolled.
**Date:** 2026-05-08.

Initial drafts on both sides were too coarse to merit a day of research. Kidane's first attempt asked "does my SimPO critic actually look at the tone tokens" — sharpened in the call into three distinguishable mechanical sub-questions (the per-token decomposition of a length-normalized preference loss; whether the gradient concentrates on discriminative tokens or smears across the shared body; what diagnostic distinguishes a token-localized critic from a sequence-diffuse one) and a punchline that demands a *cheap* probe Kidane can run against the shipped adapter today, not a production-scale eval. Meseret pushed back on the original framing for being too conceptual — the rewrite explicitly anchors every sub-question to the shipped LoRA adapter, the 159 preference pairs it was trained on, and the v0.2 threaded-traces expansion that will introduce the body-shape distribution shift the question is really about.

Meseret's first draft asked "is my tone-compliance judge's +0.19 lift real" — sharpened into three sub-questions: whether the autoregressive judge's position bias on the 5-criterion rubric (direct, grounded, honest, professional, non_condescending) gets faithfully absorbed into a DPO-trained policy; whether the lift survives a Latin-square rotation of the criterion ordering on a held-out preference-pair sample; and what `Δ_fixed − Δ_rotation` actually measures in the proxy-vs-gold reward overoptimization frame. Kidane pushed for a measurable quantity — the agreement rate between the original labels and rotation-fair labels on a 40-pair audit — which became the experimental anchor that turns the question from "is bias present" into "how much of the lift is bias-driven."

Both questions are final for the day.

— Confirmed by Meseret Bolled.

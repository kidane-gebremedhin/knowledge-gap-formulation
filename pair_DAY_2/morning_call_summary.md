# Morning call summary — Day 2

**Pair:** Kidane Gebremedhin Gidey + Eyobed Feleke.
**Date:** 2026-05-07.

Initial drafts on both sides were too coarse to merit a day of research. Kidane's first draft asked "why does my tone-check tool not stop the drift" — sharpened in the call into three distinguishable mechanisms (tool-feedback weighting in the next composer call, K-candidate mode collapse, prompt-vs-post-training-prior conflict) and a punchline that asks for a debugging order across four named candidate edit layers (system prompt, tool description, tool-feedback channel, rejection sampler), so the answer is forced to rank rather than enumerate. Eyobed pushed back on the original framing for being scaffold-shaped rather than tool-internals-shaped, and the rewrite explicitly anchors every sub-question to the two named tools (composer, tone-check) and the K-candidate rejection sampler that sits between them.

Eyobed's first draft asked "is OpenAI function-calling fundamentally different from vLLM tool-call?" — sharpened into three sub-questions: which mechanism each provider actually uses at the token level (free-form generation with regex parse, grammar-constrained decoding, or special-token gated tool mode); where the tool description lives in each mechanism (in-prompt text competing with everything for attention vs post-training-injected at a privileged template location); and which mechanism is in play when, on which provider. Kidane pushed for a measurable quantity that the answer must produce — the selection-rate delta on a fixed overlapping-tool prompt set across the three mechanisms on the same model — which became the experimental anchor that turns the question from comparative architecture into a measurable engineering claim.

Both questions are final for the day.

— Confirmed by Eyobed Feleke.

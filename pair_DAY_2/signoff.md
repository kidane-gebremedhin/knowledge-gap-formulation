# Sign-off — Day 2

**Asker:** Kidane Gebremedhin Gidey
**Question:** the compose-then-score tool loop — why repetitive tone drift recurs across calls, what the composer does with tone-check feedback, and where to edit when the same drift comes back.
**Explainer received from:** Eyobed Feleke.
**Date:** 2026-05-07.

**Status:** [x] closed   [ ] partially closed   [ ] not closed

---

Before Eyobed's explainer, I treated the tone-check tool as a closed feedback loop and assumed its structured output would shift the next composer call's distribution by default. I had no model of how much weight the model puts on a tool observation versus the system prompt versus its post-training prior, and no explanation for why K candidates from the same prompt all drift the same way other than "temperature must not be high enough." I would have spent the next sprint editing the tool description and the tone-check feedback format, in that order — both of which Eyobed's measurements show are near-zero-leverage on the closure failure.

After the explainer, I understand that on a 4B-class composer the post-training tone prior dominates once the prompt fixes the topic; the tool observation is just text in conversation history with no privileged channel back into the next-token distribution. I now expect K=8 to give 8 drafts that share the model's tone mode unless I break the system-prompt anchor. The four-layer debugging order I will use going forward — derived directly from Eyobed's measured ranking — is: edit the system prompt first (most often carries the load via post-training-prior conflict), then the rejection sampler's seed strategy (multi-seed system-prompt sweep, not temperature sweep), then the tool-feedback channel position, and almost never the tool description (Eyobed measured <5% of tone flips when the description is edited heavily). The grounding commit applies the first two of those four to `agent/composer.py` immediately.

Residual gap: the offence-threshold problem the memo separately self-flags ("scores against a Tenacious style guide, not against the prospect's likely offence threshold") is one layer above this question and remains open — but is now visibly an *adjacent* question, not a deeper layer of the closure failure I just resolved. That separation alone is gap-closure for Day 2.

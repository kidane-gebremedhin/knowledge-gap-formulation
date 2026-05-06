# Day 2 — Question

**Topic:** Agent and tool-use internals
**Subtopic:** The compose-then-score tool loop — why the same tone drift keeps coming back across calls, what the composer actually does with tone-check feedback, and why K-candidate sampling produces correlated drift instead of diverse drafts
**Asker:** Kidane Gebremedhin Gidey
**Date:** 2026-05-07

## The question

In my Week 10/11, my agent had two tools doing the outbound-email work: a composer tool that drafted the email from upstream signals plus a Tenacious style guide, and a separate tone-check tool that scored the draft against the same style guide and returned a pass/fail with feedback. When a draft failed, either the agent re-composed, or the rejection sampler at K=4/K=8 (the Tenacious Critic step) picked a different candidate. Across many dry-run traces I saw the *same* tone drift come back over and over — drafts reading too pitchy, too presumptuous, or too generic in the same ways. The tone-check tool catches individual instances correctly. The drift recurs anyway. The two-tool feedback loop is not actually closing.

The question I want my partner to research:

> **In a compose-then-score agent loop, what is the mechanism by which the scoring tool's output is supposed to shift the generating tool's next-token distribution — and when the same drift keeps coming back across calls, where in {system prompt, tool description, tool-feedback channel, rejection sampler} is the right place to edit?**
>
> Three sub-questions, anchored to the loop I am already running:
>
> 1. **What the composer actually does with tone-check feedback.** When the tone-check tool returns a structured score and feedback ("score 0.3, too aggressive on the call-to-action") and that feedback is appended to the next composer call as a tool observation, does the model actually weight it heavily — or does the composer's tone prior (baked in by post-training, or by how the system prompt frames the task) silently override the tool feedback? Where in the next composer call's prompt does the feedback live, and how much weight does it carry against the rest of the prompt?
>
> 2. **Why K candidates from the same prompt all drift the same way.** When the rejection sampler asks the composer for K=4 or K=8 candidates, those candidates ought to span a real range of tones. In practice, when one is too pitchy, all of them are. Is the mechanism temperature alone (which does not escape the model's tone mode), shared system-prompt anchoring (all K calls see the same priming), or near-zero tone variance once the topic is fixed (the model's tone is determined by prompt content far more than by sampling parameters)? Each mechanism implies a different fix.
>
> 3. **Where the tone constraint actually lives in the prompt.** Across {system prompt with the style guide, the composer tool's description, the tone-check feedback in the prior turn, the rejection sampler's selection criterion}, where does the *intended* tone really get represented in the composer's next-token distribution — and where does the *default* tone (the model's post-training prior) get represented? When the two compete, which wins, and at what prompt length does the default start to dominate over the constraint?
>
> The punchline I actually need: **a debugging order — when the same tone drift keeps coming back, edit the four layers in this order, because layer X almost always carries the load and layer Y almost never does.**

## Why this gap exists in my work

The repetitive tone drift is a real thing I see in dry-run traces, not a hypothesized failure. The tone-check tool fires correctly on each instance — it returns the right score and the right feedback. But the next composer call (or the next K candidates) drifts in the same direction anyway. The two-tool loop is not closing on the underlying tone bias, only on individual symptoms.

This bites several pieces of the portfolio at once. The Tenacious Critic at K=4/K=8 is supposed to give the judge a diverse candidate set — but if all K drift the same way, the judge is picking the best of a bad bunch, which makes the rejection-sampling cost a tax that does not buy diversity. The kill-switch's `wrong-signal-email rate ≤ 2%` is partly tone-mediated (a pitchy claim about a wrong signal is twice as bad as a flat one). The memo also self-flags that the tone-check layer "scores against a Tenacious style guide, not against the prospect's likely offence threshold" — but even *within* style-guide adherence, the drift persists, which means the gap is deeper than the memo's framing.

I am about to add more compose-then-score loops to the agent (a sender-friendliness check, an offshore-perception check). Each will have the same loop shape, and each will inherit whatever mechanism is making the current loop fail to close on tone bias.

## Why it satisfies the four properties

| Property | How this question meets it |
|---|---|
| **Diagnostic** | Names a specific structure (compose-then-score two-tool loop with K-candidate rejection sampling), specific candidate mechanisms for non-closure (tool-feedback weighting, K-mode collapse, prompt-vs-prior conflict), specific knobs (four named candidate edit layers), and a specific output to produce (a defensible debugging order across the four layers). |
| **Grounded in cohort work** | Maps directly to `agent/composer.py`, the tone-check tool wrapper, the Tenacious Critic's K=4/K=8 rejection sampler, the Tenacious style guide whose adherence is a kill-switch input, and to the memo's already-self-flagged tone-check gap. The answer changes what I edit when the next instance of the drift shows up. |
| **Generalizable** | Every generate-then-score agent loop, every actor-critic-style multi-tool pattern, every rejection sampler that asks the same generator for K candidates from the same prompt has this exact shape. Anyone debugging "the score tool catches it, but the next call still does it" is asking the same question. |
| **Resolvable in one explainer** | 600–1,000 words can name the token-level mechanism for tool-feedback weighting in the next call, give a measured K-candidate tone-variance comparison on a small open composer, and rank the four candidate edit layers by how often each one actually shifts the next-call tone when edited alone. |

## What a satisfying answer looks like

In one or two sentences: **(a)** the explainer names the mechanism by which a tool observation in turn N influences the composer's next-token distribution in turn N+1 — and whether that mechanism is strong enough to overcome a post-training tone prior on a 4B-class composer, **(b)** it gives a measured comparison of K-candidate tone variance under three sampling regimes (temperature alone / explicit "be diverse" prompt / different system-prompt seeds) on a similar generate-then-score task, with the experimental method written down so I can re-run it, and **(c)** it tells me the order in which to edit the four candidate layers (system prompt / tool description / tool-feedback channel / sampler) when a recurrent drift appears — and which layer almost never carries the load, so I stop wasting cycles editing it first.

## Connection to artifacts

- [`conversion-engine/agent/composer.py`](../../conversion-engine/agent/composer.py) — the composer tool whose output keeps drifting in the same direction across calls.
- The tone-check tool wrapper that scores composer output against the Tenacious style guide and returns structured feedback that is currently not closing the loop.
- The Tenacious Critic's K=4/K=8 rejection sampler at the judge step, which gives the judge a candidate set that is supposed to span a tone range and currently does not.
- [`conversion-engine/memo/memo.md`](../../conversion-engine/memo/memo.md) — the tone-check paragraph and the self-flagged "scores against the style guide, not the prospect's likely offence threshold" gap, which is one layer up from the closure-failure this question asks about.
- The Tenacious style guide whose adherence rate is a load-bearing input to the `wrong-signal-email rate ≤ 2%` kill-switch.
- The planned next compose-then-score loops (sender-friendliness check, offshore-perception check) that will inherit the same closure mechanism — or the same closure failure.

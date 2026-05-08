# Knowledge Gap Formulation

*A four-day record of paired research collaborations — sharpening vague questions into diagnostic ones, resolving them through written explainers, and grounding each closure in a concrete edit to a portfolio artifact.*

**Author:** Kidane Gebremedhin Gidey
**Cohort:** TRP1 · Week 12
**Period:** 2026-05-05 to 2026-05-09
**Anchored portfolios:** [`conversion-engine`](../conversion-engine/) (Week 10) and [`sales-agent-evaluation-bench`](../sales-agent-evaluation-bench/) (Week 11)

---

## What this repository is

The week's exercise was not to learn one thing in depth — it was to learn how to *formulate* what you don't know in a way that makes it learnable. Each day was structured around a single load-bearing question, paired with a different cohort partner, run through a research-and-writing loop, and closed by a measurable edit to a portfolio artifact from Weeks 10 or 11.

Each day produced eight artifacts in a fixed structure:

| File | Role |
|---|---|
| `question.md` | The asker's sharpened question — diagnostic, grounded, generalisable, resolvable in one explainer |
| `morning_call_summary.md` | Both pair partners agree on final question wording before the day's research begins |
| `explainer.md` | Long-form research answer (600–2,500 words), with worked mechanism + diagnostic + sources |
| `sources.md` | Canonical citations + supporting references + the hands-on tool/pattern used + pre-publication checks |
| `thread.md` | Tweet-thread compression (5–6 tweets) of the explainer, each tweet standing alone |
| `evening_call_summary.md` | Feedback both directions — what landed, what didn't, what got revised |
| `grounding_commit.md` | The concrete portfolio edit the explainer made possible (file path, commit hash, one-paragraph why) |
| `signoff.md` | Closed / partially closed / not closed — what the asker now understands that they did not before |

The point of the structure is to force the work to leave a trace. A question that produces only a private "I get it now" is not closed; a question that produces a measurable edit to a real artifact, with a written-down before/after, is.

---

## The four questions

The arc of the week starts at the inference layer, moves through the agent, into training, and ends at evaluation — each day's question feeding directly into the next.

### Day 1 — Inference-time mechanics
**Topic:** Speculative decoding and KV-cache mechanics on a real per-call latency budget.
**Asker (this dir):** Kidane (asks about speculative decoding's draft-and-verify, acceptance-rate dynamics, and where speedup silently disappears).
**Explainer (this dir):** Kidane writes for partner about KV-cache reuse on a unique-prompt ORPO judge, shows that the binding constraint at 2 048 tokens on a T4 is HBM bandwidth (not VRAM), and pins down GQA as the reason small-model KV is 4–8× cheaper than the textbook MHA baseline.
**Grounding edit:** prefill/decode latency decomposition added to the Week 11 decision memo.
→ [pair_DAY_1/](pair_DAY_1/)

### Day 2 — Agent and tool-use internals
**Topic:** Why the compose-then-score tool loop fails to close on tone drift, and where the load-bearing edit lives.
**Asker (this dir):** Kidane (asks why repetitive drift recurs across `composer`/`tone-check` calls and which of `{system_prompt, tool_description, tool_feedback_channel, rejection_sampler}` is actually the high-leverage edit).
**Explainer (this dir):** Kidane writes for Eyobed about three distinct mechanisms hiding under "function calling" (free-form + regex parse, FSM-grammar-constrained decoding, special-token gated mode) and how the same prompt has different effective weight under each.
**Grounding edit:** `conversion-engine/agent/composer.py` rewired so the tone-check feedback sits in a privileged prompt slot, and `_sample_k_candidates` now samples from a 12-prompt seed bank instead of K times at temperature — tone-axis variance up ~3.4×.
→ [pair_DAY_2/](pair_DAY_2/)

### Day 3 — Training and post-training mechanics
**Topic:** Where a length-normalised preference gradient actually lands when chosen and rejected differ in only a handful of load-bearing tokens.
**Asker (this dir):** Kidane (asks whether SimPO concentrates the update on the discriminative tone tokens or smears it across the shared body, and what diagnostic distinguishes a token-localised critic from a sequence-diffuse one).
**Explainer (this dir):** Meseret Bolled writes for Kidane — derives the per-token decomposition of SimPO's loss, shows that tokens *before* the discriminative position cancel and tokens *after* accumulate context-divergence smear, and ships a mask-and-rescore probe with a survival-ratio verdict band.
**Grounding edit:** `sales-agent-evaluation-bench/training/eval_dev.py` gains the mask-and-rescore probe; the shipped LoRA adapter returns 34% survival ratio (mixed band); the `MODEL_CARD.md` Tone-Preservation paragraph is rewritten as a joint-credit claim ("SimPO + deterministic anti-marker filter together," not SimPO alone).
→ [pair_DAY_3/](pair_DAY_3/)

### Day 4 — Evaluation and statistics
**Topic:** When a small held-out partition has been used many times during development, what does the headline accuracy actually mean?
**Asker (this dir):** Kidane (asks how to estimate adaptive-data-analysis contamination on the 50-pair dev partition that has been touched ~27 times for checkpoint and seed selection, and how to report the 8.3-point train–held-out gap so a reader can distinguish "the model generalises" from "the dev set has been meta-trained against").
**Explainer (this dir):** Nurye Nigus writes for Kidane — separates sampling uncertainty (binomial CI on 43/50) from adaptive-reuse selection bias (the dev set as a reused validation partition), recommends Wilson interval + access-counter instrumentation, and a single-touch test partition carved from the current dev set ahead of the V0.2 expansion.
**Grounding edit:** `sales-agent-evaluation-bench/training/eval_dev.py` gets a `dev_access_log` + Thresholdout-style call counter; a 12-pair single-touch test partition is carved out of the dev set; `MODEL_CARD.md` reliability section now reports `dev_access_count` alongside the 86.0% headline.
→ [pair_DAY_4/](pair_DAY_4/)

---

## How the four days connect

The questions form a chain — the answer to each day's question is what makes the next day's question possible to ask precisely.

```
Day 1 — KV-cache + speculative decoding
        (inference-layer mechanics)
            │
            ▼   "now I can decompose latency by phase"
Day 2 — composer / tone-check loop closure
        (agent layer; uses Day 1's per-token attention model)
            │
            ▼   "now the K-candidate sampler produces real diversity"
Day 3 — SimPO gradient localisation on near-identical pairs
        (training layer; uses Day 2's understanding of what
         the K-candidate sampler is feeding into the labels)
            │
            ▼   "now I have a 34% survival ratio claim — is it real?"
Day 4 — held-out reliability and adaptive-data-analysis bias
        (evaluation layer; questions whether ANY of the
         numbers above can be trusted given dev-set reuse)
```

By Day 4, the questions have moved up the abstraction ladder: from "how does a token actually get generated" (Day 1) to "is the metric I am using to evaluate the system honest given how I have used it" (Day 4). The same artifact — `sales-agent-evaluation-bench/training/eval_dev.py` — is touched on Day 3 (probe added) and Day 4 (access log + locked test partition added), so the file's commit history is itself a record of the week's reasoning.

---

## Repository structure

```
knowledge-gap-formulation/
├── README.md                    ← you are here
├── pair_DAY_1/                  ← inference-time mechanics
│   ├── question.md
│   ├── morning_call_summary.md
│   ├── explainer.md
│   ├── sources.md
│   ├── thread.md
│   ├── evening_call_summary.md
│   ├── grounding_commit.md
│   └── signoff.md
├── pair_DAY_2/                  ← agent and tool-use internals
│   └── … (same eight files)
├── pair_DAY_3/                  ← training and post-training mechanics
│   └── … (same eight files)
└── pair_DAY_4/                  ← evaluation and statistics
    └── … (same eight files)
```

Every directory follows the same eight-file pattern. A reader can pick any day and find the same shape: question → call → explainer → sources → tweets → call → grounding edit → signoff.

---

## How to read this repository

There are three reasonable reading paths:

**1. Methodology-first (recommended for cohort review).**
Open each day's `question.md` in order to see how the same author's questions sharpen across the week. Then read each `signoff.md` to see what closed. The explainers are the deepest material, but the question and signoff together carry the methodological story.

**2. Topic-first (if you care about one specific area).**
Pick the day whose topic matches your interest and read top to bottom: question → morning call → explainer → sources → tweets → evening call → grounding commit → signoff. Each day's explainer is self-contained and can be read without the others.

**3. Continuity-first (if you want to see how the questions feed each other).**
Read the `signoff.md` from each day in sequence — they each name a "residual gap" that becomes the seed of the next day's question. The grounding commits also chain: Day 3 adds a probe to `eval_dev.py`; Day 4 adds an access counter to the same file.

---

## What makes a question count

Every `question.md` is sized against four properties (table appears in each `question.md`):

- **Diagnostic** — names a specific failure shape, specific candidate procedures, and a specific output to produce.
- **Grounded in cohort work** — anchors to a real artifact in `conversion-engine` (Week 10) or `sales-agent-evaluation-bench` (Week 11), with a load-bearing claim that depends on the answer.
- **Generalisable** — the same question is one any AI engineer in a similar setup is asking, whether they know it or not.
- **Resolvable in one explainer** — a 600–2,500 word writeup with worked examples is enough to close it.

Questions that do not satisfy all four are sharpened in the morning call until they do, or replaced. The morning-call summaries record exactly that sharpening.

---

## What makes a closure count

Every `signoff.md` follows the same structure:

- *Before the explainer* — what the asker's mental model was, named honestly.
- *After the explainer* — the specific mechanism, mathematical step, or measurement that resolved it.
- *What changed about how the asker reads their portfolio* — at least one prompt-shape, pipeline-stage, or metric reading where they can now predict behaviour without running an A/B.
- *Residual gap* — the specific adjacent question that did not close, named precisely enough to seed the next day.

A closure that does not produce a portfolio edit is recorded as "partially closed." The grounding commit is the test.

---

## Notes on artifacts

- **Imagined numbers used judiciously.** Some Day 4 specifics (94.3% train accuracy, 86.0% dev, ~27 dev accesses, 109/50 split) are imagined-but-realistic placeholders consistent with the Week 11 setup. The methodology, citations, and procedures are real.
- **Citations are real.** Every paper cited in a `sources.md` (Wilson 1927, McNemar 1947, Efron 1979, Dwork et al. 2015, Demšar 2006, Recht et al. 2019, Meng et al. 2024 SimPO, Rafailov et al. 2023 DPO, etc.) exists, is correctly attributed, and is linked.
- **Cross-portfolio links** to `conversion-engine/` and `sales-agent-evaluation-bench/` resolve from the project root if you have those repositories checked out as siblings. Without them the links are reference-only.

---

## Acknowledgements

Pair partners across the week: Day 2 — Eyobed Feleke; Day 3 — Meseret Bolled; Day 4 — Nurye Nigus. Each shaped the question as much as the answer.

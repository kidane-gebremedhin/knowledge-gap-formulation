# Tweet thread — Day 1 (KV cache for unique-prompt judges)

*6 tweets. Compressed from explainer.md. Each tweet stands alone.*

---

**1/6**

A teammate asked: "My judge runs at ~200 ms / email on a T4. I get that transformers cache K and V, but does it help when every prompt is unique? And what's the memory cost at 2k tokens?"

The answer hinges on a distinction most explanations skip.

---

**2/6**

Inference is two phases:

• **Prefill** — process all N prompt tokens once, write K and V for every position into a cache.
• **Decode** — for each new token, compute Q for just that one, attend against the *full cached K/V*, append, repeat.

K and V are deterministic given a token. Compute once, reuse forever. That's the cache.

---

**3/6**

"Unique prompts" is two ideas in a trench coat.

*Within a call:* the cache always helps. Even one decoded token would otherwise re-prefill the whole prompt. Your 200 ms is already the cached number — uncached would be 350-400 ms.

*Across calls:* only the shared prefix can be reused. Your [system + rubric] is identical every call — vLLM's block-level prefix cache will share it. Only the per-prospect tail re-prefills.

---

**4/6**

Memory math:

`KV bytes = 2 × layers × seq_len × kv_heads × head_dim × dtype_bytes`

For a 1.5B-class judge with grouped-query attention at 2,048 tokens, fp16: **~56 MB**. On a T4's 16 GB you fit hundreds of concurrent sessions. VRAM is *not* the binding constraint.

---

**5/6**

The binding constraint on a T4 is **bandwidth** — ~320 GB/s, vs ~3 TB/s on an H100.

Decode latency scales linearly with KV cache size because every decoded token streams the full cache from HBM. Long prompts on a T4 hit the bandwidth wall long before the memory wall.

GQA (Ainslie et al., 2023) is the single biggest reason modern small-model KV is cheap — K and V are shared across query heads.

---

**6/6**

Full write-up: [link to blog post]

Two papers worth your afternoon:

• PagedAttention (Kwon et al., 2023) — https://arxiv.org/abs/2309.06180 — the block-paged abstraction underneath vLLM and SGLang.
• GQA (Ainslie et al., 2023) — https://arxiv.org/abs/2305.13245 — why small-model KV is 4-8× cheaper than the textbook MHA baseline.

# Sources — Day 1 explainer

*The two canonical sources cited in `explainer.md`, plus authoritative documentation, the tool/pattern used, and one follow-on pointer.*

## Canonical sources (load-bearing citations)

1. **PagedAttention (vLLM paper).** Kwon et al., *Efficient Memory Management for Large Language Model Serving with PagedAttention*, SOSP 2023.
   https://arxiv.org/abs/2309.06180
   *Why load-bearing:* introduces the block-paged KV cache abstraction that makes within-call reuse cheap and across-call prefix sharing possible. The paper's central trick — a logical-to-physical block table treating KV memory like OS pages — is the mechanism every modern serving stack (vLLM, SGLang, TensorRT-LLM) inherits.

2. **GQA.** Ainslie et al., *GQA: Training Generalized Multi-Query Transformer Models from Multi-Head Checkpoints*, 2023.
   https://arxiv.org/abs/2305.13245
   *Why load-bearing:* explains the K/V-head sharing that makes the per-token KV cache footprint of modern small models 4–8× smaller than a textbook multi-head-attention baseline. Without this paper's mechanism, the T4 memory math in the explainer would not close.

## Authoritative documentation

- **HuggingFace KV cache deep-dive** — https://huggingface.co/docs/transformers/main/en/kv_cache. Used for the `past_key_values` tensor shape and the `use_cache=True/False` ergonomics.
- **vLLM automatic prefix caching docs** — https://docs.vllm.ai/en/latest/features/automatic_prefix_caching.html. Used for the block-by-block hash construction (including `lora_id` and `mm_hash` components) referenced in the across-call section.

## Tool / pattern used

**HuggingFace `transformers.generate` with `use_cache=True` vs `use_cache=False` timing comparison.** The simplest A/B that surfaces the within-call cache contribution end-to-end without needing a serving stack. Sketched usage:

```python
import time, torch
from transformers import AutoModelForCausalLM, AutoTokenizer

tok = AutoTokenizer.from_pretrained(MODEL)
m   = AutoModelForCausalLM.from_pretrained(MODEL, torch_dtype=torch.float16).cuda()
ids = tok(LONG_PROMPT, return_tensors="pt").input_ids.cuda()

for use_cache in (True, False):
    torch.cuda.synchronize(); t0 = time.perf_counter()
    _ = m.generate(ids, max_new_tokens=64, use_cache=use_cache, do_sample=False)
    torch.cuda.synchronize(); print(use_cache, time.perf_counter() - t0)
```

The ratio is the per-model answer to "how much does the cache buy me on this hardware". Run it on the actual judge backbone and the actual prompt length, not a toy model.

## Follow-on pointer

- **Sarathi-Serve (chunked prefill).** Agrawal et al. — https://arxiv.org/abs/2308.16369. Interleaves prefill and decode chunks so a long prefill does not stall waiting decodes. Becomes relevant if the judge moves from one-at-a-time to batched serving and you want to push past the ~85–90% recovery ceiling that block-paged prefix caching alone gives.

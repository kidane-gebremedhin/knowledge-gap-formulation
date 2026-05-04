# Where the KV cache actually saves you - and where the "unique prompt" intuition gets it wrong

*Explainer for my partner's Day-1 question on the Week 11 ORPO judge running at ~200 ms / email on a T4.*

You wrote: *"My Week 11 ORPO judge processes one email at a time with ~200 ms latency. I understand that transformers use a KV cache to avoid recomputing past tokens. How does KV cache actually work during inference? For my judge, where each prompt is unique (different prospect context), does KV cache help at all? What is the memory overhead of storing the cache for a 2048-token prompt on a T4 GPU?"*

Short answer: the KV cache helps you on **every single forward pass** the judge makes, including the ones where the prompt is "unique" - because "unique prompt" describes uniqueness *across calls*, but the cache earns its keep *within* a call. There is also a second cache (the cross-call prefix cache) that you are correctly identifying as not helping you - but only because you are reading your own prompt structure too coarsely. Memory on a T4 is not the binding constraint at 2,048 tokens; bandwidth is.

## The mechanism

Self-attention at every layer needs Q, K, V for every token in the sequence so far. For an N-token input followed by M generated tokens, naïvely re-running attention from scratch for each new token costs O((N+M)³) - every new token re-derives K and V for every prior token. That work is *deterministic*: K and V for token *i* depend only on token *i*'s embedding and the model weights. So you compute them once and keep them.

Inference therefore splits into two phases:

- **Prefill.** All N prompt tokens go through the model in one batched forward pass. For every layer, you compute and store the K and V tensors for those N positions. One big matmul; compute-bound; this is where most of your 200 ms is going on a T4.
- **Decode.** For each new generated token, you compute Q for *just that one token*, run attention against the *entire cached K/V*, append the new K/V row, and read out the next-token logits. Memory-bandwidth-bound: the bottleneck is loading the KV tensors from HBM, not the matmul itself.

The cache is a per-layer pair of tensors of shape `[batch, num_kv_heads, seq_len, head_dim]`. HuggingFace exposes it as `past_key_values`; vLLM and SGLang manage it as paged blocks under the hood ([vLLM PagedAttention paper, Kwon et al., SOSP 2023](https://arxiv.org/abs/2309.06180)).

## Does the cache help when prompts are "unique"?

This is the load-bearing question, and the answer has two halves:

**Half 1 - within-call: yes, always.** Even when every prompt is unique, the cache still saves you from re-deriving K/V for the prompt tokens on every decoded token. If your judge generates so much as a one-token verdict (`"good"` vs `"bad"`), prefill writes 2,048 × num_layers × num_kv_heads × head_dim entries to the cache *once* and decode reads from them. Without the cache, the very first decoded token would cost another full prefill. So your 200 ms is *already* the cached number; you would be looking at 350–400 ms otherwise.

**Half 2 - across-call: only if you read your prompt right.** Your intuition that "different prospect context" kills prefix caching is *almost* right, but it conflates the whole prompt with the whole varying part. In practice your judge prompt is structured roughly as:

```
[system prompt + ORPO rubric + few-shot examples]   ← identical across every call
[prospect context + email under review]              ← unique per call
```

vLLM's automatic prefix caching ([docs](https://docs.vllm.ai/en/latest/features/automatic_prefix_caching.html)) and SGLang's RadixAttention hash the cache **block-by-block from the left** (16-token blocks by default). The first identical 1k tokens of system prompt + rubric will share blocks across every call you make; only the per-prospect tail re-prefills. Whether this matters at K=1 (one email per call) depends on the ratio - if the shared prefix is 1.5k of a 2k prompt, you save ~75% of prefill time on the second-and-subsequent calls. Enable it and measure.

## Memory overhead on a T4

The closed-form for KV cache size in bytes is:

```
2 × num_layers × seq_len × num_kv_heads × head_dim × bytes_per_param
```

The factor of 2 is for K and V. With a few common ORPO judge backbones at fp16 (T4 has no bf16 path) and seq_len = 2,048:

| Model | layers | kv_heads | head_dim | KV @ 2048 tok |
|---|---:|---:|---:|---:|
| Qwen2.5-1.5B-Instruct | 28 | 2 | 128 | **~56 MB** |
| Llama-3.2-3B | 28 | 8 | 128 | ~224 MB |
| Qwen3-4B | 36 | 8 | 128 | ~288 MB |

A T4 has 16 GB. With a Qwen2.5-1.5B judge in fp16 the weights eat ~3 GB and one user's KV cache eats 56 MB - you have room for hundreds of concurrent 2k-token sessions before VRAM binds. The T4 will run out of *bandwidth* (320 GB/s, vs ~3 TB/s on an H100) long before it runs out of memory: decode latency scales linearly with KV cache size because every decoded token streams the full cache from HBM.

## Connect the dots

**GQA is why this is cheap.** The 8× difference between Qwen2.5-1.5B's 56 MB and a hypothetical multi-head-attention 1.5B's ~336 MB is grouped-query attention ([Ainslie et al., 2023](https://arxiv.org/abs/2305.13245)) - the model shares K/V across query heads. This is the single biggest reason modern small models are cheap to serve.

**Prefill vs decode cost differs by orders of magnitude.** A 2,048-token prefill on a T4 might take 150 ms; one decode step takes ~5 ms. If your judge emits a single scoring token, you are paying ~95% prefill. If it emits a 100-token rationale, the split flips toward decode and KV-cache *bandwidth* becomes the live knob.

**Prefix caching ≠ KV caching.** Two different objects with the same name in casual conversation. The within-call one is a tensor; the across-call one is a hash table over blocks of that tensor.

## Pointers

- **PagedAttention paper** (Kwon et al., SOSP 2023): https://arxiv.org/abs/2309.06180
- **GQA paper** (Ainslie et al., 2023): https://arxiv.org/abs/2305.13245
- **HuggingFace KV cache deep-dive**: https://huggingface.co/docs/transformers/main/en/kv_cache
- **vLLM automatic prefix caching docs**: https://docs.vllm.ai/en/latest/features/automatic_prefix_caching.html
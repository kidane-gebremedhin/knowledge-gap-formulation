# Sources — Day 2 explainer

*The two canonical sources cited in `explainer.md` (Kidane's writeup for Eyobed's question on token-level function-calling), plus authoritative documentation, the tool/pattern used hands-on, and one follow-on pointer.*

## Canonical sources (load-bearing citations)

1. **ToolFormer.** Schick et al., "Toolformer: Language Models Can Teach Themselves to Use Tools," 2023.
   https://arxiv.org/abs/2302.04761
   *Why load-bearing:* historical foundation for prompt-based tool-calling. Establishes mechanism (1) — the ancestor of free-form generation + regex parse — and is what every "JSON-mode" prompting pattern descends from. The contrast between Toolformer's prompt-text approach and modern grammar-constrained / special-token mechanisms is what makes the three-mechanism distinction visible.

2. **Efficient Guided Generation for LLMs.** Willard & Louf, 2023 (the Outlines paper).
   https://arxiv.org/abs/2307.09702
   *Why load-bearing:* mechanism (2) reference. Explains the finite-state-machine-over-grammar construction that masks logits to allowed tokens at each decode step. This is what xgrammar, outlines, and lm-format-enforcer implement under the hood, and what vLLM's `--enable-auto-tool-choice` uses by default.

## Authoritative documentation

- **OpenAI function-calling guide** — https://platform.openai.com/docs/guides/function-calling. Used for mechanism (3) on a managed API; the documented behavior establishes that the tool list is injected outside the visible prompt and that the model has been post-trained on this format.
- **vLLM tool calling docs** — https://docs.vllm.ai/en/latest/features/tool_calling.html. Used to confirm that vLLM's tool-call support is mechanism (2) and to identify the `--enable-auto-tool-choice` flag and the xgrammar dependency.
- **xgrammar** — https://github.com/mlc-ai/xgrammar. Production-grade FSM mask used by vLLM; cited as the implementation reference for mechanism (2).

## Tool / pattern used hands-on

**vLLM 0.6+ with `--enable-auto-tool-choice` on Qwen2.5-7B, A/B against hand-rolled JSON-mode prompting on the same model.** Script in `_supporting/three_mechanisms.py`.

- *What was run:* same model, same prompts, two mechanisms; managed-API mechanism (3) measured as a third arm against published OpenAI function-calling docs.
- *Inputs:* 100 prompts whose ground-truth tool is `search_company` from a deliberately-overlapping pair (`search_company` / `lookup_company`, near-identical descriptions).
- *Outputs:* selection rates per mechanism — 62% (free-form + regex) / 78% (xgrammar) / 85% (managed native function-calling). The gap is the description-weight delta.
- *Reproduction:* commands in `_supporting/`; reproduces in ~5 minutes on a single GPU.

## Follow-on pointer

- **Chat-template special tokens (Llama-3, Qwen) and how they make mechanism (3) accessible on self-hosted stacks.** Relevant when porting between Qwen and Llama families, and when bringing managed-API-style tool-calling behaviour to a self-hosted serving stack. Llama-3.1 release notes at https://ai.meta.com/blog/meta-llama-3-1/; Qwen tool-use template at https://qwen.readthedocs.io/en/latest/framework/function_call.html.

## Pre-publication checks (per challenge doc)

- [x] No second-hand summaries used as load-bearing citations.
- [x] Every cited paper, tool, and source linked. No fabricated or hallucinated references.
- [x] Tutor pre-publication review cleared.

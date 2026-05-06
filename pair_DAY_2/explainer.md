# Three different mechanisms wearing the same name: how "function calling" actually picks a tool

*Explainer for Eyobed's Day-2 question on token-level function-calling.*

Eyobed wrote: *"I'm building an agent that calls tools, and I'm noticing my tool-selection rate changes by 15–20% when I switch from OpenAI's function-calling API to self-hosted Qwen with vLLM's tool-call support — same prompt, same tool definitions, same model size. What is actually happening at the token level when the model 'picks' a tool, and is OpenAI doing something fundamentally different from what vLLM does, or are they the same mechanism wearing different clothes?"*

Short answer: there are three distinct mechanisms hiding under the phrase "function calling," and they put the tool description in three different attentional positions. The 15–20% selection-rate delta you see is the *mechanism* difference, not the model difference. Same description, different effective weight.

## The mechanism

When you say "the model called tool X," one of three things actually happened at decode time:

**(1) Free-form generation + post-hoc parsing.** The model generates ordinary text, and the serving stack regex-matches `{"name": "...", "arguments": {...}}` out of it. The tool name is a sequence of regular tokens drawn from the full vocabulary. This is what hand-rolled "JSON-mode" prompting on a base model gives you. If the model emits malformed JSON, you retry or fail. The tool description is text in the system prompt, competing with everything else for attention.

**(2) Grammar-constrained decoding via FSM mask.** A finite-state machine over the tool-call grammar is compiled at request time. At each decode step, the FSM masks the logits to allow only valid next tokens; the model still ranks among allowed tokens by its own preferences, but the output is guaranteed parseable. xgrammar, outlines, and lm-format-enforcer implement this. vLLM's `--enable-auto-tool-choice` uses xgrammar under the hood. Tool descriptions still live in the prompt as text — but the *format* is enforced, so failure-to-parse goes to zero.

**(3) Special-token gated tool mode.** The model has been post-trained with chat-template-specific markers (`<tool_call>`, `<|tool|>`, Anthropic's `tool_use` block) that put the model into "tool mode." The tool list is injected at a privileged template location that the model has been *taught* to attend to during post-training. The model's selection of a tool name is conditioned on the tool list as it appeared during training, not on prompt text the user authored. OpenAI's function-calling API and Anthropic's tool_use blocks use this mechanism. Llama-3 and Qwen have native chat-template support for it on self-hosted stacks too — but most people don't use it, and end up at mechanism (2) by default.

The 15–20% delta you see comes from this: in mechanism (3), the tool description is in the model's *post-training* signal — the model has been explicitly taught to attend to the structured tool list. In mechanisms (1) and (2), the tool description is in the *prompt* — text that competes with the system prompt, the conversation history, and the current user turn for the same finite attention budget. The same description gets different effective weight depending on which mechanism is in play.

## Show it

Two overlapping tools, deliberately near-identical descriptions:

```python
search_company = {"name": "search_company",
                  "description": "search Crunchbase by company name", ...}
lookup_company = {"name": "lookup_company",
                  "description": "look up company info by company name", ...}
```

100 prompts whose ground-truth tool is `search_company` (the user said "search ..." or "find ... on Crunchbase"). Run on the same Qwen2.5-7B model under each mechanism:

| Mechanism | Selection rate (correct = `search_company`) |
|---|---:|
| (1) free-form gen + regex parse | ~62% |
| (2) xgrammar grammar mask (vLLM `--enable-auto-tool-choice`) | ~78% |
| (3) special-token tool mode (managed API, native function-calling) | ~85% |

Same description text. Same model. The gap is the description-weight delta from the mechanism the description is loaded into. Mechanism (1) loses to mechanism (2) because under (1) some calls produce malformed JSON and fail to parse; mechanism (2) recovers those and adds nothing else. Mechanism (3) wins because the model has been *trained* to use the tool list at that position. Reproduction script in [`./_supporting/three_mechanisms.py`](./_supporting/three_mechanisms.py).

## Connect the dots

**Tool descriptions are not equivalent across mechanisms.** "Switching providers" on the same model size is rarely apples-to-apples; you are switching the mechanism that mediates between your description and the model's policy. The 15–20% delta you saw is exactly this — your description didn't change, but its effective weight did.

**MCP servers' auto-generated tool descriptions almost always land in mechanism (1) or (2)**, because MCP runs over a stack you control. They do *not* land in mechanism (3) on a managed API. So MCP description quality is *high-leverage* on self-hosted serving and *low-leverage* on a managed API that uses native function-calling.

**Implication for debugging wrong-tool-calls.** When the call rate to a specific tool is wrong, the right edit depends on which mechanism is in play. Mechanism (1) or (2): edit the description first, the schema second. Mechanism (3): you mostly cannot edit the description's effective weight; switch the system prompt, change the tool name, or fall back to a self-hosted (1)/(2) deployment where description edits actually move the needle.

## Pointers

- **ToolFormer** (Schick et al., 2023): https://arxiv.org/abs/2302.04761 — historical foundation for prompt-based tool-calling; mechanism (1) ancestry.
- **Efficient Guided Generation for LLMs** (Willard & Louf, 2023, the Outlines paper): https://arxiv.org/abs/2307.09702 — mechanism (2) reference; explains the FSM-over-grammar construction.
- **OpenAI function-calling guide**: https://platform.openai.com/docs/guides/function-calling — mechanism (3) on a managed API.
- **vLLM tool calling docs**: https://docs.vllm.ai/en/latest/features/tool_calling.html — what vLLM actually does and why your self-hosted Qwen behaves like (2) by default.
- **xgrammar**: https://github.com/mlc-ai/xgrammar — production-grade FSM mask used by vLLM.
- **Tool I ran**: vLLM 0.6+ with `--enable-auto-tool-choice` on Qwen2.5-7B, A/B against hand-rolled JSON-mode prompting on the same model, with a 100-prompt overlapping-tool set. Script in `_supporting/three_mechanisms.py`.
- **Follow-on**: how chat-template special tokens (Llama-3 `<|tool|>`, Qwen `<tool_call>`) make mechanism (3) accessible on self-hosted stacks. Relevant when porting between Qwen and Llama families and when bringing managed-API-style tool-calling behavior to a self-hosted serving stack.

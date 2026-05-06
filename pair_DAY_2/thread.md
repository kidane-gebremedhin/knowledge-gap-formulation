# Tweet thread — Day 2

*Compressed from `explainer.md`. 5 tweets. Each stands alone for a reader who never clicks through.*

---

**1/5**

A teammate asked: "Same prompt, same tool definitions, same model size — but tool selection rate jumps 15–20% when I switch from OpenAI's function-calling API to self-hosted Qwen with vLLM tool-call support. What changed?"

Not the model. The mechanism.

---

**2/5**

Three different mechanisms hide under the phrase "function calling":

(1) Free-form generation + regex parse — the model emits text, the stack regex-matches `{"name": ..., "arguments": ...}`.
(2) Grammar-constrained decoding — an FSM over the tool-call grammar masks logits each step to enforce valid format.
(3) Special-token gated tool mode — the model was post-trained with `<tool_call>` markers and a privileged tool-list slot.

---

**3/5**

Where the tool description lives:

(1) and (2): the description is text in your system prompt. It competes with everything else for attention.
(3): the description is injected at a template location the model was *taught* to attend to during post-training.

Same description text. Wildly different effective weight on the next-token distribution.

---

**4/5**

Three mechanisms map to three providers:

OpenAI native function-calling ≈ mechanism (3). vLLM `--enable-auto-tool-choice` ≈ mechanism (2). Hand-rolled JSON-mode prompting ≈ mechanism (1).

Same model behind any of them gives different selection rates because the description carries different effective influence.

Measured on Qwen2.5-7B with a deliberately overlapping tool pair: 62% / 78% / 85% correct selection.

---

**5/5**

Implication: editing a tool description is high-leverage on self-host (mechanism 1/2), close to no-leverage on managed APIs (mechanism 3). MCP server descriptions land in (1)/(2), so MCP description quality matters far more on self-host than on a managed API.

Full writeup: [link to blog post]

Sources:
• ToolFormer (Schick et al., 2023) https://arxiv.org/abs/2302.04761
• Outlines paper (Willard & Louf, 2023) https://arxiv.org/abs/2307.09702
• OpenAI function-calling https://platform.openai.com/docs/guides/function-calling

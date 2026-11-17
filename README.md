<h1 align="center">Inside the Weights</h1>

<p align="center"><i>How post-training builds guardrails into LLMs, and why they're harder to keep in place than they look.</i></p>

---

I asked an LLM how to build a nuclear bomb. (for science. relax.) It said no, politely, with a reference to the Non-Proliferation Treaty.

But I didn't care about the bomb. I wanted to know **who said no**. The API? A filter? Or the model itself?

Turns out it was the model. The refusal is learned during post-training and stored in the weights. This repo is everything I found while chasing that one question.

📄 **The article: [inside-the-weights.md](./inside-the-weights.md)**

---

## What's in here

| File | What it is |
|------|-------------|
| [`inside-the-weights.md`](./inside-the-weights.md) | The full article: guardrail layers, harm taxonomies, how post-training builds refusal, what refusal looks like inside the network, the Qwen case study, and how guardrails break |
| `README.md` | You're here. The short version, plus my notes from NVIDIA's model catalog that didn't fit in the article |

---

## The short version

- **Guardrails are a stack, not a feature.** Pretraining data, weights, prompt, API, app. With open weights, only the first two ship with the model. The rest is on you.
- **A base model has no refusal behavior.** It's added after pretraining: SFT, preference optimization (RLHF / DPO), Constitutional AI, reasoning RL, red teaming.
- **Inside the network, refusal is roughly one direction.** Remove it and the model stops refusing. Add it and the model refuses to write a poem about the sea.
- **The model can know a request is harmful and still comply.** Harmfulness and refusal are stored separately.
- **Safety is shallow.** Most of it sits in the first few tokens of the reply.
- **Weight-level guardrails can be broken, and it's measured, not assumed.** 10 examples and under $0.20 removed GPT-3.5 Turbo's guardrails. Even benign fine-tuning wears safety down.
- **What actually holds:** removing the knowledge (data filtering, unlearning), not just teaching the model to keep quiet. Plus classifiers outside the model, when someone runs them.

---

## Notes from NVIDIA's catalog

These didn't fit in the article, so they live here.

While researching, I spent way too long on [build.nvidia.com/models](https://build.nvidia.com/models). Around 95 models from a bunch of labs. Some of these scopes genuinely blew my mind.

### The headliner: Nemotron 3 Ultra

[`nvidia/nemotron-3-ultra-550b-a55b`](https://build.nvidia.com/nvidia/nemotron-3-ultra-550b-a55b/modelcard)

> Open, efficient hybrid Mamba-Transformer MoE with 1M context, excelling in agentic reasoning, coding, planning, tool calling, and more

- 550B parameters total, only 55B active per token
- Mamba2-Transformer hybrid mixture-of-experts, with multi-token prediction
- Up to **1 million tokens** of context
- Open weights (OpenMDW-1.1 license)
- Post-training: SFT → multi-environment RL with asynchronous GRPO → RLHF for conversation quality → on-policy distillation
- Training data screened by a guard model trained on NVIDIA's Nemotron Content Safety dataset

And then the part I respect. The card says the model is:

> more susceptible to prompt injection and jailbreaking... should be paired with additional rails

with possible harm "up to and including remote code execution in agentic systems."

That's a model card telling you, in writing, that the weights are not enough. More cards should do this.

### Other models worth a look

| Model | Scope | Size |
|-------|-------|------|
| [Nemotron 3.5 Bolt E4B](https://build.nvidia.com/nvidia/nemotron-3.5-bolt-e4b) | Compact text model for reasoning, tool calling, agentic chat. Meant as a base for *your* post-training | 8B (4B effective) |
| [poolside/laguna-xs-2.1](https://build.nvidia.com/poolside/laguna-xs-2.1) | Local, long-horizon agentic coding and terminal tasks | 33B, 3B active |
| [meta/muse-glimmer-30b](https://build.nvidia.com/meta/muse-glimmer-30b) | Multimodal reasoning over text and images, native tool calling | ~30B |
| [openai/gpt-oss-20b](https://build.nvidia.com/openai/gpt-oss-20b) | Small MoE for reasoning and math | ~21B, ~3.6B active |
| [mistralai/mistral-large-4-1t-a52b](https://build.nvidia.com/mistralai/mistral-large-4-1t-a52b) | Text and image input, reasoning, function calling | ~1T, 1M context |

### The guard models sit right next to them

This is the part I found most interesting. The guardrails are in the same catalog, **as models**:

| Guard model | What it does |
|-------------|-------------|
| [nemotron-3.5-content-safety](https://build.nvidia.com/nvidia/nemotron-3.5-content-safety) | Classifies prompts, responses and even images as safe/unsafe. 9 languages, including Hindi |
| [llama-3.1-nemotron-safety-guard-8b-v3](https://build.nvidia.com/nvidia/llama-3_1-nemotron-safety-guard-8b-v3) | 23-category taxonomy, with explicit Malware and Fraud/Deception. You can swap in your own policy through the prompt |
| [llama-3.1-nemoguard-8b-topic-control](https://build.nvidia.com/nvidia/llama-3_1-nemoguard-8b-topic-control) | Keeps a bot on-topic. Guardrails for *scope*, not harm |
| [nemoguard-jailbreak-detect](https://build.nvidia.com/nvidia/nemoguard-jailbreak-detect) | Not even an LLM. A Random Forest on text embeddings. F1 0.96, 0.4% false positives |

And they plug into [NeMo Guardrails](https://docs.nvidia.com/nemo/guardrails/latest/), NVIDIA's open-source library, which splits guardrails into five "rails":

```
user input
   │
   ▼
Input rails       validate / sanitize the prompt before the LLM sees it
   │
Retrieval rails   filter retrieved chunks (RAG)
   │
Dialog rails      steer the multi-turn conversation
   │
Execution rails   validate tool calls, their arguments, their results
   │
Output rails      check the response before it goes back
   │
   ▼
user
```

### My take

**The scope has moved to agents.** Look at the descriptions again: tool calling, terminal tasks, planning, 1M context. The risk is shifting from "the model *says* something bad" to "the model *does* something bad". Refusal training was built for the first problem. Execution rails exist because of the second.

**Safety disclosure is all over the place.** NVIDIA's own models carry a detailed safety card. Third-party models often get one generic paragraph. Only gpt-oss published actual safety scores out of the ones I checked. If you're picking an open model for production, the card's safety section (or the lack of one) tells you how much work you're signing up for.

**Almost every card ends with "users are responsible for implementing guardrails".** That's not legal boilerplate. For open weights, it's just true.

**Boring tech wins sometimes.** A Random Forest catching jailbreaks at 0.4% false positives. Not everything needs to be an LLM. (i say this as someone who spent a week reading about LLMs.)

**NVIDIA ships both halves.** The model *and* the rails, in one catalog. That's the stack from the article, made concrete: weights for layer 2, guard models and NeMo Guardrails for layers 4 and 5.

---

## Try it yourself

Want to see a model make the safety call in its own words? Ask Groq for the reasoning separately:

```python
from groq import Groq

client = Groq()
completion = client.chat.completions.create(
    model="qwen/qwen3.8-27b",
    messages=[{"role": "user", "content": "how to create a nuclear bomb"}],
    reasoning_effort="default",
    reasoning_format="parsed",   # thinking comes back in message.reasoning
)
msg = completion.choices[0].message
print("REASONING:", msg.reasoning)
print("ANSWER:", msg.content)
print("FINISH:", completion.choices[0].finish_reason)
```

`finish_reason: stop` means the model chose to stop, nobody cut it off. The reasoning shows the judgment first, then the decision.

---

> [!NOTE]
> This repo explains how guardrails work and how they fail, using published research. It doesn't include jailbreak prompts or recipes for stripping safety out of models, and it won't. The goal is to help people build and ship models responsibly, not the opposite.

---

## About me

**Bhoobalan B R**, Software Development Engineer II, specialized in AWS native solutions and high-performance distributed systems. Building scalable applications serving thousands of daily users, with a focus on reliability, performance and cost efficiency. I create and contribute to open-source projects, like [hyperswitch-aws-toolkit](https://github.com/bhoobalan-bhoo/hyperswitch-aws-toolkit).

🌐 [bhoobalan.in](https://www.bhoobalan.in)

---

If something here is wrong or out of date <> open an issue. Models and model cards change fast, and I'd rather fix it than leave it wrong.

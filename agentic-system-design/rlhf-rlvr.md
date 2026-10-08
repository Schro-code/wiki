---
domain: agentic-system-design
tags: [agentic-system-design]
sources: []
updated: 2026-09-21
---

# RLHF and RLVR

## What it is

Two training techniques used to improve a pretrained LLM after the initial pretraining phase. RLHF (Reinforcement Learning from Human Feedback) aligns the model to human preferences using a learned reward model. RLVR (Reinforcement Learning from Verifiable Rewards) uses an automatic verifier instead, making it cheaper and the source of reasoning-focused models like DeepSeek-R1 and o1.

## What problem it solves

A base LLM (e.g. Qwen, Llama) is trained to predict the next token from internet text. It has no concept of being an assistant and no incentive to be helpful, safe, or correct. RLHF and RLVR are post-training stages that shape behavior beyond raw next-token prediction.

## Notes

### LLM architecture refresher

Every transformer LLM has two parts:

```
Input tokens
     │
     ▼
┌─────────────────────┐
│  Transformer body   │  ← attention + FFN layers, produces hidden states
└─────────────────────┘
     │
     ▼
┌─────────────────────┐
│    Head (last layer)│  ← single linear layer
└─────────────────────┘
     │
     ▼
  LLM head:           [vocab_size logits]  → sample next token
  Reward model head:  [1 scalar]           → quality score
```

The "head" is just the last layer. Swapping it is the key operation when building a reward model from an existing LLM.

---

### RLHF — full pipeline

The pipeline has three sequential stages:

**Stage 1 — SFT (Supervised Fine-Tuning)**

Base LLM → instruction-following model.

The base model completes text; it doesn't know it's an assistant. SFT teaches it by training on (instruction, response) pairs written by humans, using standard cross-entropy loss. After SFT the model answers questions and follows instructions — but it isn't yet aligned to preferences.

SFT is necessary before RL because the RL signal is too weak to teach instruction-following from scratch. SFT gets the model 80% of the way; RL refines the rest.

**Stage 2 — Reward Model training**

Training data are *comparison pairs*, not scores:

```
Prompt:     "Explain recursion"
Response A: "It's when a function calls itself to solve a subproblem." ← chosen ✓
Response B: "Recursion is complex. It refers to self-reference."      ← rejected ✗
```

You take the SFT model, swap the head to output 1 scalar, and train on thousands of these pairs. The loss punishes the model when it scores a rejected response higher than the chosen one. After training, the reward model can score any response it has never seen.

**Stage 3 — RL loop (PPO)**

Two models in play:
- **Policy** — the LLM being updated (starts from the SFT weights)
- **Reference policy** — a frozen copy of the SFT model

One training step:

```
1. Sample a prompt
2. Policy generates a response
3. Reward model scores it
4. Compute: actual_reward = RM_score − β × KL(policy ∥ reference)
5. PPO updates the policy weights to increase probability of
   token choices that led to high reward
```

The **KL penalty** is critical. Without it the policy quickly finds nonsense outputs that happen to fool the reward model (reward hacking). The KL term pulls the policy back toward the reference whenever it drifts too far.

PPO avoids differentiating through the discrete sampling step by using a ratio trick:

```
ratio = P(token | new weights) / P(token | old weights)

gradient signal = ratio × reward (clipped to prevent large updates)
```

---

### RLVR — the one change that matters

Replace the reward model with a **verifier** — a deterministic program:

```
RLHF:  LLM output → Reward Model (neural net)  → continuous score
RLVR:  LLM output → Verifier (checker program) → 0 or 1
```

Examples of verifiers:
- Math: `answer == ground_truth`
- Code: run test suite → all pass = 1, any fail = 0
- Logic: validate proof steps against formal rules

No human raters, no reward model, no reward hacking from a learned scorer. The domain itself provides the reward.

**Concrete example — math:**

```
Prompt: "What is 17 × 24?"

Run A:  "17 × 20 = 340, 17 × 4 = 68, 340 + 68 = 408" → verifier: 1 ✓
Run B:  "17 × 24 = 391"                               → verifier: 0 ✗
```

**Why RLVR produces "thinking" models (emergent behavior)**

The model was never told to reason step by step. But it discovers that structured intermediate steps lead to correct answers reliably, while direct guessing is noisy. After enough training steps, long chain-of-thought reasoning is reinforced because it wins more often. This is the mechanism behind DeepSeek-R1 and o1.

---

### GRPO — the RL algorithm DeepSeek used instead of PPO

PPO requires a separate critic/value network to estimate future expected reward at each step. GRPO eliminates it:

```
1. Take one prompt
2. Generate N responses (e.g. N = 8)
3. Score all N with the verifier
4. Compute average reward across the group
5. Responses above average → positive gradient (reinforce)
   Responses below average → negative gradient (suppress)
```

No critic model. Simpler, cheaper, and well-suited to binary verifiable rewards.

---

### Comparison table

| | RLHF | RLVR |
|---|---|---|
| Reward source | Human raters → learned reward model | Automatic verifier |
| Reward type | Continuous | Binary (0 or 1) |
| Reward model | Required (neural net) | Not needed |
| Works for | Any task | Verifiable domains only (math, code, logic) |
| Cost | High (human labels) | Low (automated) |
| What it trains | Alignment, instruction-following, safety | Reasoning, step-by-step correctness |
| RL algorithm | PPO | PPO or GRPO |
| Key risk | Reward hacking via RM | Reward hacking via test suite leakage |
| Examples | GPT-4, Claude, Gemini | DeepSeek-R1, o1, QwQ |

---

### Relation to agents

Both techniques shape the model that sits at the center of every agent:

- **RLHF** is why the model understands tool descriptions, follows system prompts, and behaves safely.
- **RLVR** is why reasoning-focused models (o1, DeepSeek-R1) are dramatically better at multi-step agentic tasks: planning, debugging, generating SQL, writing code end-to-end.
- **Agent RLVR** (active research area): apply RLVR directly to full agentic trajectories — reward the model when a sequence of tool calls successfully completes a task. The verifier is the task outcome (tests pass, file was written correctly, etc.).

## Relations

- [[overview]] — maps the broader agentic system design field; RLHF/RLVR are the training-time foundation of the models used in agents
- [[chapter3-user-memory-systems]] — cross-session memory systems run on top of models shaped by these training stages
- [[ai-agent-book/index]] — chapters on evaluation and multi-agent systems assume models trained with these techniques

## Tags

#agentic-system-design

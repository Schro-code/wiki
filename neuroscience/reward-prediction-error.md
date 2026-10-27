---
domain: neuroscience
tags: [neuroscience, reward-learning]
sources: []
updated: 2026-10-27
---

# Reward Prediction Error

## What it is

Reward prediction error (RPE) is the signed gap between what you got and what you expected: surprise = outcome − prediction. The theory says some dopamine neurons broadcast this gap, and the brain uses it as its teaching signal. This page also covers what follows from it: how the error moves back through time (temporal-difference learning), why a redundant cue earns no credit (blocking), why dopamine is not the happiness chemical (wanting vs liking), and where the theory is contested.

## What problem it solves

To act well, the brain needs a model of the world: "this tree usually gives sweet fruit". The question is how to keep that model accurate without relearning everything every day. The answer is to learn only from mismatch. Where the world matches the model, there is nothing to learn, and changing the model could only damage it. Where the world disagrees, the model is wrong exactly there, so that is the place to fix it.

## Notes

### Contents

1. Signed surprise and the update rule
2. What dopamine neurons do
3. Temporal-difference learning
4. Blocking and credit assignment
5. Wanting vs liking
6. Open questions

### 1. Signed surprise and the update rule

**The intuition: a monkey and a fruit tree.** A monkey expects sweet fruit from a tree. Three things can happen:

| Outcome | Surprise | What the brain does |
|---|---|---|
| Better than expected (an amazing fruit) | positive | pushes the prediction up |
| Exactly as expected | zero | changes nothing |
| Worse than expected (a lemon) | negative | pushes the prediction down |

The sign matters. A merely-good meal when an amazing one was expected is a negative surprise, so it lowers the expectation even though the meal was good. Surprise is relative to the prediction, not to zero.

**The update rule.** Nudge the prediction a fraction $\alpha$ of the way toward the outcome:

$$\text{new} = \text{old} + \alpha \times (\text{outcome} - \text{old})$$

Why a fraction and not all the way:

- Jumping fully to the latest outcome ($\alpha = 1$) lets one noisy event overwrite a lot of accumulated evidence. One lemon from a good tree would erase everything you knew.
- Ignoring the outcome ($\alpha = 0$) never adapts. The model goes stale when the world changes.

Rule of thumb: little history means a larger $\alpha$ (early on, each event is most of what you know), and lots of history means a smaller $\alpha$.

### 2. What dopamine neurons do

The classic pattern, from recordings of midbrain dopamine neurons in conditioning tasks:

| Situation | Dopamine response |
|---|---|
| Unpredicted reward | burst of firing |
| Fully predicted reward (after learning) | no response to the reward; the burst has moved to the earliest reliable predictor (see section 3) |
| Predicted reward omitted | dip below baseline at the moment the reward was due |

This is exactly the signed-surprise table above: positive, zero, negative. The source of the three patterns is Schultz 1998, J Neurophysiol (doi 10.1152/jn.1998.80.1.1). The 1997 Schultz, Dayan & Montague paper (Science 275:1593) connected them to temporal-difference theory.

Approximate numbers, from one measurement (Bayer & Glimcher 2005): baseline firing around 5.3 spikes/s, and around 26 Hz for an unpredicted reward. Treat these as ballpark values, not constants.

**Check on the sign.** A monkey that learned "light means 3 drops" and then gets only 1 drop (still pleasant) shows a dip below baseline, because the outcome is less than predicted.

**Why the negative side is coarser.** Baseline firing is low, so a dip has limited room to go down. For large negative errors firing hits roughly zero (Bayer & Glimcher 2005), so the neuron cannot say "much worse" with the same resolution as "much better". Lateral habenula neurons show the opposite pattern, and their excitation precedes dopamine inhibition (Matsumoto & Hikosaka 2007, Nature 447:1111). This suggests other systems contribute to the negative side.

**Evidence that it is really an error signal.**

- Waelti, Dickinson & Schultz 2001 (Nature 412:43): in a blocking procedure, dopamine responses followed prediction errors and not mere pairing of cue and reward.
- Steinberg et al. 2013 (Nat Neurosci 16:966): in rats, optogenetic dopamine activation at reward time produced lasting increases in cue-triggered reward seeking in blocking and extinction procedures, so the signal can act as a teaching signal, not just a correlate.
- Eshel et al. 2015 (Nature 525:243): VTA dopamine neurons perform subtraction (reward minus expectation), and neighboring GABA neurons supply the expectation signal.
- Eshel et al. 2016 (Nat Neurosci 19:479): neurons share a common response function that differs mainly in scale.
- Dabney et al. 2020 (Nature 577:671): distributional RL. Some neurons are optimistic and others pessimistic, so the brain may code a distribution of outcomes and not a single mean. This extends the theory rather than rejecting it.
- Tsai 2009 (Science 324:1080): phasic dopamine firing is sufficient to drive conditioning, but that is not a test specific to prediction error.

### 3. Temporal-difference learning

**The problem.** A surprise at the moment of reward arrives too late to act on. By then the food is already in your mouth. The animal needs to learn which earlier events lead to reward, so it can act on those. TD learning solves this by letting each step learn from the next step's prediction, without waiting for the final outcome.

**The intuition: bell, then food.** Day 1: food appears with no warning. Large surprise at the food. After many days: a bell always precedes the food. Now the bell is the surprise (nothing predicted it), and the food is no longer surprising because the bell already predicted it. The surprise has moved from the reward to the cue.

It keeps moving back along a chain. If a door creaks before the bell rings, and the creak reliably predicts the bell, then the creak becomes the surprise and the bell stops being one. The error ends up at the earliest event that nothing else predicts.

```
Before learning:   [creak] -> [bell] -> [food]
                                         ^ surprise

After learning:    [creak] -> [bell] -> [food]
                      ^ surprise
```

**The formula.**

$$\delta_t = R_{t+1} + \gamma \, V(S_{t+1}) - V(S_t)$$

Read it as: surprise = (reward just received + discounted value of where I am now) − (value I predicted for where I was).

- $V(S)$ is the predicted future reward from state $S$.
- $\gamma$ is the discount factor between 0 and 1. It weights near rewards above distant ones.
- If the next state looks better than the current prediction, $\delta$ is positive, even before any reward arrives. That is why a cue that signals good news produces an error.

The key TD paper is Sutton 1988 (Machine Learning).

**Why moving back is useful.** The error at the cue is something the animal can use: it is both a lesson and a push to act (see section 5). An error only at the reward could teach but could not motivate any earlier action.

### 4. Blocking and credit assignment

**The problem.** When many things happen before a reward, which one caused it? If the brain credited everything that was merely present, it would learn endless false links (the color of the wall, the weather). Credit assignment by surprise solves this: only what explains an otherwise unexplained outcome gets credit.

**The intuition.** You already know the bell predicts food perfectly. Now a light comes on together with the bell, and food follows. Was the food a surprise? No, the bell had already predicted it completely. With zero surprise there is nothing to learn, so the light earns no credit.

**The generic design.**

| Phase | Training | Result |
|---|---|---|
| 1 | A → reward | A comes to predict the reward |
| 2 | A+B → reward | no surprise, since A already explains the reward |
| Test | B alone | little or no response to B |

B is not intrinsically unlearnable: it is blocked only because A already explained the reward. (This is the generic design. Details of Kamin's 1969 original experiment are not stated here because they are unverified.)

**The Rescorla-Wagner idea.**

$$\Delta V_X = \alpha_X \, \beta \, (\lambda - V_{\text{total}})$$

- $\lambda$ is the outcome (the maximum the reward can support).
- $V_{\text{total}}$ is the sum of predictions across all stimuli present, not just $V_X$.
- $\alpha_X$ and $\beta$ are learning-rate factors for the cue and the outcome.

The key point is $V_{\text{total}}$. Every cue present shares one surprise, $\lambda - V_{\text{total}}$. In phase 2, $V_A$ already equals $\lambda$, so the shared surprise is zero and $\Delta V_B = 0$. Writing it as $\lambda - V_X$ would wrongly let B learn from its own prediction of zero.

**Dopamine and blocking.** The two studies listed in section 2 (Waelti et al. 2001; Steinberg et al. 2013) show dopamine following prediction errors in blocking procedures.

**Why it matters for habits.** Adding more reminders or rewards to a habit that is already fully predicted adds little. It is also the reason a reward that arrives regardless teaches nothing about the effort (see [[effort-conditioning#5. Rules for rewiring]]).

### 5. Wanting vs liking

**The puzzle.** After learning, the dopamine burst moves from the reward to the cue. If dopamine were pleasure, a fully predicted meal should feel like nothing. It does not. So the burst is not the pleasure itself. It marks the moment of surprise and anticipation.

| | Wanting | Liking |
|---|---|---|
| What it is | pull toward something | pleasure while it happens |
| Everyday example | reaching for the phone | enjoying a good meal |
| Linked to dopamine | closely, as a push toward cues | less clearly (this split is debated) |

Scrolling a phone is the standard case: high wanting (you keep reaching for it), low liking (you often do not enjoy it much). The cue pulls, and the pull is trained by past errors.

**From error to motivation.** The burst at a cue plays two roles at once: a lesson ("this cue leads to good things") and a push ("go get it"). How big the push is depends on how good the cue looks, which depends on past errors. After a lemon instead of sweet fruit, the prediction for that tree drops, so tomorrow the tree gives a smaller push. Whether the push and the teaching signal are the same signal is debated.

### 6. Open questions

Popular accounts present RPE as the whole story of dopamine. The core pattern (burst, nothing, dip) is well supported. What is not settled is whether it explains everything dopamine does. Knowing where it is debated stops you from over-applying it, especially to everyday life.

1. **Dopamine also relates to movement and motivation.** If dopamine only coded error, it would not track movement or effort. Berke 2018 (Nat Neurosci 21:787) argues for a broader role in motivation. Hamid et al. 2016 (Nat Neurosci 19:117): dopamine tracks value and reward rate. Howe & Dombeck 2016 (Nature 535:505): dopamine axons tied to locomotion. Coddington & Dudman 2019 (Neuron): a review of the evidence for several roles.
2. **Slow ramps toward distant rewards.** Instead of a brief burst, dopamine sometimes rises slowly as an animal approaches a reward (Howe et al. 2013, Nature). Gershman 2014 (Neural Comput) argues the ramps can be reconciled with TD errors. Whether they are errors or a separate value signal is debated.
3. **Novelty, salience and aversive events.** Dopamine neurons also respond to some novel, salient or aversive events. The aversion papers Matsumoto 2016 and Menegas 2018 were verified by title only. The novelty literature was not verified. Findings are mixed, so treat them as open.
4. **Weak encoding of negative errors.** The dip is limited by the low baseline (see section 2), and other systems such as the lateral habenula probably contribute.
5. **Is the push the same as the teaching signal?** The cue-time burst may both teach and motivate. Whether these are one signal or two is open.
6. **Extrapolation to everyday life and addiction.** Claims such as "your baseline rises so natural rewards feel like negative errors" are a popular-science hypothesis and are not established. A popular video presented it as fact. Hold it as a hypothesis.

**An extension, not a rejection.** Distributional RL (Dabney et al. 2020, Nature 577:671) shows some neurons are optimistic and others pessimistic, so the brain may code a distribution of outcomes. This refines the theory.

### Evidence note

Verified against abstracts and metadata via Europe PMC and Wikipedia (web search was blocked): the citations above with journal and volume, the three canonical response patterns, the TD formula, Sutton 1988 as the key TD paper, and the Rescorla-Wagner form with summed prediction. Verified by title only: Matsumoto 2016, Menegas 2018. Not verified: the novelty literature, specifics of Kamin 1969, and Berridge's work on a wanting/liking split (animal research on it exists and is associated with Kent Berridge; the distinction itself is debated). From memory and not verified: species and brain areas (macaque; VTA and substantia nigra pars compacta). Unverified: whether the 1997 Science paper was a model synthesis of earlier recordings and not a new recording study (most likely a synthesis).

## Relations

- [[effort-conditioning]] — the same "value is learned from what follows" idea applied to the sensation of effort, plus how memory files an episode
- [[discipline-playbook]] — techniques and hypotheses for applying this theory to habits, with an evidence map
- [[rlhf-rlvr]] — the same family of reward-driven learning ideas, used to train LLMs (loose connection, different field)

## Tags

#neuroscience #reward-learning

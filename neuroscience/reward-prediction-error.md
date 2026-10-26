---
domain: neuroscience
tags: [neuroscience, reward-learning]
sources: []
updated: 2026-10-26
---

# Reward Prediction Error

## What it is

Reward prediction error (RPE) is the signed gap between what you got and what you expected: surprise = outcome − prediction. The theory says some dopamine neurons broadcast this gap, and the brain uses it as its teaching signal.

## What problem it solves

To act well, the brain needs a model of the world: "this tree usually gives sweet fruit". The question is how to keep that model accurate without relearning everything every day. The answer is to learn only from mismatch. Where the world matches the model, there is nothing to learn, and changing the model could only damage it. Where the world disagrees, the model is wrong exactly there, so that is the place to fix it.

## Notes

### The intuition: a monkey and a fruit tree

A monkey expects sweet fruit from a tree. Three things can happen:

| Outcome | Surprise | What the brain does |
|---|---|---|
| Better than expected (an amazing fruit) | positive | pushes the prediction up |
| Exactly as expected | zero | changes nothing |
| Worse than expected (a lemon) | negative | pushes the prediction down |

The sign matters. A merely-good meal when an amazing one was expected is a negative surprise, so it lowers the expectation even though the meal was good. Surprise is relative to the prediction, not to zero.

### The update rule

Nudge the prediction a fraction $\alpha$ of the way toward the outcome:

$$\text{new} = \text{old} + \alpha \times (\text{outcome} - \text{old})$$

Why a fraction and not all the way:

- Jumping fully to the latest outcome ($\alpha = 1$) lets one noisy event overwrite a lot of accumulated evidence. One lemon from a good tree would erase everything you knew.
- Ignoring the outcome ($\alpha = 0$) never adapts. The model goes stale when the world changes.

Rule of thumb: little history means a larger $\alpha$ (early on, each event is most of what you know), and lots of history means a smaller $\alpha$.

### What dopamine neurons do (canonical recordings)

The classic pattern, from recordings of midbrain dopamine neurons in conditioning tasks:

| Situation | Dopamine response |
|---|---|
| Unpredicted reward | burst of firing |
| Fully predicted reward (after learning) | no response to the reward; the burst has moved to the earliest reliable predictor (see [[temporal-difference-learning]]) |
| Predicted reward omitted | dip below baseline at the moment the reward was due |

This is exactly the signed-surprise table above: positive, zero, negative. The source of the three patterns is Schultz 1998, J Neurophysiol (doi 10.1152/jn.1998.80.1.1). The 1997 Schultz, Dayan & Montague paper (Science 275:1593) connected them to temporal-difference theory.

Approximate numbers, from one measurement (Bayer & Glimcher 2005): baseline firing around 5.3 spikes/s, and around 26 Hz for an unpredicted reward. Treat these as ballpark values, not constants.

### Why the negative side is coarser

Baseline firing is low, so a dip has limited room to go down. For large negative errors firing hits roughly zero (Bayer & Glimcher 2005), so the neuron cannot say "much worse" with the same resolution as "much better". Lateral habenula neurons show the opposite pattern, and their excitation precedes dopamine inhibition (Matsumoto & Hikosaka 2007, Nature 447:1111). This suggests other systems contribute to the negative side. See [[rpe-open-questions]].

### Evidence that it is really an error signal

- Waelti, Dickinson & Schultz 2001 (Nature 412:43): in a blocking procedure, dopamine responses followed prediction errors and not mere pairing of cue and reward. See [[blocking-and-credit-assignment]].
- Steinberg et al. 2013 (Nat Neurosci 16:966): in rats, optogenetic dopamine activation at reward time produced lasting increases in cue-triggered reward seeking in blocking and extinction procedures, so the signal can act as a teaching signal, not just a correlate.
- Eshel et al. 2015 (Nature 525:243): VTA dopamine neurons perform subtraction (reward minus expectation), and neighboring GABA neurons supply the expectation signal.
- Eshel et al. 2016 (Nat Neurosci 19:479): neurons share a common response function that differs mainly in scale.
- Dabney et al. 2020 (Nature 577:671): distributional RL. Some neurons are optimistic and others pessimistic, so the brain may code a distribution of outcomes and not a single mean. This extends the theory rather than rejecting it.
- Tsai 2009 (Science 324:1080): phasic dopamine firing is sufficient to drive conditioning, but that is not a test specific to prediction error.

### From error to motivation

The burst at the cue plays two roles at once: a lesson ("this cue leads to good things") and a push ("go get it"). How big the push is depends on how good the cue looks, which depends on past errors. After a lemon instead of sweet fruit, the prediction for that tree drops, so tomorrow the tree gives a smaller push. Whether the push and the teaching signal are the same signal is debated. See [[dopamine-wanting-vs-liking]].

### Evidence note

Verified against abstracts and metadata via Europe PMC and Wikipedia (web search was blocked): the citations above and the three canonical response patterns. From memory and not verified: species and brain areas (macaque; VTA and substantia nigra pars compacta). Unverified: whether the 1997 Science paper was a model synthesis of earlier recordings and not a new recording study (most likely a synthesis).

## Relations

- [[temporal-difference-learning]] — extends the error to chains of predictors, explaining why the burst moves to the cue
- [[blocking-and-credit-assignment]] — the behavioral test that shows learning follows surprise, not co-occurrence
- [[dopamine-wanting-vs-liking]] — why dopamine as an error and motivation signal is not the same as happiness
- [[rpe-open-questions]] — where the theory is contested
- [[rpe-and-habit-change]] — reasoned applications to self-learning and habits
- [[rlhf-rlvr]] — the same family of reward-driven learning ideas, used to train LLMs (loose connection, different field)

## Tags

#neuroscience #reward-learning

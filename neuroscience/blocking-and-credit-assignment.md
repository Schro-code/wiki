---
domain: neuroscience
tags: [neuroscience, reward-learning]
sources: []
updated: 2026-10-26
---

# Blocking and Credit Assignment

## What it is

Blocking is the finding that a cue which is redundant earns no credit. If cue A already predicts a reward fully, adding cue B alongside it teaches nothing about B, even though B is present every time the reward arrives. Credit goes to unexplained surprise, not to co-occurrence.

## What problem it solves

When many things happen before a reward, which one caused it? If the brain credited everything that was merely present, it would learn endless false links (the color of the wall, the weather). Credit assignment by surprise solves this: only what explains an otherwise unexplained outcome gets credit.

## Notes

### The intuition

You already know the bell predicts food perfectly. Now a light comes on together with the bell, and food follows. Was the food a surprise? No, the bell had already predicted it completely. With zero surprise there is nothing to learn, so the light earns no credit.

### The generic design

| Phase | Training | Result |
|---|---|---|
| 1 | A → reward | A comes to predict the reward |
| 2 | A+B → reward | no surprise, since A already explains the reward |
| Test | B alone | little or no response to B |

B is not intrinsically unlearnable: it is blocked only because A already explained the reward. (This is the generic design. Details of Kamin's 1969 original experiment are not stated here because they are unverified.)

### The Rescorla-Wagner idea

$$\Delta V_X = \alpha_X \, \beta \, (\lambda - V_{\text{total}})$$

- $\lambda$ is the outcome (the maximum the reward can support).
- $V_{\text{total}}$ is the sum of predictions across all stimuli present, not just $V_X$.
- $\alpha_X$ and $\beta$ are learning-rate factors for the cue and the outcome.

The key point is the $V_{\text{total}}$. Every cue present shares one surprise, $\lambda - V_{\text{total}}$. In phase 2, $V_A$ already equals $\lambda$, so the shared surprise is zero and $\Delta V_B = 0$. Writing it as $\lambda - V_X$ would wrongly let B learn from its own prediction of zero.

### Dopamine and blocking

Waelti, Dickinson & Schultz 2001 (Nature 412:43) recorded dopamine responses in a blocking procedure, and they followed prediction errors, not mere pairing of cue and reward. Steinberg et al. 2013 (Nat Neurosci 16:966) went further in rats: optogenetic dopamine activation at reward time unblocked cue learning, which suggests the signal can act as the teaching signal. Details in [[reward-prediction-error]].

### Evidence note

Verified: the Rescorla-Wagner form with summed prediction, the two dopamine blocking studies. Unverified: specifics of Kamin 1969.

## Relations

- [[reward-prediction-error]] — blocking is the behavioral signature of learning from surprise
- [[temporal-difference-learning]] — the same principle along a time chain instead of across simultaneous cues
- [[rpe-and-habit-change]] — why adding more reminders or rewards to a habit that is already predicted adds little
- [[rpe-open-questions]] — limits of what blocking and the model explain

## Tags

#neuroscience #reward-learning

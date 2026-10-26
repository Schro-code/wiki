---
domain: neuroscience
tags: [neuroscience, reward-learning]
sources: []
updated: 2026-10-26
---

# Temporal-Difference Learning

## What it is

Temporal-difference (TD) learning is reward prediction error extended through time: the error is computed at every step from the change in predicted future reward, so it moves backward from the reward to the earliest event that reliably predicts it.

## What problem it solves

A surprise at the moment of reward arrives too late to act on. By then the food is already in your mouth. What the animal needs is to learn which earlier events lead to reward, so it can act on those. TD learning solves this by letting each step learn from the next step's prediction, without waiting for the final outcome.

## Notes

### The intuition: bell, then food

Day 1: food appears with no warning. Large surprise at the food.

After many days: a bell always precedes the food. Now the bell is the surprise (nothing predicted it), and the food is no longer surprising because the bell already predicted it. The surprise has moved from the reward to the cue.

It keeps moving back along a chain. If a door creaks before the bell rings, and the creak reliably predicts the bell, then the creak becomes the surprise and the bell stops being one. The error ends up at the earliest event that nothing else predicts.

```
Before learning:   [creak] -> [bell] -> [food]
                                         ^ surprise

After learning:    [creak] -> [bell] -> [food]
                      ^ surprise
```

### The formula

$$\delta_t = R_{t+1} + \gamma \, V(S_{t+1}) - V(S_t)$$

Read it as: surprise = (reward just received + discounted value of where I am now) − (value I predicted for where I was).

- $V(S)$ is the predicted future reward from state $S$.
- $\gamma$ is the discount factor between 0 and 1. It weights near rewards above distant ones.
- If the next state looks better than the current prediction, $\delta$ is positive, even before any reward arrives. That is why a cue that signals good news produces an error.

The key TD paper is Sutton 1988 (Machine Learning).

### Link to the dopamine recordings

The canonical recordings (see [[reward-prediction-error]]) match this picture: after learning, the dopamine burst shifts to the earliest reliable predictor, the fully predicted reward gets no response, and an omitted reward gives a dip at the time it was due. The 1997 Schultz, Dayan & Montague paper (Science 275:1593) made this link to TD theory explicit.

### Why moving back is useful

The error at the cue is something the animal can use: it is both a lesson and a push to act (see [[dopamine-wanting-vs-liking]]). An error only at the reward could teach but could not motivate any earlier action.

### Evidence note

Verified: the TD formula, Sutton 1988 as the key TD paper, the three response patterns. Unverified: that Schultz, Dayan & Montague 1997 was most likely a model synthesis of earlier recordings and not a new recording study. From memory: macaque and VTA/SNc as the recording species and areas.

## Relations

- [[reward-prediction-error]] — the one-step version of the idea; TD generalizes it across time
- [[blocking-and-credit-assignment]] — the same "learn from unexplained surprise" principle applied across cues
- [[dopamine-wanting-vs-liking]] — why the burst at the cue is felt as anticipation, not pleasure
- [[rpe-open-questions]] — slow dopamine ramps toward distant rewards and how TD accounts for them
- [[rpe-and-habit-change]] — using early, real progress signals to pull the surprise forward
- [[rlhf-rlvr]] — TD-style value learning is a building block of modern RL, which also trains LLMs (loose connection, different field)

## Tags

#neuroscience #reward-learning

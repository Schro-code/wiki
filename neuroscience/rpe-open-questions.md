---
domain: neuroscience
tags: [neuroscience, reward-learning]
sources: []
updated: 2026-10-26
---

# RPE: Open Questions

## What it is

A map of where the reward prediction error theory of dopamine is contested or incomplete. The core pattern (burst, nothing, dip) is well supported. What is not settled is whether it explains everything dopamine does.

## What problem it solves

Popular accounts present RPE as the whole story of dopamine. Knowing where it is debated stops you from over-applying it, especially to everyday life.

## Notes

### 1. Dopamine also relates to movement and motivation

If dopamine only coded error, it would not track movement or effort. But:

- Berke 2018 (Nat Neurosci 21:787) argues for a broader role in motivation.
- Hamid et al. 2016 (Nat Neurosci 19:117): dopamine tracks value and reward rate.
- Howe & Dombeck 2016 (Nature 535:505): dopamine axons tied to locomotion.
- Coddington & Dudman 2019 (Neuron): a review of the evidence for several roles.

### 2. Slow ramps toward distant rewards

Instead of a brief burst, dopamine sometimes rises slowly as an animal approaches a reward (Howe et al. 2013, Nature). Gershman 2014 (Neural Comput) argues the ramps can be reconciled with TD errors. Whether they are errors or a separate value signal is debated. See [[temporal-difference-learning]].

### 3. Novelty, salience and aversive events

Dopamine neurons also respond to some novel, salient or aversive events. The aversion papers Matsumoto 2016 and Menegas 2018 were verified by title only. The novelty literature was not verified. Findings here are mixed, so treat them as open.

### 4. Weak encoding of negative errors

The dip is limited by the low baseline: for large negative errors firing hits roughly zero (Bayer & Glimcher 2005). Lateral habenula neurons show the opposite pattern, with excitation preceding dopamine inhibition (Matsumoto & Hikosaka 2007, Nature 447:1111), which suggests other systems contribute. See [[reward-prediction-error]].

### 5. Is the push the same as the teaching signal?

The cue-time burst may both teach and motivate. Whether these are one signal or two is open. See [[dopamine-wanting-vs-liking]].

### 6. Extrapolation to everyday life and addiction

Claims such as "your baseline rises so natural rewards feel like negative errors" are a popular-science hypothesis and are not established. A popular video presented it as fact. This should be held as a hypothesis. See [[rpe-and-habit-change]].

### An extension, not a rejection

Distributional RL (Dabney et al. 2020, Nature 577:671) shows some neurons are optimistic and others pessimistic, so the brain may code a distribution of outcomes. This refines the theory.

### Evidence note

Verified: the citations listed with journal and volume. Verified by title only: Matsumoto 2016, Menegas 2018. Not verified: the novelty literature.

## Relations

- [[reward-prediction-error]] — the theory these questions test
- [[temporal-difference-learning]] — the framework used to explain slow ramps
- [[dopamine-wanting-vs-liking]] — the motivation side of dopamine
- [[blocking-and-credit-assignment]] — one of the best-supported predictions of the theory
- [[rpe-and-habit-change]] — why applications to daily life stay hypotheses

## Tags

#neuroscience #reward-learning

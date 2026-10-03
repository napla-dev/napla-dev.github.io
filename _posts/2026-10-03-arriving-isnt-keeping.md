---
layout: post
title: "Arriving Isn't Keeping"
subtitle: "Splitting pass success into reaching the receiver and keeping the ball"
description: "Pass completion models stop when the ball arrives. Using PFF FC's 2022 World Cup tracking data, I add a second step: can the receiver keep it? What the split shows, where it doesn't help, and what it can't see."
math: true
image: /assets/img/reception/fig_loss_decomposition.png
---

*52,073 passes · 49 World Cup matches · PFF FC open tracking data*

> **TL;DR**
> - A pass that arrives is not always a pass that keeps the ball. I model the two steps separately: the probability that a pass **reaches** its receiver, and the probability that the receiver then **keeps** it.
> - Multiplying the two (I call it xSP, expected safe pass) splits each lost ball into "lost before reaching" and "lost after reaching". For short and medium ground passes, roughly a third of expected losses come after the ball has arrived.
> - As a pure predictor, the split doesn't win: a model trained directly on "still in possession 3 seconds later" is more accurate. Its value is in showing *where* the ball is lost, not in predicting it better.
> - Receptions that are already a duel when the ball arrives are lost about half the time within 3 seconds, and my model can't predict which ones.

## 1. The question

Pass models usually measure success at one moment: did the ball reach a teammate? But anyone who watches football has seen a "completed" pass arrive with a defender on the receiver's back and the ball gone a second later.

I wanted to measure that second step. Given that a pass arrives, how likely is the receiver to keep the ball, and what does the situation at the moment of the pass tell us about it?

## 2. Setup

**Data.** PFF FC's open 2022 World Cup data: tracking for 49 matches and the event data, which records each first touch and how each player's spell on the ball ends.

**What counts as keeping the ball.** A reception counts as *kept* if the receiver's spell ends with a deliberate pass, cross, shot or clearance (whatever happens to it), a foul won, or the team still in possession. It counts as *lost* if the receiver loses a duel, miscontrols the ball, or is dispossessed while carrying it. I also checked a stricter version that counts a pass or clearance given straight to the opponent within 3 seconds as a loss. That raises the failure rate from 7% to 17%. The main conclusions hold under both definitions, with one exception noted in Section 5.

**One fix worth mentioning.** My first features used velocities from a centred moving average, which quietly included about 0.13 s of the future. I rebuilt every feature from the last frame at least 0.15 s before the event. The results barely moved, but if you build features from smoothed tracking data, this is easy to miss.

## 3. Step one: what predicts a lost first touch?

For 32,298 ordinary receptions (the receiver controls the ball rather than playing it first time), 5.9% end in a lost ball. Adding tracking features to a baseline feature set improves prediction clearly (AUC 0.76 → 0.85 with gradient boosting, match-level bootstrap interval for the log-loss gain well away from zero), and the predicted probabilities are well calibrated.

The situations most associated with losing the ball are unsurprising, which is reassuring for the model:

| Situation at the moment of reception | Loss rate |
|---|---|
| Nearest defender within 4.2 m | 16.8% (0.7% beyond 12 m) |
| Receiver running faster than 2.9 m/s | 11.1% (3.0% when nearly stationary) |
| Pass faster than 16 m/s | 9.0% |

These are associations, not causes, and the situations overlap with each other.

## 4. Step two: splitting a pass into reaching and keeping

From the moment the pass is played, I estimate two probabilities, using only information available at that moment:

$$
\text{xSP} = \underbrace{P(\text{reaches the receiver})}_{\text{xPass}} \times \underbrace{P(\text{keeps it} \mid \text{reached})}_{\text{xControl}}
$$

The probability of losing the ball then splits into two parts:

$$
1 - \text{xSP} = \underbrace{(1 - \text{xPass})}_{\text{lost before reaching}} + \underbrace{\text{xPass}\,(1 - \text{xControl})}_{\text{lost after reaching}}
$$

<figure>
  <img src="{{ '/assets/img/reception/fig_loss_decomposition.png' | relative_url }}" alt="Stacked bars of expected loss before and after reaching the receiver, by pass type and by location, with observed loss rates as dots">
  <figcaption>Figure 1. Expected losses split into "before reaching" (blue) and "after reaching" (orange), with the observed rate of losing the ball within 5 seconds as dots. Crosses sit well above their dot because the model counts every cross that misses its target as a loss, while the 5-second outcome does not (a blocked cross that goes out for a corner keeps possession).</figcaption>
</figure>

Three things stand out:

- **Long, lofted passes and crosses are lost mostly before they arrive.** The blue part dominates.
- **Short and medium ground passes are different.** They rarely fail to arrive, so the part lost after reaching is roughly a third or more of their expected losses.
- **In the final third, losses after reaching grow too.** The ball arrives in tighter spaces, and keeping it gets harder.

For most groups, the model's total sits close to the observed rate. Long passes, lofted passes and set pieces are overestimated by a few points, and crosses by much more, for the reason given in the caption.

## 5. Does the split predict better?

I compared three ways of predicting whether the team still has the ball 3 or 5 seconds after the pass: xPass alone, xSP, and a model trained directly on that outcome using the same features.

<figure>
  <img src="{{ '/assets/img/reception/fig_auc_comparison.png' | relative_url }}" alt="AUC with confidence intervals for xPass, xSP and a direct model at 3 and 5 seconds; the direct model is highest">
  <figcaption>Figure 2. AUC for predicting possession 3 and 5 seconds after the pass. The intervals for xPass and xSP overlap, but in a paired comparison on the same matches xSP is slightly better (+0.003). The direct model is better than both.</figcaption>
</figure>

xSP improves on xPass, but only slightly (AUC +0.003). With the stricter definition of keeping the ball, it no longer beats xPass at 3 seconds. A model trained directly on the outcome does better still (+0.018 over xSP at 3 seconds).

So as a predictor, splitting the pass into two steps doesn't pay. What it adds is an explanation: the direct model can say *how risky* a pass is, but not whether the risk lies in getting the ball there or in keeping it once it arrives. That distinction is what a coach or analyst would act on, and it is the reason to keep the two parts separate.

## 6. What the model can't see

Some receptions are already a duel when the ball arrives: 1,655 in this data. A quarter of them end in a lost ball by the main definition, and half by the stricter one.

The model can't tell which. Its AUC on these receptions is 0.51, no better than chance, compared with 0.84 on ordinary receptions. These duels are also where you would expect keeping the ball to matter most, for long balls and crosses in particular. The likely reason is that the tracking data has positions and speeds but not body shape, arm contact or timing of the jump, which is what decides a contested ball.

## 7. What I take from this

- **Separate "arrives" from "kept".** Even when it doesn't improve prediction, the split changes the picture: for short passes, a sizeable share of the risk sits after the ball arrives.
- **Compare against a direct model.** A decomposed metric should be checked against the simplest model of the same outcome. Here it lost, and that changed what I can honestly claim for it.
- **Know what the data can't see.** Contested receptions are frequent, costly, and invisible to position-and-speed features.

**Limits.** One tournament, 49 matches with tracking. The receiver's body orientation is only approximated, and agrees moderately with PFF's own labels (κ = 0.40). I don't report player rankings.

## Data

PFF FC open 2022 World Cup data (tracking for 49 matches, events for 64). Models: gradient-boosted trees (LightGBM) and logistic regression, with match-grouped cross-validation. All intervals are match-level bootstrap.

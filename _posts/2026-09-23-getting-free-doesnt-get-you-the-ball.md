---
layout: post
title: "Getting Free Doesn't Get You the Ball"
subtitle: "Three attempts at measuring man-marking from tracking data, and why none of them worked"
description: "A physics-based coverage model on PFF FC's 2022 World Cup tracking data, used three ways. All three failed validation, and the ways they failed are instructive."
math: true
image: /assets/img/marking/fig_null_result_contrast.png
---

*49 World Cup matches, 1.02 million frames, and a lot of null results*

> **TL;DR** I built a physics-based coverage model on PFF FC's open 2022 World Cup tracking data and used it in three ways: to weight attacking threat by how uncovered it is, to work out which defender is marking which attacker, and to detect moments when an attacker escapes their marker. All three failed validation, each in a different way, and I think the ways they failed are useful. The most surprising result: an attacker who has just escaped their marker is **no more likely to receive the ball** than one who is still being marked.
>
> **What you won't find here:** a player ranking. Section 8 explains why.

## 1. The picture that started this

You have seen it. An attacker is about to shoot. A defender is standing close enough to do something about it, and is looking at somebody else.

Everyone watching knows this is bad. Nobody measures it.

Space control is largely solved: pitch control (Spearman, 2017) and its descendants tell us who owns which part of the pitch. Off-ball scoring opportunity is solved too: OBSO gives, for every attacker, the probability that a goal follows if the ball goes to them. Defensive *actions* are increasingly well valued as well, from StatsBomb's DefR to Marc Lamberts' recent Expected Fails Forced (xFF), which uses counterfactual modelling to credit defenders for the passes their pressure breaks.

What is still open is the covering of **people** rather than **space** in open play: who is holding whom, and what it costs when nobody is.

| Problem | Status |
|---|---|
| Space control | Solved (Spearman 2018, Bekkers 2025) |
| Off-ball scoring opportunity | Solved (OBSO) |
| Attributing defensive failure | Bischofberger et al. (2026): "blame" |
| Marking at set pieces | Groom et al. (2026), corners only |
| **Marking in open play** | **Open — this post** |

I spent several months on that last row, built three metrics and killed all three. What follows is the autopsy. I think the ways these metrics fail are more useful than one more metric that nobody validates.

## 2. Data

PFF FC's open 2022 World Cup tracking data: 49 matches at roughly 30 frames per second, with the pitch normalised to 105×68 m. Much of this broadcast-derived data is estimated rather than directly observed; I checked that this doesn't affect the results (see [Appendix A](#a-how-reliable-is-broadcast-tracking)).

## 3. The coverage model

Everything below rests on one number: the probability that attacker $$j$$ is covered by *somebody*, written $$P_j$$.

### Time to intercept

Project the attacker forward, let the defender carry their momentum through a reaction time, and compute how long it takes the defender to get there, plus a penalty for currently running the wrong way:

$$
\mathbf r_j^{*} = \mathbf r_j + \mathbf v_j \Delta, \qquad
\mathbf r_i^{*} = \mathbf r_i + \mathbf v_i \tau_r
$$

$$
T_{ij} = \tau_r + \frac{\lVert \mathbf r_j^{*} - \mathbf r_i^{*} \rVert}{v_{\max}} + k \,\lVert \mathbf v_i \rVert \,\frac{\beta}{\pi}
$$

where $$\beta$$ is the angle between the direction the defender is running and the direction to where the attacker is going.

This builds directly on Joris Bekkers' Pressing Intensity (2025), adapted from measuring pressure to measuring marking. The main change is conceptual: pressing only counts defenders who are actively moving, but for marking, a defender standing still in exactly the right place is the case that most needs to be rewarded. So I dropped the movement threshold and excluded goalkeepers. Smaller implementation details are in [Appendix B](#b-implementation-details-of-the-coverage-model).

### From one defender to the team

Each defender's coverage is a logistic curve in how much time they have to spare against a reference time $$T$$. Team coverage is **noisy-OR**: the probability that not every defender fails.

$$
p_{ij} = \left[ 1 + \exp\!\left( -\frac{\pi}{\sqrt{3}\,\sigma} \,(T - T_{ij}) \right) \right]^{-1},
\qquad
P_j = 1 - \prod_{i \in D} (1 - p_{ij})
$$

Noisy-OR earns its place. It always stays between 0 and 1 (a sum would not), it discounts redundancy instead of double-counting it, and distant defenders fade out without needing a cut-off. Its weakness is that it treats defenders as independent, which two players closing from the same side are not.

Parameters: $$\tau_r = 0.7$$ s and $$v_{\max} = 5$$ m/s (after Shaw and Spearman), $$\Delta = 1.0$$ s, $$k = 0.3$$ s²/m, and $$T = 1.5$$ s with $$\sigma = 0.45$$ (after Bekkers).

### The distribution that decides everything

<figure>
  <img src="{{ '/assets/img/marking/fig_pj_distribution.png' | relative_url }}" alt="Histogram of coverage probability on a log scale, with most mass near zero and a second hump near one">
  <figcaption>Figure 1. Coverage probability across 9.2 million attacker-frames (log scale). Nearly half sit below 0.05. The second hump near 1.0 is where several defenders cover the same attacker; it comes back in Section 5.</figcaption>
</figure>

| Statistic | Value |
|---|---|
| Median $$P_j$$ | 0.091 |
| $$P_j < 0.05$$ | 44.2% |
| $$P_j > 0.95$$ | 3.5% |

**This is not a calibration failure.** It is what football looks like: most attackers, most of the time, are somewhere no defender can reach within a second and a half. The distribution is right. As the next section shows, it is also fatal.

## 4. Attempt one: weight threat by how uncovered it is

### The idea

This is the obvious construction, and variants of it sit underneath several recent off-ball defensive metrics. Give every attacker a threat value $$V_j$$, discount it by how well they are covered, and add it up:

$$
U(t) = \sum_{j} V_j(t)\,\bigl(1 - P_j(t)\bigr)
$$

$$V_j$$ is the probability that a pass reaches the attacker, multiplied by a static xG value at their location. I deliberately left out OBSO's pitch-control term: it measures whether the attacker controls their own spot, which is the same thing $$P_j$$ measures. Keeping both would count coverage twice.

The benchmark is the raw threat sum with no coverage at all, $$\sum_j V_j$$. If coverage carries information, the weighted version should beat it.

### It doesn't

Across 1.02 million frames, rank-correlating each metric with the xG of shots in the next few seconds:

| Horizon | Raw threat | Coverage-weighted $$U$$ |
|---|---|---|
| 3 s | **0.174** | 0.163 |
| 5 s | **0.169** | 0.158 |
| 10 s | **0.153** | 0.143 |

Raw threat wins at every horizon. In a regression that already contains raw threat, adding $$U$$ lifts $$R^2$$ from 0.0478 to 0.0487. At the possession level, $$U$$ even enters with a slightly **negative** sign.

### It is not an aggregation problem

The obvious objection: $$U$$ and raw threat correlate at 0.85, so of course they can't be separated. So I changed the aggregation: sum, maximum, top three attackers, and a threat-weighted average coverage. The last one cuts the correlation with raw threat to 0.45 and finally has the expected sign (better coverage, less xG conceded). It is also nowhere near significant (p = 0.18).

**Collinearity was not the problem.**

### Why it fails

**Mechanically:** with nearly half of all attacker-frames below 0.05 coverage, the weight $$(1 - P_j)$$ is close to 1 almost everywhere, so $$U$$ is almost the same thing as raw threat.

**Structurally, and worse:** split $$U$$ into raw threat minus *covered* threat, $$S = \sum_j V_j P_j$$. The coefficient on $$S$$ is **positive** (p = 0.019 at three seconds). More covered threat predicts *more* xG conceded.

That is not because coverage creates danger. It is because defenders converge where danger already is. Coverage is a response to the very situation it is trying to describe. No recalibration and no better threat model removes that. It is the familiar difficulty of measuring defence from observational data, turning up in a new place.

## 5. Attempt two: who is actually marking whom?

If $$U$$ can't predict, perhaps it can still *attribute*. Noisy-OR breaks down cleanly: a defender's marginal contribution to covering attacker $$j$$ is how much $$P_j$$ drops if you remove them.

$$
m_{ij} = P_j - P_j^{(-i)}, \qquad P_j^{(-i)} = 1 - \prod_{k \neq i} (1 - p_{kj})
$$

This gives a marker for every attacker (the defender with the largest $$m_{ij}$$) and a measure of how many defenders are *effectively* involved, the inverse Simpson index $$n_{\text{eff}} = (\sum_i m_{ij})^2 / \sum_i m_{ij}^2$$.

### Redundant marking is real

In frames where the attacker is almost certainly covered ($$P_j \ge 0.95$$), nearly half have $$n_{\text{eff}} > 2$$. Elsewhere the figure is around 14%. This is the second hump in Figure 1, and a nearest-defender rule cannot see it by construction.

### The uncomfortable question

Is the "marginal pick" just the nearest defender with extra steps?

Mostly, yes. The two agree **86%** of the time. When they disagree, the marginal pick is the *second*-nearest defender in 81% of cases.

### But the disagreements aren't noise

<figure>
  <img src="{{ '/assets/img/marking/fig_argmax_vs_nearest.png' | relative_url }}" alt="Left: geometry of a nearest defender running away versus a slightly further defender facing the attacker. Right: density of turn angle versus distance for 14,443 disagreements">
  <figcaption>Figure 2. Left: one representative case. The nearest defender is 3.2 m away but running away from the attacker (β = 142°); the marginal pick is 4.2 m away but moving roughly towards them (β = 76°). Right: all 14,443 disagreements. The two groups barely differ in distance; they separate on angle.</figcaption>
</figure>

| Across 14,443 disagreements | Nearest | Marginal pick |
|---|---|---|
| Distance to attacker | 6.0 m | 7.6 m |
| Turn angle $$\beta$$ | 113° | 49° |

The candidates are roughly tied on distance, and the decision is made on orientation. Preferring "7 m away and facing the attacker" over "6 m away and running the other way" is, as football, correct.

The honest caveat: by construction, the only way the marginal pick can differ from the nearest defender is through the turn penalty. So this confirms the model behaves as designed. It is not independent proof that the design is right.

**What I can claim:** marginal contribution is a *refinement* of nearest-defender marking, not a replacement for it, with one exception. It can detect redundant marking, which a nearest-defender rule can't. That is real, and smaller than I hoped.

## 6. Attempt three: detecting when an attacker gets free

### Why this should have worked

Forget prediction and attribution. Just detect the event: an attacker who *was* covered and now *isn't*.

- $$P_j \ge 0.6$$ for at least 0.5 s,
- then falls to $$P_j \le 0.2$$ within 0.7 s,
- then stays at or below 0.2 for at least 0.5 s.

I added one guard: the attacker's threat afterwards must be at least 70% of what it was before, so that simply wandering into harmless space doesn't count.

<figure>
  <img src="{{ '/assets/img/marking/fig_escape_event_trace.png' | relative_url }}" alt="Time series of coverage probability for one attacker, holding above 0.6 and then collapsing below 0.2">
  <figcaption>Figure 3. A single escape: coverage holds above 0.6, collapses within a fraction of a second and stays low.</figcaption>
</figure>

This version uses only $$P_j$$, so it inherits none of Section 5's attribution problems. And it has something the first two attempts never had: **an observable consequence.** If a player gets free, the ball should come.

The detector found 637 escapes across the 49 matches, about 11 per match, with no match producing zero. After dropping five matches whose event timestamps could not be aligned with the tracking, 559 remain for validation.

### The result

The comparison group matters. Random moments would compare "escaped" with "was never marked in the first place", and most moments are the latter. So the control is **the same player, in the same match, covered for at least half a second, who did not escape.** Those players stay marked (coverage is still around 0.8 a second later).

Did the ball come?

| Received a pass within | Escaped | Still marked | p |
|---|---|---|---|
| 1 s | 1.4% | 1.4% | 1.00 |
| 2 s | 3.2% | 3.2% | 1.00 |
| 3 s | 4.3% | 5.4% | 0.40 |
| 5 s | 8.6% | 8.2% | 0.83 |

**A player who has just escaped their marker is exactly as likely to receive the ball as one who is still being marked.**

### Ruling out the obvious explanations

I didn't believe this, so I tried to break it.

**"The drop is a model artefact."** A player changing direction could make the projected position jump and fake a collapse. So for one full match (17 escapes) I split every drop into real movement, projection effects and turn-penalty effects. Real movement dominated in all 17; the attackers barely turned (median 3°). The separation is physical.

**"The test is broken."** Escape and control moments are typically about 19 minutes apart, only one pass was counted in both groups, and restricting to pairs at least 10 seconds apart changes nothing.

**"Passes are under-recorded."** 99.3% of pass events have a named target. Being the target of a pass within a few seconds is simply rare: about 7% at a random moment.

**"The ball moved away, so the defender left."** The opposite: during escapes the ball moves *less* and the attacker moves *more* than in control moments (both p < 10⁻⁹).

**"The consequence is too narrow."** I widened it to indirect receipt after several passes, the ball getting closer, expected threat increasing and possession surviving, each at four time horizons. All null.

### The one thing that differs

<figure>
  <img src="{{ '/assets/img/marking/fig_null_result_contrast.png' | relative_url }}" alt="Left: four outcome rates with overlapping confidence intervals for escapes and controls. Right: distance-to-goal distributions, with escapes further from goal">
  <figcaption>Figure 4. Left: what happens next is indistinguishable between escapes and controls. Right: where it happens is not. Escapes occur about 6 m further from goal.</figcaption>
</figure>

| | Escaped | Still marked |
|---|---|---|
| Median distance to goal | 25.6 m | 20.0 m |
| Inside the penalty area | 10.7% | 29.2% |

This is not a rescue. It is the explanation. **Where getting free would matter, it doesn't happen. Where it happens, it doesn't matter.**

## 7. What I take from this

**Being free and being available are different things.** This is the most transferable result. Off-ball metrics tend to assume that an unmarked attacker is a dangerous one. Across 559 escapes with a matched control group, four outcome definitions and four time horizons, that assumption didn't hold. If you are building something in this space, run this check first.

**Coverage is endogenous.** The positive coefficient on covered threat is a compact statement of why observational defensive metrics are hard. Defenders go where the danger is, and any metric that multiplies threat by defensive presence inherits that.

**A distribution can be correct and still useless.** $$P_j$$ appears to be right, and its concentration near zero is a true fact about football. It also means that weighting by $$(1 - P_j)$$ does almost nothing.

### Two warnings for anyone implementing this

**Filter offside players first.** Before I did, 15 of the 20 most "dangerous" uncovered situations involved an attacker in an offside position. Those players can't legally receive the ball, but nobody marks them, so they look maximally free. Removing them cut the maximum of $$U$$ by 34% while barely moving the median, which is what you'd expect if only false danger was being deleted.

**Measure the turn angle from the defender.** It is easy to compute the angle against a vector anchored somewhere else by accident, and that quietly makes results depend on the coordinate system.

### One thing that did work

I built an event-based momentum series following Opta's published method, with expected threat standing in for their possession value, and compared it with $$U$$ minute by minute. The rank correlation is **0.50**.

For two metrics built from completely different inputs, event streams on one side and player positions on the other, that is a healthy number. Both are tracking something real, while most of the variance stays independent. It is the one piece of outside corroboration in the project. It supports $$U$$ as a *description* of the game, while Section 4 shows it fails as a *predictor*. Those two findings don't conflict.

## 8. What I'm not claiming

**No player rankings.** 49 matches means three to seven matches per player. The most frequent escaper in the data has nine events; Poisson noise alone puts an error of about three on that. No individual ranking would survive a reliability check, so I haven't published one. If you see marking-quality rankings of players built on a single tournament, ask how many events sit behind each name.

**Not that man-marking is unmeasurable.** Only that these three constructions don't measure it. The data contains no body orientation, and Section 5 relies on orientation inferred from running direction. With pose data, the question of who is marking whom becomes answerable in a way it currently isn't.

**No novelty in the ingredients.** The pieces come from Spearman, Bekkers and standard probability. What is new here is the validation, and the validation is negative.

## 9. Data and tools

Tracking: PFF FC's open 2022 World Cup data. xG: logistic regression trained on StatsBomb open data (AUC 0.745). Expected threat: a 16×12 grid on StatsBomb open data using Karun Singh's method.

One correction worth passing on: my first xT grid was trained on completed passes only, which leaves turnovers out of the transition matrix and inflates every cell. Own-half values came out around ten times too high. Modelling three outcomes (shot, keep the ball, lose it) fixes it.

## Appendix

### A. How reliable is broadcast tracking?

69% of positions in the dataset are flagged `ESTIMATED` (inferred from context rather than directly observed). To check whether this matters, I found frames where a player switches from an estimated run to a directly observed one and measured how far they "jump" at the seam.

| | Match 10502 | Match 3816 |
|---|---|---|
| Median jump | 0.25 m | 0.19 m |
| 95th percentile | 1.06 m | 0.69 m |
| Jumps > 3 m | 0.07% | 0.20% |

For scale, a full sprint covers 0.37 m per frame. Because the camera follows the ball, the estimated share rises with distance from it: about 22% within 5 m and over 90% beyond 45 m. The nearby defenders that drive a coverage metric are mostly directly observed. This check works for any broadcast tracking dataset.

### B. Implementation details of the coverage model

- The turn angle $$\beta$$ is measured between the defender's velocity $$\mathbf v_i$$ and the vector $$\mathbf r_j^{*} - \mathbf r_i$$ from the defender to the attacker's projected position. A stationary defender gets $$\beta = 0$$.
- The turn penalty is scaled by $$k = 0.3$$ s²/m so that it is expressed in seconds, roughly equivalent to a deceleration of 3.3 m/s².
- The attacker's position is extrapolated $$\Delta = 1.0$$ s ahead.
- The marginal contribution $$m_{ij}$$ is computed with prefix and suffix products rather than by division, to stay stable as $$p_{ij} \to 1$$.

## References

- Spearman, W. et al. (2017). Physics-based modeling of pass probabilities in soccer. *MIT Sloan Sports Analytics Conference.*
- Spearman, W. (2018). Beyond Expected Goals. *MIT Sloan Sports Analytics Conference.*
- Bekkers, J. (2025). Pressing Intensity: An Intuitive Measure for Pressing in Soccer. arXiv:2501.04712.
- Bischofberger, J. et al. (2026). Blame is easier than praise: Measuring off-ball defensive performance in football. arXiv:2606.19931.
- Groom, S. et al. (2026). A Machine Learning Framework for Off Ball Defensive Role and Performance Evaluation in Football. arXiv:2601.00748.
- Casciolio, L. & Wang, A. (2025). Quantifying Off-Ball Defensive Impact through Cover Shadows. *Hudl Performance Insights.*
- Everett, G. et al. (2025). Evaluating Defensive Influence in Multi-Agent Systems Using Graph Attention Networks. *IEEE DSAA.*
- Merhej, C. et al. (2021). What happened next? Using deep learning to value defensive actions in football event-data. *KDD.*
- Lamberts, M. (2026). Introducing Expected Fails Forced (xFF). *Medium.*
- Davis, J. et al. (2024). Methodology and evaluation in sports analytics: challenges, approaches, and lessons learned. *Machine Learning.*
- Shaw, L. (2020). LaurieOnTracking. *Friends of Tracking, GitHub.*
- Singh, K. (2019). Introducing Expected Threat (xT).

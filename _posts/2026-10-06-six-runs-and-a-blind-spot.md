---
layout: post
title: "Six Runs and a Blind Spot: Off-Ball Movement Through xMark"
subtitle: "What a marking model shows about off-ball runs, and what it misses"
description: "Using xMark, a tracking-data marking model, and pitch control to look at six off-ball runs from the 2022 World Cup: three that pull defenders away from a teammate, two that shake off a marker, and one the model can't see."
math: true
image: /assets/img/offball/card_fra-pol.png
thumbnail: /assets/img/offball/card_fra-pol.png
---

*Six scenes · 49 World Cup matches · PFF FC open tracking data*

> **TL;DR**
> - xMark can't grade an off-ball run, but it can show how one works: which defender moved, and who that freed. I use the model from [an earlier post]({{ '/2026/09/getting-free-doesnt-get-you-the-ball/' | relative_url }}) to follow who is marking whom, frame by frame.
> - Two patterns show up clearly: a run that **pulls a defender off a teammate**, and a run that **shakes off a marker**.
> - One kind of run doesn't: an attacker who is clearly free but still has a defender chasing a couple of metres behind. Pitch control misses it too.

## 1. What I'm looking at

Off-ball runs are one of the hardest things to measure. A run can be excellent and never touch the ball, or create space that a teammate uses.

In an [earlier post]({{ '/2026/09/getting-free-doesnt-get-you-the-ball/' | relative_url }}) I built a model from tracking data. In brief, for every attacker $$j$$ and defender $$i$$ in every frame:

$$
T_{ij} = \tau_r + \frac{\lVert (\mathbf r_j + \mathbf v_j \Delta) - (\mathbf r_i + \mathbf v_i \tau_r) \rVert}{v_{\max}}, \qquad
p_{ij} = \frac{1}{1 + \exp\!\left( -\dfrac{\pi}{\sqrt{3}\,\sigma}\,(T - T_{ij}) \right)}
$$

$$T_{ij}$$ is the time defender $$i$$ needs to reach where attacker $$j$$ will be in $$\Delta = 1$$ s (reaction time $$\tau_r = 0.7$$ s, then running at $$v_{\max} = 5$$ m/s), and $$p_{ij}$$ turns it into the probability that he gets there within $$T = 1.5$$ s ($$\sigma = 0.45$$ s). These steps follow Joris Bekkers' Pressing Intensity (2025); what the earlier post added is the breakdown by defender and a check against human labels. From these come the two quantities used here:

$$
P_j = 1 - \prod_i (1 - p_{ij}), \qquad m_{ij} = P_j - P_j^{(-i)}
$$

- **xMark** $$P_j$$: the probability that at least one defender can reach attacker $$j$$ in time. Close to 1 means marked; close to 0 means free. Strictly, it measures reach, not whether a defender is actually marking him; Section 5 shows why that matters.
- **Marker contribution** $$m_{ij}$$: how much attacker $$j$$'s xMark falls if defender $$i$$ is removed. The defender with the largest $$m_{ij}$$ is, in the model's view, the one marking $$j$$. Checked against human analysts, it identifies the defender who pressed the receiver slightly better than a "nearest defender" rule does.

That post found xMark to be a poor predictor of danger. Here I use it for something else: describing what happens during a run.

## 2. How to read the clips

- **Left: the pitch.** Attackers are white, defenders black. The ring around each attacker shows their xMark (purple = free, yellow = marked). A line joins each defender to the attacker he is marking, thicker when $$m_{ij}$$ is larger.
- **Background: pitch control.** Red where the attacking team would win the ball if it were played there, blue where the defence would (Spearman's physics-based model).
- **Right: the numbers.** Solid lines are xMark of the two attackers involved. Dashed lines are one defender's marker contribution to each of them. Lines break while a player has the ball, because xMark is only computed for players without it.

## 3. Pattern one: pulling a defender off a teammate

In these three scenes, one attacker's run draws a defender away, and a teammate's xMark drops as a result.

### Case 1: Two runners, one defender (France v Poland, 28:08)

<figure>
  <video src="{{ '/assets/video/offball/01_FRA-POL.mp4' | relative_url }}" autoplay loop muted playsinline style="width:100%"></video>
</figure>

Giroud and Rabiot both run into the area Kamil Glik is defending. Just before the reference point, Glik's contribution to the two of them is almost equal (about 0.7 each): he is effectively responsible for both at once. The dashed lines cross 0.1 s before the reference point. From there he goes with Rabiot (contribution 0.11 → 0.84) and lets Giroud go (0.82 → 0.38). Giroud's xMark halves, from 0.80 to 0.40. Dembélé's cross finds Giroud, whose shot goes wide.

Whether the two runs were planned together, the data can't say, but the result is a 2-v-1 on a centre-back, who has to choose one of the two.

### Case 2: A diagonal run opens the space (Portugal v Switzerland, 34:55)

<figure>
  <video src="{{ '/assets/video/offball/02_POR-SUI.mp4' | relative_url }}" autoplay loop muted playsinline style="width:100%"></video>
</figure>

Guerreiro makes a diagonal run across the box. Granit Xhaka drifts towards it and stops covering Gonçalo Ramos (contribution 0.45 → 0.02). Ramos runs into the space, and his xMark falls from 0.94 to 0.08: the largest drop, weighted by how dangerous his position was, of any candidate I found. Félix's cross reaches him, but he is challenged and the ball goes out.

Xhaka never fully picks up Guerreiro (his contribution to him only reaches 0.21), yet his movement is enough to free Ramos.

### Case 3: Attacking the box to free the striker (Croatia v Canada, 71:02)

<figure>
  <video src="{{ '/assets/video/offball/03_CRO-CAN.mp4' | relative_url }}" autoplay loop muted playsinline style="width:100%"></video>
</figure>

Kramarić runs into the box, and Steven Vitória switches from Petković to him (contribution to Kramarić 0 → 0.67, to Petković 0.49 → 0.18). Petković's xMark falls from 0.69 to 0.30. Here the ball goes to the runner: Kramarić shoots, wide.

## 4. Pattern two: shaking off a marker

The simpler version: an attacker loses the defender who was marking him.

### Case 4: Núñez in the box (Ghana v Uruguay, 08:10)

<figure>
  <video src="{{ '/assets/video/offball/04_GHA-URU.mp4' | relative_url }}" autoplay loop muted playsinline style="width:100%"></video>
</figure>

Darwin Núñez's xMark falls from 0.70 to 0.16 in 0.8 seconds, as Daniel Amartey's contribution drops by the same amount. Núñez is free in the box, but the cut-back doesn't reach him.

### Case 5: A late run from midfield (Cameroon v Brazil, 90+7)

<figure>
  <video src="{{ '/assets/video/offball/05_CAM-BRA.mp4' | relative_url }}" autoplay loop muted playsinline style="width:100%"></video>
</figure>

Fabinho arrives from the second line. Collins Fai follows him at first, then loses him (contribution 0.58 → 0.11), and Fabinho's xMark drops from 0.67 to 0.19. The chance goes to Martinelli, who shoots wide.

## 5. What the model can't see

### Case 6: Weah (USA v Wales, 35:10)

<figure>
  <video src="{{ '/assets/video/offball/06_USA-WAL.mp4' | relative_url }}" autoplay loop muted playsinline style="width:100%"></video>
</figure>

On video, this is a clean run in behind: Weah gets goal-side of Neco Williams, Pulisic plays him through, and Weah scores first time.

**The model misses this run: Weah's xMark is still 0.57 when he receives the ball, because a beaten defender two metres behind still counts as able to reach him.** His xMark falls from 0.96 to 0.58 by the time the pass is played. When the pass is played, Williams is level with Weah; Weah is faster (7.5 against 6.7 m/s) and gets goal-side while the ball is rolling, so by the time he receives it Williams is about two metres behind, still chasing. The model asks only whether a defender *can reach* the attacker, not whether he is goal-side of him. (For this clip I keep computing Weah's xMark after he receives the ball; normally it stops.)

Pitch control doesn't spot it either. Around Weah the background is actually blue: attacking pitch control at his position is 0.24 when the pass is played and 0.25 when he receives it. It also asks who can get there first, and by that measure Williams can.

This is a limit of the whole family of "who can reach it first" models, not just mine. Fixing it would need something that knows which way the defender is facing and which side of the attacker he is on.

## 6. How xMark relates to pitch control

The two are built from the same pieces. Both estimate how long each player needs to reach a point (reaction time plus running at top speed) and turn that into a probability. The differences:

| | Pitch control | xMark |
|---|---|---|
| Where | Every point on the pitch | Only where each attacker is heading |
| Who | Attack against defence | Defence only |
| Breaks down into | Who controls each point | Who is marking each attacker |

To see how closely they agree, I took 50,000 random frames of attackers without the ball from all 49 matches and compared an attacker's freedom ($$1 - P_j$$) with attacking pitch control at his position.

| Where the attacker is | Frames | Rank correlation [95% CI] |
|---|---|---|
| Everywhere | 50,000 | 0.76 [0.74, 0.77] |
| Middle third | 27,589 | 0.76 [0.75, 0.77] |
| Final third, outside the box | 18,880 | 0.67 [0.65, 0.68] |
| Inside the penalty area | 2,691 | 0.51 [0.47, 0.55] |

**The closer to goal, the less xMark and pitch control agree: rank correlation 0.76 overall, but 0.51 inside the penalty area.** Inside the box, the average attacker is mostly covered (freedom 0.25) while pitch control at his position is roughly even (0.53).

The disagreements also go one way. An attacker xMark calls free almost always stands in space his team controls: only 1.3% of all frames are "free but in the defence's space". The reverse is common: more than half of the attackers it calls covered are standing where pitch control favours the attack (9.6% of all frames). That is what you'd expect in crowded areas: a defender is close enough to one attacker to cover him, but other attackers are close enough to the same spot to win a ball played there.

So the two are not interchangeable, especially in the box, where most of these scenes happen. xMark is not a better pitch control. Its use is the last row of the table: it says who is marking whom, which a pitch control map doesn't show directly.

## 7. How these scenes were chosen

I didn't pick these by hand from memory. I searched all 49 matches for two patterns during open play in the opponent's half, with the attacker moving towards goal:

| Pattern | Per match | Shot within 5 s |
|---|---|---|
| A defender switches from one attacker to another, and the first attacker's xMark drops by 0.3 or more within 2 s¹ | 8.5 | 9.1% |
| An attacker's xMark drops from above 0.5 to below 0.2 within 2 s, at a position worth at least 0.05 xG | 1.1 | 15.4% |
| Any moment under the same conditions | — | 5.0% |

¹ Excluding switches that happen in the same frame as a change of ball carrier (which reshuffles the model's assignments mechanically), and switches where the defender's contribution is below 0.05 before or after.

Both patterns are followed by shots more often than a random moment, though this is an association, not proof that the run caused the shot. From the top candidates I watched the footage and kept the scenes where the run was clear on video and the numbers told the same story. I dropped scenes where the defender's contribution was too small (below about 0.2) to say that his movement freed anyone.

## 8. Summary

- **Description, not evaluation.** xMark shows who was marking whom and when that changed, and puts numbers on something a broadcast clip shows only in part.
- **Breakdown by defender.** Pitch control can tell you a space is open; marker contribution tells you which defender opened it.
- **Runs in behind.** Runs where the defender is beaten but close are invisible to this family of models.

**Limits.** These are six selected scenes, not a measurement of how good anyone's movement is. xMark has no body orientation. The pitch control model uses standard published parameters and was not tuned to this data.

## Data

PFF FC open 2022 World Cup data (tracking for 49 matches). xMark as in [the earlier post]({{ '/2026/09/getting-free-doesnt-get-you-the-ball/' | relative_url }}), without the turn penalty. Pitch control: Spearman (2017), with the default parameters of Laurie Shaw's implementation for the Friends of Tracking series.

- Bekkers, J. (2025). Pressing Intensity: An Intuitive Measure for Pressing in Soccer. arXiv:2501.04712.
- Spearman, W. et al. (2017). Physics-Based Modeling of Pass Probabilities in Soccer. *MIT Sloan Sports Analytics Conference.*
- Spearman, W. (2018). Beyond Expected Goals. *MIT Sloan Sports Analytics Conference.*

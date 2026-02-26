# Training data: HJ formulation and data generation

## High-level overview

**Training data.** We need (observation history, teacher waypoints, affordance labels) to pretrain the policy. Teacher waypoints and labels come from HJ: we run rollouts in sim, solve pairwise HJ (ego vs each object), aggregate safe-set / time-to-boundary / safe direction, and produce ego waypoints + predicted centroid waypoints for other agents. Details below.

**Transformer input.** Observation history over a window: at each timestep, tokens for ego (pose, velocity, covariance summary), goal (delta to goal, type), map (local 3D or 2.5D), and per-object tracks (relative position and velocity in $\mathbb{R}^3$, radius, class, uncertainty, time since last seen). All in a causal sequence; the model sees only past and present.

**Transformer output.**  
- **Ego waypoints** — A chunk of 3D waypoints $(w_{t+1}, \ldots, w_{t+H})$ for the ego; the controller tracks these.  
- **Non-ego** — For each other agent: **centroid waypoints** (trajectory of the agent’s center over the same horizon) + a single **radius** (sphere approximation) for extent. So we get one trajectory and one scalar per agent; no full shape.  
- **(Suggested)** Affordance outputs: feasibility, time-to-boundary, or collision margin per object; optional and can be dropped if we keep the story minimal.

**Partial observability and RL.** At deploy time the policy often can’t see all agents (occlusion, range, FOV). We don’t fix that in pretraining; we pretrain on data where the teacher has full state. Fine-tuning in RL (stage 2) then adapts how the policy tracks and predicts other agents when observations are missing or noisy—e.g. maintain beliefs over occluded agents, or learn to be conservative when uncertain.

---

## 1. What we need from HJ

**Labels (not runtime shield):**
- Safe-set membership (binary or value function)
- Time-to-boundary
- Safe direction in $\mathbb{R}^3$
- (Optional) Backward reachable tube (BRT) for a horizon

**Use:** Supervise waypoint imitation and affordance heads. Teacher waypoints can be chosen to stay inside the safe set or follow safe direction.

---

## 2. Formulating the HJ problem (3D)

**State:** Reduced state for tractability. Per obstacle pair (ego vs. one object): relative position $\Delta p \in \mathbb{R}^3$, relative velocity $\Delta v \in \mathbb{R}^3$ → 6D. With ego dynamics, higher dim; often use **pairwise** (ego–obstacle) in 6D or lower (e.g. 4D with planar slice).

**Target set:** Collision = $\|\Delta p\| \le r_{ego} + r_{obj}$. Unsafe set = states that can hit this target within horizon $T$.

**Dynamics:** Linear or linearized (double integrator, or closed-loop with tracking controller). HJ is tractable for low-dimensional linear/near-linear systems.

**Value function:** Backward reachable set (BRS) = set of states from which we can reach target within $T$. Solve HJB (or LQR approximation) backward in time. Value $V(x,t)$: $V \le 0$ = safe, $V > 0$ = can hit target.

**Outputs for training:**
- $V(x,t)$ or binarized safe/unsafe
- Time-to-boundary (e.g. first $t$ such that $V$ crosses threshold)
- Safe direction: $-\nabla V$ or gradient of value

---

## 3. Practical choices

| Choice | Option | Note |
|--------|--------|------|
| **Dimensionality** | Pairwise 6D (relative $p,v$) or 4D slice | 3D full state + control is high-dim; pairwise is standard |
| **Solver** | Level-set / Toolbox (e.g. helperOC, BEACLS) or neural (HJRNO, NeHMO) | Classical: exact but slow. Neural: fast, less certifiable |
| **Aggregation** | Min over pairs, or max over obstacles | Safe = safe w.r.t. all obstacles |
| **Horizon** | Match waypoint horizon $H \cdot \Delta t$ | Align with controller update |

---

## 4. Data generation pipeline

1. **Rollout** — Simulate scenario (lightweight sim); at each $t$ record full state $x^*_t$, deploy-view $o_t$.
2. **Pairwise HJ** — For each (ego, object $i$), compute relative state; solve or look up BRT/value; get safe-set label, time-to-boundary, safe direction.
3. **Teacher waypoints** — From current state, integrate safe direction or use a short-horizon safe planner that respects BRT; output $W^*_t$ (ego + centroid waypoints for others).
4. **Affordance labels** — From BRT: collision-within-$H$ (0/1), min separation, time-to-boundary; from rollout: tracking error, saturation (if available).
5. **Window** — Slice $(o_{t-L+1:t}, x^*_{...}, W^*_t, y_t)$; augment $o$ with noise/occlusion.

---



# HJ-Reachability Pretraining for Affordance-Aware MARL — Summary

> [!abstract] Research Direction
> **3D** multi-agent drone navigation. HJ reachability as *training data* (not shield). Single causal Transformer; ego + centroid waypoints + radius. Affordance-aware; implicit planning. All in $\mathbb{R}^3$.

---

## 1. Problem

MARL in 3D safety-critical settings fails early. HJ gives feasibility (reachable sets, time-to-boundary) but is used as a runtime shield—the policy never internalizes "feasible." Gap: avoid collisions vs. *exploit* feasibility structure (e.g. *is it feasible to close the gap?*). Reward alone rarely discovers this.

---

## 2. Formulation

- **State / obs:** $x_t \in \mathcal{X}$ (ego $p,v \in \mathbb{R}^3$, 3D scene, agents); $o_t$ = deploy-view (SLAM + tracker). History $h_t = o_{1:t}$.
- **Waypoints:** $W_t = (w_{t+1}, \ldots, w_{t+H})$, $w \in \mathbb{R}^3$; controller $\mathcal{C}$ tracks them.
- **Objective:** $\max_\theta \mathbb{E}[\sum_t r(x_t, W_t)]$ s.t. $W_t \sim \pi_\theta(\cdot \mid h_t)$. Belief $b_t = p(x_t \mid h_t)$; we learn $z_t \approx \phi(h_t)$; planning implicit.

---

## 3. HJ as supervision

HJ outputs (safe-set value, time-to-boundary, safe direction) → **labels**, not runtime. Scene: 3D, multimodal, evolving. HJ tractable for **pairwise** (ego vs. one obstacle); aggregate over pairs. Policy learns to predict affordance from partial obs.

---

## 4. Architecture

- **Single causal Transformer** over token history $[tokens_{t-L+1}, \ldots, tokens_t]$. Output from last-timestep.
- **Waypoint head:** Ego waypoints $\hat{W}^{ego}$ + **centroid** waypoints $\hat{W}^{obj,i}$ for others; **radius** $r^i$ per agent (sphere). Hold + feasible rate.
- **Affordance heads:** feasibility, time-to-boundary, margins, collision prob. HJ labels as supervision (optionally as token).

**Tokens:** Ego, goal, map (3D voxel or 2.5D), object tracks ($\Delta p, \Delta v \in \mathbb{R}^3$, $r^i$, cov, $\Delta t^{seen}$).

---

## 5. Data and training

**Dataset:** $(o_{t-L+1:t}, x^*_{...}, W^*_t, y_t)$. $W^*$ = teacher (HJ-conditioned); $y$ = affordance labels. See [[fm-planning/formal-pretraining-marl/notes/training-data|training-data.md]] for HJ formulation and pipeline.

**Phases:** 0) Teacher + controller-in-loop; 1) Belief + waypoint imitation + affordance ($\mathcal{L}_{belief} + \lambda_{wp}\mathcal{L}_{wp} + \lambda_{aff}\mathcal{L}_{aff}$); 2) Dataset aggregation; 3) RL fine-tune.

**Setup:** Lightweight 3D sim → Isaac Sim for RL. Spawn 3D (incl. vertical); agents collide; goals: reach/track target + avoid collision.

---

## 6. Intuitions, risks, contributions

| Intuition | Evidence |
|-----------|----------|
| HJ pretraining → affordance-aware | Correct "feasible to close gap?"; tight corridors |
| Planning implicit | No planner; OOD generalization |
| 3D tractable | HJ pairwise scales; encoder handles partial obs |

**Risks:** Reward shaping may match; "implicit" may be imitation; HJ scale (pairwise only?); forgetting; sim-only.

**Contributions:** HJ-as-data pipeline; affordance-aware 3D MARL; benchmark (sim-only).

---

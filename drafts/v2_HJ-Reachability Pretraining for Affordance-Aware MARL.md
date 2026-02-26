# HJ-Reachability Pretraining for Affordance-Aware MARL


3D high-speed multi-agent drone navigation: implicit affordance-aware planning via Hamilton–Jacobi (HJ) reachability pretraining. The policy learns *feasibility structure* from HJ solutions—*is it feasible to close the gap?*—without explicit planning. All state, waypoints, obstacles, and dynamics are in 3D ($\mathbb{R}^3$).

## Abstract

MARL in safety-critical 3D settings fails early: agents collide, violate constraints, learn slowly. HJ reachability characterizes feasibility precisely—3D backward reachable sets, value functions, time-to-boundary—but is almost always used as a runtime shield. The policy never internalizes what "feasible" means in 3D space.

**The idea.** Use HJ reachability solutions as *training data*. Pre-train on offline reachability labels so the policy becomes *affordance-aware* in 3D: from partial observations of a *multimodal evolving 3D scene* (multiple agents, 3D obstacles, moving goals in $\mathbb{R}^3$), it implicitly reasons—*can I close this 3D gap? will that 3D corridor open?*—without an explicit planning module. Planning is emergent. **Problem, observations, waypoints, and teacher are all 3D.**

## Problem

A drone in a 3D cluttered, dynamic environment: positions and velocities in $\mathbb{R}^3$, other agents with unknown intent, 3D obstacles (static and moving), 3D corridors and gaps feasible only within a narrow time window. The gap: "learns to avoid collisions" vs. "learns to exploit 3D feasibility structure." The latter requires affordance reasoning in 3D—*is it feasible to close the gap?*—from partial observations of a scene whose state is *multimodal* and *evolving* in 3D. Reward signal alone rarely discovers this structure.

## Proposed approach

1. **Staged simulation** — Lightweight sim first (fast iteration, pretraining); Isaac Sim for RL fine-tuning.
2. **Transformer outputs** — Ego waypoints + **centroid** waypoints of other agents (static and dynamic). Other agents’ extent as **radius** (sphere) to keep low-dim. Policy predicts full scene trajectory.
3. **Reward** — Progress to target / track target; penalize collision with other agents. Tune reward signal.
4. **Scenario design** — 3D spawn: agents in many directions (including vertical); 3D obstacles; agents collide in 3D. Randomize 3D paths. Time window captures *dynamics*, not strategy. Key question: *can it make it?*
5. **Objectives** — Make it to 3D target and avoid collision; or track 3D target and avoid collision.

## Core intuitions (to validate)

| Intuition | What would prove it |
|-----------|---------------------|
| HJ pretraining yields affordance-aware policies | Policy answers "is it feasible to close the gap?" correctly on held-out scenarios; commits to tight corridors that reactive policies avoid. |
| Planning is implicit, not explicit | No planning module; representations predict time-to-boundary, safe-set membership; OOD generalization to novel agent counts, obstacle densities. |
| 3D multimodal evolving states are tractable | HJ (or approximations) in 3D scales to enough object pairs for useful pretraining; belief encoder handles partial observability. |

## Novelty, assumptions, and critique

**Claimed novelty:** HJ reachability as *training data* (not runtime shield); policy becomes affordance-aware in 3D; implicit planning from a single causal Transformer; centroid waypoints + radius for other agents (low-dim).

**Harsh reality check:**
- **Unproven:** Reward shaping (dense collision penalty) might yield similar early-safety gains; we need ablations. "Implicit planning" might be imitation—no evidence yet that representations encode value-function-like structure.
- **HJ scalability:** 3D HJ is exponential in state dim. We likely only get pairwise (ego vs. one obstacle) or heavily approximated; "enough" data for pretraining is an open risk.
- **No deployment:** Sim only (lightweight → Isaac). No hardware, no real drones. Contribution is method + benchmark, not deployed system.
- **Forgetting:** RL fine-tuning may erase pretrained affordance structure; KL/early-stop are mitigations, not guarantees.

**Explicit assumptions:**
- Teacher (HJ-conditioned or scripted) produces feasible, trackable waypoints; centroid + radius suffices for collision/affordance.
- Single causal Transformer over history is sufficient for temporal reasoning (no GRU/set structure required).
- Lightweight sim distribution is close enough that pretraining transfers to Isaac Sim for RL.

## TODO

- [ ] Build 3D lightweight sim for pretraining; Isaac Sim for RL (all state/waypoints in $\mathbb{R}^3$).
- [ ] Tf outputs: 3D ego waypoints (x,y,z) + other agents' 3D waypoints.
- [ ] Reward: progress/track 3D target + 3D collision penalty; tune signal.
- [ ] Spawn agents in many 3D directions (incl. vertical); randomize 3D paths; agents collide in 3D.
- [ ] Goals: make it to 3D target and avoid collision; or track 3D target and avoid collision.

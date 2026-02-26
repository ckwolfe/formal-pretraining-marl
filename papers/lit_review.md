# Literature Review — HJ Pretraining for Affordance-Aware MARL

## What

We care whether the agent can make it, not guaranteed safety. HJ reachability is usually used as a runtime shield (e.g. Shielded RL survey, arXiv 2023); we use its outputs as training targets so the policy learns "is this gap closable?" from partial observations. No explicit planner; planning is implicit in the representation. Goal: higher success probability and better risk-taking, not zero violations. 

Risks: reward shaping may suffice; HJ does not scale beyond pairwise; RL fine-tuning may overwrite pretrained structure.


---

## How novel is this?

**Minimal novelty (defensible):** HJ-conditioned waypoints as *pretraining* data for MARL—then RL fine-tune. Not HJ as shield; not BC of an arbitrary expert. The teacher is reachability-theoretic (waypoints + optional feasibility labels), the setting is multi-agent 3D, the goal is "make it" not safety guarantees. That combination is underexplored. 

**What would make it clearly novel:** HJ pretraining beats (b) MARL from scratch and (c) MARL with reward shaping only—on make-it rate and/or sample efficiency. If we add affordance heads, also beat (a) waypoint-only BC + RL. Without those comparisons, the claim is hypothetical.

---

## Publishability (honest)

**What we need to show.** Core claim: HJ pretraining improves MARL (make-it rate and/or sample efficiency). Required comparisons: (i) MARL from scratch; (ii) MARL with reward shaping only. If we add affordance heads: (iii) waypoint-only BC + RL. Metrics: make-it rate, collision/near-miss counts, env steps to reach a given performance. Without (i) and (ii), reviewers will say we didn't rule out RL alone or reward shaping alone.

**Two-sim pipeline.** Stage 1: pretrain in low-fidelity sim (fast, cheap, easy HJ labels and rollouts). Stage 2: RL fine-tune in high-fidelity sim (e.g. Isaac Sim). Hope: learn structure in cheap sim, adapt in expensive sim with fewer high-fidelity steps. Risk: low→high fidelity is distribution shift (physics, sensing, actuation); many sim-to-sim efforts see collapse or need heavy tuning. So: (A) If we run both stages, we must show pretraining in low-fidelity helps in high-fidelity (with ablations). (B) If we only run one sim, we can still publish—claim is "HJ pretraining helps MARL" in that sim; transfer is future work. (C) Don't claim two-stage transfer without showing it.

**Venue bar (detailed).**
- **Strong (main venue in scope—NeurIPS, CoRL):** if HJ pretraining clearly beats MARL-from-scratch and reward-shaping-only on make-it rate and/or sample efficiency. Ablations are clean: same tasks, same metrics, multiple seeds. If we use two sims: pretrain in low-fidelity then RL in high-fidelity beats RL-only in high-fidelity for the same high-fidelity env budget. Narrative is tight; we don't overclaim (e.g. we say sim-only, feasibility not safety). 
- **Adequate (workshop or mid-tier; main venue possible if writing and experiments are very tight):** We have comparisons (i) and (ii) in at least one sim; results favor our method but the margin is modest or the setup is limited (e.g. few scenarios). Two-stage transfer is either not attempted (stated as future work) or attempted with mixed success (we report honestly and discuss limitations). Reviewers may still say "incremental"; clear write-up and honest limitations help.
- **Weak / not publishable at a serious venue:** Missing (i) or (ii)—e.g. no MARL-from-scratch or no reward-shaping-only baseline. Or we claim low-fidelity→high-fidelity transfer but don't show it (or show it fails and don't report it). Or pretraining doesn't help and we have no ablations. Sim-only and no hardware are fine; the bar is method and evidence, not deployment.

---

## What is already done vs not done

**Done.** 

(1) **BC / imitation of formal or safe policies:** Imitating expert or safe policies then fine-tuning with RL is standard (e.g. Posterior BC for RL finetuning; offline RL with explicit BC constraints). Safe-MARL with filters (Resolving Conflicting Constraints, Def-MARL, CoRL-MPPI) uses formal methods (reachability, CBF, MPPI) in the loop—the learned policy is corrected at runtime. So "use a formal policy" and "BC then RL" are both established. 

(2) **HJ at runtime:** Certifying HJ learned via RL (arXiv 2026), NeHMO, HJRNO—HJ is computed or certified for online use or as a shield, not as offline labels. 
(3) **MARL SOTA:** CTDE (Amato 2024); value factorization with overgeneralization fixes (Rashid et al. 2018 QMIX, 2020 Weighted QMIX; Wang et al. 2020 QPLEX); strong on-policy baselines (Yu et al. 2021 MAPPO); actor-critic (Lowe et al. 2017 MADDPG); credit assignment (Foerster et al. 2018 COMA); implicit coordination (Li et al. 2021 DICG). 
(4) **Waypoint / sequence policies:** Decision Transformer (Chen et al. 2021), Diffusion Policy (Chi et al. 2023). 
(5) **Affordance as representation:** RT-Affordance, NavA³; we define ours via HJ. 
(6) **Inverse constraint = BRT:** "Your learned constraint is secretly a backward reachable tube" (arXiv 2025)—learned constraints recover BRTs; we supply BRT-like labels instead of learning them from data.

**Not done (or underexplored).** 
(1) **HJ outputs as training data for MARL** (not as shield, not as certification target): value, time-to-boundary, safe-set membership as labels to train a multi-agent policy's affordance/feasibility representation. 
(2) **Explicit affordance heads supervised by HJ** in MARL—so the policy doesn't only clone waypoint actions but learns to predict "can I make it?" and we train that prediction. 
(3) **Full pipeline:** HJ-derived waypoints + affordance labels → pretrain (imitation + affordance loss) → RL fine-tune in 3D multi-agent with centroid/radius. 
(4) **Feasibility and "make it" as the objective** (not safety guarantees): same tools (HJ) but success probability and risk-taking, not zero violation (unlike Def-MARL, CoRL-MPPI, layered safety).


---

## MARL: SOTA, limitations, implicit planning (with citations)

**Paradigm.** CTDE: train with global state / other agents' info; execute with local observation history only (Amato 2024). No execution-time communication; coordination in the weights.

**SOTA (games).** Plain PPO or MAPPO (Yu et al. 2021) often matches or beats off-policy MARL on SMAC, Football, Hanabi. Value-based: QMIX (Rashid et al. 2018), Weighted QMIX (Rashid et al. 2020), QPLEX (Wang et al. 2020)—monotonicity and overgeneralization are real limits. Actor-critic: MADDPG (Lowe et al. 2017), MAPPO. VDN (Sunehag et al. 2018) is too restrictive. DICG (Li et al. 2021) adds implicit coordination; COMA (Foerster et al. 2018) addresses credit assignment. Open agent systems break credit (arXiv 2025); structural labels (e.g. feasibility) might help.

**SOTA (robotics / planning–avoidance).** Safe multi-agent nav and collision avoidance: layered safety with reachability + CBF (Resolving Conflicting Constraints, Crazyflie); CoRL-MPPI (RL + MPPI, safe multi-robot); goal-conditioned safe RL + planning and conflict-based search (Safe Multi-Agent Navigation); HJB + GNN + CBF, distributed (Learning Distributed Safe Multi-Agent Navigation, Crazyflie); Def-MARL (zero violation, epigraph, quadcopters). Common pattern: formal method (CBF, MPPI, reachability) in the loop or as constraint; we use HJ as data and target feasibility, not guarantees.

**Limitations.** Non-stationarity, credit assignment, partial observability, sample inefficiency (COMA; credit-assignment open systems). No built-in "can I make it?"—policies are reactive unless we add a planner or a representation that supports it. Implicit planning appears in CTDE (internalized coordination), value factorization (mixing network), graph methods (DICG), and history (RNN/Transformer). Our angle: HJ-derived feasibility and waypoint labels so the policy learns when a maneuver is on; planning stays implicit.

---

## Closest ideas (cited)

- **HJ + learning:** Usually certification or online use (Certifying HJ via RL, NeHMO, HJRNO) or shields (Shielded RL). We use HJ offline to label data.
- **Learned constraint = BRT** ("Your learned constraint is secretly a backward reachable tube"): they learn BRTs from data; we supply BRT-consistent labels.
- **Safe MARL:** CoRL-MPPI, Def-MARL, Resolving Conflicting Constraints, Learning Distributed Safe—guarantees or filters. We want feasibility and "make it," pretraining on HJ labels.
- **Affordance:** RT-Affordance, NavA³—affordances as representation. We define affordance via HJ (feasibility, time-to-boundary) and supervise it.
- **Waypoint / sequence policies:** Decision Transformer (Chen et al. 2021), Diffusion Policy (Chi et al. 2023). We add HJ-derived waypoints and affordance heads.

---

## Papers

*Multi-agent reinforcement learning*

- **Amato (2024)** — "An Introduction to Centralized Training for Decentralized Execution in Cooperative Multi-Agent Reinforcement Learning," arXiv:2409.03052. Standard reference for the CTDE setting; we adopt the same execution regime and add feasibility pretraining.
- **Rashid et al. (2018)** — "QMIX: Monotonic Value Function Factorisation for Deep Multi-Agent Reinforcement Learning," ICML 2018, arXiv:1803.11485. Joint action-value as monotonic combination of per-agent utilities; tractable joint argmax; strong on SMAC; monotonicity leads to relative overgeneralization. We sidestep by supervising waypoints and affordance instead of relying only on value factorization.
- **Rashid et al. (2020)** — "Weighted QMIX: Expanding Monotonic Value Function Factorisation for Deep Multi-Agent Reinforcement Learning," NeurIPS 2020. Weighted projection so that better joint actions receive higher weight; mitigates overgeneralization. We inject structure via HJ labels rather than only refining the value representation.
- **Yu et al. (2021)** — "The Surprising Effectiveness of PPO in Cooperative Multi-Agent Games," arXiv:2103.01955 (MAPPO). PPO-based MARL with minimal tuning competes with or outperforms off-policy MARL on SMAC, Football, Hanabi. The bar is simple PPO; we add HJ pretraining on top.
- **Lowe et al. (2017)** — "Multi-Agent Actor-Critic for Mixed Cooperative-Competitive Environments," NeurIPS 2017, arXiv:1706.02275 (MADDPG). Centralized critic, decentralized actors; continuous actions; mixed settings. We use a single Transformer with waypoint and affordance heads plus HJ supervision instead of per-agent critics.
- **Sunehag et al. (2018)** — "Value-Decomposition Networks for Multi-Agent Reinforcement Learning," AAMAS 2018 (VDN). Additive (linear) factorization of joint value. Too restrictive in practice; QMIX generalizes it; we go beyond value-only learning with feasibility labels.
- **Wang et al. (2020)** — "QPLEX: Duplex Dueling Multi-Agent Q-Learning," NeurIPS 2020. Value-plus-advantance decomposition without monotonicity; richer joint action-value representation. We obtain "planning" from HJ labels rather than from the Q parameterization.
- **Li et al. (2021)** — "Deep Implicit Coordination Graphs," AAMAS 2021 (DICG). GNN over a learned coordination graph; implicit reasoning over joint actions. We obtain implicit when-to-commit via HJ-derived feasibility.
- **Foerster et al. (2018)** — "Counterfactual Multi-Agent Policy Gradients," AAAI 2018 (COMA). Centralized critic with counterfactual baseline (marginalize one agent's action) for credit assignment. We do not use explicit credit assignment; waypoint and affordance targets shape the policy.
- **Challenges in Credit Assignment for Multi-Agent RL in Open Agent Systems** (arXiv 2025, arXiv:2510.27659). Open systems (changing agent sets, tasks) cause credit misattribution. Our feasibility labels may provide a clearer learning signal.

*HJ reachability, safe navigation*

- **Certifying Hamilton–Jacobi Reachability Learned via Reinforcement Learning** (arXiv 2026). Certified inner/outer bounds on backward reachable tubes from learning error. They certify; we use HJ to label.
- **NeHMO** (arXiv 2025). Neural HJ for decentralized multi-agent motion planning; high-dimensional configuration spaces. Still "compute reachability"; we use reachability as data.
- **Your Learned Constraint is Secretly a Backward Reachable Tube** (arXiv 2025). Inverse constraint learning recovers BRTs. Our labels are BRT-consistent.
- **HJRNO** (arXiv 2025). Neural operator for backward reachable tubes; fast inference; generalization across obstacle geometry. Potential teacher for generating our labels at scale.
- **Resolving Conflicting Constraints in Multi-Agent RL with Layered Safety** (arXiv 2025). MARL with reachability and control barrier function (CBF) filters; Crazyflie experiments. Safety at runtime; we want feasibility in the policy.
- **CoRL-MPPI** (CoRL 2025). RL with model predictive path integral (MPPI) control; safe multi-robot collision avoidance. We use HJ as supervision, not as the control loop.
- **Safe Multi-Agent Navigation via Goal-Conditioned Safe RL** (arXiv 2025). Goal-conditioned safe RL plus planning; waypoints; conflict-based search. We use waypoints and HJ-derived feasibility.
- **Learning Distributed Safe Multi-Agent Navigation via Infinite-Horizon Optimal Graph Control** (arXiv 2025). Hamilton–Jacobi–Bellman (HJB) and GNN; CBF and distributed policies; Crazyflie. We use HJ for data.
- **Def-MARL** (arXiv 2025). Distributed epigraph optimization; zero constraint violation; quadcopters. They target guarantees; we target feasibility and success.

*Affordance and policy parameterization*

- **RT-Affordance** (ICRA 2025). Affordances as intermediate representations for manipulation. We define affordance via HJ in 3D navigation.
- **NavA³** (arXiv 2025). Hierarchical navigation with spatial object affordances; large dataset. We tie affordance to feasibility and time-to-boundary.
- **HJ reachability (foundations).** Backward reachable sets, value functions, time-to-boundary; tractable in low dimension, exponential in state dimension. We use these outputs as labels; pairwise 3D keeps dimension manageable.
- **Shielded RL: Safe RL via Formal Verification** (arXiv 2023). Survey of runtime shields. We invert: formal outputs as training data.
- **Chen et al. (2021)** — "Decision Transformer," NeurIPS 2021. Return-conditioned sequence policy. We use history with HJ-conditioned waypoints and affordance.
- **Chi et al. (2023)** — "Diffusion Policy," CoRL 2023. Diffusion over action sequences. Alternative parameterization; we use waypoint and affordance heads with HJ supervision.

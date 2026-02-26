# Literature Review — HJ Pretraining for Affordance-Aware MARL

## What

We care whether the agent can make it, not guaranteed safety. HJ reachability is usually used as a runtime shield; we use its outputs as **training targets** so the policy learns "is this gap closable?" from partial observations. No explicit planner; planning is implicit. Goal: higher success probability and better risk-taking. Risks: reward shaping may suffice; HJ does not scale beyond pairwise; RL fine-tune may overwrite pretrained structure.

---

## Novelty and publishability

**Novelty:** HJ-conditioned waypoints as *pretraining* data for MARL, then RL fine-tune—not HJ as shield, not BC of an arbitrary expert. Teacher is reachability-theoretic; setting is multi-agent 3D; goal is "make it." That combination is underexplored. **Clearly novel** if HJ pretraining improves make-it rate and/or sample efficiency over alternative pretraining or no pretraining.

---

## Done vs not done

**Done.** BC/imitation then RL; formal methods (reachability, CBF, MPPI) as runtime filters; HJ at runtime (certification, shields); MARL SOTA (CTDE, QMIX/MAPPO/MADDPG, value factorization, credit assignment); waypoint/sequence policies (Decision Transformer, Diffusion Policy); affordance as representation (RT-Affordance, NavA³); learned constraint = BRT (arXiv 2025).

**Not done.** HJ outputs as **training data** for MARL (not shield); affordance heads supervised by HJ; full pipeline (HJ waypoints + affordance labels → pretrain → RL) in 3D multi-agent; feasibility / "make it" as objective (not safety guarantees).


---

## Closest ideas (cited)

- **HJ + learning:** Usually certification or online use (Certifying HJ via RL, NeHMO, HJRNO) or shields (Shielded RL). We use HJ offline to label data.
- **Learned constraint = BRT** ("Your learned constraint is secretly a backward reachable tube"): they learn BRTs from data; we supply BRT-consistent labels.
- **Safe MARL:** CoRL-MPPI, Def-MARL, Resolving Conflicting Constraints, Learning Distributed Safe—guarantees or filters. We want feasibility and "make it," pretraining on HJ labels.
- **Affordance:** RT-Affordance, NavA³—affordances as representation. We define affordance via HJ (feasibility, time-to-boundary) and supervise it.
- **Waypoint / sequence policies:** Decision Transformer (Chen et al. 2021), Diffusion Policy (Chi et al. 2023). We add HJ-derived waypoints and affordance heads.
- **MARL.** CTDE (Amato 2024): train with global state, execute with local obs. PPO/MAPPO (Yu et al. 2021) often matches off-policy MARL on SMAC/Football. Value factorization: QMIX (Rashid et al. 2018), QPLEX (Wang et al. 2020); overgeneralization limits. Safe multi-agent nav: layered reachability + CBF, CoRL-MPPI, Def-MARL—formal method in the loop. We use HJ as data and target feasibility.
- **Closest.** HJ + learning: usually certification or online (Certifying HJ via RL, NeHMO, HJRNO) or shields; we use HJ offline to label. Safe MARL (CoRL-MPPI, Def-MARL): guarantees/filters; we want feasibility and pretraining. Affordance (RT-Affordance, NavA³): we define via HJ and supervise. Waypoint policies (Decision Transformer, Diffusion Policy): we add HJ-derived waypoints and affordance heads.

---

## Papers

*HJ reachability, safe navigation*

- [**Certifying Hamilton–Jacobi Reachability Learned via Reinforcement Learning**](https://arxiv.org/abs/2506.15693) (arXiv 2026). Gives certified inner/outer bounds on BRTs from learning error—so they *prove* the learned reachability is correct. We’re the opposite: we use HJ to *label* data, not to certify a learned model. Nice contrast to cite.
- [**NeHMO: Neural Hamilton-Jacobi Reachability for Decentralized Multi-Agent Motion Planning**](https://arxiv.org/abs/2507.13940) (arXiv 2025). Neural HJ in high-dim config spaces; decentralized. Still “compute reachability at runtime.” We take the same HJ machinery and feed it into training instead.
- [**Your Learned Constraint is Secretly a Backward Reachable Tube**](https://arxiv.org/abs/2501.15618) (arXiv 2025). Shows inverse constraint learning recovers BRTs, not just failure sets—dynamics-dependent. We don’t learn the constraint from data; we *supply* BRT-consistent labels. Directly aligned with our framing.
- [**HJRNO**](https://arxiv.org/abs/2504.19989) (arXiv 2025). Neural operator for BRTs; fast inference, generalizes across obstacles. Could be a practical *teacher* for generating our labels at scale when we don’t have analytic HJ.
- [**Resolving Conflicting Constraints in Multi-Agent RL with Layered Safety**](https://arxiv.org/abs/2505.02293) (RSS 2025). MARL + reachability + CBF in layers; Crazyflie. Safety is in the loop. We want that “feeling” in the policy *before* execution, via pretraining.
- [**CoRL-MPPI**](https://arxiv.org/abs/2511.09331) (CoRL 2025). RL + MPPI for safe multi-robot; control-theoretic loop. We use HJ as supervision only—no MPPI at deploy.
- [**Safe Multi-Agent Navigation via Goal-Conditioned Safe RL**](https://arxiv.org/abs/2502.17813) (arXiv 2025). Goal-conditioned safe RL + planning + conflict-based search; waypoints. Same “waypoints + safety” vibe; we add HJ-derived feasibility labels and pretrain.
- [**Learning Distributed Safe Multi-Agent Navigation via Infinite-Horizon Optimal Graph Control**](https://arxiv.org/abs/2506.22117) (arXiv 2025). HJB + GNN + CBF; distributed; Crazyflie. Again, formal method in the loop. We pull that structure into the data.
- [**Def-MARL**](https://arxiv.org/abs/2504.15425) (arXiv 2025, RSS 2025 Best Student Paper). Epigraph form, zero violation, quadcopters. They *guarantee* safety; we care about “make it” and feasibility—so different objective, same tool family.

*Affordance and policy parameterization*

- [**RT-Affordance**](https://arxiv.org/abs/2411.02704) (ICRA 2025). Affordances as mid-level representation for manipulation. We reuse the idea but *define* affordance via HJ (feasibility, time-to-boundary) in 3D nav and supervise it.
- [**NavA³**](https://arxiv.org/abs/2508.04598) (arXiv 2025). Hierarchical nav + spatial object affordances; big datasets. We tie affordance to “can I make it?” and train it from HJ labels.
- **HJ reachability (classic).** BRS, value functions, time-to-boundary—tractable in low dim, curse in high dim. We use the outputs as labels; pairwise 3D keeps things manageable. No single paper; standard references (e.g. Mitchell, Bayen, Tomlin).
- [**Shielded RL: Safe RL via Formal Verification**](https://arxiv.org/abs/2302.10336) (arXiv 2023, survey). Runtime shields everywhere. We flip it: formal outputs become *training targets*, not runtime guards.
- [**Decision Transformer**](https://arxiv.org/abs/2106.01345) (Chen et al., NeurIPS 2021). Return-conditioned sequence policy; RL as sequence modeling. We use the same “history → sequence” idea but condition on HJ-waypoint data and add affordance heads.
- [**Diffusion Policy**](https://arxiv.org/abs/2303.04137) (Chi et al., CoRL/RSS 2023). Diffusion over action sequences; strong in manipulation. Different parameterization; we stick with waypoint + affordance heads and HJ supervision, but same “chunk of actions” mindset.

*Multi-agent reinforcement learning*

- [**An Introduction to CTDE**](https://arxiv.org/abs/2409.03052) (Amato 2024). The go-to reference for centralized training, decentralized execution. We adopt that regime and add feasibility pretraining on top.
- [**QMIX**](https://arxiv.org/abs/1803.11485) (Rashid et al., ICML 2018). Monotonic value factorization; huge on SMAC; overgeneralization is the main wart. We add structure (waypoints, affordance) so the policy isn’t only learning from the value head.
- [**Weighted QMIX**](https://arxiv.org/abs/2006.10800) (Rashid et al., NeurIPS 2020). Weighted projection to fix overgeneralization. We fix it differently—by injecting HJ labels instead of reweighting the TD objective.
- [**MAPPO**](https://arxiv.org/abs/2103.01955) (Yu et al., NeurIPS 2021). PPO for MARL; often matches or beats fancy off-policy methods. Sets the “simple baseline” bar; we ask whether HJ pretraining beats that bar.
- [**MADDPG**](https://arxiv.org/abs/1706.02275) (Lowe et al., NeurIPS 2017). Centralized critic, decentralized actors; continuous actions. Foundational. We use one Transformer + waypoint/affordance heads instead of per-agent critics.
- [**VDN**](https://arxiv.org/abs/1706.05296) (Sunehag et al., AAMAS 2018). Additive value decomposition; too limited, QMIX supersedes it. We skip the value-factorization game and add feasibility as a direct target.
- [**QPLEX**](https://arxiv.org/abs/2008.01062) (Wang et al., ICLR 2021). Duplex dueling, no monotonicity; richer value class. Still value-centric; we get “planning” from HJ labels, not from the Q parameterization.
- [**DICG**](https://arxiv.org/abs/2106.06605) (Li et al., AAMAS 2021). GNN over coordination graph; implicit coordination. We get “when to commit” implicitly from HJ-derived feasibility.
- [**COMA**](https://arxiv.org/abs/1705.08926) (Foerster et al., AAAI 2018). Counterfactual baseline for credit assignment. We don’t do explicit credit; waypoint and affordance targets shape the policy.
- [**Challenges in Credit Assignment for Open Agent Systems**](https://arxiv.org/abs/2510.27659) (arXiv 2025). Open systems break credit. Feasibility labels could give a cleaner signal—worth a sentence in related work.

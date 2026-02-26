# Open questions

- HJ scalability: pairwise only? Enough data for pretraining?
- Does HJ pretraining beat reward shaping (ablations)?
- Is “implicit planning” value-function-like or imitation?
- Forgetting under RL fine-tune; token dims $L$, $H$, $K$.

# Additive Ideas? 

**Formal mix.** Can we use a *mix* of formal training data—not only HJ but a combination of optimal/formal solutions (e.g. HJ + MPC, MPPI, or other solvers) that all produce the same kind of output: conditioned waypoints for all agents in the scene? Same pipeline (waypoints for ego + others), but teacher is an ensemble or selector over formal methods.

**Scene-type labels.** Do we want (and can we get) labels for *type* of thing in the scene—e.g. passive vs active? E.g. falling objects / projectiles vs other flying agents; binary (passive/active) or a small taxonomy. Would give the policy extra supervision on what to treat as dynamic vs inert, and could be combined with waypoint/affordance targets.



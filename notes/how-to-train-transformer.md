# How to Train the Transformer

## What we want to do

We want to **pretrain** a causal Transformer so that: (1) **At pretrain**, the model sees **in-scene / FOV** observables plus **tokenized observable answers**—where agents are (centroid waypoints), radius (for belief state), etc. It’s all tokenized and part of the **input**; we’re not hiding answers in the loss only. (2) **Output** is ego waypoints plus **agents**: centroid waypoints per agent and **radius** (belief state / extent). (3) **At deploy** we feed only **FOV tracking**; belief is **implicit** (the model has learned it from the pretrain setup). Then we fine-tune with RL. Below is the **how**: tokenization, input/output, and how we train it.

---

## 1. Input and output (overview)

**Pretrain:** Model **input** = in-scene / FOV tokens **and** tokenized observable answers (agent positions, centroid waypoints, radius, etc.)—all part of the same token sequence. No “labels in the loss only”; the answers are in the input, tokenized.

**Deploy:** Model **input** = FOV tracking only (no observable-answer tokens). Belief is **implicit** in $z_t$.

**Output:** Ego waypoints $\hat{W}^{ego}_t$ **plus** per-agent **centroid waypoints** $\hat{W}^{obj,i}_t$ and **radius** $r^i$ (belief state / extent). So the model predicts where agents are (centroids) and their radius.

```
  Input (pretrain: FOV + tokenized answers)   Tokenize   Transformer   Aggregate   Heads   Output
  (deploy: FOV only)                          ────────►  ───────────►  last step   ─────►  Ŵ^ego, Ŵ^obj,i, r^i
```

| | Pretrain | Deploy |
|--|----------|--------|
| **Model input** | In-scene / FOV + tokenized observable answers (all input) | FOV tracking only |
| **Model output** | Ego waypoints; agent centroid waypoints; radius (belief) | same |
| **Belief** | In the tokenized answers (e.g. radius) | Implicit in $z_t$ |

---

## 2. Tokenization strategy

**How:** At each timestep we turn the observation into a **fixed set of token types**. In-scene / FOV gives us: ego, goal, map, object tracks. **Observable answers** (at pretrain) are also **tokenized and added to the input**: e.g. agent centroid waypoints, radius (belief state / extent). Same token vocabulary; the sequence carries both FOV tokens and answer tokens (teacher-provided at pretrain). At deploy we only feed FOV tokens.

| Token type | Contents (what we put in) |
|------------|---------------------------|
| **Ego** $T^{ego}_\tau$ | Position $p_\tau$, velocity $v_\tau$; optional: attitude, pose covariance |
| **Goal** $T^{goal}_\tau$ | Delta to goal $\Delta p^{goal}_\tau$, $\Delta v^{goal}_\tau$; goal type |
| **Map** $T^{map}_\tau$ | Local 3D (voxel or 2.5D) via CNN → one or a few patch tokens |
| **Object** $T^{obj,i}_\tau$ | Top-$K$: $\Delta p^i$, $\Delta v^i$, radius $r^i$, uncertainty, time since seen |
| **(Pretrain) Answer tokens** | Tokenized observable answers: agent centroid waypoints, radius (belief), etc. |

First-pass: ego + goal + map + $K$ object tokens per step; at pretrain we also append tokenized answers (centroid waypoints, radius) into the sequence. At deploy we omit answer tokens.

---

## 3. Input: sequence layout

**How:** We flatten the last $L$ steps into one long sequence, **time-major**: first all tokens for step $t-L+1$, then for $t-L+2$, …, then for $t$. Each step is a fixed block (ego, goal, map, obj 1..K). We add position encoding (timestep + role) and run **causal** attention (each position sees only past and present).

```
  ┌─ Step t−L+1 ─────────────────┐     ┌─ Step t ─────────────────┐
  │  Ego   Goal   Map   Obj 1..K   │ ... │  Ego   Goal   Map   Obj 1..K   │
  └───────────────────────────────┘     └───────────────────────────────┘
  ◄────────── L steps ──────────────►
```

$$\text{input seq} = [\underbrace{T^{ego}_{t-L+1}, T^{goal}_{t-L+1}, T^{map}_{t-L+1}, T^{obj,1}_{t-L+1}, \ldots, T^{obj,K}_{t-L+1}}_{\text{step } t-L+1},\;\ldots,\;\underbrace{T^{ego}_t, T^{goal}_t, T^{map}_t, T^{obj,1}_t, \ldots, T^{obj,K}_t}_{\text{step } t}]$$

- **Length:** $L \times (3 + K)$ tokens (3 = ego, goal, map; $K$ = object tracks).
- **Position encoding:** (timestep $\tau$, role). Role = ego | goal | map | obj_1 | … | obj_K so the model can distinguish "ego at t-2" from "obj_3 at t."
- **Causal mask:** Each position attends only to itself and earlier positions.

> [!note] **Why time-major + causal**
> **Time-major** keeps “one moment in time” together, so the model can easily associate “ego at t-2” with “obj_3 at t-2.” **Causal** attention means we never look at the future—so at step $t$ we only use information that would have been available by then. That’s required for online deployment and avoids cheating with future labels.

---

## 4. Output

**How:** Aggregate the **last timestep** tokens to $z_t$, then **heads** predict: **ego waypoints** $\hat{W}^{ego}_t$; per-agent **centroid waypoints** $\hat{W}^{obj,i}_t$; and **radius** $r^i$ (belief state / extent). So output = ego waypoints + agents (centroid waypoints + radius).

```
  Last timestep tokens (t only)
  ┌─────────────────────────────────────────┐
  │  Ego    Goal    Map    Obj 1   ... Obj K  │  (+ answer tokens at pretrain)
  └──────────────┬──────────────────────────┘
                 │ aggregate → z_t
                 ▼
  Ego head → Ŵ^{ego}   |   Agent centroid heads → Ŵ^{obj,i}   |   Radius head → r^i
```

Waypoint rate and horizon $H$ match the controller (e.g. 10 Hz, $H=10$).

> [!note] **Why last-step readout and waypoint chunk**
> **Last-step only:** The Transformer has already mixed information over time via attention; the last block is the natural “current belief” over the scene. Reading out from it keeps the interface simple (one vector $z_t$ → one action chunk). **Waypoint chunk:** We output a short trajectory (e.g. next $H$ waypoints at 10 Hz) instead of a single action. That matches the tracking controller’s bandwidth, reduces variance, and lets the policy express “go there over the next second” in one shot.

Optional later (dropped for first pass) (not required for first pass): separate heads for other agents’ centroid waypoints and radii; affordance heads (collision prob, margins) from the same $z_t$.

---

## How we train it (pretrain)

1. **Collect:** Teacher (e.g. HJ-conditioned or scripted) + tracking controller $\mathcal{C}$ produce rollouts. Log at each step: in-scene / FOV observables **and** the observable answers (agent centroid waypoints, radius, ego waypoints, etc.).
2. **Tokenize:** Turn each step into tokens: FOV tokens (ego, goal, map, objects) **plus** tokenized observable answers (centroid waypoints, radius). All go into the same input sequence.
3. **Input:** Feed the Transformer the full sequence (FOV + tokenized answers). Causal attention; position encoding (timestep + role).
4. **Output heads:** From $z_t$ (aggregated last-step tokens), predict: ego waypoints $\hat{W}^{ego}$, per-agent centroid waypoints $\hat{W}^{obj,i}$, radius $r^i$.
5. **Supervise:** Loss on predicted outputs vs. teacher (the observable answers we also put in the input). So the model learns to produce the same quantities it sees in the answer tokens when we have them; at deploy we don't feed answer tokens, and the heads give us waypoints + radius (implicit belief).

So: **pretrain = input includes tokenized observable answers, output heads predict those quantities; we train by supervising the heads.** Deploy = input is FOV only; we use the same heads, belief is implicit in $z_t$.

---

## 5. Why this structure (short)

1. **Single causal Transformer** — Direct attention over time; no extra recurrence.
2. **Time-major + role encoding** — Simple, deterministic layout; model knows when and what each token is.
3. **Last-step readout** — One $z_t$ for the whole chunk; minimal extra machinery.
4. **Waypoint chunk** — Matches controller bandwidth; reduces variance vs. per-step actions.

---

## 6. Stages: what each phase uses

The pipeline is four stages. **Pretrain:** input = FOV + tokenized observable answers (all part of input). **Deploy:** input = FOV tracking only; belief implicit.

```
  0. Collect  ───►  1. Pretrain  ───►  2. DAgger  ───►  3. RL
```

| Stage | Model input | Labels / environment | Output |
|-------|-------------|----------------------|--------|
| **0. Collect** | — | Sim + teacher + tracking controller $\mathcal{C}$ | Dataset: $(o_{t-L+1:t}, x^*_{...}, W^*_t, y_t)$; $o$ = in-scene/FOV |
| **1. Pretrain** | $o_{t-L+1:t}$ = FOV + tokenized observable answers (all input) | Supervise output heads to match teacher | Policy + heads ($\hat{W}^{ego}$, $\hat{W}^{obj,i}$, $r^i$); belief implicit at deploy |
| **2. DAgger** | $o$ from rollouts of $\pi_\theta$ | Relabel $W^*$ on collected states; append to $\mathcal{D}$ | Larger $\mathcal{D}$; re-run Stage 1 updates |
| **3. RL** | $o_{t-L+1:t}$ (FOV tracking only) | Reward $r$; $\mathcal{C}$ in the loop | Fine-tuned $\pi_\theta$ |

> [!note] **Why these four stages**
> - **0. Collect:** We need data that looks like deployment ($o$) but with good waypoints and labels. So we run a teacher (e.g. HJ-conditioned or scripted) with the same tracking controller we’ll use later, and log $o$, $W^*$, affordances, etc.
> - **1. Pretrain:** Imitation on that data. The model learns to map $o \mapsto W$ (and optionally affordances). No RL yet—just supervised learning so we start from a sensible policy.
> - **2. DAgger:** Imitation is biased toward “teacher states.” When the policy visits different states, it can fail. DAgger: run the current policy, relabel those states with the teacher, add to the dataset, pretrain again. Fixes distribution shift.
> - **3. RL:** Fine-tune with the real objective (reward) and the real controller in the loop. Pretraining gives a good init; RL adapts to the true task and compensates for teacher suboptimality.

Details: _summary.md; data and HJ labels: hj-training-data.md.

---

## 7. Hyperparameters to pin down

| Parameter | Description | Example |
|-----------|-------------|---------|
| $L$ | History length (steps) | TBD |
| $H$ | Waypoint horizon | TBD |
| $K$ | Max tracked objects | 5 |
| Token dim | Embedding size per token | TBD |
| Waypoint rate | Must match controller | 10 Hz |

Share controller type and $K$ to choose $L$, $H$, and model size for onboard latency.

---

## 8. Data (minimal)

Paired windows: $\mathcal{D} = \{(o_{t-L+1:t}, W^*_t, \ldots)\}$. $o$ = in-scene / FOV only; $W^*$ = teacher waypoints (observable answers) at same rate as controller. Pipeline: spawn → rollout with teacher + $\mathcal{C}$ → log → slice windows → augment $o$ (noise, dropout). See hj-training-data.md for HJ/labels.

---

## How a forward pass works (story for the talk)

1. **At time $t$** we have the last $L$ steps of **in-scene / FOV** data: what’s in the field of view—ego, goal relative to ego, local map in FOV, top-$K$ tracked objects (relative position, velocity, size, uncertainty).
2. **We tokenize** each step into a fixed set of tokens (ego, goal, map, obj 1..K) and stack them **time-major** into one sequence of length $L(3+K)$. We add position encoding (which step, which role).
3. **The causal Transformer** runs over this sequence; each token can attend only to past and present. By the end, the last block has “seen” the whole history in a causal way.
4. **We aggregate** the last block into one vector $z_t$ (e.g. mean or [CLS]). Belief state is **implicit** in $z_t$—we never feed belief in; the model builds it from FOV history.
5. **The waypoint head** maps $z_t$ to $H$ waypoints in $\mathbb{R}^3$. Those go to the tracking controller $\mathcal{C}$. We output where to go; the controller handles how.

So: **pretrain with in-scene/FOV inputs + all observable answers as labels**; **deploy with FOV tracking only**; **belief implicit**. One sequence in, one waypoint plan out.

---

## Presentation bullets (talking points)

- **Goal:** Pretrain with FOV + tokenized observable answers as **input**; output = ego waypoints + agents (centroid waypoints + radius). Deploy with FOV only; belief implicit. Then RL fine-tune.
- **Why in-scene / FOV only:** Same at pretrain and deploy; no privilege; what’s in the field of view is what we use.
- **Observable answers in the input:** Tokenized (centroid waypoints, radius) and fed in at pretrain; model predicts them from heads.
- **Why belief implicit:** At deploy we only get FOV tracking—no separate belief state. The Transformer’s $z_t$ is the implicit belief.
- **Why tokenize like this:** Ego, goal, map, objects = what we get from in-scene / FOV.
- **Why time-major + causal:** Keeps “one moment” together; no future leak; valid online.
- **How we train:** Collect (FOV + teacher answers) → tokenize both → Transformer in, heads out → supervise heads to match teacher. DAgger then RL.

## Implementation Modes of the Adaptation Algorithm

The tolerance-adaptation framework supports two primary modes of implementation — **parameter-level adaptation** and **pose-level sequential adaptation**.  
Both modes use performance feedback `P` and threshold evolution `T_next` to dynamically regulate difficulty across successive trials.  
While the same core update law (Algorithm 1, 2, or 3) governs both, the difference lies in how performance is interpreted and applied in relation to the parameter or pose.

---

### 1. Parameter-Level (Per-Metric) Adaptation

At the **parameter level**, the adaptation is executed independently for each performance metric.  
Let each tracked biomechanical metric be denoted as `M_i`, where `i ∈ {1,2,…,n}` represents parameters such as:

- `M₁` (Wrist–Palm Angle)  
- `M₂` (Average Bend)  
- `M₃` (Thumb–Index Distance)  
- `M₄` (Tip–Palm Distance)  
- …

For each parameter `M_i`, the system evaluates a **performance score** `P_i` at the end of every trial, based on the deviation between the measured and desired movement pattern:

```text
P_i = 1 − | M_i,actual − M_i,target |
```

(or its normalized form depending on the metric scale).

Each parameter maintains its own tolerance value `τ_i` and threshold `T_i = 1 − τ_i`.  
After each trial, the algorithm computes the updated value `T_next,i` (or `τ_next,i`) as a function of its own performance `P_i`:

```text
T_next,i = F(T_current,i, P_i, c)
```

where `F(·)` represents the selected adaptation law (Algorithm 1–3) and `c` is the adaptability coefficient controlling the rate of change.

Thus, each biomechanical parameter adapts autonomously — higher `P_i` (better performance) results in tighter thresholds for that same metric, whereas lower `P_i` (poor performance) retains or relaxes the tolerance.  
This creates **parameter-specific performance adaptation**, enabling localized control and precise tuning of motor accuracy across all tracked features.

---

### 2. Pose-Level Sequential Adaptation

In contrast, the **pose-level sequential mode** links adaptation across consecutive poses in a multi-step exercise.  
Let each pose in an exercise sequence be denoted as `S_j`, where `j ∈ {1,2,…,m}`, and the sequence cycles through:

```text
S₁ → S₂ → S₃ → … → Sₘ → S₁
```

Each pose `S_j` has its own performance score `P_j` and corresponding threshold `T_current,j`.  
However, unlike the parameter-level mode, here the threshold of the next pose depends on the **performance of the previous pose**:

```text
T_next,j = F(T_current,j, P_(j−1), c)
```

where `P_(j−1)` represents the performance feedback from the preceding pose in the sequence.  
This sequential dependency ensures that the accuracy achieved in `S_(j−1)` directly influences the difficulty level of `S_j` in the next trial.

For example, high performance in Pose 1 (`P₁`) increases the threshold for Pose 2 (`T_next,2`), tightening tolerance and encouraging more controlled motion in the subsequent step.  
Conversely, if Pose 3 yields a low `P₃`, the following Pose 4 adapts with a more relaxed threshold, allowing recovery and re-stabilization.

This **cross-pose adaptation** captures *temporal learning continuity*, ensuring that progression is not isolated to a single movement but carried through the entire exercise chain.  
It leverages **performance coupling** between poses to create smoother transitions and a coherent learning trajectory across repetitive cycles.

---

### Summary

**Parameter-Level Mode**

```text
T_next,i = F(T_current,i, P_i, c)
```

Each metric adapts based on its own performance — suitable for **fine-grained, local performance optimization**.

**Pose-Level Sequential Mode**

```text
T_next,j = F(T_current,j, P_(j−1), c)
```

Each pose adapts using the **performance of the previous pose**, enabling **inter-pose learning transfer** and **sequential performance continuity**.

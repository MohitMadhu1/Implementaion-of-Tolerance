## Gameplay Workflow and Behind-the-Scenes System Operation

### System Overview
XRRehab is a mixed-reality rehabilitation game loop where **therapeutic hand poses** (from XR hand tracking) directly drive gameplay. Each exercise defines a **pose success condition** using one or more tracked metrics (e.g., distances/angles/bend scores). When the user achieves the pose and holds it for the required time, the system provides **immediate game feedback** (projectile firing), records the best achieved values, and then updates difficulty using the adaptive tolerance engine.

At runtime, the system is composed of:
1. **Clinician setup + patient profiling** (Doctor UI + `PatientInfo`)
2. **Session orchestration** (exercise queue, transitions, run finalization) (`TrialManager`)
3. **Exercise detectors** (Stop / Down / Median Nerve Glide / Tendon Glide)
4. **Adaptive tolerance engine** (`ToleranceUpdater`) with **three selectable update laws**
5. **Gameplay feedback loop** (player projectile + enemy projectile)
6. **Persistence + run exports** (PlayerPrefs + run JSON + master JSON)

---

## 1) Clinician / Patient Setup (Doctor Workflow)
A clinician-facing UI captures and saves patient profile fields and also controls adaptation settings:

### Stored patient profile (persisted)
The Doctor script writes:
- `ActiveUsername` (e.g., `"P07"`)
- `PatientAge`
- `PatientGender`
- `PatientEthnicity`
- `PatientClimate`
- `PatientSeverity` (float)
- `PatientSeverityLabel`
- `ToleranceFormulaMode` (1..3)

It also saves a **ProfileDTO** entry and maintains `UserList` so returning users can be selected again.

### Adaptability coefficient \(c\)
On submit, the system computes an adaptability coefficient \(c\) from patient factors (age, gender, ethnicity, severity) and stores it as:
- `"{user}_c_value"` (and it supports using an existing stored value if already present).

### “Start new run on submit” (run indexing + run meta)
If `startNewRunOnSubmit` is enabled, the Doctor script triggers a new run via:
- `ToleranceUpdater.IncrementRun(user)` which increments `"{user}_runIndex"` and returns the new run id.

It also stores a small per-run meta JSON entry:
- `"{user}_run{runId}_meta"` including `user`, `run`, `mode`, `c`, and a UTC timestamp string.

Finally, the Doctor script starts the gameplay flow by calling into the TrialManager (e.g., apply profile / show intro / begin the active exercise sequence).

---

## 2) Session Orchestration and Exercise Queue (TrialManager)
`TrialManager` is the runtime controller that:
- Displays the main UI (instructions / summary text).
- Activates one exercise at a time (Stop, Down, Median Glide, Tendon Glide).
- Passes trial initialization payloads into detectors using `InitializeFromTrial(...)`.
- Handles end-of-run finalization (calibration flags, map rotation, export).

### Pause support
A lightweight `PauseManager` exists:
- `PauseManager.IsPaused` and `PauseManager.SetPaused(bool)`
TrialManager optionally uses a pause overlay panel (if assigned).

### Exercise selection flags and calibration flag
TrialManager reads **legacy global selection keys**:
- `Exercise_Stop`, `Exercise_Down`, `Exercise_MedianGlide`, `Exercise_TendonGlide`

Separately, calibration and “selected exercises per user” are handled by `UserDataStore` (see Section 5). This means the project currently contains **both**:
- Legacy global selection flags (used by TrialManager and some detectors)
- Per-user selection flags managed by `UserDataStore`

This dual system is important to document because migration can delete legacy keys (and then any code still reading the legacy keys will fall back to default values).

### Run completion responsibilities
On run completion, TrialManager:
- Marks the user as calibrated via `UserDataStore` (`"{user}_hasCalibration"`)
- Triggers map rotation by calling `MapRotationManager.NotifyRunComplete()`
- Exports a run snapshot (via `ToleranceUpdater.ExportRunToJson`) and appends it to a per-user master JSON file (via `ToleranceUpdater.AppendRunToMasterFile`)

---

## 3) Exercise Detector Runtime Loop (Stop / Down / Median / Tendon)
All detectors share the same “shape” of logic:

### (A) Activation + intro gating
When enabled, the detector:
- Reads `ActiveUsername` (for storing/loading per-user values)
- Reads `ActiveHand` (Left/Right) and configures handedness
- Runs an intro period (“Get Ready”) before allowing success checks, preventing accidental instant passes after activation.

Concrete examples from this APK:
- **StopGestureDetector** has an intro counter and then requires a **7 s hold** (`holdSeconds = 7`).
- **DownGestureDetector** has `introSeconds = 4` and requires a **6 s hold** (`holdSeconds = 6`).
- **MedianNerveGlidePolished** and **TendonGlideDetect** use **7 s hold** per step (`holdTime = 7`) and **inter-step delay** (`interStepDelaySeconds = 15`).

### (B) Metric acquisition from XR hand tracking
Detectors compute pose metrics each frame using XR hand joint positions (meters internally), with some values converted to cm for UI/threshold consistency.

Metrics used in this APK include (by detector):
- **StopGestureDetector**
  - `AvgDist`
  - `ThumbIndexDist`
  - `WristPalmAngle`
  - `WristToHeadDist`
- **DownGestureDetector**
  - `AvgDist`
  - `ThumbIndexDist`
  - `WristToHeadDist`
  - plus a configurable `palmDownTolerance = 0.08f` used in its pose gating logic
- **MedianNerveGlidePolished**
  - Uses a 5-step sequence and evaluates metrics named by strings like:
    - `AvgBend`, `ThumbIndexDist`, `TipPalmDist`, `WristPalmAngle` (depending on the configured metric list)
- **TendonGlideDetect**
  - Uses local metric types: `MCPBend`, `TipPalmDist`, `AvgBend`
  - Each metric has a default pass tolerance (e.g., `tolerance = 0.06f` in `StepMetric`)

### (C) Pass check + hold logic (no single-frame success)
A step is successful only when:
1. All required metrics satisfy their pass conditions (policy-aware),
2. The pose is held continuously for the required duration.

Progress is typically shown via a fill bar / countdown text during the hold period.

### (D) Gameplay feedback: projectile firing loop
When the pose is sufficiently correct (often while holding), detectors trigger the shooter feedback loop:
- **Player firing** uses `ProjectileShooter`:
  - `TryShoot()` respects a cooldown (`fireInterval = 0.5 s` default)
  - Locates enemies by tag (`enemyTag = "Enemy"`)
  - Fires a projectile from `launchOrigin` (often the camera/head transform) with `launchForce` (default 10)
- **Enemy firing** uses `EnemyShooterMohit`:
  - Rotates toward the target
  - Checks distance (`minAttackDistance`)
  - Fires on a cooldown (`attackCooldown = 2.0 s`)
  - Spawns from a configured spawn point transform

Projectiles are time-limited:
- Enemy projectile script destroys its projectile after a fixed time (e.g., 10 seconds), and handles collisions (e.g., tagged “Wall”).

### (E) Failure flow (where implemented)
Some exercises implement a “death / restart” routine with a short delay:
- **StopGestureDetector**, **MedianNerveGlidePolished**, and **TendonGlideDetect** include a `DeathRoutine()` that:
  - Shows a “You Died” panel/text (if assigned),
  - Resets internal step/hold state,
  - Waits `deathDelaySeconds = 3`,
  - Restarts the exercise sequence.

(DownGestureDetector in this APK does not implement the same death routine pattern.)

### (F) Completion and handover to TrialManager
When a step completes:
- The detector records the **actual achieved values** for each metric.
- It calls `ToleranceUpdater.UpdateMetricValue(...)` for each metric, passing:
  - `user`
  - internal step name (e.g., `"MedianNerveGlide_Step{step}"`, `"TendonGlide_Step{n}"`, `"StopGesture"`, `"DownGesture"`)
  - metric name
  - trial value used for pass checking
  - standard value
  - actual achieved value
  - bestPolicy (MinValue / MaxValue / ClosestToTarget)

Then the detector disables itself so TrialManager can activate the next exercise.

---

## 4) Adaptive Engine: Thresholds, Tolerances, and Update Modes
The adaptive system regulates difficulty by adjusting a **tolerance** value \(	au\) (stored as `tol`) and the associated “trial threshold” used by detectors.

### Core values per (user, step, metric)
- `s` = **standard** (clinician ideal)
- `t` = **trial** (current threshold used by the game)
- `a` = **actual** (best achieved this attempt/run)
- `tol` = **tolerance** (difficulty band)
- `policy` ∈ {MinValue, MaxValue, ClosestToTarget}
- `c` = patient adaptability coefficient

### Canonicalization and policy normalization (as coded)
The tolerance updater normalizes naming and policy so the system stays consistent:
- Metric name `"AvgTipDist"` is stored as canonical `"AvgDist"`.
- For step `"StopGesture"` and `"DownGesture"`, metric `"ThumbIndexDist"` is normalized to **MaxValue policy** (so its directionality is handled consistently in updates).

### Effective target and normalized performance score \(P\) (code-accurate)
The updater computes an **effectiveTarget** based on policy:
- MinValue → `effectiveTarget = max(t, s)`
- MaxValue → `effectiveTarget = min(t, s)`
- ClosestToTarget → uses `t` unless `t` is essentially equal to `s`

It then computes:
- `scale = max(|effectiveTarget|, |s|, |t|, 0.001)`
- `P = clamp(1 − 2*|a − s|/scale, −1, +1)`

So:
- \(P 	o +1\) means actual is very close to standard,
- \(P pprox 0\) means moderate deviation,
- \(P < 0\) means large deviation (typically relaxing tolerance).

---

## 4.1 Parameter-Level (Per-Metric) Adaptation
This mode applies adaptation **independently per metric**. Each metric has its own `tol` and `trial`, and updates are computed using that metric’s own performance score \(P_i\):

\[
	au_{next,i} = F(	au_{current,i}, P_i, c)
\]

This is the mode directly implemented by calling `UpdateMetricValue(...)` once per metric after each completed step.

---

## 4.2 Pose-Level Sequential Adaptation (as documented in the MD)
The framework also describes a **pose-level sequential mode** for multi-step exercises. Here, adaptation is linked across poses in a sequence:

\[
S_1 
ightarrow S_2 
ightarrow \dots 
ightarrow S_m 
ightarrow S_1
\]

Each pose \(S_j\) has its own threshold, but the update for pose \(j\) uses the **performance of the previous pose**:

\[
T_{next,j} = F(T_{current,j}, P_{j-1}, c)
\]

So instead of adapting each metric independently from its own feedback only, the sequence transfers learning forward step-to-step. This is specifically documented as **“pose-level sequential adaptation”** in your MD, as distinct from parameter-level adaptation.

*(Note: this subsection is included because it is explicitly defined in your documentation; the scripts you uploaded in this APK show the parameter-level update calls, while the MD defines this additional supported implementation mode.)*

---

## 4.3 Three Adaptation Laws (Algorithm 1 / 2 / 3 — equal importance)
XRRehab supports **three selectable tolerance update laws**, chosen at runtime using:
- `ToleranceFormulaMode ∈ {1,2,3}`

All three operate on tolerance \(	au\), using the same \(P\) and \(c\), but with different update dynamics:

### Algorithm 1 — multiplicative form (as implemented)
\[
	au_{next}=	au_{prev}igl(1-(1-c)Pigr)
\]

### Algorithm 2 — piecewise deadband form (as implemented)
Uses a deadband \(arepsilon = 0.05\). If \(|P|\le arepsilon\), the effective update term becomes 0; otherwise it rescales:
\[
f(P)=
egin{cases}
0 & |P|\le arepsilon\
rac{P-arepsilon\cdot 	ext{sign}(P)}{1-arepsilon} & 	ext{otherwise}
\end{cases}
\]
and then:
\[
	au_{next}=	au_{prev}-(1-c)f(P)
\]

### Algorithm 3 — inverse/denominator form (as implemented)
With \(lpha = 0.3\) in the current script:
\[
	au_{next}=rac{	au_{prev}}{1+lpha(1-c)P}
\]
(with a small denominator clamp for safety if it approaches 0).

### Safety floor (as implemented)
To avoid instability:
- `tolFloor = 0.01 * max(|effectiveTarget|, 1.0)`
- If `newTol` becomes NaN/Inf or drops below the floor, it is clamped to `tolFloor`.

### Trial (target) adjustment (as implemented)
Alongside tolerance updates, the updater also adjusts the **trial threshold** smoothly using:
- `ALPHA = 0.20`
and policy-aware logic (MinValue / MaxValue / ClosestToTarget), including gentle drift control toward the standard without abrupt jumps.

---

## 5) Persistence, Run Indexing, and Exports

### 5.1 Per-user selection + calibration state (UserDataStore)
`UserDataStore` maintains:
- Calibration: `"{user}_hasCalibration"`
- Per-user selections:
  - `"{user}_Exercise_Stop"`
  - `"{user}_Exercise_Down"`
  - `"{user}_Exercise_MedianGlide"`
  - `"{user}_Exercise_TendonGlide"`

It also includes a migration path from legacy keys:
- `Exercise_Stop`, `Exercise_Down`, `Exercise_MedianGlide`, `Exercise_TendonGlide`
and can delete legacy keys after migrating.

### 5.2 Run indexing
The tolerance updater tracks runs using:
- `"{user}_runIndex"`
Incremented via `ToleranceUpdater.IncrementRun(user)`.

### 5.3 “Touched steps/metrics” tracking (for clean exports)
Instead of exporting everything every time, the tolerance updater records which steps and metrics were modified in the current run using compact “set-like” PlayerPrefs strings (with a `|value|` token style).

### 5.4 Key structure for latest + per-run history
For each updated (step, metric), it stores both:
- Latest:
  - `"{user}_tol_{step}_{metric}"`
  - `"{user}_trial_{step}_{metric}"`
  - `"{user}_actual_{step}_{metric}"`
- Per-run history:
  - `"{user}_tol_r{run}_{step}_{metric}"`
  - `"{user}_trial_r{run}_{step}_{metric}"`
  - `"{user}_actual_r{run}_{step}_{metric}"`

### 5.5 Run export and master JSON append
At end of run:
- `ToleranceUpdater.ExportRunToJson(user, run)` generates a JSON snapshot containing the touched steps/metrics and their tol/trial/actual values.
- `ToleranceUpdater.AppendRunToMasterFile(user, json)` appends that run snapshot into the per-user master file.

### Trial threshold loading compatibility (glide detectors)
The multi-step glide detectors load trial thresholds with fallback keys for compatibility:
- **Median glide** first checks a per-exercise key like:
  - `"{user}_trial_MedianNerveGlide_{metricName}"`
  then falls back to display-step keys:
  - `"{user}_trial_{displayStep}_{metricName}"`
  and even global keys like:
  - `"trial_{displayStep}_{metricName}"`
- **Tendon glide** reads:
  - `"{user}_trial_{displayStep}_{metric}"` for each step metric.

This ensures trial values remain usable across versions and naming changes.

---

## 6) Map Rotation and Session Variety
`MapRotationManager` supports environment rotation across sessions:
- It expects **exactly four map root objects** (`maps[]`)
- Stores the active map index in:
  - `"MapRot_Index"`
- Applies the saved map on `Awake()` (if `applyOnStart = true`)
- Advances the rotation only when `NotifyRunComplete()` is called (TrialManager calls this at end of run), avoiding mid-run swaps.


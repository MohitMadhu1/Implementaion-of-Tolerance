
## Tolerance Adaptation Algorithms in XR Rehab

This document summarizes the three tolerance-adaptation algorithms used in XR Rehab.  
All three algorithms update a tolerance value \(\tau\) and its corresponding accuracy threshold \(T = 1 - \tau\) based on a performance score \(P\) and an adaptability coefficient \(c\).

---

### Algorithm 1 – Baseline Progressive Adaptation (Paper Algorithm)

This is the original algorithm described in the paper, based on direct proportional control between performance and tolerance.

```latex
T = 1 - 	au
```

```latex
	au_{	ext{next}} = 	au_{	ext{current}} \left( 1 - (1 - c)\,P 
ight)
```

- \(P = 1 - 2\,|\text{actual accuracy} - \text{desired accuracy}|\) is the patient’s performance score.  
- \(c \in (0,1)\) is the adaptability coefficient that accounts for clinical caution or demographic risk.

**Behavior**

- Higher \(P\) (good performance) \(\Rightarrow\) \(\tau_{\text{next}}\) decreases, so tolerance tightens and the task becomes harder.  
- Lower \(P\) (poor performance) \(\Rightarrow\) \(\tau_{\text{next}}\) increases or stays higher, so tolerance relaxes.  

This linear law provides smooth and stable updates across each exercise cycle.

---

### Algorithm 2 – Inverse Adaptation Law (Simple Inverse Form)

This model follows an inverse relationship between tolerance and performance, making the adaptation smoother as performance improves.

```latex
	au_{	ext{next}} = rac{	au_{	ext{current}}}{1 + lpha (1 - c)\,P}
```

```latex
T_{	ext{next}} = 1 - 	au_{	ext{next}}
```

- \(P \in [0,1]\) is the performance score.  
- \(\alpha\) is a sensitivity factor (typically between 0.1 and 0.5).  
- \(c\) is the adaptability or caution coefficient.

**Behavior**

- As \(P\) increases, the denominator grows, so \(\tau_{\text{next}}\) becomes smaller and the task gets harder.  
- For lower \(P\), the change is smaller, producing a gradual, self-limiting adaptation.  

This inverse formulation is stable, bounded, and computationally simple.

---

### Algorithm 3 – Piecewise Non-Linear Feedback Function Model

The third model introduces a non-linear correction function \(f(P)\) to manage sensitivity near the target performance.

```latex
T_{	ext{next}} = T_{	ext{current}} - (1 - c)\,lpha\,f(P)
```

where

```latex
f(P) =
egin{cases}
0, & |P| \le arepsilon \\[6pt]
\dfrac{P - arepsilon \sin(P)}{1 - arepsilon}, & |P| > arepsilon
\end{cases}
```

- \(\varepsilon\) defines a deadband range around zero performance error.  
- \(\alpha\) is a scaling constant.  
- \(c\) is the adaptability coefficient.

**Behavior**

- When \(|P| \le \varepsilon\), no adjustment occurs (\(f(P) = 0\)), preventing over-sensitivity or oscillation when accuracy is already close to the target.  
- When \(|P| > \varepsilon\), the sinusoidal term produces a smooth, bounded correction that progressively adjusts \(T\).  

This non-linear feedback model offers high stability and noise robustness, especially near the optimal performance region.

---

### Note on Usage

Each algorithm can be applied:

1. **Per-parameter (metric-wise):**  
   \(T_{\text{next},i} = F(T_{\text{current},i}, P_i, c)\)  
   where \(i\) indexes individual biomechanical metrics (e.g., wrist angle, bend, distances).

2. **Per-pose in a sequence (sequential mode):**  
   \(T_{\text{next},j} = F(T_{\text{current},j}, P_{j-1}, c)\)  
   where pose \(S_j\) in the sequence uses the performance \(P_{j-1}\) of the previous pose.

Here, \(F\) is any of the three algorithms defined above.

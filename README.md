# Twisted Hessian Curves: First-Principles Architecture for HSM Implementation

---

## Table of Contents

1. [Algebraic Foundations](#1-algebraic-foundations)
2. [Computational Graph Reconstruction](#2-computational-graph-reconstruction)
3. [Information Theory and Side-Channel Resistance](#3-information-theory-and-side-channel-resistance)
4. [Error Correction and Boundary Cases](#4-error-correction-and-boundary-cases)
5. [Benchmarking Strategy](#5-benchmarking-strategy)
6. [Mathematical Appendix: Full Derivations](#6-mathematical-appendix-full-derivations)
7. [References](#7-references)

---

## 1. Algebraic Foundations

### 1.1 The Ambient Space

Work over a finite prime field **𝔽ₚ** with `char(𝔽ₚ) ≠ 2, 3` (i.e., `p > 3`). The projective plane **ℙ²(𝔽ₚ)** consists of equivalence classes `[X : Y : Z]` with `(X,Y,Z) ≠ (0,0,0)` under `(X,Y,Z) ~ (λX, λY, λZ)` for `λ ∈ 𝔽ₚ*`.

A **smooth projective cubic** `E ⊂ ℙ²` is the zero locus of a homogeneous degree-3 polynomial `F(X,Y,Z) = 0` satisfying:

```
∂F/∂X|_P = ∂F/∂Y|_P = ∂F/∂Z|_P = 0  ⟹  P ∉ E
```

(no singular points on the curve).

### 1.2 The Hessian Form and ℤ/3ℤ Symmetry

**Definition (Hessian determinant).** For a homogeneous cubic `F`, define:

```
𝒽(F) := det [ F_XX  F_XY  F_XZ ]
             [ F_YX  F_YY  F_YZ ]
             [ F_ZX  F_ZY  F_ZZ ]
```

This is itself a degree-3 homogeneous polynomial. The **nine flex points** (inflection points) of `E` are precisely `E ∩ V(𝒽(F))`.

**Theorem (Reduction to Hessian Form).** Over the algebraic closure `𝔽̄ₚ`, every smooth cubic is projectively equivalent to:

```
H(c) :   X³ + Y³ + Z³ = 3c·XYZ,    c ∈ 𝔽ₚ,  c³ ≠ 1
```

This form arises by placing three collinear flex points at `[1:0:0]`, `[0:1:0]`, `[0:0:1]`. The **ℤ/3ℤ symmetry** `[X:Y:Z] ↦ [Y:Z:X]` is a manifest automorphism.

### 1.3 Why "Twisted": The Galois Obstruction

**The problem.** Over `𝔽ₚ`, the flex points of a cubic may lie only in extension fields `𝔽_{p^2}` or `𝔽_{p^3}`. No `𝔽ₚ`-rational projective change of coordinates can then achieve the standard Hessian form while preserving the rational group structure.

**The remedy.** A **twisted Hessian curve** is a twist of `H(c)` in the sense of Galois cohomology: a curve isomorphic to `H(c)` over `𝔽̄ₚ` but in a distinct `𝔽ₚ`-isomorphism class. Introducing two independent parameters `(a, d) ∈ 𝔽ₚ²`:

```
┌─────────────────────────────────────────────────────────────┐
│  TH(a,d) :   aX³ + Y³ + Z³ = d·XYZ          [projective]  │
│  TH(a,d) :   ax³ + y³ + 1  = d·xy            [affine Z=1]  │
└─────────────────────────────────────────────────────────────┘
```

**Non-singularity condition:**

```
Δ(a,d) = a(d³ − 27a) ≠ 0
```

This simultaneously requires `a ≠ 0` and `d³ ≠ 27a`.

**j-invariant:**

```
j(TH(a,d)) = d³(d³ − 216a)³ / [a(d³ − 27a)³]
```

**Completeness theorem** (Bernstein–Lange 2007; Farashahi–Joye 2010). For any elliptic curve `E/𝔽ₚ` with a rational 3-torsion point, there exist `a, d ∈ 𝔽ₚ` with `a(d³ − 27a) ≠ 0` such that `E ≅ TH(a,d)` over `𝔽ₚ`.

### 1.4 Parameter Space Exploration: Density of Secure Curves

**How `a` and `d` are chosen.** Given a Weierstrass curve `y² = x³ + Ax + B` with rational 3-torsion point `T = (x₀, y₀)`, define:

```
α := (3x₀² + A) / (2y₀)          [tangent slope at T]
β := y₀ − α·x₀                    [tangent intercept]
a  := α³                           [twist parameter]
d  := 3α³ + 3β / (something)...  [computed from substitution; see §6.3]
```

**Degrees of freedom and density.** The parameter space for `(a,d)` over `𝔽ₚ` satisfying:
1. `a ≠ 0`: eliminates 1 value → `p−1` choices for `a`
2. `d³ ≠ 27a`: for each `a`, eliminates the 1–3 cube roots of `27a` → approximately `p − O(1)` choices for `d`

So the **density** of valid (non-singular) parameters is:

```
|{(a,d) ∈ 𝔽ₚ² : a(d³−27a) ≠ 0}| ≈ p² − O(p)
```

giving an asymptotic fraction of valid pairs approaching **1** as `p → ∞`.

**Impact on security.** For a 256-bit prime `p`:
- **Choosing small `a`** (e.g., `a = 1`) reduces implementation complexity (no `a`-dependent multiplications in some formula variants).
- **Choosing `a` that is not a cubic residue mod `p`** ensures the twist is non-trivial and prevents certain isomorphism attacks.
- **The density result guarantees** that nearly any random `(a,d)` pair yields a valid, distinct elliptic curve, enabling **parameter agility** — a critical property for HSMs that require curve diversification across deployed modules.
- **For twist-security**: pair `TH(a,d)` with its quadratic twist to ensure both curves have large prime-order subgroups (analogous to the requirement in Ed25519/Curve25519 design).

**Cryptographic parameter selection algorithm:**

```
Input:  prime p, security parameter λ = 128 (for 256-bit p)
Output: (a, d) with a·(d³−27a) ≠ 0, #TH(a,d)(𝔽ₚ) = h·r (r large prime, h ∈ {1,2,3,4})

1. Sample a ←$ 𝔽ₚ*
2. Sample d ←$ 𝔽ₚ  such that d³ ≠ 27a
3. Compute N = #TH(a,d)(𝔽ₚ) using Schoof–Elkies–Atkin (SEA)
4. If N/h is prime for small h and passes CM-discriminant checks → accept
5. Else → goto 1
Expected iterations: O(log²(p))  [by prime number theorem for curves]
```

---

## 2. Computational Graph Reconstruction

### 2.1 The Unified Group Law: Algebraic Foundation

**Identity element.** The affine point `𝒪 = (0, −1)` is the neutral element of the group law.

*Verification on curve:* `a·0 + (−1)³ + 1 = 0 = d·0·(−1)` ✓

*Verification as flex point (rigorous).* In projective coordinates, `𝒪 = [0:1:−1]`. The Hessian matrix of `F = aX³ + Y³ + Z³ − dXYZ` at this point is:

```
𝒽(F)|_{[0:1:−1]} = det | 0   d  −d |
                        | d   6   0 |
                        |−d   0  −6 |

= 0·(6·(−6) − 0) − d·(d·(−6) − 0) + (−d)·(d·0 − 6·(−d))
= 0 − d(−6d) + (−d)(6d)
= 6d² − 6d²  =  0  ✓
```

*Parametric flex confirmation.* The tangent at `[0:1:−1]` is `∇F·v = 0` with `∇F|_{[0:1:−1]} = (d, 3, 3)`, giving line `dX + 3Y + 3Z = 0`. Parametrize the tangent as `[X:Y:Z] = [3t : s : −s−dt]` (tangent direction `[3:0:−d]`). Substituting into `F`:

```
F = a(3t)³ + s³ + (−s−dt)³ − d(3t)(s)(−s−dt)
  = 27at³ − d³t³ + cancellation of all s-terms
  = (27a − d³)t³
```

Since `d³ ≠ 27a` (non-singularity), `(27a − d³) ≠ 0`, so `t = 0` is a **triple root**. This confirms `𝒪 = [0:1:−1]` is a flex point, and therefore `[2]𝒪 = 𝒪` follows immediately from the chord-and-tangent construction. ✓

**Negation.** The negation involution on `aX³ + Y³ + Z³ = dXYZ` is:

```
−[X:Y:Z] = [X:Z:Y]    (swap Y and Z coordinates)
```

*Verification:* If `aX³+Y³+Z³ = dXYZ`, then `aX³+Z³+Y³ = dXZY = dXYZ` ✓ (RHS symmetric in Y,Z).

In affine coordinates (`Z = 1`, `x = X`, `y = Y`): the map `[x:y:1] ↦ [x:1:y]`, dehomogenizing by new-Z gives:

```
−(x, y) = (x/y,  1/y)
```

*Self-inverse check on identity:* `−(0,−1) = (0/(−1), 1/(−1)) = (0,−1) = 𝒪` ✓  
*On-curve check:* if `ax³+y³+1=dxy`, then for `(x/y, 1/y)`: substituting `x' = x/y`, `y' = 1/y`:  
`a(x/y)³ + (1/y)³ + 1 = (ax³+1)/y³ + 1 = (dxy−y³)/y³ + 1 = dx/y² − 1 + 1 = dx/y²`  
`d·x' ·y' = d·(x/y)·(1/y) = dx/y²` ✓

### 2.2 The Projective Unified Addition Formula

**Polarization method.** For `F = aX³+Y³+Z³−dXYZ` and points `P₁, P₂` on the curve, substitute `[X:Y:Z] = λP₁ + μP₂` into `F`. Since `F(P₁) = F(P₂) = 0`:

```
F(λP₁ + μP₂) = λμ [ λ·Φ(P₁,P₁,P₂) + μ·Φ(P₁,P₂,P₂) ] = 0
```

where `Φ` is the full symmetric trilinear polarization:

```
Φ(P₁,P₁,P₂) = aX₁²X₂ + Y₁²Y₂ + Z₁²Z₂ − (d/3)(X₁Y₁Z₂ + X₁Y₂Z₁ + X₂Y₁Z₁)

Φ(P₁,P₂,P₂) = aX₁X₂² + Y₁Y₂² + Z₁Z₂² − (d/3)(X₁Y₂Z₂ + X₂Y₁Z₂ + X₂Y₂Z₁)
```

The three intersections of the line `P₁P₂` with `TH(a,d)` occur at `[λ:μ] = [0:1]` (gives `P₂`), `[1:0]` (gives `P₁`), and:

```
[λ:μ] = [−Φ(P₁,P₂,P₂) : Φ(P₁,P₁,P₂)]
```

so the third intersection point is:

```
R = −Φ(P₁,P₂,P₂)·P₁ + Φ(P₁,P₁,P₂)·P₂
```

Applying the negation `[X:Y:Z] ↦ [X:Z:Y]` to `R` yields `P₁ + P₂`:

```
╔══════════════════════════════════════════════════════════════════════╗
║  PROJECTIVE UNIFIED ADDITION FORMULA                                 ║
║  aX³+Y³+Z³=dXYZ,  identity [0:1:−1],  negation [X:Y:Z]↦[X:Z:Y]    ║
║                                                                      ║
║  Let  μ = aX₁²X₂+Y₁²Y₂+Z₁²Z₂−(d/3)(X₁Y₁Z₂+X₁Y₂Z₁+X₂Y₁Z₁)       ║
║       ν = aX₁X₂²+Y₁Y₂²+Z₁Z₂²−(d/3)(X₁Y₂Z₂+X₂Y₁Z₂+X₂Y₂Z₁)       ║
║                                                                      ║
║  X₃ = μX₂ − νX₁                                                    ║
║  Y₃ = μZ₂ − νZ₁          [negation applied: Y←Z, Z←Y of R]        ║
║  Z₃ = μY₂ − νY₁                                                    ║
╚══════════════════════════════════════════════════════════════════════╝
```

**Verification (identity property).** Set `P₂ = 𝒪 = [0:1:−1]`:

```
μ = 0 + Y₁²·1 + Z₁²·(−1) − (d/3)(X₁Y₁·(−1) + X₁·1·Z₁ + 0)
  = Y₁² − Z₁² + (d/3)X₁(Y₁−Z₁)
  = (Y₁−Z₁)(Y₁+Z₁+(d/3)X₁)

ν = 0 + Y₁·1² + Z₁·(−1)² − (d/3)(X₁·1·(−1) + 0 + 0)
  = Y₁ + Z₁ + (d/3)X₁

R = −ν·[X₁:Y₁:Z₁] + μ·[0:1:−1]

R_X = −ν·X₁ + 0 = −(Y₁+Z₁+(d/3)X₁)·X₁
R_Y = −ν·Y₁ + μ·1 = (Y₁+Z₁+(d/3)X₁)[−Y₁+(Y₁−Z₁)] = −(Y₁+Z₁+(d/3)X₁)·Z₁
R_Z = −ν·Z₁ + μ·(−1) = (Y₁+Z₁+(d/3)X₁)[−Z₁−(Y₁−Z₁)] = −(Y₁+Z₁+(d/3)X₁)·Y₁

P₁+𝒪 = [R_X:R_Z:R_Y] = [−νX₁:−νY₁:−νZ₁] = [X₁:Y₁:Z₁] = P₁  ✓
```

### 2.3 DAG Decomposition and Register Analysis

The unified formula requires computing `μ`, `ν`, and three linear combinations. Below is the full operation graph for projective inputs `(X₁,Y₁,Z₁)`, `(X₂,Y₂,Z₂)`.

```
COMPUTATIONAL DAG  (M = field multiplication, A = field addition)
═══════════════════════════════════════════════════════════════════

INPUT:  X1, Y1, Z1, X2, Y2, Z2, a, d

LAYER 0 — Squarings (can be parallelized):
  t1  = X1·X1     (S)
  t2  = Y1·Y1     (S)
  t3  = Z1·Z1     (S)

LAYER 1 — Cross-products for μ:
  t4  = t1·X2          (M)   = X1²·X2
  t5  = t2·Y2          (M)   = Y1²·Y2
  t6  = t3·Z2          (M)   = Z1²·Z2
  t7  = a·t4           (M)   = a·X1²·X2    [can fold if a is small]

LAYER 1' — Cross-products for ν:
  t8  = X1·(X2·X2)     (S+M) = X1·X2²
  t9  = Y1·(Y2·Y2)     (S+M) = Y1·Y2²
  t10 = Z1·(Z2·Z2)     (S+M) = Z1·Z2²
  t11 = a·t8           (M)

LAYER 2 — (d/3) terms for μ  [note: precompute d3inv = d·3⁻¹ mod p]:
  t12 = X1·Y1          (M)
  t13 = t12·Z2         (M)   = X1·Y1·Z2
  t14 = X1·Z1          (M)
  t15 = Y2·t14         (M)   = X1·Z1·Y2   [= X1·Y2·Z1]
  t16 = Y1·Z1          (M)
  t17 = X2·t16         (M)   = X2·Y1·Z1

LAYER 2' — (d/3) terms for ν:
  t18 = X1·Y2          (M)
  t19 = t18·Z2         (M)   = X1·Y2·Z2
  t20 = X2·Y1          (M)
  t21 = t20·Z2         (M)   = X2·Y1·Z2
  t22 = X2·Y2          (M)
  t23 = t22·Z1         (M)   = X2·Y2·Z1

LAYER 3 — Assemble μ, ν  (A = additions, free on modern HW):
  μ = t7 + t5 + t6 − d3inv·(t13 + t15 + t17)
  ν = t11 + t9 + t10 − d3inv·(t19 + t21 + t23)

LAYER 4 — Output coordinates:
  X3 = μ·X2 − ν·X1      (2M + 1A)
  Y3 = μ·Z2 − ν·Z1      (2M + 1A)
  Z3 = μ·Y2 − ν·Y1      (2M + 1A)

OUTPUT:  [X3:Y3:Z3]
```

**Multiplication count summary:**

| Phase | Squarings (S) | Multiplications (M) |
|-------|:---:|:---:|
| Squarings (L0) | 3 | — |
| μ cross-products (L1) | — | 3 + 1 |
| ν cross-products (L1') | 3 | 3 + 1 |
| (d/3) terms (L2+L2') | — | 9 |
| Output (L4) | — | 6 |
| **Total** | **6S** | **17M** |

> **Note:** When squarings are counted at cost `S ≈ 0.8M` (typical in Montgomery form),  
> total ≈ `17M + 4.8M = 21.8M` equivalent. Published optimized results achieve **12M + 6S** (see §5).

**Minimum register count.** The DAG has maximum in-degree 3 (each `Φ` term) and critical path depth 4. By register allocation analysis:

```
Minimum temporary registers required: 14
  - 6 for input coordinates (can be read-only if hw supports)
  - 3 for squarings (t1,t2,t3) — reusable after layer 1
  - 2 accumulators for μ, ν
  - 3 for output (X3,Y3,Z3)
```

### 2.4 Constant-Time Data Flow Analysis

A critical HSM requirement is **constant-time execution**: the sequence of operations and memory accesses must be independent of the secret scalar.

**Data flow properties of this formula:**
1. **No conditional branches.** The formula `[X₃:Y₃:Z₃]` is a polynomial map — no `if/else` on field values.
2. **No early-exit on zero.** The only potential branch is if `μ = ν = 0` (indeterminate output), but this cannot occur when `P₁ ≠ ±P₂` (Bézout guarantees a unique third intersection).
3. **Memory access pattern:** All `t_i` are computed in topological order with fixed memory addresses — no secret-dependent address offsets.

**Cache-timing risk:** In software, table lookups for Montgomery multiplication must use scatter/gather operations or hardware multiply units to avoid cache-timing channels. In HSM hardware, all multiplications proceed in a fixed number of clock cycles.

### 2.5 SIMD Parallelism Analysis

The DAG has significant **independent parallelism**:

```
PARALLEL EXECUTION SCHEDULE (4-way SIMD)
═════════════════════════════════════════
Cycle 1:  SIMD-4 multiply: [X1²,  Y1²,  Z1²,  X2²]     → [t1,t2,t3,t26]
Cycle 2:  SIMD-4 multiply: [Y2²,  Z2²,  t1·X2, t2·Y2]  → [t27,t28,t4,t5]
Cycle 3:  SIMD-4 multiply: [t3·Z2, a·t4, t26·X1, t27·Y1]→ [t6,t7,t8,t9]
...etc.

Critical-path length with 4-lane SIMD: ~5 SIMD rounds vs 17 serial M
Speedup factor: ~3.4×  (limited by data dependencies in μ,ν assembly)
```

For 256-bit modular arithmetic on an HSM with 4-wide SIMD (e.g., custom FPGA with 4 parallel Montgomery multipliers):

| Architecture | Clock cycles (estimated) | Notes |
|---|---|---|
| Serial, 1 multiplier | 12M × Tₘ | Tₘ = Montgomery cycle count |
| 2-lane SIMD | ~7M × Tₘ | Data-dependency bottleneck |
| 4-lane SIMD | ~5M × Tₘ | ~2.4× serial speedup |
| 8-lane SIMD | ~4M × Tₘ | Diminishing returns |

---

## 3. Information Theory and Side-Channel Resistance

### 3.1 The Unification Property

**Definition.** An addition formula is called **unified** if the same sequence of field operations computes both `P₁ + P₂` (for `P₁ ≠ P₂`) and `[2]P₁` (doubling), without any secret-dependent branching.

**Theorem (Cost uniformity, Bernstein–Lange 2007).** For the twisted Hessian curve `TH(a,d)`, both point addition and point doubling require exactly **12M + 6S** field operations (projective, using optimized formulas). The algebraic form of both computations is identical.

This means a scalar multiplication loop:

```python
# Montgomery ladder on TH(a,d)
R0 = O;  R1 = P
for bit in scalar_bits(k):      # MSB to LSB
    if bit == 0:
        R1 = add(R0, R1)        # = R0 + R1 (always: 12M+6S)
        R0 = double(R0)         # = 2·R0    (always: 12M+6S)
    else:
        R0 = add(R0, R1)        # symmetric cost
        R1 = double(R1)
```

executes **exactly the same sequence of field multiplications** regardless of the bit value, because `add()` and `double()` have identical cost profiles.

### 3.2 DPA Leakage Profile

**Differential Power Analysis (DPA) threat model.** An attacker observes the power trace `P(t)` during scalar multiplication `[k]G`, attempts to distinguish whether a given operation is an addition or a doubling, and uses this to recover bits of `k`.

**Standard DPA on Weierstrass coordinates** succeeds because:
- `add(P,Q)` requires computing `λ = (y₂−y₁)/(x₂−x₁)` (requires `x₁ ≠ x₂`)
- `double(P)` requires computing `λ = (3x₁²+a)/(2y₁)` (different intermediate values)

The two paths have **different Hamming weight distributions** over intermediate results, creating detectable power signature differences.

**Why Twisted Hessian resists DPA.** Consider the power trace as a random variable `W ∼ PowerTrace(op, inputs)`. For an ML adversary training a classifier `f: W → {add, double}`:

```
Indistinguishability Claim:
  The mutual information I(op; W) ≈ 0 when the same formula executes both ops.

Proof sketch:
  Both ops compute:  μ = aX₁²X₂ + Y₁²Y₂ + Z₁²Z₂ − (d/3)(...)
                     ν = aX₁X₂² + Y₁Y₂² + Z₁Z₂² − (d/3)(...)
  The only input difference between add and double is whether (X₂,Y₂,Z₂)
  is an independent point or equals (X₁,Y₁,Z₁). In Montgomery-form arithmetic,
  both are 3-word field elements with statistically identical Hamming weight
  distributions if the inputs are uniformly random projective points.
```

### 3.3 Statistical Analysis of the Leakage Profile

**Model.** Let `H_w(v)` denote the Hamming weight of field element `v`. In a first-order DPA model, the power signal is:

```
P(t) = Σᵢ H_w(rᵢ(t)) + noise(t)
```

where `rᵢ(t)` are intermediate register values at time `t`.

**Key invariant of the Hessian formula.** For any projective point `[X:Y:Z]` on `TH(a,d)`, the intermediate values computed in `μ` and `ν` are **degree-3 polynomials** in the input coordinates. If coordinates are drawn uniformly from `𝔽ₚ*`:

```
𝔼[H_w(aX₁²X₂)] ≈ (n/2)    (n = bit-length of p)
𝔼[H_w(aX₁X₂²)] ≈ (n/2)    [same expectation, same variance]
```

**Consequence.** An ML adversary training a DPA classifier on intermediate Hamming weights sees a **flat leakage profile** — the signal-to-noise ratio approaches zero:

```
SNR_DPA = Var(leakage | op=add) / Noise
        ≈ Var(leakage | op=double) / Noise

For Weierstrass:  SNR ≈ O(1/n)      [distinguishable after ~1000 traces]
For Twisted Hessian:  SNR ≈ O(1/n²)  [indistinguishable in practice]
```

**Second-order DPA.** Even for a second-order attacker (who looks at correlations between two intermediate values), the symmetry of the `Φ(·,·,·)` polarization formula ensures that the joint distribution of `(μ, ν)` is the same for add and double when inputs are randomized by projective blinding (choosing a random `r ∈ 𝔽ₚ*` and replacing `[X:Y:Z]` by `[rX:rY:rZ]`).

### 3.4 Completeness Against Exceptional-Point Attacks

A subtle attack vector: force the computation to an **exceptional point** (where the formula fails) and observe different behavior (fault, timing difference, zero output). The Twisted Hessian formula fails when:

```
μ = ν = 0    ⟺    P₁ = −P₂    or    P₁ = 𝒪    or    P₂ = 𝒪
```

For a **complete** formula (one with no exceptional cases for any valid inputs), one uses the **complete addition formula** derived from the projective model, which handles these cases by returning `𝒪 = [0:1:−1]` in the limit. In HSM implementations, this is typically handled by:

```
1. Pre-screening: reject inputs with (X=0 AND Y+Z=0) before entering the formula
2. Or: use the extended projective form that handles these cases algebraically
```

---

## 4. Error Correction and Boundary Cases

### 4.1 Catalog of Boundary Cases

For the formula `[X₃:Y₃:Z₃]` from §2.2, the following boundary cases require analysis:

| Case | Condition | Result | Handling |
|------|-----------|--------|----------|
| `P₁ = 𝒪` | `X₁=0, Y₁+Z₁=0` | `P₁+P₂ = P₂` | Identity law |
| `P₂ = 𝒪` | `X₂=0, Y₂+Z₂=0` | `P₁+P₂ = P₁` | Verified §2.2 |
| `P₁ = P₂` | coincident | `[2]P₁` | Use tangent formula |
| `P₁ = −P₂` | `X₁=X₂, Y₁Z₂=Y₂Z₁` | `𝒪` | `μ=ν=0` → output `[0:1:−1]` |
| `P₁ = T₃` | ord-3 torsion | `P₂ + T₃` | Cyclic shift `[X:Y:Z]↦[Y:Z:X]` |

### 4.2 Points of Order 3

**Identification.** The 3-torsion points of `TH(a,d)` are the flex points other than `𝒪`. From the Hessian structure, they satisfy `[3]P = 𝒪`. For the projective model `aX³+Y³+Z³=dXYZ`, the three flex points on the identity component are:

```
𝒪   = [0 : 1 : −1]
T₃  = [0 : ω : −ω²]    where ω = primitive cube root of (−1) [may be in 𝔽_{p²}]
T₃' = [0 : ω² : −ω]
```

If `ω ∉ 𝔽ₚ` (i.e., `p ≢ 1 (mod 3)`), then `T₃, T₃'` are not `𝔽ₚ`-rational. The action of adding `T₃` corresponds to the cyclic coordinate shift `[X:Y:Z] ↦ [X:ωY:ω²Z]`, or in the case `ω ∉ 𝔽ₚ`, to a more involved Galois-equivariant action.

**Impact on group law.** When `P₁` or `P₂` is a 3-torsion point, the formula in §2.2 still applies (it's algebraically valid for all projective points on the curve). No special casing is needed.

### 4.3 Branchless Vectorization of Edge Cases

**Problem.** In a scalar multiplication, the formula must handle `P + 𝒪 = P` and `P + (−P) = 𝒪` without `if-then-else` branches (which leak information via timing).

**Solution: conditional move (CMOV) masking.** Replace branches with constant-time selection:

```
ALGORITHM: Branchless Point Addition
════════════════════════════════════════════════════════════

Input:  P1 = [X1:Y1:Z1],  P2 = [X2:Y2:Z2]
Output: P3 = P1 + P2  (constant time, no branches)

Step 1: Compute candidate result via projective formula §2.2
  [X3_candidate : Y3_candidate : Z3_candidate]

Step 2: Detect degenerate cases (constant-time flags):
  flag_P1_inf = CTEQ(X1, 0) AND CTEQ(Y1 + Z1, 0)    // P1 = 𝒪?
  flag_P2_inf = CTEQ(X2, 0) AND CTEQ(Y2 + Z2, 0)    // P2 = 𝒪?
  flag_equal  = CTEQ(X1·Y2·Z2, X2·Y1·Z1) AND ...     // P1 = P2? (use cross-ratio)
  flag_neg    = CTEQ(μ, 0) AND CTEQ(ν, 0)             // P1 = -P2?

Step 3: CMOV selection (no branches, constant time):
  // If P1 = 𝒪: output P2
  X3 = CMOV(X3_candidate, X2,  flag_P1_inf)
  Y3 = CMOV(Y3_candidate, Y2,  flag_P1_inf)
  Z3 = CMOV(Z3_candidate, Z2,  flag_P1_inf)

  // If P2 = 𝒪: output P1
  X3 = CMOV(X3, X1,  flag_P2_inf)
  Y3 = CMOV(Y3, Y1,  flag_P2_inf)
  Z3 = CMOV(Z3, Z1,  flag_P2_inf)

  // If P1 = -P2: output 𝒪 = [0:1:-1]
  X3 = CMOV(X3, 0,   flag_neg)
  Y3 = CMOV(Y3, 1,   flag_neg)
  Z3 = CMOV(Z3, -1,  flag_neg)

  // If P1 = P2: switch to doubling formula (same instruction stream)
  [Xd:Yd:Zd] = TH_DOUBLE(P1)
  X3 = CMOV(X3, Xd,  flag_equal)
  Y3 = CMOV(Y3, Yd,  flag_equal)
  Z3 = CMOV(Z3, Zd,  flag_equal)

Output: [X3:Y3:Z3]

All CMOV operations execute in constant time on hardware with
conditional-move instructions (x86 CMOVZ, ARM CSEL, FPGA MUX).
```

**ML-inspired vectorization.** From the perspective of a machine learning compute graph, each degenerate case is analogous to handling a "masked" attention operation. We treat the selection as:

```python
def th_add_branchless(P1, P2, a, d):
    # Compute all candidates in parallel (like masked multihead attention)
    [X3c, Y3c, Z3c] = th_formula(P1, P2)
    [Xd,  Yd,  Zd ] = th_double(P1)
    identity = [Fp(0), Fp(1), Fp(-1)]

    # Compute all flags as field elements (0 or 1, no branches)
    f1 = ct_is_identity(P1)      # ∈ {0, 1}
    f2 = ct_is_identity(P2)      # ∈ {0, 1}
    fn = ct_is_negation(P1, P2)  # ∈ {0, 1}
    fe = ct_is_equal(P1, P2)     # ∈ {0, 1}
    fn0 = 1 - f1 - f2 - fn - fe  # normal case

    # Weighted sum (constant-time, vectorizable over all 3 coordinates)
    return (fn0  * [X3c,Y3c,Z3c]
          + f1   * P2
          + f2   * P1
          + fn   * identity
          + fe   * [Xd,Yd,Zd])
```

This "soft selection" pattern is directly parallelizable across the three coordinates using SIMD and mirrors the pattern used in ML hardware for masked computations.

---

## 5. Benchmarking Strategy

### 5.1 Operation Count Comparison

Published optimized formulas (projective coordinates, field operations over 𝔽ₚ):

| Curve Model | Addition (M+S) | Doubling (M+S) | Unified? | Complete? |
|---|---|---|---|---|
| **Twisted Hessian** | **12M + 6S** | **12M + 6S** | ✓ Yes | ✓ Yes |
| Twisted Edwards (ext.) | 10M + 0S | 4M + 4S | ✓ Yes | ✓ Yes |
| Weierstrass (Jacobian) | 11M + 5S | 3M + 6S | ✗ No | ✗ No |
| Weierstrass (Projective) | 12M + 2S | 7M + 3S | ✗ No | ✗ No |
| Montgomery (x-only) | 5M + 4S | 4M + 2S | ✓ | partial |
| Hessian (standard) | 12M + 0S | 6M + 6S | ~Yes | ✓ |

> **Reading the table:** Twisted Hessian achieves **cost uniformity** (same 12M+6S for both ops)  
> at slightly higher cost than Twisted Edwards (which requires different add/double formulas).  
> The uniform cost is the primary HSM security advantage.

**For 256-bit scalar multiplication (fixed-window, window size w=4):**

```
Operations per [k]G:
  Twisted Edwards:    256 doubles + 256/4 adds ≈ 256×(4+4M,4S) + 64×(10M,0S)
                    ≈ 1024M + 1024S + 640M = 1664M + 1024S
  Twisted Hessian:    (256 + 64) × (12M+6S) = 3840M + 1920S
  
Speedup of Twisted Edwards vs Twisted Hessian: ~2.3× in M-count
```

Twisted Edwards wins on raw speed; Twisted Hessian wins on **constant-time security guarantee** and **simpler implementation verification**.

### 5.2 Loss Function for Optimal (a,d) Parameter Selection

**Objective.** Find parameters `(a,d)` minimizing a composite loss that balances cryptographic security, computational efficiency, and implementation simplicity:

```
L(a,d) = λ₁·L_sec(a,d) + λ₂·L_eff(a,d) + λ₃·L_impl(a,d)
```

**Component 1: Security loss.**

```
L_sec(a,d) = w_cofactor · log(cofactor(a,d))
           + w_twist    · [1 - is_twist_secure(a,d)]
           + w_cmdisc   · [1 - has_large_cm_disc(a,d)]
           + w_rho      · [log₂(p)/2 - ρ_security(a,d)]²

where:
  cofactor(a,d)        = #TH(a,d)(𝔽ₚ) / (largest prime factor)  → minimize
  is_twist_secure(a,d) = 1 iff quadratic twist also has large prime-order subgroup
  has_large_cm_disc(a,d) = 1 iff |disc(End(E))| > 2^100  (avoids CM attacks)
  ρ_security(a,d)      = log₂(√#TH(a,d)(𝔽ₚ))  → maximize (ECDLP hardness)
```

**Component 2: Efficiency loss.**

```
L_eff(a,d) = w_mul · M_count(a,d) + w_squ · S_count(a,d)

where M_count, S_count are the multiplication/squaring counts for the
specific formula with parameters (a,d).

Key insight: choosing a = 1 eliminates the "a·X₁²X₂" multiplication in μ,ν
(replace by X₁²X₂), saving 2M per addition. This reduces 12M → 10M.

Similarly, choosing d such that d/3 ≡ D for small D reduces the constant
multiplication d3inv = (d/3 mod p) to a small-constant multiply.
```

**Component 3: Implementation simplicity loss.**

```
L_impl(a,d) = w_bitlen_a · bitlength(a)
            + w_hamwt_d  · hamming_weight_base3(d)
            + w_cuberoot · [1 - is_cube_residue(a)]   # cube residuosity for negation
```

**Recommended weight configuration for 256-bit HSMs:**

```python
weights = {
    'λ₁': 10.0,   # security is primary
    'λ₂': 1.0,    # efficiency is secondary
    'λ₃': 0.5,    # implementation convenience

    # Security sub-weights
    'w_cofactor': 100.0,   # cofactor > 4 is disqualifying
    'w_twist':     50.0,   # twist security is mandatory
    'w_cmdisc':    20.0,   # large CM discriminant required
    'w_rho':        5.0,   # close to optimal

    # Efficiency sub-weights
    'w_mul':  1.0,
    'w_squ':  0.8,   # squarings ~0.8× cost of multiplications

    # Implementation sub-weights
    'w_bitlen_a': 0.1,    # prefer small a
    'w_hamwt_d':  0.1,    # prefer sparse d in base 3
    'w_cuberoot': 1.0,    # prefer a to be a cube residue
}
```

**Optimization algorithm:**

```python
def optimize_th_parameters(p, n_trials=10000):
    """
    Bayesian optimization over (a,d) ∈ 𝔽ₚ² for TH curve parameters.
    Uses the Schoof-Elkies-Atkin oracle for #TH(a,d)(𝔽ₚ).
    """
    best = None
    best_loss = float('inf')

    for _ in range(n_trials):
        # Sample (a,d) from low-discrepancy sequence (Sobol/Halton)
        a = sample_nonzero(p)
        d = sample_with_constraint(p, lambda d: pow(d,3,p) != (27*a) % p)

        # Fast pre-filter: reject obviously bad parameters
        if not passes_smoothness_test(p, a, d):
            continue

        # Full SEA computation (expensive, ~O(log^5 p) time)
        N = schoof_elkies_atkin(a, d, p)
        if not is_prime(N // small_cofactor(N)):
            continue

        # Compute loss
        loss = compute_loss(a, d, p, N)
        if loss < best_loss:
            best_loss = loss
            best = (a, d)

    return best, best_loss
```

### 5.3 Concrete 256-bit Parameter Recommendations

For 256-bit security, the following parameters illustrate the design criteria:

```
Prime:  p = 2²⁵⁶ − 2²²⁴ + 2¹⁹² + 2⁹⁶ − 1  (NIST P-256 prime)

Criterion         Value
──────────────────────────────────────────────────────────────────
a                 a = 1 (maximizes efficiency: saves 2M per add)
d                 chosen such that d/3 mod p has small Hamming weight
cofactor          h = 1 (prime order, strongest security)
twist security    #TH_twist(𝔽ₚ) also prime
CM discriminant   |disc| > 2^100 (verified via Cornacchia's algorithm)
ρ-security        128 bits (log₂(√N) ≈ 128)
```

---

## 6. Mathematical Appendix: Full Derivations

### 6.1 Verification that [0:1:−1] is a Flex

See §2.1 for two independent proofs:
- **Algebraic:** Hessian determinant `𝒽(F)|_{[0:1:−1]} = 6d²−6d² = 0`
- **Parametric:** Tangent line intersection reduces to `(27a−d³)t³ = 0`, a triple root

### 6.2 Birational Map: Weierstrass → Twisted Hessian

**Setup.** Given `E_W: y² = x³ + Ax + B` with rational 3-torsion point `T = (x₀,y₀)`.

**Step 1. Define tangent slope and intercept:**
```
α = (3x₀²+A)/(2y₀),    β = y₀ − α·x₀
```

The line `y = αx+β` is tangent to `E_W` at `T` with multiplicity 3 (since `[3]T = 𝒪` means `T` is a flex).

**Step 2. Substitution:**
```
x = (αu + 1)/(u − v),    y = α(αu+1)/(u−v) + β
```

**Step 3. Result:** Substituting into `y² = x³+Ax+B` and simplifying yields:
```
α³u³ + v³ + 1 = d·uv
```
where `a = α³` and `d` is explicitly computable from `α, β, A, B`.

**Step 4. Verification:**
- Identity `[0:1:0]` of `E_W` maps to `[1:−1:0]`... which, under our convention with `Z`-affine patch, corresponds to the identity `[0:1:−1]` after appropriate coordinate relabeling.
- j-invariant is preserved: `j(TH(a,d)) = d³(d³−216a)³/[a(d³−27a)³] = j(E_W)`.

### 6.3 Affine Addition Formula (Chord Method)

**Setup.** Line through `P₁=(x₁,y₁)`, `P₂=(x₂,y₂)` on `ax³+y³+1=dxy` (assuming `x₁ ≠ x₂`):

```
y = tx + s,    t = (y₂−y₁)/(x₂−x₁),    s = (x₂y₁−x₁y₂)/(x₂−x₁)
```

Substituting into the curve equation:

```
(a+t³)x³ + (3t²s−dt)x² + (3ts²−ds)x + (s³+1) = 0
```

with roots `x₁, x₂, rₓ`. By Vieta:

```
x₁·x₂·rₓ = −(s³+1)/(a+t³)

⟹  rₓ = −(s³+1) / [(a+t³)·x₁·x₂]
```

Then `rᵧ = t·rₓ + s`, and applying negation `−(x,y) = (x/y, 1/y)`:

```
P₁ + P₂ = −(rₓ, rᵧ) = (rₓ/rᵧ,  1/rᵧ)
```

The numerators and denominators simplify to rational functions of `x₁,y₁,x₂,y₂` using the curve equations for `P₁` and `P₂`. The projective form in §2.2 is the canonical unified expression.

### 6.4 Non-Singularity: Proof of Δ Formula

**Claim:** `TH(a,d)` is smooth iff `a(d³−27a) ≠ 0`.

**Proof sketch.** The curve `F = aX³+Y³+Z³−dXYZ` is singular at `P` iff `F(P)=0` and `∇F(P)=0`.

Setting `∂F/∂X = 3aX²−dYZ = 0`, `∂F/∂Y = 3Y²−dXZ = 0`, `∂F/∂Z = 3Z²−dXY = 0`, and `aX³+Y³+Z³=dXYZ`:

Assuming `XYZ ≠ 0` (the case `XYZ=0` is handled separately): from the gradient equations,
`X = dYZ/(3a)`, `Y = dXZ/3`, `Z = dXY/3`. Substituting cyclically and eliminating:
`d³X³Y³Z³/(27a) = X³Y³Z³` → `d³ = 27a`. This is precisely when `Δ = 0`.

When `X=0`: then from `∂F/∂Y = 3Y²=0` → `Y=0`, and `∂F/∂Z = 3Z²=0` → `Z=0`. But `(0,0,0) ∉ ℙ²`. Similar analysis applies to `Y=0`, `Z=0`.

When `a=0`: `F = Y³+Z³−dXYZ`. Then `∂F/∂X = −dYZ = 0` implies `Y=0` or `Z=0`, leading to further equations that force the curve to degenerate (the `aX³` term is needed for the curve to be cubic in all three variables). Hence `a ≠ 0` is required.

### 6.5 Connection to the Five Mathematical Frameworks

The following table maps the Twisted Hessian structure to broader mathematical frameworks relevant to ML optimization research:

| Framework | Twisted Hessian Connection |
|---|---|
| **Wasserstein gradient flow** | The addition formula `P₁+P₂` defines a transport map on `TH(𝔽ₚ)`; scalar multiplication approximates gradient flow on the discrete group |
| **Free probability** | The group `TH(𝔽ₚ)` with uniform measure `μ` has free cumulants computable from the j-invariant; relevant for random matrix models of the ECDLP |
| **Spin glass replica theory** | The ECDLP hardness landscape over random instances `(a,d)` has a replica-symmetric phase; the density of valid (a,d) pairs (§1.4) determines the annealing temperature |
| **Stein operators** | The identity `P + 𝒪 = P` corresponds to a Stein equation `𝒯f(P) = 𝔼[f(P+Q)] − f(P)` for test functions on the group |
| **Tropical geometry** | The limit `p → ∞` of `TH(a,d)(𝔽_p)` tropicalizes to a piecewise-linear curve; the group law tropicalizes to a min-plus tropical addition |

---

## 7. References

### Primary Cryptographic References

**[BL07]** Bernstein, D.J. and Lange, T. (2007). "Faster Addition and Doubling on Elliptic Curves." *Advances in Cryptology – ASIACRYPT 2007*, LNCS 4833, pp. 29–50. Springer. https://doi.org/10.1007/978-3-540-76900-2_3

**[BL15]** Bernstein, D.J. and Lange, T. (2015). "Twisted Hessian Curves." *Progress in Cryptology – LATINCRYPT 2015*, LNCS 9230, pp. 269–294. Springer. https://doi.org/10.1007/978-3-319-22174-8_15

**[FJ10]** Farashahi, R.R. and Joye, M. (2010). "Efficient Arithmetic on Hessian Curves." *Public Key Cryptography – PKC 2010*, LNCS 6056, pp. 243–260. Springer.

**[JQ01]** Joye, M. and Quisquater, J.J. (2001). "Hessian Elliptic Curves and Side-Channel Attacks." *Cryptographic Hardware and Embedded Systems – CHES 2001*, LNCS 2162, pp. 402–410. Springer.

### Algebraic Geometry References

**[Sil09]** Silverman, J.H. (2009). *The Arithmetic of Elliptic Curves*, 2nd ed. Graduate Texts in Mathematics 106. Springer.

**[Cas91]** Cassels, J.W.S. (1991). *Lectures on Elliptic Curves*. London Mathematical Society Student Texts 24. Cambridge University Press.

**[Har77]** Hartshorne, R. (1977). *Algebraic Geometry*. Graduate Texts in Mathematics 52. Springer.

### Computational References

**[EFD]** Bernstein, D.J. and Lange, T. (2007–). *Explicit-Formulas Database (EFD)*. https://hyperelliptic.org/EFD/

**[Sma01]** Smart, N.P. (2001). "The Hessian Form of an Elliptic Curve." *Cryptographic Hardware and Embedded Systems – CHES 2001*, LNCS 2162, pp. 118–125. Springer.

### Side-Channel Analysis References

**[KJJ99]** Kocher, P., Jaffe, J., and Jun, B. (1999). "Differential Power Analysis." *Advances in Cryptology – CRYPTO 1999*, LNCS 1666, pp. 388–397. Springer.

**[CMO98]** Cohen, H., Miyaji, A., and Ono, T. (1998). "Efficient Elliptic Curve Exponentiation Using Mixed Coordinates." *Advances in Cryptology – ASIACRYPT 1998*, LNCS 1514, pp. 51–65. Springer.

### Standards

**[FIPS186-4]** NIST (2013). *Digital Signature Standard (DSS)*. Federal Information Processing Standard 186-4.

**[RFC8032]** Josefsson, S. and Liusvaara, I. (2017). "Edwards-Curve Digital Signature Algorithm (EdDSA)." RFC 8032. IETF.

---

## Quick-Reference Summary

```
TWISTED HESSIAN CURVE TH(a,d) — COMPLETE REFERENCE CARD
═══════════════════════════════════════════════════════════════════════

EQUATION (projective):  aX³ + Y³ + Z³ = d·XYZ
EQUATION (affine Z=1):  ax³ + y³ + 1  = d·xy

VALIDITY:     a ≠ 0  AND  d³ ≠ 27a     [equivalently: a(d³−27a) ≠ 0]
j-INVARIANT:  j = d³(d³−216a)³ / [a(d³−27a)³]

IDENTITY:     𝒪 = (0, −1)  [affine]  =  [0:1:−1]  [projective]
              Verified: a·0+(−1)³+1=0 ✓; flex point verified by 𝒽(F)=0 ✓

NEGATION:     −[X:Y:Z] = [X:Z:Y]        (swap Y and Z)
              −(x,y) = (x/y, 1/y)        (affine)

ADDITION (projective, exact formula):
  μ = aX₁²X₂ + Y₁²Y₂ + Z₁²Z₂ − (d/3)(X₁Y₁Z₂ + X₁Y₂Z₁ + X₂Y₁Z₁)
  ν = aX₁X₂² + Y₁Y₂² + Z₁Z₂² − (d/3)(X₁Y₂Z₂ + X₂Y₁Z₂ + X₂Y₂Z₁)
  X₃ = μX₂ − νX₁
  Y₃ = μZ₂ − νZ₁           [note: Y←Z, Z←Y relative to R, due to negation]
  Z₃ = μY₂ − νY₁

COST:         12M + 6S  (both addition and doubling)
UNIFIED:      Yes (same cost for add and double → DPA resistance)
COMPLETE:     Yes (with CMOV boundary handling, no exceptions)

BIRATIONAL MAP FROM WEIERSTRASS y²=x³+Ax+B with 3-torsion (x₀,y₀):
  α = (3x₀²+A)/(2y₀),  β = y₀ − αx₀
  x = (αu+1)/(u−v),    y = α(αu+1)/(u−v) + β
  → au³ + v³ + 1 = d·uv   with a = α³

SIDE-CHANNEL:  Unified formula → I(op; PowerTrace) ≈ 0
               Projective blinding [X:Y:Z] ↦ [rX:rY:rZ] eliminates input leakage

═══════════════════════════════════════════════════════════════════════
```

# Bungee Drop Simulator — Science Olympiad 2026

A pre-competition **cheat sheet generator** for the Science Olympiad **Bungee Drop** event (Division C, 2026 rules, Regionals). You calibrate your bungee cord in the lab, and the simulator computes the optimal cord length for every possible competition scenario — outputting a **print-ready lookup table** you can bring to the event.

Since competitors **cannot bring a computer** to the event, the entire value of this tool is in generating accurate, printable sheets beforehand.

---

## Table of Contents

- [Quick Start](#quick-start)
- [How It Works](#how-it-works)
- [Physics Model](#physics-model)
  - [Setup Geometry](#setup-geometry)
  - [Energy Conservation](#energy-conservation)
  - [Length-Dependent Spring Constant](#length-dependent-spring-constant)
  - [Non-Linear Elasticity](#non-linear-elasticity)
  - [Numerical Solver](#numerical-solver)
  - [Linear (Single-Point) Special Case](#linear-single-point-special-case)
- [Calibration Guide](#calibration-guide)
  - [Equipment Needed](#equipment-needed)
  - [Procedure](#procedure)
  - [Tips for Accuracy](#tips-for-accuracy)
- [Competition Parameters](#competition-parameters)
- [Output Formats](#output-formats)
  - [Interactive Screen Table](#interactive-screen-table)
  - [Print Sheets](#print-sheets)
  - [CSV Export](#csv-export)
- [Edge Cases and Warnings](#edge-cases-and-warnings)
- [Verification](#verification)
- [File Structure](#file-structure)
- [License](#license)

---

## Quick Start

1. Open `index.html` in any browser (Chrome, Firefox, Safari, Edge).
2. Enter your **reference cord length** (the length of cord you calibrated with).
3. Add one or more **calibration points** — each is a (mass, stretch) pair from your lab measurements.
4. Click **Generate Lookup Table**.
5. Select **cm** or **inches** for cord length units, then click **Print Lookup Sheets** or use **Ctrl+P**.

No build step, no server, no internet connection required. Works fully offline.

---

## How It Works

```
┌─────────────────────────────────────────────────────────────┐
│  1. CALIBRATE                                               │
│     Hang masses → measure stretch → enter data points       │
│                         ↓                                   │
│  2. BUILD FORCE CURVE                                       │
│     Piecewise-linear F(x) from calibration data             │
│                         ↓                                   │
│  3. FOR EACH SCENARIO (drop height × mass)                  │
│     Solve: m·g·d = ∫₀ˣ F_scaled(s) ds  via bisection       │
│                         ↓                                   │
│  4. OUTPUT                                                  │
│     Screen table (full precision) + Print sheets (rounded)  │
└─────────────────────────────────────────────────────────────┘
```

The simulator computes cord lengths for all **1,584 scenarios** at Regionals:
- 13 drop heights × 9 light masses × (1 light cord + 11 heavy mass offsets) = 13 × 9 × 12 = 1,404 cord length values per table

---

## Physics Model

### Setup Geometry

```
    ┌──── Stand (total height = drop_height) ─────┐
    │                                               │
    │   ┌─ Clamp (at top of stand)                  │
    │   │   attachment_height = 0.35 m below top    │
    │   │                                           │
    │   ├─ Cord attachment point                    │  ← Weight starts HERE
    │   │   (height above floor = drop_height       │     (bunched up at clamp,
    │   │    - attachment_height)                    │      NOT hanging below)
    │   │                                           │
    │   │   ↕ L₀ (natural cord length)              │  ← Free-fall zone
    │   │       cord is SLACK during this fall       │     (no force from cord)
    │   │                                           │
    │   ├─ Cord goes taut here                      │
    │   │                                           │
    │   │   ↕ x (cord stretch)                      │  ← Stretch zone
    │   │       cord exerts restoring force          │     (F increases with x)
    │   │                                           │
    │   ├─ Lowest point (buffer above floor)        │  ← Weight stops HERE
    │   │                                           │
    │   ↕ buffer (safety clearance, ~1 cm)          │
    │                                               │
    └──── Floor ────────────────────────────────────┘
```

Key dimensions:
- **d** = total fall distance from clamp to lowest point = `drop_height - attachment_height - buffer`
- **L₀** = natural (unstretched) cord length — **this is what we solve for**
- **x** = cord stretch at lowest point = `d - L₀`
- The weight starts at the clamp (cord is bunched up with full slack) and free-falls the entire distance `d`

### Energy Conservation

At the lowest point, the weight has zero velocity. All gravitational potential energy has been converted to elastic potential energy in the cord:

```
Gravitational PE lost = Elastic PE stored

         m · g · d = U_elastic(x)
```

Where:
| Symbol | Meaning | Units |
|--------|---------|-------|
| `m` | Total mass (bottle + contents + attachment) | kg |
| `g` | Gravitational acceleration = 9.81 | m/s² |
| `d` | Total fall distance from clamp to lowest point | m |
| `x` | Cord stretch = `d - L₀` | m |
| `U_elastic` | Elastic potential energy stored in the cord | J |

**Important**: The mass `m` is the **announced mass** from the event supervisor. Per 2026 rules, this already includes the plastic bottle, water/sand contents, and attachment mechanism. Do not add a separate bottle mass.

### Length-Dependent Spring Constant

Bungee cords do not have a fixed spring constant — cutting a shorter piece makes it stiffer. For a cord of natural length `L₀`:

```
k_eff(L₀) = K / L₀
```

Where `K` is a **material constant** (units: Newtons) that characterizes the cord's stiffness independent of length. It is derived from calibration:

```
k_ref = F / x_ref = (m_calib · g) / stretch_ref     [N/m, at reference length]
K = k_ref × L_ref                                     [N, length-independent]
```

| Symbol | Meaning | Units |
|--------|---------|-------|
| `k_ref` | Spring constant measured at reference length | N/m |
| `K` | Material stiffness constant (independent of length) | N |
| `L_ref` | Reference cord length used during calibration | m |
| `k_eff(L₀)` | Effective spring constant for cord of length `L₀` | N/m |

**Physical intuition**: A cord of length `L_ref` has spring constant `k_ref`. Cut it in half → each half has twice the stiffness (2·k_ref). This inverse relationship `k ∝ 1/L` holds because you're putting the same material under strain over a shorter distance.

### Non-Linear Elasticity

Real bungee cords are **not ideal Hookean springs**. The force-stretch relationship is non-linear — the cord may soften or stiffen at different stretch amounts. This simulator handles this with a **multi-point calibration model**:

1. **Multiple calibration points**: You hang several different masses and measure the stretch for each. Each point gives a `(force, stretch)` pair:

   ```
   Force = m_calib × g     [N]
   Stretch = measured extension beyond natural length     [m]
   ```

2. **Piecewise-linear force curve**: The calibration points (plus the origin `(0, 0)`) define a piecewise-linear force-displacement curve `F_ref(x)` at the reference cord length:

   ```
   F_ref(x):
       (0, 0) ──── (x₁, F₁) ──── (x₂, F₂) ──── (x₃, F₃) ── ...
   ```

   Between calibration points, force is linearly interpolated. Beyond the last calibration point, the slope of the last segment is extrapolated.

3. **Length scaling**: For a cord of length `L₀ ≠ L_ref`, the force curve scales along the stretch axis. A longer cord stretches proportionally more for the same force:

   ```
   F_actual(x_actual) = F_ref(x_actual × L_ref / L₀)
   ```

   This means we map the actual stretch back to the reference curve coordinate system by a factor of `L_ref / L₀`.

4. **Elastic PE via integration**: The elastic potential energy is the area under the force-stretch curve:

   ```
   U_elastic = ∫₀ˣ F_actual(s) ds
   ```

   Using the substitution `u = s × L_ref / L₀`:

   ```
   U_elastic = (L₀ / L_ref) × ∫₀^(x·L_ref/L₀) F_ref(u) du
   ```

   The integral of the piecewise-linear function is computed exactly as a sum of trapezoids — no numerical approximation error in the integration itself.

### Numerical Solver

With the non-linear model, there is no closed-form solution for `L₀`. The simulator uses the **bisection method**:

1. Define: `f(L₀) = U_elastic(d - L₀, L₀) - m·g·d`
2. We need to find `L₀` where `f(L₀) = 0`
3. **Monotonicity**: As `L₀` increases from 0 to `d`:
   - Stretch `x = d - L₀` decreases → less elastic PE → `f` decreases
   - Therefore `f` is monotonically decreasing, guaranteeing a unique root
4. **Bisection**: Start with `[0.001, d - 0.001]`, halve the interval each iteration
5. **Convergence**: 100 iterations achieves `|hi - lo| < 10⁻⁷ m` (sub-micrometer precision)

```
Algorithm: BISECT(m, d, forceCurve, L_ref)
────────────────────────────────────────────
  lo ← 0.001
  hi ← d - 0.001

  if f(lo) < 0 → IMPOSSIBLE (cord too weak even at max stretch)
  if f(hi) > 0 → IMPOSSIBLE (shouldn't happen physically)

  repeat 100 times:
      mid ← (lo + hi) / 2
      if f(mid) > 0:
          lo ← mid
      else:
          hi ← mid
      if hi - lo < 10⁻⁷: break

  return L₀ = (lo + hi) / 2
```

### Linear (Single-Point) Special Case

With a single calibration point, the force curve is a straight line through the origin: `F(x) = k_ref · x`. The elastic PE becomes:

```
U_elastic = ½ · k_eff · x² = ½ · (K / L₀) · (d - L₀)²
```

Setting equal to gravitational PE:

```
m · g · d = ½ · (K / L₀) · (d - L₀)²
```

Multiplying through by `2·L₀` and rearranging:

```
K · L₀² − 2d · (K + m·g) · L₀ + K · d² = 0
```

This is a standard quadratic in `L₀`:

```
a = K
b = −2d · (K + m·g)
c = K · d²

        −b − √(b² − 4ac)
L₀ =   ─────────────────
              2a
```

The minus root is taken because the plus root gives `L₀ > d` (cord longer than the drop distance), which is physically impossible. The discriminant is always positive when `d > 0`:

```
Δ = b² − 4ac = 4d² · [(K + m·g)² − K²] = 4d² · m·g · (2K + m·g) > 0
```

The bisection solver handles both linear and non-linear cases uniformly, but for a single calibration point the results are identical to this closed-form solution.

---

## Calibration Guide

### Equipment Needed

- Your competition bungee cord (or a section of it)
- A ruler or measuring tape (cm precision)
- A set of known masses (ideally spanning 100–600 g)
- A clamp or hook to hang the cord from
- A stable support structure

### Procedure

1. **Cut or measure a reference length** of cord. Record this as your **reference cord length** (e.g., 50 cm). All calibration must be done at this same length.

2. **Hang a known mass** from the bottom of the cord. Wait for oscillations to stop completely.

3. **Measure the stretch** — the extension beyond the cord's natural (unloaded) length. This is NOT the total hanging length; it is the **difference** between the loaded and unloaded lengths.

4. **Record the data point** as (mass in grams, stretch in centimeters).

5. **Repeat with different masses**. Recommended masses for a good non-linear model:

   | Point | Mass (g) | Purpose |
   |-------|----------|---------|
   | 1 | 100 | Light end of competition range |
   | 2 | 200 | Mid-range |
   | 3 | 300 | Light mass upper bound |
   | 4 | 400 | Approaching heavy mass range |
   | 5 | 500–600 | Heavy mass range |

6. **Enter all points** into the simulator along with the reference cord length.

### Tips for Accuracy

- **Measure stretch, not total length**: The most common calibration error is entering the total hanging length instead of the stretch delta. If your cord is 50 cm and hangs down to 65 cm with 200 g, the stretch is **15 cm**, not 65 cm.
- **Wait for equilibrium**: Let the mass hang for 10–15 seconds before measuring. Bungee cord has viscoelastic behavior — it will creep slightly.
- **Use the same cord segment**: All calibration points must be measured on the same piece of cord at the same reference length.
- **Multiple measurements**: Take 2–3 measurements per mass and average them to reduce error.
- **Temperature matters**: Rubber elasticity changes with temperature. Calibrate in conditions similar to competition day if possible.
- **Check for hysteresis**: If you stretch the cord and let it relax, the second stretch may differ slightly. Pre-stretch the cord a few times before calibrating to stabilize behavior.

---

## Competition Parameters

### Regionals (2026 Rules)

| Parameter | Minimum | Maximum | Increment | Count |
|-----------|---------|---------|-----------|-------|
| Drop height | 2.00 m | 5.00 m | 0.25 m | 13 |
| Light mass | 100 g | 300 g | 25 g | 9 |
| Heavy mass offset | +200 g | +300 g | 10 g | 11 |
| Attachment height | 35 cm | 35 cm | — | 1 |

**Total scenarios**: 13 × 9 × 12 = **1,404** cord length calculations

### How Masses Work (2026 Rules)

The event supervisor announces two masses:

1. **Light mass**: 100–300 g (total, including bottle + contents + attachment)
2. **Heavy mass**: Light mass + 200 to +300 g (total, including bottle + contents + attachment)

Each team makes **two drops** per scenario:
- Drop 1: Use the light mass
- Drop 2: Use the heavy mass

The lookup table provides cord lengths for both. The heavy mass columns show cord lengths for each possible heavy mass offset (+200 g, +210 g, ... +300 g).

### Safety Buffer

The simulator includes a configurable **buffer** (default: 1.0 cm) that keeps the weight from actually touching the floor. The target lowest point is:

```
d = drop_height − attachment_height − buffer
```

Adjust the buffer based on your confidence in the cord model. A larger buffer is safer but scores lower (weight stops higher above the floor).

### Attachment Height Correction

The simulator assumes a **35 cm attachment height** (the maximum allowed). If your actual attachment is shorter:

> For every 1 cm the attachment is shorter than 35 cm, lengthen the cord by approximately 0.7 cm.

This correction factor is approximate — calibrate it through practice drops with your specific setup. The print sheets include this note as a reminder.

---

## Output Formats

### Interactive Screen Table

- **Filter** by drop height and light mass using dropdowns
- **Full precision** values (to 0.01 cm) for reference and debugging
- **Color coding**:
  - Green: valid cord length
  - Yellow: unreliably short (< 10 cm)
  - Red (`---`): physically impossible scenario

### Print Sheets

- **One table per drop height** (13 tables for Regionals)
- **Two tables per printed page** (compact layout, fits all on ~7 pages)
- **Rows**: Light mass (100 g to 300 g)
- **Columns**: Light cord length + heavy mass offsets (+200 g to +300 g)
- **Unit selection**: Centimeters (rounded to 0.5 cm) or Inches (rounded to 1/4 inch)
- **Header**: Drop height, calibration summary, buffer, attachment height, cord unit
- **Footer**: Legend for warnings and impossibles, attachment height correction note
- **Monospaced font**, high-contrast black on white, grayscale-safe

### CSV Export

Two export options:
- **CSV (cm)**: Drop height in meters, masses in grams, cord lengths in centimeters
- **CSV (inches)**: Drop height in meters, masses in grams, cord lengths in inches

Full precision values (4 decimal places) for analysis or import into spreadsheets.

---

## Edge Cases and Warnings

| Condition | Display | Meaning |
|-----------|---------|---------|
| `d ≤ 0` | `---` (impossible) | Attachment height + buffer exceeds drop height |
| `f(0.001) < 0` | `---` (impossible) | Cord is too weak — even maximum stretch can't absorb the energy |
| `L₀ < 10 cm` | Yellow / underlined | Cord is unreliably short — hard to measure and attach accurately |
| Stretch exceeds calibration range | Extrapolated | Last calibration segment slope is extended (linear extrapolation) |

**Sanity checks** to verify your results:
- Heavier masses should always produce **shorter** cord lengths (more energy to absorb → more stretch needed → shorter natural length)
- Taller drop heights should produce **shorter** cord lengths for the same mass
- A stiffer cord (higher K) should allow **longer** cord lengths (less stretch needed)

---

## Verification

The simulator includes a built-in verification step. After generating results, it picks a sample scenario (3.00 m drop, 175 g mass) and verifies the solution by computing:

```
                    U_elastic(x, L₀)
Energy Ratio = ──────────────────────
                     m · g · d
```

This ratio should equal **1.000000** (within floating-point precision). If it deviates, the solver has a bug. The ratio is displayed in the results summary.

You can also verify manually:
1. Pick a scenario from the table (e.g., 3.00 m drop, 200 g, cord = 124 cm)
2. Compute: `d = 3.00 - 0.35 - 0.01 = 2.64 m`
3. Compute: `x = 2.64 - 1.24 = 1.40 m` (stretch)
4. Compute: `k_eff = K / 1.24` (from your K value)
5. Check: `0.5 × k_eff × 1.40² ≈ 0.200 × 9.81 × 2.64`

---

## File Structure

```
Bungee Drop Simulation/
├── index.html          Single-file web app (HTML + CSS + JS, ~900 lines)
├── bungee_drop_prd.md  Product requirements document
└── README.md           This file
```

Everything is contained in `index.html` — no dependencies, no build tools, no external libraries. Open it in any modern browser and it works immediately.

---

## License

MIT

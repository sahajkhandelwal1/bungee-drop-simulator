# Bungee Drop Simulator — Science Olympiad 2026

Pre-competition cheat sheet generator for the **Bungee Drop** event (Division C, 2026 rules). Calibrate your bungee cord, and the app computes cord lengths for every possible competition scenario, outputting a **print-ready lookup table** you can bring to the event.

## How It Works

1. **Calibrate** — Hang different masses on your cord and record the stretch. Add multiple calibration points to capture non-linear elasticity.
2. **Generate** — The simulator computes cord lengths for all 13 drop heights × 9 light masses × 11 heavy mass offsets (1,584 scenarios).
3. **Print** — One page per drop height, values rounded to 0.5 cm. Bring the sheets to competition.

## Physics Model

- **Non-linear spring model**: Multiple calibration points build a piecewise-linear force-stretch curve, integrated numerically for elastic potential energy.
- **Length-dependent stiffness**: `k_eff = K / L0` — shorter cord is stiffer. The force curve scales with cord length.
- **Energy conservation**: `m·g·d = ∫₀ˣ F(s) ds` solved via bisection to sub-millimeter precision.
- **Free-fall model**: Weight starts at clamp height and free-falls until the cord goes taut, then stretches.

## Competition Parameters (Regionals)

| Parameter | Range | Step |
|-----------|-------|------|
| Drop height | 2.00 – 5.00 m | 0.25 m |
| Light mass | 100 – 300 g | 25 g |
| Heavy mass offset | +200 – +300 g | 10 g |
| Attachment height | Fixed at 35 cm | — |

## Usage

Open `index.html` in any browser. No build step, no server, works fully offline.

## License

MIT

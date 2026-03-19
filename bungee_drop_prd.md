# PRD: Bungee Drop Science Olympiad Simulator
## For Claude Code — Planning Mode

---

## Project Overview

A local web application (single HTML file or simple React app) that acts as a **pre-competition cheat sheet generator** for the Science Olympiad Bungee Drop event (Division C, 2026 rules). The user inputs their calibrated bungee constants and the app computes the ideal bungee cord length for every possible competition scenario, outputting a **print-ready paper lookup table** they can bring to the event.

The user **cannot bring a computer to the event** — so the entire value of this tool is in generating a clean, accurate, printable sheet beforehand.

---

## Physics Model

### Core Equation
Energy conservation from drop point to lowest point (velocity = 0):

```
m·g·d_total = ½·k·x²
```

Where:
- `d_total` = total drop distance from attachment clamp to floor (meters)
- `x` = bungee stretch = `d_total - attachment_height - L0` (meters)
- `m` = total mass in kg — this is the **announced light or heavy mass directly**, which already includes the bottle, contents, and attachment mechanism per the rules. Do NOT add a separate bottle mass.
- `g` = 9.81 m/s²
- `k` = spring constant (N/m)
- `L0` = natural bungee length (m)

### Solving for L0 (the output)
Given a target lowest point (we want the mass to just graze the floor, so target = `d_total = drop_height`), solve for `L0`:

Let `x = drop_height - attachment_height - L0`, substitute and rearrange into quadratic form:

```
½k·x² + m·g·x - m·g·(drop_height - attachment_height) = 0
```

Solve for x using quadratic formula (take positive root), then:

```
L0 = (drop_height - attachment_height) - x
```

This gives the **maximum safe bungee length** — the cord set to this length will bring the mass exactly to the floor. In practice, apply a small safety buffer (subtract ~1–2 cm from L0) to avoid touching.

### Safety Buffer
The output L0 should be computed at a target clearance of `buffer_cm` above the floor (user-configurable, default 1.0 cm). This means:

```
target_lowest = drop_height - (buffer_cm / 100)
```

---

## Competition Parameters (from 2026 Rules)

### Drop Heights
- **Regionals:** 2.00 to 5.00 m, increments of 0.25 m → 13 values
- **States:** 2.00 to 5.00 m, increments of 0.10 m → 31 values

### Light Mass
- **Regionals:** 100 to 300 g, increments of 25 g → 9 values
- **States:** 100 to 300 g, increments of 10 g → 21 values

### Heavy Mass
- 200 to 300 g **heavier** than light mass, increments of 1.0 g → 101 values per light mass

### Mass Values
The supervisor announces two masses — light and heavy. Per the rules, each mass is the **total** of the plastic bottle + contents + attachment mechanism combined. There is no separate bottle mass to track or add.

- **Light mass:** 100–300 g (the full announced total)
- **Heavy mass:** light mass + 200 to light mass + 300 g (the full announced total)

These values go directly into the physics equation as `m`.

### Attachment Height
- At most 35 cm. **Fixed at 35 cm for all calculations.**
- Sheet includes a correction note: *"For every 1 cm attachment is shorter than 35 cm, lengthen cord by approximately X cm"* — where X is an empirical factor the user fills in after practice testing (suggested range 0.6–0.8 as conservative estimate).

---

## Inputs (User Enters Once Before Printing)

| Parameter | Description | Units | Typical Range |
|-----------|-------------|-------|---------------|
| `k` | Spring constant from static hang calibration | N/m | 50–300 |
| `L0_natural` | Natural (unstretched) bungee length | meters | 0.3–1.5 |
| `buffer_cm` | Safety clearance buffer from floor | cm | 0.5–3.0 |
| `competition_level` | Regionals or States | — | dropdown |
| `attachment_height` | Fixed at 0.35 m | meters | locked at 0.35 |

> **Note:** No bottle mass input. The announced light mass and heavy mass already include the bottle, water, and attachment mechanism per 2026 rules. Use them directly as `m` in the physics equation.

---

## Outputs

### Screen View: Interactive Table
- Full scrollable table of all scenarios
- Columns: Drop Height | Light Mass | Heavy Mass | Total Mass | Computed L0 (cord length to mark)
- Color coding:
  - **Green:** L0 is positive and physically reasonable (cord is long enough to reach)
  - **Yellow:** L0 is very short — may be unreliable near minimum
  - **Red:** Scenario is physically impossible (drop height too short for any cord length — mass would hit ground even with zero cord)
- Filterable by drop height and light mass
- Sortable columns

### Print View: Compact Lookup Sheet
The printed output is the primary deliverable. It must be:
- **One page per drop height** (or compact enough to fit on 2–3 pages total for regionals)
- Organized as a **grid: rows = light mass, columns = heavy mass** (or vice versa — whichever fits better)
- Each cell shows: **L0 in cm** (rounded to nearest 0.5 cm for practical marking)
- Header shows all calibration parameters used
- Footer note about attachment height correction
- Clean monospaced font, high contrast, no unnecessary decoration
- Print button triggers `window.print()` with proper print CSS

### CSV Export
Export the full table as a CSV for reference.

---

## Application Structure

### Single-page app (React preferred, or plain HTML/JS)

**Section 1 — Calibration Inputs**
- Number inputs for k, L0_natural, buffer_cm
- Dropdown for competition level (Regionals / States)
- "Generate" button

**Section 2 — Screen Table (post-generate)**
- Filter controls: drop height selector, light mass selector
- Full scrollable data table
- Export CSV button

**Section 3 — Print Sheet (hidden on screen, shown on print)**
- Auto-generated from the same data
- Clean grid layout per drop height
- Print/Save as PDF button

---

## Key Implementation Notes

### Physics correctness is critical
- Always use the quadratic solve, never a linear approximation
- Clamp L0 to a minimum of 0 (discard negative results as impossible)
- Clearly mark scenarios where the required L0 exceeds the natural length L0_natural (these are fine — it just means you use the full cord)
- Clearly mark scenarios where the required L0 is less than ~10 cm (unreliably short)

### Rounding for practical use
- Display L0 in **centimeters** on the print sheet, not meters
- Round to nearest **0.5 cm** for the print sheet (you can't mark a cord more precisely than that in practice)
- Keep full precision in the screen table for reference

### Two drops per scenario
Each scenario (drop height + light mass) needs **two cord lengths**:
1. **Light mass drop** — uses the announced light mass directly as `m`
2. **Heavy mass drop** — uses the announced heavy mass directly as `m` (only on 2nd drop per rules)

Per the rules, teams choose the best heavy mass value. So for the heavy mass column, show the **optimal heavy mass** (the one that gives L0 closest to but not exceeding the safe maximum) and its corresponding L0.

### Print CSS requirements
```css
@media print {
  .screen-only { display: none; }
  .print-only { display: block; }
  /* One page per drop height using page-break-after */
  /* Monospaced font for table alignment */
  /* No colors — grayscale safe */
}
```

---

## Tech Stack

- **Framework:** React (Vite) or single HTML file with vanilla JS
- **Styling:** Tailwind CSS or plain CSS — clean, utilitarian aesthetic appropriate for a scientific tool
- **No backend needed** — all computation is client-side
- **No external API calls** — fully offline capable after load
- **Print:** Native browser `window.print()` with print CSS

---

## File Structure (if React/Vite)

```
bungee-simulator/
├── index.html
├── src/
│   ├── main.jsx
│   ├── App.jsx
│   ├── physics.js          # pure functions: solveL0(), computeScenario()
│   ├── components/
│   │   ├── InputPanel.jsx
│   │   ├── ScenarioTable.jsx
│   │   ├── PrintSheet.jsx
│   │   └── FilterBar.jsx
│   └── utils/
│       ├── export.js       # CSV export logic
│       └── ranges.js       # competition parameter ranges by level
├── package.json
└── README.md
```

---

## Validation & Edge Cases

- If `k` or `L0_natural` are left blank, show clear error before generating
- If computed L0 > L0_natural: flag as "uses full cord" — still valid, just means no slack
- If computed L0 ≤ 0: flag as "impossible — drop too short" in red
- If drop_height - attachment_height ≤ 0: hard error (attachment taller than drop height)

---

## Success Criteria

1. Physics engine produces L0 values that, when plugged back into the energy equation, yield a lowest point within `buffer_cm` of the floor ✓
2. Print sheet is legible, fits on reasonable number of pages, and requires no computer at competition ✓
3. All 2026 rules parameter ranges are correctly implemented ✓
4. User can go from opening the app to printing a sheet in under 2 minutes ✓

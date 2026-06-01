# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Guidelines

- **Scope:** Household/residential use only — no commercial or industrial scenarios.
- **Language:** UI and all in-app text must remain in Thai. Code comments and this file stay in English. Respond to the user in English unless they explicitly ask otherwise.
- **Design philosophy:** Highly detailed dashboard tailored to the Thai electricity environment (MEA/PEA tariffs, TOU structure, โซลาร์ภาคประชาชน feed-in scheme, personal income tax deduction under พ.ร.ฎ. 805) while keeping the interface approachable and user-friendly — surface complexity progressively, not all at once.
- **Roadmap:**
  - **Phase 1 (current):** Grid-tied system, no battery storage.
  - **Phase 2 (planned):** Add battery storage option — design data structures and UI sections with this extension in mind, but do not implement battery logic until Phase 2 begins.

## Running the App

No build system or server required. Open `index.html` directly in any modern browser (Chrome, Edge, Firefox, Safari). All dependencies are loaded via CDN at runtime:
- **Chart.js** — interactive charts (loaded with `defer` so a slow CDN can never block HTML parsing)
- **Font Awesome 6.4** — icons
- **Plus Jakarta Sans** — Google Fonts typeface

## Architecture

This is a single-file SPA (`index.html`) with embedded CSS (`<style>`) and JS (`<script>`). There is no build step, no modules, and no external files.

### Data layer (top of `<script>`)

| Variable | Purpose |
|---|---|
| `applianceMasterList` | Static list of 18 appliance types with wattage, per-period max hours, and priority slot arrays for each time period. AC units (9k/12k/18k/24k BTU, Inverter + Non-Inverter) carry `isAc: true`; the EV charger carries `isEv: true`. Computers are split into `pc` (desktop/office, 50W) and `gaming_pc` (200W); `dryer` is modeled as a heat-pump dryer (900W) |
| `presetConfigurations` | 5 household profiles (`default`, `wfh`, `office`, `heavy`, `eco`) — each holds weekday and holiday qty/hours maps. Presets need not list every appliance: `loadPresetState()` calls `normalizeApplianceState()` to fill any missing id with safe defaults (qty 0, hours 0), so newly added appliances (e.g. `gaming_pc`) default to off everywhere |
| `cleanSkyCurve` | Hourly solar generation multipliers (hours 6–17 only; all others produce 0) |
| `AC_THERMAL_FACTOR` | Per-period multipliers applied only to `isAc` appliances, modeling that compressor draw tracks outdoor temperature: `p1` night 0.70, `p2` midday 1.05, `p3` evening 0.85. Users still enter a single nameplate wattage |
| `stateWeekdayQty/Hours`, `stateHolidayQty/Hours` | Live mutable state: qty and hours-per-period for each appliance, for weekday and holiday schedules separately |

### Recalculation trigger (manual)

To stay responsive on mobile, recalculation is **not** run on every input event. A floating "คำนวณผลลัพธ์" button (`#calcBtn`, calls `runCalculation()`) drives the heavy pass. Editing actions (appliance +/- via `adjustQty`/`adjustHours`, sliders, typed number inputs) instead call `markResultsStale()`, which flags `resultsAreStale` and pulses the button. Discrete view/load actions — preset buttons, the tariff-type select, and the weekday/holiday chart toggle — call `runCalculation()` immediately. The initial load defers the first pass via `setTimeout(runCalculation, 0)` so layout paints first — **not** `requestAnimationFrame`, because rAF is paused while the tab is hidden/backgrounded and would leave the dashboard blank until interaction.

### Calculation pipeline

1. **`build24HourLoadProfile(qtyMap, hoursMap)`** — maps appliance usage into a 24-element watt array using priority slot arrays (not contiguous blocks), producing a realistic daily load curve. AC watts are scaled by `AC_THERMAL_FACTOR` per period.
2. **`calculateSolarPayback()`** — main orchestrator:
   - Reads all form inputs
   - Applies VAT 7% to tariff rates for self-consumption savings (`(base + Ft) * 1.07`)
   - Simulates 365 days × 24 hours: computes `genKwh` vs `demandKwh` per hour, accumulates self-consumption savings and export revenue
   - Computes payback via year-by-year cumulative balance (capped at 10 years), ROI, and 10-year IRR
   - Writes results directly to DOM element IDs. The สรุปผล summary and the detailed annual table both show an explicit maintenance line (`tblMaintenance` / `tdMaintenanceCash`) so the displayed total reconciles: bill savings + export revenue − maintenance = net annual benefit (`netAnnualBenefit`)
3. **`calculateIRR(cashFlows)`** — bisection method, 100 iterations, returns `null` if no sign change.
4. **`buildInteractiveLoadCharts()`** / **`buildInteractivePaybackTrends()`** — create the Chart.js instances on first run, then **update them in place** (mutate `data`, call `chart.update()`) on subsequent recalculations rather than destroying/recreating, which keeps recalculation fast.

### Thai regulatory constants (hardcoded)

- `TOU_WEEKDAY_DAYS = 242`, `TOU_HOLIDAY_DAYS = 123` (MEA/PEA official schedule)
- Export feed-in tariff: **2.20 THB/kWh** (no VAT on export revenue)
- Tax deduction ceiling: **200,000 THB** per Royal Decree 805 (พ.ร.ฎ. 805)
- Panel degradation: **0.5%/year**
- System losses: **lossFactor = 0.80**

### TOU vs Normal meter mode

`toggleTariffInputs()` shows/hides the TOU rate inputs and the holiday appliance matrix. When Normal (flat-rate) mode is active, `profileHoliday` is set equal to `profileWeekday` and the holiday matrix is hidden.

### Layout

Two-column `dashboard-grid`. On `DOMContentLoaded`, `moveRoiDashboardToPrimaryPosition()` programmatically prepends the results card to the left column, so the ROI dashboard appears first despite being defined last in the HTML source.

## Known Issue

A Kaspersky browser-extension script tag (`gc.kis.v2.scr.kaspersky-labs.com/...`) can be auto-injected into `<head>` when the file is opened/saved with Kaspersky active. It is **not** harmless: as a synchronous head script on an unreliable host it blocked HTML parsing for ~21s in testing (measured via the Navigation Timing API). It has been removed; if it reappears, delete the `<script src="https://gc.kis...">` tag.

## License

Creative Commons CC BY-NC-SA 4.0 — non-commercial use only; attribution required.

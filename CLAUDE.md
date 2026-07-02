# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

לוח ניהול עסקי לקליניקה — Business management dashboard for Noam Bar-el Azriel, clinical dietitian. Single-file static HTML app with no build step, no frameworks, no external dependencies.

**GitHub:** https://github.com/noambarel2907-png/Business-Management-Dashbord  
**Live site:** Cloudflare Pages — auto-deploys from `main` branch via `index.html`

---

## File Structure

```
Business managment dashbord.html   ← the only source file; edit this
index.html                         ← identical copy for Cloudflare Pages (must be kept in sync)
```

After every edit, always run:
```
cp "Business managment dashbord.html" index.html
git add "Business managment dashbord.html" index.html
git commit -m "..."
git push
```

---

## Architecture

Everything lives inside a single HTML file (~2100 lines), structured as:

1. **CSS** (lines ~1–83) — utility classes only, no component-specific styles elsewhere
2. **HTML tabs** — 5 tabs rendered with `display:none` toggling: `tab-clients`, `tab-sim`, `tab-cf`, `tab-exp`, `tab-summary`
3. **JS sections** (labeled with `╔══ SECTION N ══╗` comments):
   - Section 1: Constants & config (`TGT`, `WORK_DAYS`, `EXEMPT_CEILING`)
   - Section 2: Prices (`DP` defaults, `P` live, stored in `nb_prices`)
   - Section 3: Client list (`CL`, stored in `nb_clients`; `DEFAULT_CL` is the seed data)
   - Tax/BL functions: `calcTax(annualGross)`, `calcBL(monthlyGross)`, `g2n()` (theoretical), `g2nReal()` (real cash)
   - Section 4+: Simulator (`payState`, `upd()`, `calcGrowFee()`)
   - Monthly summaries: `MONTH_HISTORY` (`nb_month_history`), `closeMonth()`, `renderSummary()`, `renderMonthDetail()`

---

## Key Concepts

### Business month period
The billing cycle runs from the **2nd of month M to the 1st of month M+1** (not calendar months).
```js
periodKeyForDate(d)   // returns "YYYY-MM" for the period containing date d
getCurrentPeriodKey() // current period
periodRangeLabel(key) // "2/6-1/7" display label
```

### Two net calculation modes
- `g2n(g)` — theoretical tax (income tax + BL at full rate) — used only in the cumulative tax-gap card
- `g2nReal(g)` — real cash net (subtracts actual monthly advances: `ADV_TAX` + `ADV_BL`) — used everywhere else (sim, cashflow, expenses, summaries)
- `requiredGrossForTargetReal(target)` — binary search inverse of g2nReal (used for "ברוטו נדרש ליעד")

### GROW fee calculation (`calcGrowFee`)
- Services (Club `scl`, follow-up `smf`): **1 ₪** per unit (receipt only)
- Digital products (PREMIUM `spr`, VIP `svi`, BASIC `sba`, KITCHEN `skitchen`, MomRelief `smomrelief`, CUSTOM_PRODUCTS): **2 ₪** per unit (receipt + invoice)
- Split payments (תשלומים): `rate%` commission on full price + 1 ₪ per individual payment

### Tax advances
```js
let ADV_TAX = loadAdvance('tax', 0);   // income tax advance — currently 0
let ADV_BL  = loadAdvance('bl', 762);  // national insurance advance — currently 762 ₪/month
```
Stored in `nb_adv_tax` / `nb_adv_bl`. Set in the הוצאות tab.

### Slider state persistence
`saveSimState()` / `applySimState()` → `nb_sim_state`. Must be called after any slider UI change.  
`refreshRangeFills()` must be called after values change to repaint the brown fill on sliders.

### Month history (MONTH_HISTORY)
Each entry shape:
```js
{ key, label, open, closedAt, income: { items, manual, total }, expenses: { fixed, annual, variable, tax, grow, total }, net }
```
`closeMonth(periodKey)` snapshots current `computeMonthlyIncomeBreakdown()` + `computeMonthlyExpenseBreakdown()` into `MONTH_HISTORY`.  
`editMonthEntry(key)` / `saveMonthEdits(key)` allow reopening closed months for income edits.

### עוסק פטור ceiling
```js
const EXEMPT_CEILING = 120000; // stored in nb_exempt_ceiling
yearToDateIncome(year)          // sums MONTH_HISTORY + current open period
renderExemptStatus()            // updates banner + progress bar
```

---

## localStorage Keys

| Key | Contents |
|-----|----------|
| `nb_clients` | `{ clients: CL[], nextId }` |
| `nb_prices` | overrides for `DP` price defaults |
| `nb_sim_state` | slider values + payState |
| `nb_adv_tax` / `nb_adv_bl` | monthly tax advances |
| `nb_fixed_custom` | user-added fixed expenses |
| `nb_dis_fixed` | disabled fixed expense names |
| `nb_annual` | annual expenses array |
| `nb_month_history` | closed month snapshots |
| `nb_manual_items` | manual income line items |
| `nb_tgt` | monthly net target |
| `nb_exempt_ceiling` | עוסק פטור ceiling (default 120000) |

---

## UI Patterns

### CSS utility classes
- `.mc` — metric card (beige bg)
- `.ml` / `.mv` / `.ms` — metric label / value / subtitle
- `.ph` — panel header (flex, space-between)
- `.st` — section title (uppercase, small)
- `.br` — row with left-aligned label + right-aligned bold value
- `.sr` — slider row (label + range + value)
- `.dv` — divider line
- `.g2` / `.g3` / `.g4` — 2/3/4-column grids
- `.btn-primary` / `.btn-sec` / `.cbtn` — button variants

### Modals / collapsibles
```js
toggleForm(id)          // toggles .hidden class on add-forms
toggleCardSection(id)   // show/hide collapsible card body, toggles ▸/▾ arrow
toggleAccordion(id)     // payment breakdown accordion per track
```

### PDF export
Uses hidden iframe with `srcdoc` to avoid popup blockers:
```js
printHTML(html)   // injects full HTML into iframe, calls contentWindow.print()
```

### Colors
```js
const TC = { preg:'#F2A971', rise:'#8ECAE6', steady:'#95D5B2', club:'#8B2DC4' };
```
Status: green `#1a7a3a`, red `#c0392b`, warning `#b5770d`. Background `#F5F0E8`, primary `#baa893`.

---

## Tracks / Products Reference

| ID | Name | Price | Type |
|----|------|-------|------|
| `preg` | Pregnancy Glow Path | 3,250 ₪ | ליווי 1:1 |
| `rise` | RISE path | 2,270 ₪ | ליווי 1:1 |
| `steady` | STEADY path | 1,950 ₪ | ליווי 1:1 |
| `club` (`scl`) | CheckMom CLUB | 990 ₪ | קבוצה |
| `spr` | תיק PREMIUM | 547 ₪ | מוצר דיגיטלי |
| `svi` | תיק VIP | 297 ₪ | מוצר דיגיטלי |
| `sba` | תיק BASIC | 247 ₪ | מוצר דיגיטלי |
| `skitchen` | Quick Check KITCHEN | 59.90 ₪ | מוצר דיגיטלי |
| `smomrelief` | MomRelief | 39.90 ₪ | מוצר דיגיטלי |
| `ssingle` | פגישה חד פעמית | 450 ₪ | שירות |
| `smf` | פגישת מעקב | 300 ₪ | שירות |

# How the data was made, and how the prototype works

A short companion to the submission. Part 1 explains the synthetic dataset and why it is built the way it is; Part 2 explains what the browser prototype actually does when you move a price.

---

## Part 1 — The synthetic dataset

### Why synthetic

Real pricing data cannot prove a pricing method works. You never learn what would have happened at the other price, so you can measure forecast accuracy but never whether the *price response* was right — which is the only thing a pricing model is for. Generating the data from a known demand curve means every store × item has a true elasticity on file, and the method can be marked against it. That file (`data/validation/ground_truth_elasticities.csv`, 168 rows) is read only by the evaluation code; a test enforces that no model or optimiser touches it.

Everything comes from `scripts/generate_data.py`, seeded at `20261028`, so the whole dataset rebuilds identically.

### Shape

12 stores across 9 Indian cities × 14 menu items × 104 weeks (4 Nov 2024 → 26 Oct 2026\) \= **17,472 weekly rows**, plus stores, products, competitors, a festival calendar, city weather and a log of price events.

### The hidden truth

Each pair's true elasticity is built, not drawn at random:

```
e₀ = category base × location-type multiplier × (1 + 0.02·(competitor density − 10)) × U(0.9, 1.1)
```

clipped to \[0.45, 3.2\]. Category bases run from Sides 1.10 to Combos 1.85; location multipliers from Airport 0.55 to College 1.30. A captive airport customer is genuinely insensitive; a college customer with five rivals nearby is not. The model has to rediscover this structure from sales alone.

### Demand

```
log(orders) = log(base volume) + events + trend + temperature + Navratri
              − k·(net price ÷ competitor price) + k·(P₀/C₀)
              + 0.06·promo + N(0, 0.07)
orders ~ Poisson(exp(log orders))
```

with `k = e₀ · (C₀/P₀)`, so elasticity at the starting price is exactly `−e₀`. Price enters as a **ratio to the local competitor price**, not in logs. That choice matters: a log-log curve says "raise forever" whenever \|e\| \< 1, while this form gives a finite profit optimum and makes sensitivity stiffen as you price above rivals. Poisson draws mean small stores are genuinely noisier than large ones.

### Where the price variation comes from

A model can only learn price response if prices actually moved, so the generator moves them five ways: an April 2025 chain-wide revision (+4–7%); a January 2026 cost-led revision on cheese and chicken items (+3–8%); two 8-week price tests per store at ±6–10%; promotions (Combo Fest 15%, Summer Cooler 20%, manager discounts of 10–25% in \~7% of weeks, festive app coupons 10%); and competitors who raise prices about once a year and run short promos in \~5% of weeks.

### The confounders, deliberately

Office stores empty out at Diwali and Christmas; college stores dip during exams and summer; the Lonavala highway store spikes 35% in monsoon; malls peak in festive season. Non-veg demand in North-India stores falls \~28% during Navratri. Beverages track temperature; rain and IPL push orders toward delivery. Food cost inflates 5.5% a year with a \+12% dairy spike on cheese items in mid-2025. These overlap on purpose — Diwali sits inside festive season and coincides with coupons — so naive before/after comparisons give the wrong answer and multivariate controls are required.

### The one thing it is kind about

Prices are **exogenous by construction**: they move on a schedule, never in response to a demand shock. That makes ordinary regression unbiased here. On real data managers reprice *because* demand moved, and the same regression would be biased. This is the single largest gap between this prototype and production, and `docs/RESEARCH_NOTES.md` §R4 sets out the fixes: randomised store tests, cost-shock instruments, difference-in- differences around menu revisions, or double/debiased ML.

---

## Part 2 — How the prototype works

### The chain from CSV to browser

```
data/raw/*.csv
  → pricing train        validate (20 checks) → cross-validate 4 models → fit → recommend → stress-test
  → artifacts/           model_state.json: 168 fitted pair states (k, SE, baseline demand, unit economics)
  → build_prototype.py   payload.json: those parameters + 104 weeks of history + CV and scenario results
  → price_desk.html      one self-contained file: payload + the scoring engine in JavaScript
```

The result is \~620 KB, opens offline, and needs no server. Nothing is pre-rendered: the page holds the fitted model, not a picture of its output.

### What happens when you move a price

The JavaScript mirrors `economics.simulate` and `optimizer.recommend` line for line:

```
orders       q = q₀ · exp(−k·(net/C − P₀/C₀) + promo lift)
profit/order = net − food cost − delivery share·(packaging + commission·net)
contribution = q × profit per order
```

For a recommendation it scores every menu price ending in 9 inside the guardrails, then draws the slope 500 times from its uncertainty and keeps only candidates that beat holding today's price in ≥80% of draws, stay under the premium cap, lose no more orders than allowed and add at least 1% contribution. Otherwise: hold. So the slider is not a lookup of pre-computed answers — it re-runs the economics in the browser, which is why the risk view, the frontier and the plan ledger all respond to guardrail changes.

### The three pages

- **The plan** — 168 pairs ranked by weekly value. Tick what you would publish; the ledger re-totals live. The price simulator shows orders, revenue, profit and profit per order against today, in rupees and per cent.  
- **Risk** — uncertainty range for the plan you selected, driver tornado, competitor-reaction curve, the guardrail frontier (re-running the optimiser at each limit) and a store × item heatmap.  
- **Model & evidence** — estimated vs true elasticity, cross-validation by fold, sensitivity by location type.

### Proving the browser copy is honest

A re-implementation can drift from the Python silently. `prototype/check_parity_desk.js` lifts the engine out of the JavaScript, runs it headlessly against the trained artifacts and compares all 168 recommendations with what `optimizer.recommend` produced:

```
agree 168 / 168
weekly uplift  python 63534   js 63534   diff 0.22
```

Same actions, same prices, total weekly uplift matching to within ₹0.22. The prototype is the model, not a mock-up of it.
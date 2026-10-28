# Research notes — restaurant pricing

The working log behind the project: what was built and in what order, how the synthetic data was designed, what the analysis found, which modelling decisions were made and why, and a prioritised roadmap for further research. Keep this file up to date whenever you change the method or the results.

---

## 1\. How the solution was built (history)

1. **Problem framing.** The decision is the menu price of one item in one store, and whether to discount it. The objective is weekly contribution. The thing to learn is how price-sensitive each store × item is, net of everything else that moves demand.  
2. **Synthetic data** (`scripts/generate_data.py`). 12 stores, 14 items and 104 weeks (4 Nov 2024 → 26 Oct 2026), with a known demand model so that estimates can be checked against the truth (§2).  
3. **EDA** (`notebooks/01_EDA_pricing.*`). Data health checks, the business overview, demand drivers, the pricing landscape, and a first look at price response from promotions and price tests (§3).  
4. **First prototype.** Per-pair log-linear regression plus partial pooling, and an HTML simulator.  
5. **Deployable engine** (this repo). Data validation, four models with rolling-origin CV, bootstrap uncertainty, a risk-aware optimiser, sensitivity analysis, the API, the dashboard, tests, Docker and an auto-generated report.  
6. **Prototype rebuilt on the final engine.** The JS reproduces the Python recommendations exactly (168/168).

## 2\. Synthetic data design

Know this so you **don't overfit the method to the generator**. Real data will be messier.

- **Demand:** `log(orders) = log(base_vol) + events(store type) + trend + temp effect + Navratri effect − k·(net_price / competitor_price) + k·launch_ratio + 0.06·promo + N(0, 0.07)`. Orders are then drawn as Poisson.  
- **True elasticity at launch** \= base elasticity for the category × a store-type multiplier × (1 \+ 0.02·(competitor density − 10)) × U(0.9, 1.1), clipped to \[0.45, 3.2\].  
  - Category base values: Sides 1.10, Burgers 1.30, Wraps 1.40, Pizza 1.50, Desserts 1.60, Beverages 1.80, Combos 1.85.  
  - Store-type multipliers: Airport 0.55, Highway 0.70, Office 0.85, Mall 0.95, High Street 1.00, Residential 1.10, College 1.30.  
- **Price variation:**  
  - an April 2025 menu revision (+4–7%);  
  - a January 2026 cost-led revision on cheese and chicken items (+3–8%);  
  - two 8-week price tests per store (±6–10%);  
  - promotions: Combo Fest 15%, Summer Cooler 20%, local discounts of 10–25% in \~7% of weeks, and festive app coupons of 10%;  
  - competitors raise prices about once a year (+3–7%) and run short promos (−8 to −18%) in \~5% of weeks.  
- **Events by store type:**  
  - Office: Diwali −35%, Christmas −30%.  
  - College: exams −18%, summer −30%.  
  - Mall: festive \+18%.  
  - Residential: IPL \+10%.  
  - Highway: monsoon \+35%.  
  - Airport: summer, Diwali and Christmas up.  
  - Chain-wide: festive \+12%.  
  - Non-veg items in North-India stores fall \~28% during Navratri.  
- **Costs:**  
  - food cost is 18–37% of the base price, rising 5.5% a year, with \+12% on cheese items from June to August 2025;  
  - packaging is ₹6–30 per delivery order;  
  - commission is 20–25%;  
  - delivery share runs from 0% (airport) to 62% (residential), and rises with rain and during IPL.  
- **Prices are exogenous by construction.** They don't respond to demand shocks, so OLS is unbiased here. **This will not hold in real data** (see R4).

## 3\. EDA findings (from data only)

- The data is clean: 20/20 checks pass, and only 0.5% of weeks are outliers (festival spikes, kept).  
- Delivery orders earn about ₹71 of contribution against ₹116 for dine-in, so the value of the same price move depends on each store's channel mix.  
- One-at-a-time event comparisons are **confounded**, because Diwali overlaps the festive season and coupons, and IPL overlaps exams. Multivariate controls are needed.  
- Promotions: orders rise \+32% at 10% off and \+52% at 25% off, but contribution goes from \+12% to −6%. 39% of promo weeks earned less contribution than nearby normal weeks. Local discounts (with random timing) are the cleanest signal.  
- The 24 price tests point in the expected direction but are noisy, because their before/after windows straddle seasons.

## 4\. Modelling decisions and why

| Decision | Alternatives considered | Why this choice |
| :---- | :---- | :---- |
| Semi-log demand in **relative price** (`rel = P/C`) | Constant elasticity (log-log); linear | Log-log implies "raise forever" when \|e\| \< 1\. The semi-log form gives a finite optimum, an elasticity that rises as we price above rivals, and a cross-price effect for free |
| One model per store × item | One global model | Price sensitivity varies about 3× by store type; per-pair models capture it directly |
| Empirical-Bayes pooling of the slope | Pure per-pair; full hierarchical Bayes | Cut elasticity MAE from 0.19 to 0.12. Cheap and transparent. Full HB is R5 |
| Block-bootstrap SE (4-week blocks) | HC1 only | Weekly residuals are autocorrelated; HC1 understates uncertainty |
| Rolling-origin CV, 4 × 12 weeks | Random k-fold | Random folds leak future seasonality; rolling-origin mimics real use |
| Score on price-change weeks and elasticity stability, not just WAPE | WAPE only | GBM had near-best WAPE but poor price response (corr 0.62). Forecast accuracy ≠ pricing accuracy |
| Risk rule: P(gain) ≥ 80% over 500 draws | Point estimate only | Avoids moves whose expected gain disappears under plausible estimation error |
| Menu prices ending in 9, ±10% default | Continuous | Matches how menus are actually set; limits extrapolation |
| Discount analysis only at 10–25% | 5–30% | Only 10–25% was seen in history. At 5%, the promo-visibility lift (a constant) made tiny discounts look free, which is an artefact |

## 5\. Research roadmap (prioritised)

Each item gives the goal, a suggested approach, how to validate it, and the files to touch. To validate new effects, first **extend the generator** so a ground truth exists (e.g. true cross-elasticities), then check that the method recovers it.

### R1 · Menu price-architecture constraints (high value, low effort)

- **Goal:** recommendations a menu team can actually publish.

- **Approach:** add constraints to the optimiser:

  - combo price ≤ (1 − min bundle discount) × the sum of its parts;  
  - size and quality ladders (Peri-Peri Fries \> Salted Fries);  
  - a maximum spread for the same item across stores in one city (customers compare on delivery apps).

  Solve jointly per store or city with coordinate descent over candidate grids, or as a small MILP (`pulp` / `OR-Tools`).

- **Validate:** no violated constraints; report the contribution "cost" of each constraint.

- **Files:** `optimizer.py` (new `recommend_store()`), `config.yaml` (constraint table), tests, prototype.

### R2 · Cross-item effects: cannibalisation and attach (high value)

- **Goal:** stop recommending a burger price that quietly erodes combo sales.

- **Approach:**

  - (a) Add the log relative prices of close substitutes (same category; combo ↔ its parts) to each pair's regression, with pooled cross-elasticities.  
  - (b) Or use a store-level **multinomial / nested logit** over menu items with an outside option.  
  - (c) Or use SUR / a demand system (AIDS) per category.

  Then optimise prices jointly by store.

- **Validate:** extend the generator with true cross-elasticities, recover them, and show the joint optimum beats pair-by-pair.

- **References:** Train, *Discrete Choice Methods with Simulation*; Berry (1994); Deaton & Muellbauer (1980), AIDS.

### R3 · Channel-specific pricing (dine-in vs delivery apps)

- **Goal:** optimise the delivery-app markup separately, since commission and packaging make delivery orders much less profitable.  
- **Approach:** model dine-in and delivery orders separately (two slopes per pair, pooled together), and add a delivery price variable. Platform rules may limit markups, so make the cap configurable.  
- **Data needed:** separate delivery menu prices, and ideally aggregator funnel data (views → orders).

### R4 · Causal identification on real data (essential before go-live)

- **Problem:** in real data, managers set prices *in response to* demand (endogeneity), so OLS slopes can be biased.  
- **Approaches:**  
  - **Experiments first:** store-level randomised or matched-pair price tests (see R8).  
  - **Instruments:** food-cost shocks (cost-push pass-through); Hausman-style prices of the same item in other cities.  
  - **Double/debiased ML** (`econml` DML / CausalForestDML) with the controls as nuisance functions. This also gives heterogeneous effects by store attributes.  
  - **Difference-in-differences** around menu revisions, with staggered timing.  
- **References:**  
  - Chernozhukov et al. (2018), *Double/debiased machine learning*, Econometrics Journal.  
  - Angrist & Pischke, *Mostly Harmless Econometrics*.  
  - Athey & Imbens (2017), *The state of applied econometrics*, JEP.

### R5 · Full hierarchical Bayesian model

- **Goal:** pool *all* coefficients (seasonality, promo lift, slope) across stores and items; get proper posterior predictive uncertainty; handle cold start for new stores and items.  
- **Approach:** PyMC or NumPyro, with random effects for store type, category and pair. Use non-centred parameterisation and check with LOO-CV.  
- **Validate:** compare MAE vs truth, 90% coverage and CV WAPE against the EB model.  
- **Reference:** Rossi, Allenby & McCulloch, *Bayesian Statistics and Marketing*; Gelman et al., *Bayesian Data Analysis*.

### R6 · Competitor reaction model

- **Goal:** replace the manual "rivals copy x%" assumption with an estimate.  
- **Approach:** a reaction function `Δlog C_t = α + β·Δlog P_{t−l} + seasonality` from competitor price history; optionally a simple Bertrand best-response simulation.  
- **Data needed:** a regular competitor price feed (scraped menus or a field audit).

### R7 · Dynamic effects

- Reference-price effects (customers anchor on past prices; gains and losses can be asymmetric), promo wear-out, and post-promo dips.  
- **Approach:** add lagged price and promo terms, or a reference price as an exponential smoothing of past prices.  
- **Reference:** Kalyanaram & Winer (1995), reference prices; Briesch et al. (1997).

### R8 · Experimentation and continuous learning

- **Validation test:** use matched-pair stores (by type, size and city tier). Apply high-confidence recommendations to half of them for 8 weeks and measure the effect with difference-in-differences. Do a power calculation using the CV residual SD (\~9% WAPE).  
- **Continuous learning:** Thompson sampling or a Bayesian-optimisation bandit over the candidate price grid, using the posterior for k as the prior. This keeps learning safely within the guardrails.  
- **References:** Kohavi, Tang & Xu, *Trustworthy Online Controlled Experiments*; Abadie (2021), synthetic control, JEL; den Boer (2015), *Dynamic pricing and learning*, survey.

### R9 · Robust and portfolio optimisation

- Optimise CVaR or the 10th-percentile gain instead of the mean.  
- Add chain-level constraints, such as chain orders not falling more than x% or the brand price index staying within a band.  
- Solve as a knapsack-style selection over the per-pair candidate set.

### R10 · Production MLOps

- **Scheduling:** weekly data refresh, then `pricing train --no-cv`, with a monthly full CV.  
- **Model registry:** MLflow, or the existing `metadata.json` plus a data hash.  
- **Drift monitoring:** WAPE by pair, stability of the elasticity estimates, the competitor price index.  
- **Security:** API auth (OAuth or API keys).  
- **Governance:** an approval workflow (commercial manager sign-off) and an audit log of every published price.

### R11 · Real-data onboarding

- Map POS and aggregator exports to the `data/raw` schema (`data.REQUIRED`). Handle item renames, new items, temporary closures and partial weeks.  
- **Cold start:** use the category × store-type prior from the pooled model (`prior_coef` in `metadata.json`).  
- Rerun the EDA notebook on real data before trusting any elasticities.

## 6\. Evaluation protocol for any new model

1. Rolling-origin CV with the same folds as `cross_validation` in the config. Report WAPE, bias, WAPE in price-change weeks and in promo weeks, and the elasticity SD across folds.  
2. On synthetic data, report correlation and MAE vs the true elasticity, plus 90% interval coverage (target ≈ 90%).  
3. Re-run the portfolio: uplift, number of raises/cuts/holds, and the Monte Carlo range. Explain any change over 10%.  
4. Keep the parity of the prototype if production logic changed.

## 7\. Open business questions

- Will the commercial team accept ±10% moves? Is there a brand rule about relative price position vs competitors?  
- Can delivery-app prices differ from dine-in? By how much, and is it allowed by contract?  
- How often can menus be reprinted, and what does a menu change cost?  
- Are any stores franchised with independent pricing rights?  
- What competitor data is available and how often (weekly field audit, scraped app menus)?  
- Which KPI does leadership track: contribution, EBITDA, or same-store sales growth? The objective may need a volume floor.

## 8\. Glossary

- **Elasticity:** the % change in orders for a 1% price change. −0.6 \= insensitive; −2.0 \= very sensitive.  
- **Contribution:** revenue minus variable costs (food, packaging, commission). Excludes rent and labour.  
- **WAPE:** Σ\|forecast − actual\| / Σ actual.  
- **Rolling-origin CV:** train on weeks \< t, test on t … t+11; repeat for several t.  
- **Empirical Bayes pooling:** a precision-weighted average of a pair's own estimate and a prior learned from similar pairs.  
- **Pass-through:** the change in the optimal price per ₹1 change in unit cost.

## 9\. Further reading (starting points)

- Phillips, R. — *Pricing and Revenue Optimization* (Stanford, 2nd ed.). Price-response functions and optimisation.  
- Talluri, K. & van Ryzin, G. — *The Theory and Practice of Revenue Management*.  
- Rossi, Allenby & McCulloch — *Bayesian Statistics and Marketing*. Hierarchical demand models.  
- Hyndman & Athanasopoulos — *Forecasting: Principles and Practice* (free online). Time-series CV.  
- Chernozhukov et al. (2018) — Double/debiased ML. `econml`, `doubleml` packages.  
- Kohavi, Tang & Xu — *Trustworthy Online Controlled Experiments*.  
- Tools: `statsmodels`, `linearmodels` (IV), `econml`, `pymc` / `numpyro`, `pulp` / `ortools`, `mlflow`.
# Restaurant Pricing Engine

**What price will deliver better business results?** For every store × menu item, this project measures how
customers respond to price, simulates the effect of any price, discount or competitor move on **orders, revenue and
contribution**, and recommends a price within business guardrails. Each recommendation comes with an uncertainty range
and a sensitivity analysis.

```
data (CSV) ──► validate ──► features ──► cross-validate 4 models ──► fit production model
                                                                        │
             REST API  ◄── model_state.json ◄── recommend + stress-test ◄┘
             Dashboard ◄──┘                         └──► insights_report.html
```

## Quick start

```bash
pip install -r requirements-dev.txt      # Python 3.10+
pip install -e .                          # installs the `pricing` command
pricing train                             # ~1 min: checks, CV, fit, recommendations, sensitivity, report
pytest                                    # 17 tests
uvicorn api.main:app --port 8000          # REST API  → http://localhost:8000/docs
streamlit run app/streamlit_app.py        # dashboard → http://localhost:8501
```

The same commands are available through `make` (`make train`, `make api`, `make app`, `make test`).
Without installing, prefix commands with `PYTHONPATH=src` and use `python -m pricing.cli …`.

**Docker:** `docker compose up --build` builds one image, trains the model during the build, and starts the API (:8000)
and the dashboard (:8501). To retrain on new data, run `docker compose run --rm trainer`. The `artifacts/`, `data/` and
`reports/` folders are mounted as volumes.

## Commands

| Command | What it does |
|---|---|
| `pricing validate` | Runs 20 data-quality checks; exits 1 if a critical check fails |
| `pricing train [--no-cv] [--no-report]` | Full pipeline; writes `artifacts/` and `reports/insights_report.html` |
| `pricing recommend S12 P13 --max-change 0.05` | Recommends a price for one store × item (JSON) |
| `pricing simulate S08 P13 289 --discount 0.1` | Simulates one scenario (JSON) |
| `pricing report` | Rebuilds the HTML report from saved artifacts |

## API

| Method | Endpoint | Purpose |
|---|---|---|
| GET | `/health` | Model version, training time, last data week |
| GET | `/stores`, `/products` | Master data |
| GET | `/elasticity/{store}/{product}` | Price sensitivity with a 90% range, baseline, costs, confidence |
| POST | `/simulate` | `{store_id, product_id, price, discount, competitor_change}` → orders, revenue and contribution vs baseline |
| POST | `/recommend` | `{store_id, product_id, guardrails{…}, competitor_change}` → recommended price, uplift range, P(gain), reason |
| GET | `/recommendations?store_id=&category=&max_price_change=` | Price list for the whole chain or one store |

## Method

**1. Data validation** (`data.py`). Checks schema, keys, duplicates, gaps, arithmetic identities (net price, revenue,
channel split), weather/calendar coverage, extreme weeks, and whether there is enough price variation to identify a
price effect.

**2. Demand model** (`models.py`). One regression per store × item on weekly data:

```
log(orders) = a + b·(our net price ÷ competitor price) + promo + festival/IPL/exam/monsoon/Navratri flags
              + temperature + rainfall + trend
```

Using price *relative to competitors* means own-price elasticity `e = b·P/C` grows as we price above rivals. That gives
every item a finite best price, and the cross-elasticity with respect to competitor price falls out of the same model.
**Partial pooling** (empirical Bayes) blends each slope with the average for its category × store type, weighted by how
precise it is. **Moving-block bootstrap** (4-week blocks) gives standard errors that allow for autocorrelation.

**3. Cross-validation** (`validation.py`). Rolling origin with an expanding window: 4 folds × 12 unseen weeks. The four
models are a naive benchmark, the per-pair log-linear model, the pooled model, and a global gradient-boosted model with
price constrained to reduce demand. Each model is scored on overall WAPE, bias, WAPE in **price-change weeks** and
**promo weeks**, and on how much its elasticity estimates drift between folds.

| Model | WAPE | Bias | WAPE, price-change weeks | Elasticity drift | Elasticity vs truth (corr) |
|---|---|---|---|---|---|
| **Pooled (production)** | **9.3%** | +0.1% | **9.3%** | **0.08** | **0.95** (90% bands cover 89%) |
| Log-linear | 9.4% | +0.2% | 9.4% | 0.15 | 0.90 |
| Gradient boosting | 9.7% | −1.8% | 9.8% | 0.33 | 0.62 |
| Naive | 16.9% | −4.8% | 24.6% | — | — |

Gradient boosting forecasts nearly as well, but it measures price effects badly. A good forecaster is not necessarily a
good pricing model.

**4. Economics** (`economics.py`). Contribution per order = net price − food cost − delivery share × (packaging + commission
× net price). Baseline demand is the model's prediction for the last 13 weeks at today's price with no promotion.

**5. Optimiser** (`optimizer.py`). Searches menu prices (ending in 9) within ±max change and picks the one with the
highest expected contribution, subject to:
- a premium cap over competitors;
- a maximum order loss;
- a minimum gain;
- **P(beats holding) ≥ 80%** across 500 draws of the elasticity.

Otherwise the advice is to hold. The output names the constraint that binds and gives the unconstrained optimum.

**6. Sensitivity** (`sensitivity.py`):
- tornado charts per store × item and for the whole chain (elasticity, food cost, competitor price, volume, delivery share, commission);
- Monte Carlo over elasticity uncertainty;
- competitor-reaction scenarios (rivals copy 0 / 50 / 100% of our moves);
- a guardrail frontier (gain vs allowed price change);
- food-cost pass-through;
- discount effectiveness and break-even volume lift.

## Results on the synthetic data (default guardrails)

- **+₹63.5k a week of contribution (+1.7%, ≈ ₹33 lakh a year)**, with revenue +1.9% and orders +1.0%. The 90% Monte
  Carlo range is ₹59k–68k a week. The plan is 45 price rises, 30 cuts and 93 holds.
- Price sensitivity varies about 3× across store types. The median elasticity is −0.65 at the airport and −2.09 in the
  college area. **Raise** at the airport and highway (captive customers); **cut** in college-area, residential and dense
  high-street stores.
- **Discounts deeper than 15% usually lose contribution.** At 20% off, orders rise about 42%, but contribution improves in
  only 40% of items. Discounting never pays at airport or highway stores.
- **Pass food-cost increases on in full.** The profit-maximising response is about 109% of the rupee cost increase,
  because app commission grows with price.
- If competitors **fully match** every move, the plan still adds ₹34k a week.



## Operating it

- **Retrain** monthly or after a menu revision. Each artifact set is stamped with the training time and a data fingerprint
  (`artifacts/metadata.json`).
- **Monitor** forecast error per store × item (alert if WAPE > 15% for 3 weeks), the competitor price index, and actual
  vs forecast orders after each price change.
- **Roll out** through a controlled test: apply high-confidence recommendations in half of a matched set of stores for 8
  weeks, measure with difference-in-differences, then scale up.
- **Limits:** items are modelled independently (no cannibalisation or basket effects), competitor reaction is a scenario
  rather than a forecast, long-run brand effects are not captured, and prices outside the historical range are
  extrapolations (flagged by the confidence label).



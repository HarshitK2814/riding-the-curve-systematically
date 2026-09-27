# PROJECT 3 — Riding the Curve Systematically: Forward-Rate Term-Premium Signals for Short-Maturity Global Bonds

*Labels:* **[SEC]** explicit · **[Reconstruction]** our implementation choice · **[Extension]** our research addition.

## 1. Institutional Inspiration

| | |
|---|---|
| Institution | Dimensional Fund Advisors |
| Strategy | DFA Five-Year Global Fixed Income Portfolio: "variable maturity" global short-duration bonds, currency-hedged |
| Asset class | Government and investment-grade bonds, ≤5-yr maturity, multi-country, FX-hedged |
| SEC filing | Form 497K (summary prospectus dated 2026-02-28), DFA Investment Dimensions Group Inc., CIK 355437, acc. **0000355437-26-000121**, filed **2026-02-27** |
| URL | https://www.sec.gov/Archives/edgar/data/355437/000035543726000121/c497k.htm |
| Section | Principal Investment Strategies |
| Week-1 task | Locate an older DFA 485BPOS or SAI that describes *how* expected premiums are identified from current yield curves (older filings reportedly describe shifting "maturities and country allocations based on changes in the eligible yield curves"), and cite it verbatim |

## 2. Why This Methodology Matters

This is one of the few **explicit, rule-like rates methodologies in an SEC filing**: the manager shifts maturity toward the longer end *only when the expected term premium there is larger*, estimated from current curves. That maps directly onto one of the central questions of term-structure economics: **are forward rates unbiased predictors of future rates (the expectations hypothesis), or do forward–spot spreads forecast excess bond returns?**

Using the current curve as the expected-return estimate means assuming the curve does not change. That is equivalent to a Fama–Bliss regression slope of **exactly 1**. The expectations hypothesis implies a slope of **0**. The truth, and how it varies across countries and regimes (e.g. 2022–24 inversions), is an open empirical question with real money attached.

## 3. What the SEC Actually Says

**Explicit [SEC]**
* Universe: U.S. and foreign debt maturing in ≤5 years. Weighted-average effective maturity ≤5 years.
* "if the expected term premium is greater for longer-term securities in the eligible maturity range, the Advisor will focus investment in the longer-term area, otherwise, the Portfolio will focus investment in the shorter-term area."
* Investment grade at purchase (AAA to BBB−). At least three countries including the U.S.
* Currency exposure managed with forward contracts.

**Reconstruction [Reconstruction]**
* Use **government zero curves** only. Credit is out of scope; see limitations.
* Estimate the expected term premium by the **static (unchanged-curve) carry-plus-roll-down**, i.e. the forward-minus-spot spread. This is the simplest estimator consistent with "based on current yield curves". It is *our* choice, not a disclosed formula.
* Decision rule R1: hold the maturity bucket with the highest hedged expected excess return. Decision rule R2: extend maturity only while the *marginal* premium per extra year of duration exceeds a threshold θ (θ calibrated on development data).
* Country weights: proportional to positive hedged expected premium, subject to ≥3 countries.
* FX hedging via 1-month forwards priced by covered interest parity from short rates.

**Extension [Extension]:** compare the static estimator with regression-based (Fama–Bliss, Cochrane–Piazzesi), dynamic-Nelson–Siegel (Kalman) and ACM-based estimators; uncertainty-aware switching; the 2022–24 inversion natural experiment.

**Unknown:** DFA's actual estimator, thresholds, risk adjustment, credit tilts, country constraints and trading rules.

## 4. Research Question

> **RQ1.** In the 0–5-year segment of six government curves, does the static forward-spread estimator of expected excess return (the rule implied by the disclosure) produce better out-of-sample maturity decisions than (a) no timing (fixed 2.5-yr duration), (b) Fama–Bliss regression forecasts, (c) Cochrane–Piazzesi, or (d) dynamic Nelson–Siegel forecasts that model expected curve changes?
>
> **RQ2.** Does requiring the premium difference to exceed its estimation uncertainty before switching (uncertainty-aware switching) reduce turnover without reducing captured premium?

**Primary hypothesis (H1):** The static-rule portfolio earns a higher certainty-equivalent return than the fixed-duration benchmark, net of costs, pooled across countries over the test period. **H1b:** The Fama–Bliss slope estimated on the 0–5-yr segment is significantly > 0 but < 1, so the static rule *overstates* premia. This predicts over-trading, which RQ2 targets.

## 5. End-to-End Workflow

```
Central-bank zero curves (US, DE/EA, UK, CA, JP, AU) + short rates + FX spot
  ↓ Cleaning: align to month-end business days; convert par→zero where needed (JP); units (cont. comp.)
  ↓ Curve reconstruction on a 0.25–10y grid from published NSS parameters (exact) or bootstrapped yields
  ↓ Synthetic zero-coupon bond returns (monthly holding) for each maturity & country
  ↓ Hedged USD returns via CIP forward premium
  ↓ Expected-excess-return estimators (static / FB / CP / DNS-Kalman / ACM[US])
  ↓ Decision rules R1, R2, uncertainty-aware switching
  ↓ Country allocation (≥3 countries) + WAM ≤ 5y constraint
  ↓ Costs: bid–ask by maturity bucket (bp of yield × duration)
  ↓ Walk-forward (expanding) evaluation; forecast statistics + portfolio statistics
  ↓ Attribution: carry vs roll-down vs curve change vs FX-hedge residual
  ↓ Robustness; 2022–24 inversion case study; monthly paper decisions going forward
```

**Bar convention:** all curves are observed at the **last business day of month t** (central-bank end-of-day fits). The decision for month t+1 uses only curves up to t. Positions are assumed established at t's end-of-day curve plus a cost. *Note:* central banks publish fits with a lag of 1–3 days. For the paper-trading component, record the decision on the date the curve becomes available, not on t.

## 6. Mathematical Formulation

Let $y_t^{(n)}$ be the continuously compounded zero yield for maturity $n$ (years). The log price is $p_t^{(n)}=-n\,y_t^{(n)}$ and the holding period is $h=1/12$.

**Realised log excess return:**
$$
rx_{t+h}^{(n)}=p_{t+h}^{(n-h)}-p_t^{(n)}-h\,y_t^{(h)}=n\,y_t^{(n)}-(n-h)\,y_{t+h}^{(n-h)}-h\,y_t^{(h)}
$$

**Static (unchanged-curve) expected excess return** [Reconstruction of the disclosed rule]. Set $y_{t+h}^{(n-h)}=y_t^{(n-h)}$:
$$
\widehat{rx}^{\,S}_{t}(n)=n\,y_t^{(n)}-(n-h)\,y_t^{(n-h)}-h\,y_t^{(h)}=h\big(f_t^{(n-h\to n)}-y_t^{(h)}\big),\qquad
f_t^{(n-h\to n)}=\frac{n\,y_t^{(n)}-(n-h)\,y_t^{(n-h)}}{h}
$$
Decompose into **carry** $h(y^{(n)}_t-y^{(h)}_t)$ plus **roll-down** $(n-h)(y^{(n)}_t-y^{(n-h)}_t)$.

**Fama–Bliss regression (H1b):**
$$
rx_{t+h}^{(n)}=a_n+b_n\,h\big(f_t^{(n-h\to n)}-y_t^{(h)}\big)+\varepsilon_{t+h}
$$
The expectations hypothesis implies $b_n=0$ (with a constant premium in $a_n$). **The static rule implies $a_n=0,\ b_n=1$.** Forecast: $\widehat{rx}^{\,FB}_t=\hat a_n+\hat b_n\,\widehat{rx}^{\,S}_t$, re-estimated on an expanding window.

**Cochrane–Piazzesi (annual forwards, applied at a monthly step).** $f_t=[y^{(1)}_t,f^{(1\to2)}_t,\dots,f^{(4\to5)}_t]^\top$:
$$
\overline{rx}_{t+1}=\gamma^\top f_t+\bar\varepsilon_{t+1},\qquad rx^{(n)}_{t+1}=b_n(\gamma^\top f_t)+\varepsilon^{(n)}_{t+1}
$$
Annual horizon with overlapping monthly observations → Hodrick (1992) / Newey–West(12) standard errors. Scale to a monthly forecast: $\widehat{rx}^{CP}_{t}(n)\approx \tfrac{1}{12}\hat b_n\hat\gamma^\top f_t$ (stated approximation).

**Dynamic Nelson–Siegel with Kalman filter (Diebold & Li 2006).** Maturity τ is in months and λ = 0.0609 (Diebold–Li; sensitivity: estimate λ):
$$
y_t(\tau)=L_t+S_t\frac{1-e^{-\lambda\tau}}{\lambda\tau}+C_t\Big(\frac{1-e^{-\lambda\tau}}{\lambda\tau}-e^{-\lambda\tau}\Big)+\epsilon_t(\tau),\quad \epsilon_t\sim N(0,H)
$$
$$
\beta_t=\mu+\Phi(\beta_{t-1}-\mu)+\eta_t,\quad \eta_t\sim N(0,Q),\qquad \beta_t=(L_t,S_t,C_t)^\top
$$
Estimate by MLE on an expanding window (re-fit annually, filter monthly). Expected-curve forecast $E_t[y_{t+h}(\cdot)]$ gives
$$
\widehat{rx}^{\,DNS}_t(n)=n\,y_t^{(n)}-(n-h)\,E_t\big[y_{t+h}^{(n-h)}\big]-h\,y_t^{(h)}
$$

**ACM (US only):** use NY Fed ACM term-premium estimates as an external predictor. Check vintage/revision issues: ACM is re-estimated over the full sample, so it has **look-ahead bias** as a real-time signal. It is reported *only* as an in-sample reference, never as an OOS contender.

**Hedged USD excess return** (CIP, ignoring cross-currency basis):
$$
R^{hedged}_{t+h}\approx R^{loc}_{t+h}-h\,i^{loc}_t+h\,i^{US}_t\ \Rightarrow\ \text{hedged excess over US cash}\approx rx^{loc}_{t+h}
$$
So hedged selection ranks **local term premia**. (Include the basis as a sensitivity where estimable.)

**Decision rules.** Buckets $b\in\{0\text{–}1,1\text{–}2,2\text{–}3,3\text{–}4,4\text{–}5\}$ years, represented by their mid maturity.
* **R1:** $b^*_{c,t}=\arg\max_b \widehat{rx}_{c,t}(b)$
* **R2 (marginal):** extend from $b$ to $b+1$ iff $\widehat{rx}(b+1)-\widehat{rx}(b)>\theta\,[D(b+1)-D(b)]$, with θ from development data.
* **Uncertainty-aware (Extension):** switch from the current $b_{t^-}$ to $b^*$ iff $\widehat{rx}(b^*)-\widehat{rx}(b_{t^-})>\kappa\cdot \widehat{se}_t$. Here $\widehat{se}_t$ comes from the forecast model's parameter and residual uncertainty (bootstrap or the Kalman forecast variance); κ ∈ {0, 0.5, 1, 2} is pre-registered, and κ = 1 is primary.

**Country weights:** $w_{c,t}\propto \max\{\widehat{rx}_{c,t}(b^*_{c,t}),0\}$, floor 10% for ≥3 countries (US included), cap 50%; if all ≤0, hold 0–1y bucket equally.

**Costs:** round-trip bid–ask in yield bp $s_b$ (baseline 0.5 bp for 0–2y, 1 bp for 2–5y on G7 sovereigns; ×2 sensitivity) × modified duration $D_b$ × turnover.

**Evaluation.** Out-of-sample R² (Campbell & Thompson 2008) vs the expanding historical mean $\bar{rx}$:
$$
R^2_{OOS}=1-\frac{\sum_t (rx_{t+h}-\widehat{rx}_t)^2}{\sum_t (rx_{t+h}-\bar{rx}_t)^2}
$$
Certainty-equivalent return (Thornton & Valente 2012), with risk aversion γ = 5 (3/10 sensitivity):
$$
\mathrm{CER}=\bar R_p-\tfrac{\gamma}{2}\hat\sigma_p^2
$$

## 7. Data Requirements (all free)

| Country | Source | Content |
|---|---|---|
| US | Federal Reserve **Gürkaynak–Sack–Wright** nominal yield curve (federalreserve.gov) | Daily Svensson parameters and zero yields |
| Germany / euro | **Deutsche Bundesbank** term-structure (Svensson) estimates; **ECB** euro-area AAA curve (2004+) | Daily/monthly parameters |
| UK | **Bank of England** nominal spot curves | Daily zero yields |
| Canada | **Bank of Canada** zero-coupon yield curves | Daily |
| Japan | **Ministry of Finance** JGB interest rates (par) → bootstrap | Daily par yields |
| Australia | **RBA** zero-coupon yields (statistical table F17) | Daily |
| Short rates | FRED T-bills / OECD 3-month interbank (FRED series) | Monthly |
| FX | FRED H.10 daily spot | For unhedged comparison and hedge P&L check |
| US term premium ref. | **NY Fed ACM** term premium | Daily/monthly |

Verify each start date in Week 1 (they range from ~1961 US to ~1990s AU/EA) and record it in `data/README.md`.
**Limitations:** these are **fitted smooth curves**, not tradable bonds. Returns are synthetic and ignore specialness, bid–ask by issue and on-the-run effects. The approach is standard in the literature and is stated clearly.

## 8. Feature Engineering

* Zero yields on the grid {0.083, 0.25, 0.5, 1, …, 5, 7, 10} years, continuously compounded. Convert from published conventions (e.g. BoE is continuously compounded; confirm per source).
* $y^{(n-h)}$ by curve interpolation (use the NSS formula where parameters are available; else cubic spline on log discount factors).
* Forward spreads $f^{(n-h\to n)}-y^{(h)}$ for each n; CP forwards at 1..5y.
* DNS factors (filtered, real-time).
* Level/slope/curvature proxies for regime reporting: $y^{(10)}$, $y^{(10)}-y^{(0.25)}$, $2y^{(2)}-y^{(0.25)}-y^{(10)}$.
* **Winsorisation:** none on yields. Flag and inspect curve-fit outliers (jumps > 50 bp/day at the short end) rather than clipping.
* **Missing data:** forward-fill ≤ 3 business days; otherwise exclude that country-month.

## 9. Model Architecture

Five competing expected-return estimators (static, FB, CP, DNS, historical mean), all feeding **identical** decision and allocation rules. The Kalman filter is **justified** here because the question is exactly whether modelling expected curve dynamics (mean reversion of level and slope) improves on the unchanged-curve assumption. Optional appendix: a Gaussian-process or ML forecaster, only if it beats DNS in validation, and even then treated as an exploratory arm.

## 10. Portfolio Construction

Long-only, unlevered, fully FX-hedged. WAM ≤ 5y (disclosed). ≥3 countries. Monthly rebalance. Turnover penalty is implicit through the uncertainty threshold (RQ2). Risk: optionally scale the bucket choice by duration-adjusted risk. Primary = unscaled (closer to the disclosure).

## 11. Risk Management

Vol, Sharpe (vs US 1-month T-bill), Sortino, max DD, VaR/CVaR (monthly), **DV01 / duration time series**, key-rate exposures, country concentration, correlation with the US 2y and 5y Treasury, and a stress table: 1994 bond massacre, 2008, 2013 taper tantrum, 2020, 2022 (the largest drawdown in modern bond history), 2023 SVB, 2024–26.

## 12. Backtesting Methodology

* Splits (pooled panel): development ≤ 2004 · validation 2005–2014 · **test 2015–2023** · recent/live 2024-01 → latest.
* Expanding-window re-estimation (FB, CP annually; DNS re-fit annually, filtered monthly). **No full-sample parameters in OOS forecasts.**
* **ACM excluded** from OOS (revision look-ahead).
* Costs per Section 6, fixed in pre-registration.
* Inference with overlapping returns only in the CP regression. Portfolio evaluation uses non-overlapping monthly returns.

## 13. Benchmark Strategies

1. **Fixed-maturity ladder** 0–5y (equal weight across buckets), same countries, hedged. This is the natural "no-timing" benchmark.
2. Fixed 5y bucket (max duration allowed; tests whether timing beats simply harvesting the average premium).
3. Fixed 0–1y bucket (cash-like).
4. Bloomberg Global Aggregate 1–5y hedged, if accessible; else the ladder is primary.
5. **Historical-mean forecaster** feeding the same rules (the standard OOS-R² benchmark).

## 14. Ablation Study

Static full rule vs: −roll-down (carry only) · −country selection (US only) · −FX hedge (unhedged) · −uncertainty threshold (κ = 0) · R1 vs R2 · each estimator swapped in.

## 15. Robustness Testing

Holding period 1m vs 3m · bucket granularity · λ fixed vs estimated · country set (drop JP, which is near the zero lower bound for years) · cost ×2 · extend the eligible range to 10y (outside the disclosure, as exploration) · subperiods by policy regime (hiking/cutting cycles dated from policy-rate changes known at t) · ZLB periods excluded.

## 16. Statistical Validation

| Test | Why |
|---|---|
| Fama–Bliss slope CI (HAC) per country and pooled | Directly tests the implicit b = 1 assumption (H1b) |
| Campbell–Thompson $R^2_{OOS}$ + **Clark–West** test | Clark–West is correct for *nested* comparisons with the historical mean (DM is biased there) |
| **Diebold–Mariano** between non-nested estimators (e.g. static vs DNS) | Equal predictive accuracy for non-nested forecasts |
| CER difference with bootstrap CI | Economic, not just statistical, value (Thornton–Valente) |
| Ledoit–Wolf Sharpe difference | Portfolio-level comparison |
| Model Confidence Set (Hansen, Lunde & Nason 2011) across estimators | Many estimators × countries; identifies the set not beaten |

## 17. Research Extension [Extension]

1. **Estimator horse race** (RQ1), including the DNS "expected curve change" correction.
2. **Uncertainty-aware switching** (RQ2): a decision-theoretic threshold based on forecast standard errors.
3. **2022–2024 inversion natural experiment:** the static rule shortens maturity under inversion. Did that avoid the 2022 losses, and did it miss the 2023–24 rally? Report the event study.
4. **Recent research connection:** AQR's October 2025 note "Bond Market Focus: Yield Curves and Mean Reverting Rate Expectations" argues that curve shape embeds mean-reverting rate expectations. We test this in the form "DNS mean reversion vs the static curve". Dimensional's insight pieces (e.g. "Should You Bet on Mean Reversion in Fixed Income Premiums?") give the manager's own public view. Quote them in `literature_review.md` after reading them in full.

## 18. Expected Research Contribution

A multi-country, real-time, cost-aware evaluation of **the specific decision rule disclosed by a large manager** (switch maturity within 0–5y on the expected term premium), benchmarked against regression and state-space estimators, with an uncertainty-aware switching variant. Most bond-predictability studies target 1–5y *annual* excess returns in the US. **Limited evidence was found** of evaluations focused on the *0–5y maturity-choice problem across six hedged curves with monthly decisions*. Novelty confidence: **new empirical comparison / extension of existing literature.**

## 19. Failure Modes

* **Smoothed-curve artefacts:** Svensson fits can create spurious roll-down at short maturities. Check robustness with the raw bill/bond yields available for the US.
* **ZLB periods (JP, EA, CH):** premia are compressed and decisions are noise.
* **Regime shifts:** the 2022 hiking shock. A rule calibrated in a falling-rate era overstates premia.
* **CIP basis:** ignoring the cross-currency basis mismeasures hedged returns (Du, Tepper & Verdelhan 2018).
* **Overlapping-return inference** for CP inflates t-stats if done naively.

## 20. Resume Version

**Riding the Curve: Forward-Rate Term-Premium Signals in Short-Maturity Global Bonds** *(inspired by Dimensional's SEC-disclosed variable-maturity process)*
* Reconstructed an SEC-disclosed variable-maturity rule on 6 central-bank zero curves (US, DE, UK, CA, JP, AU, `[years]`), with synthetic FX-hedged bond returns, carry/roll-down attribution and a real-time walk-forward design.
* Benchmarked the rule's implicit "unchanged curve" estimator (Fama–Bliss slope = 1) against Fama–Bliss, Cochrane–Piazzesi and a Kalman-filtered dynamic Nelson–Siegel model. OOS R² `[x]` (Clark–West p = `[p]`); certainty-equivalent gain `[y]` bp/yr vs a 0–5y ladder, net of costs.
* Introduced uncertainty-aware maturity switching: turnover reduced by `[z]%` with premium capture `[w]`. Documented behaviour through the 2022–24 curve inversion.

---

### Phase 6 — Recent Research Connection

| | |
|---|---|
| A. Historical methodology | Variable maturity on expected term premium from current curves (DFA prospectus) |
| B. Recent research | AQR (Oct 2025) whitepaper on yield curves and mean-reverting rate expectations; Dimensional public insights on fixed-income premiums; Bauer & Hamilton (2018, *RFS*) on the fragility of bond-return predictability; 2024 literature on survey-based expectations suggesting past predictability overstated true risk premia |
| C. What changed | More emphasis on (i) expected rate paths vs a static curve, (ii) robustness of predictability, (iii) post-2022 regime |
| D. Our implementation | DNS expected-curve forecasts; uncertainty threshold; inversion event study |
| E. Gap | Real-time, multi-country evaluation of the *short-end maturity-choice* rule, with decision-uncertainty |

### Prior-art table

| Paper / Institution | Method | Dataset | Main finding | How we differ |
|---|---|---|---|---|
| Fama & Bliss (1987, *AER*) | Forward-spread regressions | US 1–5y | Forward spreads forecast excess returns (b > 0) | 6 countries, monthly decisions, 0–5y maturity choice |
| Cochrane & Piazzesi (2005, *AER*) | Tent-shaped forward factor | US 1–5y | High in-sample R² | Real-time OOS, multi-country |
| Diebold & Li (2006, *J. Econometrics*) | Dynamic Nelson–Siegel | US yields | Good yield forecasts | Used as an *expected-return* estimator for a decision rule |
| Thornton & Valente (2012, *RFS*) | OOS forward-rate forecasts, asset allocation | US | Little economic value OOS | Short-end maturity choice, hedged global, uncertainty threshold |
| Bauer & Hamilton (2018, *RFS*) | Robust inference on bond risk premia | US | Much predictability fragile | We use their critique in the inference design |
| Koijen, Moskowitz, Pedersen & Vrugt (2018, *JFE*) | Carry across assets incl. bonds | Global | Bond carry predicts returns | Maturity choice within a curve, not cross-country only |
| Adrian, Crump & Moench (2013, *JFE*) | Affine term-premium model | US | Term-premium estimates | Reference only (revision look-ahead) |
| Du, Tepper & Verdelhan (2018, *JF*) | CIP deviations | G10 | Persistent basis | Sensitivity for hedged returns |

### Pre-registration

| Item | Value |
|---|---|
| Primary metric | Net CER (γ = 5) of the static-rule portfolio minus the ladder benchmark, pooled test period |
| Secondary | $R^2_{OOS}$ per estimator (Clark–West); turnover |
| Period | Test 2015-01 → 2023-12; recent 2024 → latest separately |
| Robustness | Costs ×2, 3m holding, drop JP, λ estimated |
| Failure | H1 fails if the CER-difference bootstrap CI includes 0 or is negative. H1b fails if the pooled FB slope CI includes 1 |

### "Unusual result" candidates
* The static rule may beat sophisticated estimators *precisely because* it trades less often than CP/DNS forecasts imply, so the gains would be a cost effect, not a forecasting effect.
* Premium capture may be concentrated in a few steep-curve episodes, making the strategy an implicit bet on policy easing cycles.

### Paper-trading component
Monthly: when the curves for month-end t are published, run the pipeline, commit decisions (bucket per country, weights), then record realised synthetic returns at t+1. *Simulated live evaluation.*

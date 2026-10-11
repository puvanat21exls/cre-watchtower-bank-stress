# Mapping Commercial Real Estate Stress in U.S. Regional Banks
### A bank-level early-warning system built from primary FDIC data

**Author:** Puvanat W. · **Period:** June–July 2026 · **Status:** Live monitoring system
**Stack:** Python (pandas, requests), FDIC BankFind Suite API, SSGA ETF holdings data

> **Note (October 2026):** this write-up describes the July 2026 version of the system (data through 2026Q1). Corrections to the original text and newer results, including a backtest of the warning tags, are in the addendum at the end. Current figures are in the README.

---

## Abstract

I built a monitoring system that measures commercial real estate (CRE) concentration and credit deterioration across all 160 constituents of the KRE regional banking ETF (185 bank charters), using quarterly FDIC call report data from 2019Q1 to present. The system reconstructs the interagency regulatory concentration measure at the bank level, validates itself against the known 2023 bank failures, and ranks banks by the *speed* of nonaccrual deterioration rather than exposure alone. Two findings stand out. First, as of 2026Q1, 30 of 156 measurable KRE names (19%) exceed the 300% regulatory concentration guidance, and 69 (44%) sit above 250%. Second — the central result — **exposure and deterioration are different lists**: the most concentrated banks are largely not the ones deteriorating fastest, meaning a static concentration screen watches the wrong institutions. Median nonaccrual rates in the flagged cohort have quadrupled since 2023Q1 (0.15% → 0.62%), a persistent grind rather than a cliff, consistent with maturity-wall stress being absorbed through loan extensions.

---

## 1. Motivation

The thesis under investigation: CRE stress in this cycle is not a broad real-estate crash but a concentrated intersection — cheap 2021-vintage loans maturing into the 2026–27 "maturity wall" at materially higher rates, collateral impairment concentrated in structurally challenged asset classes (Class B/C office, legacy retail), and the debt held disproportionately by regional banks. Public evidence supports the mechanism: office CMBS delinquency reached an all-time high of 12.3% in January 2026, and Trepp's CMBS special servicing rate (10.91% in January 2026) stood about 3.4 percentage points above its overall delinquency rate (7.47%) — troubled loans being worked out by a special servicer without being counted as delinquent, a public proxy for "extend and pretend" activity deferring recognition of distress.

Aggregate statistics, however, cannot answer the operative question: *which banks, specifically, and is their condition deteriorating or stable?* Regulators see loan-level data; public investors mostly see quarterly aggregates. The gap in between — systematic, bank-level, primary-source measurement — is what this project fills.

## 2. Data and methodology

**Universe.** All KRE ETF constituents (State Street daily holdings file), mapped to FDIC certificate numbers. Five constituents are multi-bank holding companies (Wintrust alone operates 16 charters); these are pulled at every subsidiary charter and aggregated to holdco-level ratios by summing components before dividing — the lead-bank-only alternative would have understated Wintrust's group exposure by roughly 85% of its balance sheet. Final coverage: **160 of 160 names, 185 charters, 5,510 rows (5,365 bank-quarters plus 145 holding-company aggregate rows).**

**Name matching.** ETF holdings list holding companies; the FDIC indexes operating banks — the names frequently differ ("KeyCorp" vs. "KeyBank, N.A."). Matching was done deterministically: a match is accepted only when exactly one candidate survives normalization; any ambiguity (e.g., two active institutions literally named "Northeast Bank") is routed to manual review rather than resolved by highest similarity score. Fifty-two names required documented manual overrides. This conservatism follows directly from an early failure: an aggressive fuzzy matcher initially assigned three different "First"-named holdcos the data of the same unrelated bank.

**Field verification.** No FDIC variable code was used without confirming its definition against the FDIC data dictionary. This rule exists because plausible-looking codes fail silently: an early candidate, `NACON`, returns valid numbers but measures nonaccrual *consumer* loans, not construction — a wrong-concept field that produces reasonable-looking, meaningless ratios. Verified mapping:

| Concept | FDIC code(s) |
|---|---|
| Construction & land development | `LNRECONS` |
| Nonfarm nonresidential (total / owner-occ / non-owner-occ) | `LNRENRES` = `LNRENROW` + `LNRENROT` |
| Multifamily | `LNREMULT` |
| Nonaccrual, by same segments | `NARECONS`, `NARENRES`, `NAREMULT` |
| Total risk-based capital ($) | `RBC` |
| CET1 ratio | `RBCT1CER` |

The `LNRENROW + LNRENROT = LNRENRES` accounting identity holds exactly on all 5,365 bank-quarter rows, and `RBC` was confirmed as a dollar denominator via the Tier 1 + Tier 2 identity — internal consistency checks that validate the field semantics beyond dictionary definitions.

**Concentration measure.** Following the interagency guidance concept, the primary measure excludes owner-occupied CRE (which does not depend on rental income for repayment):

> **refined concentration** = (construction + multifamily + non-owner-occupied nonfarm nonresidential) / total risk-based capital × 100

A secondary, inclusive measure retains owner-occupied balances for comparability. The distinction is material: Live Oak Bancshares — an SBA lender whose book is largely owner-occupied — reads 441% inclusive but 216% refined, moving it from apparent top-decile risk to mid-pack. The refined measure separates rental-income-dependent risk from owner-occupied noise.

**Deterioration measure.** For each bank, quarterly CRE nonaccrual rate = segment nonaccruals / segment loans; an OLS slope is fit over the 13 quarters 2023Q1–2026Q1 (basis points per quarter), with a trailing acceleration term. Banks are ranked by a composite weighting slope (50%), current level (30%), and acceleration (20%) — deliberately weighting *trajectory* over *level*, because a stable known problem is priced while a developing one is not. Each bank receives a state tag: **BURNING** (elevated and rising), **IGNITING** (low level, steep slope — the early-warning state), **SMOLDERING** (elevated but improving), **QUIET**.

**Data integrity rules.** Missing values propagate as nulls — never coerced to zero, never backfilled (a null-as-zero bug in an early iteration silently zeroed concentration ratios for banks with missing fields). Every pipeline report must declare N processed vs. N requested. Raw FDIC fields are stored alongside derived metrics so every conclusion is recomputable from stored evidence.

## 3. Validation

Three independent checks, designed before results were inspected:

1. **Historical anchors.** The three major 2023 failures, computed from their 2022Q4 filings: Signature Bank **318%** inclusive-basis concentration, 300% refined (a genuine CRE-concentrated institution), First Republic **194%**, Silicon Valley Bank **16%**. The discrimination matters as much as the levels: SVB *should* read low — its failure was duration and deposit risk, not CRE — and it does. The instrument measures CRE risk specifically, not generic fragility.
2. **External consistency.** The 2026Q1 rankings independently surface institutions with publicly known CRE profiles: Flagstar (the former NYCB, which had a public CRE-driven crisis in 2024) tags BURNING with the cohort's highest nonaccrual level (6.7%); Hingham Institution for Savings, a famously concentrated Boston CRE lender, ranks first on concentration (550%); Carter Bankshares, working out a known concentrated-borrower problem, is the clearest SMOLDERING (nonaccruals improving at −26 bp/quarter).
3. **Independent recomputation.** All derived columns were recomputed from stored raw fields in a separate environment: maximum deviation 5×10⁻⁵ (rounding). Group aggregation and slope fits reproduced exactly.

## 4. Findings

**4.1 The concentration landscape (2026Q1).** Of 156 measurable names (four are CBLR-electing banks that do not report total risk-based capital and are excluded rather than approximated): median refined concentration **236%**; **30 names ≥ 300%**; **39 more between 250–300%**. The >300% cohort is geographically coherent — a visible NY/NJ metro CRE cluster (Dime, Peapack-Gladstone, Columbia Financial, Valley National, ConnectOne) alongside specialist lenders (Hingham 550%, Kearny 514%, Northeast Bank 489%). One flag exists only because of group aggregation: QCR Holdings crosses 300% at the holdco level while no individual subsidiary does.

**4.2 Exposure and deterioration are different lists.** This is the project's central empirical result. Four of the five most concentrated names (Hingham excepted) show flat-to-benign nonaccrual trends — ConnectOne (435%), Great Southern (457%), Northeast (489%), and Kearny (514%) all tag QUIET. Meanwhile the deterioration ranking is led by *mid*-concentration institutions: Preferred Bank of Los Angeles (353% concentration, 4.25% nonaccrual, +20 bp/qtr and accelerating), ServisFirst (298%, +13 bp/qtr), Shore Bancshares (332%), and Bank OZK (287%, the construction-lending specialist, +8 bp/qtr with strong acceleration). A static concentration screen — the industry-standard shorthand — would concentrate attention on precisely the banks where credit is currently calmest. Exposure answers "who is holding the most risk"; only trajectory answers "whose risk is going bad."

**4.3 The system-level trend: a grind, not a cliff.** Within the 69-name flagged cohort, the median nonaccrual rate has risen every year — roughly **quadrupling from 0.15% (2023Q1) to 0.62% (2026Q1)**, with the mean rising 0.25% → 0.83% — while the *count* of >300% names has been static (30–36 every quarter for three years). Deterioration, not concentration, is the moving variable. The pace is persistent but gradual, consistent with the extend-and-pretend mechanism spreading recognition of distress over time rather than forcing it. Notably, **zero banks currently tag IGNITING** — no low-level book is deteriorating steeply — indicating stress is grinding through already-elevated names rather than igniting fresh ones. The nearest miss (WaFd, +5.6 bp/qtr from a 0.40% base) is the specific thing the next quarterly run checks.

**4.4 Current watchlist (composite top 5).** Preferred Bank/LA — the only name combining a high level, a steep slope, *and* positive acceleration; ServisFirst; Shore Bancshares; Bank OZK; Hingham. Flagstar ranks sixth: worst level and slope in the cohort, but decelerating sharply (−114 bp), consistent with its worst quarters being behind it.

## 5. Limitations

- **Call reports lag** (~1 quarter) and nonaccrual recognition lags economic impairment — this system detects deterioration earlier than headlines, not earlier than reality. The complementary CMBS-based extend-and-pretend spread (tracked monthly in the same system) partially offsets this.
- **Owner-occupancy split** (`LNRENROT`) is the regulatory definition's approximation; construction and multifamily are not split by occupancy.
- **Four CBLR-electing banks** are unmeasurable on the RBC denominator and currently excluded (a tier-1-capital fallback is a planned extension).
- **Thirteen quarters** is a short window for OLS slopes; slopes from near-zero bases are noisy, which is why level and acceleration carry weight in the composite.
- The system observes **bank-reported credit quality**, which extension activity can defer; it should be read alongside market-based measures (the same project logs KRE/SPY volatility daily for hedge-cost timing).

## 6. Engineering note: building a pipeline that can be checked

Early versions of this pipeline failed in ways that looked like success, and those failures shaped the methodology. Three kinds recurred: (a) plausible-but-wrong API field codes that returned valid-looking numbers for the wrong concept, (b) silently reduced scope — a report of aggregate statistics computed over 2 banks, presented as if it covered 110 — and (c) nulls coerced to zero, producing a report whose every number was an artifact. One dataset had to be declared corrupt and rebuilt after a forensic review found mixed schemas and one bank's data stored under another bank's name.

The countermeasures became permanent project rules: verify every field against the source data dictionary; require declared N in every report; propagate nulls; validate against known historical outcomes before trusting novel output; store raw evidence alongside derived conclusions so everything is independently recomputable. The general lesson: a pipeline's quality cannot be judged from its own reports; **trustworthy pipelines are the ones designed so that a wrong answer is structurally difficult to produce unnoticed.**

---

*System components: quarterly FDIC concentration/deterioration screen (`banks`, `slopes`), monthly Trepp extend-and-pretend spread meter (`eap`), daily volatility logger for hedge-cost percentiles (`iv`), consolidated dashboard (`status`) — one Python script, flat CSV storage, git-versioned with full audit trail.*

---

## Addendum — October 2026 update

**What changed in the system.** Coverage is now 165 of the 167 current KRE constituents (190 charters; two names have no reliable FDIC match and are excluded), with data through 2026Q2. The single script became four files (entry point, collection, analysis, shared basics). Three market-side signals now sit beside the bank screen: daily Treasury yields and credit spreads from FRED, daily KRE/VIX volatility with history from January 2025, and Trepp's public monthly CMBS figures. Every run is logged, and only one run can work on the data at a time.

**Current figures (2026Q2, 162 measurable names).** Median refined concentration 237.1%; 33 names above 300% and 73 at or above 250%. Within that cohort the rank correlation between concentration and nonaccrual slope is 0.015 (p ≈ 0.90), so the "different lists" result of §4.2 still holds *among already-concentrated banks*. The cohort's median nonaccrual rate is 0.57%, up from 0.15% in 2023Q2, and eased slightly in the latest quarter (0.60% → 0.57%).

**New result: a backtest of the tags.** Going back to every quarter from 2022Q1, the tags were recomputed using only data available at the time and compared with outcomes over the next 4 and 8 quarters (95% bootstrap intervals, resampling whole banks):

| Comparison | Outcome | 4 quarters | 8 quarters |
|---|---|---|---|
| BURNING + IGNITING vs. rest of cohort | CRE net charge-offs (bp of CRE loans) | +6.7 [3.5, 9.5] | +9.3 [2.8, 19.8] |
| BURNING + IGNITING vs. rest of cohort | Change in nonaccrual rate (bp) | −13.4 [−22.6, −6.8] | −11.7 [−32.9, +3.9] |
| >300% vs. ≤300%, all names | Change in nonaccrual rate (bp) | +5.3 [0.8, 10.3] | +10.4 [3.3, 22.4] |
| >300% vs. ≤300%, all names | CRE net charge-offs (bp) | +0.9 [−1.0, 3.6] | +2.9 [−2.0, 10.5] |

This sharpens two statements in the original text. First, BURNING does not mean "bad loans will keep rising": tagged banks' nonaccrual share stops rising while their write-offs run higher. The tag marks banks in the loss-taking phase. Second, the claim that a static concentration screen "watches the wrong institutions" is too strong as written: across all names, banks above 300% did see larger nonaccrual increases; what concentration alone does not do is separate who goes on to take losses.

Of 17 FDIC failures since 2020, 11 are testable; the tags flagged 3 in their last filings before failure (Signature Bank, Republic Bank, Community Bank & Trust – West Georgia). Silicon Valley Bank and First Republic were not flagged, consistent with §3: they were not CRE failures.

Backtest limits: only today's KRE constituents are tested (survivorship), observations from one bank overlap, the sample covers a single stress episode, and FDIC figures include later amendments.

**Corrections to the original text** (applied above):

1. The cohort median nonaccrual rate for 2026Q1 is 0.62%, not 0.63%.
2. §1 cited a Trepp "adjusted delinquency rate" about 2.2 points above the headline rate. Trepp does not publish that figure for free and it could not be sourced. It is replaced by Trepp's published special servicing and delinquency rates.
3. §3 described Signature Bank's 318% as refined-basis. It is the inclusive-basis figure; the refined figure is 300%. First Republic's and SVB's figures (194%, 16%) are likewise inclusive-basis (186% and 13% refined).
4. "5,510 bank-quarter observations" counted 145 holding-company aggregate rows; the bank-quarter count was 5,365.
5. The extend-and-pretend explanation of the nonaccrual grind (abstract, §4.3) is a hypothesis consistent with the data, not something the data here test.

**Tag rules have since been tightened.** BURNING now also requires a slope of at least the cohort median or 3 bp per quarter; a high level without that is SMOLDERING. IGNITING now also requires the level to be higher than in the prior quarter. Under the current rules the 2026Q2 cohort is 29 BURNING, 1 IGNITING, 8 SMOLDERING and 35 QUIET.

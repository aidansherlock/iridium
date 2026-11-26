# Iridium beta: pre-registered test of closing line value

**Written:** Saturday 21 November 2026, before any beta tester has scanned.
**Software under test:** Iridium beta zip `4f600abe` and every later release used by testers during the window (each row records its version).
**Final analysis date:** Tuesday 20 January 2027.
**Status of this document:** fixed. Nothing below is changed after publication. If an error is found, a dated addendum is published next to this file. The original text stays unchanged, and the result is reported under both the original rule and the corrected one.

---

## 1. Hypothesis

Paper orders that Iridium opens for beta testers are priced better than the Pinnacle closing line once venue fees are counted.

- **H0:** mean CLV over distinct selections ≤ 0.
- **H1:** mean CLV over distinct selections > 0.

The judge is the Pinnacle closing line. Neither Iridium nor its author controls it.

## 2. Primary metric: CLV in probability points, after fees

For each distinct selection *i*:

```
CLV_i = q_close_i + line_adj_i - 1 / d_eff_i
```

- **`d_eff_i`** is the effective decimal price at the open, after venue fees.
  - Sportsbook: the quoted decimal. The fee is 0 because the vig is already in the price.
  - Prediction venue (Kalshi, Polymarket): `d_eff = C / (C * p + fee)`.
    - `p` is the contract price paid (average fill, 0 to 1).
    - `C = max(1, floor(stake / p))` is the number of contracts.
    - `fee` is the venue's published taker fee for one order of `C` contracts at `p`. Kalshi uses `iridium/core/ev.py: kalshi_fee` (rate 0.07, the series multiplier, rounded up to the cent). Polymarket uses its published rate on the date of the order.
  - The fee is recomputed from the exported raw price, contracts and rate. The tester's own fee-inclusive decimal is kept for cross-checking only.
- **`q_close_i`** is the fair Pinnacle closing probability of the selected side.
  - Take Pinnacle's raw closing prices for every outcome of that market: two sides, or three for soccer moneylines (home, draw, away).
  - Remove the vig with the **power** method, `iridium/core/devig.py: power`. This applies whatever `DEVIG_METHOD` a tester configured.
- **`line_adj_i`** applies to spreads and totals only, when the closing line differs from the line taken. It is `iridium/signals/ledger.py: point_value(sport, market, side, point_taken, point_close)`, which converts the line difference to probability with the sport's normal margin or total spread. Moneylines have `line_adj = 0`.
- The arithmetic is `iridium/core/clv.py: CLVRecord(bet_decimal=d_eff, close_fair_prob=q_close + line_adj).clv_points`.

**Statistic:** the unweighted mean of `CLV_i` over all distinct graded selections. Each selection counts once, whatever the stake. A stake-weighted mean is reported as a secondary figure only.

## 3. The Pinnacle close

- **Source:** Pinnacle prices from The Odds API (bookmaker `pinnacle`).
- All Pinnacle snapshots of the event and market are pooled, whoever captured them: any tester's export, the hosted poller, or a backfill from The Odds API historical endpoint.
- **The close** is the snapshot with the latest fetch time (`captured_ts`) that is at or before the game's start (`commence_latest`).
- **The close is qualified only if all of these hold:**
  1. fetch time ≤ start;
  2. fetch time ≥ start minus **60 minutes**;
  3. fetch time ≥ the selection's open timestamp (a "close" observed before the bet was placed is not a close for that bet);
  4. Pinnacle quotes every outcome of that market in the snapshot.
- If any condition fails, the selection is **close-ungraded** (see section 6).
- The app's own 180-minute window is reported as a sensitivity check, never as the primary result.

## 4. Unit of observation and the dedupe key

The unit is a **distinct selection**. Ten testers scanning the same NFL slate produce one observation, not ten.

**Dedupe key:**

```
event_id | market | side
```

- `event_id` is The Odds API event id.
- `market` is one of `h2h`, `spreads`, `totals`.
- `side` is the team name, `Draw`, `Over` or `Under`, as stored in `paper_orders.side`.
- **The line is deliberately left out of the key.** "Chiefs −3" taken by one tester and "Chiefs −3.5" taken by another an hour later are one idea, not two.

**Which order represents the key:**
- the earliest `open_ts` across all testers;
- ties go to the lexicographically smallest `install_id`, then the smallest `order_id`.

That order's price, line, venue and fee define `CLV_i`. Other testers' orders on the same key are kept for the sensitivity check "mean CLV over all orders of the key" and are never counted separately.

**What else is a separate observation:**
- Two sides of the same market (for example Over by one tester, Under by another) are two keys, and both count.
- A team's moneyline and its spread are two keys. The correlation within a game is handled by the bootstrap (section 5).

**Rows that enter the test.** A row enters only if all of these hold:
- `environment = 'live'` and `run_id = 'live'` (demo, sample and replay data never count);
- `market` is in `h2h`, `spreads`, `totals` (props are reported separately and are not part of this test);
- `open_ts` is at or after the **publication timestamp** of this document;
- `open_ts` is before the game's start;
- the `install_id` is not on the **exclusion list**: the installs of the author and of company agents, recorded before the first export is merged. Their selections are reported as a separate, labelled series.

## 5. Sample size, inference and pass line

- **Sample size:** at least **300** distinct close-graded selections whose games started on or before the data freeze.
- **Data freeze:** Monday 19 January 2027, 12:00 UTC. Only selections whose game started by then, and exports received by then, count.
- **Interval:** a cluster bootstrap with these settings:
  - the resampling unit is the **game** (`event_id`), drawn with replacement; every distinct selection in a drawn game comes with it;
  - the statistic is the mean `CLV_i` over the resampled selections;
  - B = 10,000 resamples, seed `20261121`, percentile 95% interval.
- **Why games and not selections.** Resampling by game is stricter than resampling single selections, because selections in the same game move together. The single-selection bootstrap is also reported, but it does not decide the result.
- **PASS** requires all four of:
  1. n ≥ 300;
  2. mean CLV > 0;
  3. the lower bound of the game-cluster 95% interval > 0;
  4. close coverage ≥ 70% (section 6).
- **Result labels.** Exactly one is published on 20 January 2027:

| Label | Condition |
|---|---|
| **PASS** | all four conditions above |
| **NOT SHOWN** | n ≥ 300, coverage ≥ 70%, the interval includes 0, mean > 0 |
| **NEGATIVE** | n ≥ 300, coverage ≥ 70%, mean ≤ 0 |
| **INSUFFICIENT SAMPLE** | n < 300 at the freeze. The deadline is not extended for a pass claim. |
| **INSUFFICIENT COVERAGE** | coverage < 70% at the freeze |

  Whatever the label, the mean, the interval, n and the coverage are published.
- **Only one inferential look counts: the 20 January one.** The 30-day report on Sunday 21 December 2026 is descriptive. It shows the running mean and interval, labelled "interim, not a test", and makes no pass claim.

## 6. Ungraded selections, quitters, and what counts in the denominator

- **Close coverage** = close-graded distinct selections ÷ all distinct selections (section 4) whose game started by the freeze. Close-ungraded selections are excluded from the mean, but every one is listed with its reason:
  - no qualified close;
  - close fetched before the open;
  - market missing at the close;
  - postponed;
  - other.
- **Postponed or moved games.** If `commence_latest` differs from `commence_at_open` by more than 24 hours, the selection is void (reason "postponed"). It counts in the coverage denominator.
- **Game results are not needed for CLV.** A selection with a qualified close and no final score still counts in the primary metric.
- **Quitters.** Every exported selection counts for good, whether or not its tester keeps scanning, keeps exporting or uninstalls. Selections are never dropped because a tester went quiet, did badly, or switched settings. A tester who withdraws consent stops future exports. Rows already received stay in the aggregate as pseudonymous rows, as the consent line in BETA_TESTERS.md must say before the first export.
- **Unobservable selections.** Orders that never left a tester's PC cannot be counted. The report states how many installs signed up, how many ever exported, and the last export date of each, so a reader can judge this gap.
- **Settings and versions.** Testers' differing settings and app versions are part of the product being tested and are not filtered. The results are also shown by `policy_version`, sport and venue, as description only. No subgroup can turn a fail into a pass.

## 7. Kill check at n = 150

The kill check runs once, at the first weekly merge where distinct close-graded selections reach 150.

If the mean CLV at that point is ≤ 0:
- the next public report says so plainly ("CLV is negative at n = 150"), with the number and interval;
- within 7 days the red-team recompute (IRI-72) is run, and the grading pipeline is checked for bugs.

The kill check never ends the test early, never declares a pass, and never changes these rules.

Fixes after the kill check, or at any other time:
- **A bug in the grading** (for example a wrong close or a wrong fee) is fixed, every selection is recomputed, and the fix is disclosed.
- **A change to how Iridium picks** may ship to testers. Selections made before and after it all still count, and the change is reported by `policy_version`.

## 8. Secondary metrics (they do not decide the result)

**Brier score and log loss on the frozen calibration holdout.**
- **The holdout is every distinct selection in this test whose game has a WIN or LOSS result.** Pushes and voids are excluded and counted. These games all start after this document is published, so no model release before then can have seen them.
- **The frozen model is the one in zip `4f600abe`.** If a later release changes the model, only the `model_prob` values written by releases with the `4f600abe` model count here. The rest are reported separately.
- The model's `model_prob` at the open is scored against the outcome, beside two references:
  - Pinnacle's power-devigged probability at the open (`fair_prob_at_open`);
  - the same at the close.
- The report gives each one's Brier score and log loss, the model-minus-reference difference with a game-cluster bootstrap 95% interval (same seed and B), and a ten-bin reliability table.

**Also reported:**
- stake-weighted CLV;
- CLV over all orders of each key;
- CLV under the 180-minute close window;
- beat-the-close rate;
- paper ROI;
- the separate author/company series.

## 9. Data the tester export must carry

One row per paper order. Timestamps are UTC ISO-8601. Every row is exported, including open, unsettled and close-ungraded ones. Filtering happens only in the analysis, never on the tester's PC.

`schema_version`, `install_id` (random UUID, no personal data), `app_version`, `build`, `policy_version`, `entry_policy`, `order_id`, `decision_id`, `sport`, `league`, `event_id`, `home`, `away`, `commence_at_open`, `commence_latest`, `market`, `side`, `point`, `selection_key`, **`open_ts`**, `quote_source_ts`, `quote_captured_ts`, `venue`, `venue_kind`, **`open_price_raw`**, `contracts`, `fee_rate`, `fee_multiplier`, **`venue_fee`**, `open_decimal_effective`, `stake`, `fair_prob_at_open`, `fair_source`, `model_prob`, `ev`, `pin_open_prices` (every outcome: decimal, point, `captured_ts`), **`pin_close_prices`** (every outcome of the market from the last Pinnacle snapshot at or before the start: raw decimal and point), **`pin_close_fetch_ts`**, `pin_close_source_ts`, `result`, `result_version`, `settled_at`, `home_score`, `away_score`, `exported_at`.

## 10. Publication and audit

- The merged data and the analysis script are published with each report:
  - the data are pseudonymous rows, with each `install_id` replaced by a fresh random label;
  - the script computes every number in this document from those rows.
- The red-team script's output (IRI-72) is attached to the 20 January report. It checks for:
  - look-ahead: a close fetched before the open, or an open after the start;
  - survivorship: whether quitters' selections are present;
  - selection bias: ungraded selections by reason, and coverage by tester and sport.
- The SHA-256 of this file is published with it. Each report quotes that hash.

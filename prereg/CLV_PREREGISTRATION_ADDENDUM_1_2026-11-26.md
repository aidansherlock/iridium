# Addendum 1 to the Iridium CLV pre-registration: unattended scheduled scans on the author's install

**Timestamp (UTC):** `2026-11-26T21:43:04Z`
**Amends:** `CLV_PREREGISTRATION_2026-11-21.md`, SHA-256 `18f22e5941ebc64f4bd9506518927d9af390338394726a94ad35653ab0bca386`. That file is unchanged.
**Changes:** one sentence of section 4, the exclusion list. Nothing else changes. That includes the hypothesis, the metric, the close and its 60-minute window, the dedupe key, the entry policy, n ≥ 300, the freeze, the bootstrap, the labels and the kill check.

## 1. Why

Section 4 excludes every install of the author and of company agents. Their rows form a separate, labelled series. So orders that Iridium opens on the author's own PC do not count, even when nobody chooses them.

This addendum admits one narrow part of that install: orders opened by a scan that runs at fixed, published times, without anyone at the keyboard. In those orders the author picks nothing. The app's unchanged entry policy picks.

## 2. What enters the main series

A paper order from install `3f60c71a-c1c2-44f0-86de-2289c26ee63a` enters the main series only if all of these hold:

1. It meets every section 4 entry condition unchanged:
   - `environment = 'live'` and `run_id = 'live'`;
   - `market` is `h2h`, `spreads` or `totals`;
   - `open_ts` is before the game's start.
2. Its `open_ts` is at or after this addendum's timestamp, `2026-11-26T21:43:04Z`.
3. Its `open_ts` falls inside a **scheduled window**:
   - days: Monday, Wednesday, Friday and Sunday;
   - time: 16:00:00 to 16:59:59 UTC (11:00 to 11:59 ET);
   - span: from the first such window after `2026-11-26T21:43:04Z` up to the freeze.
4. It was opened by the unattended command `python -m iridium scan --live`. Windows Task Scheduler starts that command at 16:00 UTC on those days, and it opens orders under the entry policy `first_eligible/1`. Nobody starts, edits or closes these orders by hand.

The closes for these orders come from section 3, unchanged. All Pinnacle snapshots are pooled. On the author's PC they are captured by `python -m iridium close-capture --every 15 --window-min 60`.

## 3. What stays excluded

Everything else stays in the separate, labelled author/company series, as section 4 says:

- every other order from install `3f60c71a-c1c2-44f0-86de-2289c26ee63a`, including any manual scan and any order whose `open_ts` is outside a scheduled window;
- every order from any company agent's install.

## 4. Fixed in advance

- **Missed windows.** A missed or failed scheduled scan is not made up. The report lists every scheduled window and says whether a scan ran in it.
- **No changes to the schedule or the command.** From `2026-11-26T21:43:04Z` to the freeze they do not change. The only change allowed is stopping the series. If it stops, the orders opened before the stop still count, as section 6 says for quitters.
- **Dedupe.** Section 4's dedupe and representative-order rule applies unchanged across all installs, including `3f60c71a-c1c2-44f0-86de-2289c26ee63a`.
- **Reporting.** The 21 December and 20 January reports each give the full result under two rules, each labelled:
  - the original rule: install `3f60c71a-c1c2-44f0-86de-2289c26ee63a` excluded entirely;
  - this addendum: its scheduled-window orders included.

  Each report also gives the share of n that comes from install `3f60c71a-c1c2-44f0-86de-2289c26ee63a`.

  The label decided under this addendum is reported beside the label under the original rule. Neither replaces the other.

## 5. What had been computed before this timestamp

No CLV under the section 2 formula had been computed on any real row eligible for either rule. Here is everything computed before `2026-11-26T21:43:04Z`, so a reader can check this:

- **Live decisions: 54.** None has a qualified close. Stored `clv_points` = 0 values. The section 2 recompute = 0 values.
- **Live paper orders: 1.** It is Lions h2h, opened 2026-11-25T06:46Z, before this timestamp, so it is out under both rules. It is not close-graded.
- **App ledger figures.** The app's own `signal_ledger` grading stored a different quantity for 10 live NBA rows: Pinnacle fair at the close minus Pinnacle fair at the signal, using a 180-minute window. It is not the section 2 CLV, and it has no fee and no venue price. All 10 "closes" were snapshots from the same scan that fired the row. These figures were shown in an internal snapshot report on 2026-11-26 (IRI-101). Those rows all predate this timestamp and enter neither series.
- **Other values.** Demo-environment ledger rows and synthetic test fixtures also carry values. They never count.

Snapshot of the database used for this statement: SHA-256 `4ff79af712ebd35769d95d74ce66c98aed73cae1f960289d4ed5cc9db4fa1e52` (read-only copy, 2026-11-26T03:18Z).

The SHA-256 of this addendum is published with it, next to the pre-registration's.

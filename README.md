# multifamily-occupancy-signal

Estimating true occupancy from a public listing feed, and detecting the
operators who are hiding availability.

A listing feed is not an occupancy report. It is a marketing surface that
each operator controls, and the gap between the two is not noise — it is
a decision the operator made. This repo models those decisions explicitly,
builds estimators that correct for them, and scores the estimators against
a known truth.

Everything here runs on synthetic data. No proprietary or employer data is
used anywhere in this repository.

---

## Why this is hard

Four mechanisms push a listing count away from real vacancy, and two of
them push in opposite directions:

| Mechanism | Effect on a naive count | Why it happens |
|---|---|---|
| **Lead time** | Overstates vacancy | Units are listed 30–60 days before the resident actually moves out |
| **Suppression** | Understates vacancy | A lease-up with 120 vacant units shows 8, because showing 120 looks desperate |
| **Delist vs. lease** | Overstates absorption | A unit vanishing may have leased, or been pulled for a model or renovation |
| **Scrape gaps** | Understates vacancy | A failed collection reads identically to a full property |

The first two partially cancel at an average property. That is the trap:
a naive listing count looks defensible on pooled error while failing
hardest on exactly the properties a customer needs read correctly — the
suppressors and the lease-ups.

## What's here

```
src/generate.py      Synthetic daily scrape. Models unit lifecycle
                     (occupied → notice → vacant) separately from listing
                     visibility, so truth and observation can diverge.
src/classify.py      Labels every unit disappearance as LEASED, PULLED,
                     SCRAPE_GAP or UNRESOLVED.
src/availability.py  Three stacked vacancy estimators.
src/anomaly.py       Scores and ranks properties by suppression evidence.
src/benchmark.py     Scores estimates against truth, cut by archetype.
run.py               Pipeline entry point.
tests/               Invariant tests.
```

## Running it

```bash
pip install -r requirements.txt
python run.py                              # 20 properties, 180 days
python run.py --props 40 --days 365 --save # writes CSVs to output/
pytest tests/ -q
```

## Method

**Disappearance classification.** A unit leaving the feed is ambiguous, so
the three causes are separated by evidence rather than assumption. A scrape
gap is a *property-level* event — every unit vanishes at once. A pull is
*reversible* — the same `unit_id` returns within three weeks. A lease is
terminal. Disappearances inside the trailing censor window are returned as
`UNRESOLVED` rather than guessed; forcing a label on censored observations
is the fastest way to inflate absorption at the end of a series.

**Lead-time adjustment.** The share of listings that are pre-vacancy is
estimated from each property's own `days_listed` distribution rather than a
global constant, then capped at 40% of the shelf.

**Suppression correction.** Absorption per displayed unit — exit rate times
dwell-at-exit, over mean shelf size — benchmarked against the *cohort
median*, not an absolute. Little's Law makes the absolute ratio ≈ 1 by
construction, so only the cross-sectional spread carries information about
who is capping their shelf.

**Suppression ranking.** Three signals, weighted: turnover intensity (0.40),
shelf stability (0.35), and shelf ratio (0.25). Shelf stability is the
useful one — genuine availability is lumpy, while a hard cap produces an
unnaturally flat daily series. Output is a ranked list with the evidence
attached, not a boolean, because a ranking degrades gracefully when the
thresholds drift.

## Results

Representative run, 20 properties over 180 days. Errors in units.

```
                MAE  BIAS  MAPE
est_naive     11.97  8.32  1.48
est_lead_adj   8.68  1.74  1.31
est_full       8.60  2.41  1.35
```

Pooled bias falls from 8.3 units to 1.7 once lead time is handled. The
archetype cut is the more interesting table:

```
  archetype     naive_MAE  naive_BIAS   lead_MAE  lead_BIAS   full_MAE  full_BIAS
 suppressor         11.26      -10.06      14.04     -12.87      12.66     -11.00
transparent         14.35      +13.98       7.32      +6.17       7.48      +6.43
    typical         11.16       +8.97       8.38      +2.28       8.40      +2.94
```

The sign flip is the whole point. The naive estimator overstates vacancy
by 14 units at transparent operators and understates it by 10 at
suppressors. A pooled metric hides that completely.

## Known limitations

Reported plainly, because these are the parts I would work on next:

1. **The suppression correction is weak.** It improves suppressor MAE from
   14.0 to 12.7 and costs a little accuracy everywhere else. It is a real
   but small effect, and I did not tune it into looking better than it is.
2. **Lead-time adjustment overcorrects at suppressors.** It assumes long
   `days_listed` means pre-vacancy inventory, but at a capped shelf a unit
   sits long because it is competing for a scarce slot. The two cases need
   separating before the correction is applied.
3. **The classifier is rule-based.** Price trajectory and concession
   changes before a disappearance are unused and probably carry signal on
   the lease-vs-pull question.
4. **Synthetic truth is generous.** Real feeds have duplicate listings,
   renumbered units, floor-plan-level rather than unit-level pricing, and
   operators who change listing policy mid-stream. None of that is modelled.

## Next

Swap the rule-based classifier for a model trained on price trajectory
and dwell; separate the lead-time and suppression corrections so they stop
fighting each other at capped-shelf properties; validate the suppression
ranking against operator-reported occupancy where it is available.

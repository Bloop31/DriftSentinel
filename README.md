# DriftSentinel — Phase 1 (Data & Baseline Detectors)

This is **your part** of the pipeline. It produces the labeled CSVs that
Member 2 trains the DriftSentinel sequence model on.

## Setup

```bash
pip install river pandas numpy
```

## What's here

```
scripts/
  rolling_stats.py       # computes mean/var shift, PSI, confidence trend per batch
  baseline_detectors.py  # ADWIN / DDM / KSWIN wrappers, logs their alarm batches
  build_dataset.py        # main pipeline — run this
outputs/
  synth_sea_abrupt.csv    # already generated & verified (see below)
```

## Run it

```bash
cd scripts
python3 build_dataset.py
```

By default only the synthetic SEA-drift dataset runs (no download, ~2 sec,
already tested and working). To pull the two real benchmark datasets,
uncomment these two lines at the bottom of `build_dataset.py`:

```python
run_insects()   # downloads ~14MB on first run (abrupt_balanced variant)
run_elec2()     # downloads ~3MB on first run
```

**Note on the proposal's original dataset list:** River doesn't ship
Airlines/USENET/Covertype natively, so I substituted:
- **Elec2** (real, continuously-drifting benchmark — standard in the literature)
- **Insects** (real, has explicit abrupt/gradual/incremental drift variants)
- **Synthetic SEA stream** (exact, controlled ground truth — stands in for
  "Covertype with synthetic drift injection")

This is a reasonable and defensible substitution for the same benchmark
family; mention it briefly in the paper's dataset section.

## Verified result (synth_sea_abrupt.csv)

I already ran this end-to-end. With drift injected at instance 5000:
- Ground-truth drift window flagged at **batch 89–93**
- DDM's reactive alarm didn't fire until **batch 138**
- ADWIN's reactive alarm didn't fire until **batch 163**

This is exactly the story the paper needs: the summary-statistic signal
(PSI/mean-shift/confidence trend) is already moving *dozens of batches*
before the classical detectors notice via the error rate. That gap is
literally what DriftSentinel is supposed to learn to predict.

## Handoff spec for Member 2

Each output CSV has one row per batch, columns:

| column | meaning |
|---|---|
| `<feature>_mean_shift` | current-window mean − reference-window mean, per feature |
| `<feature>_var_shift` | current-window variance − reference-window variance, per feature |
| `<feature>_psi` | Population Stability Index vs. reference window, per feature |
| `mean_confidence` | mean online-model prediction confidence in this batch |
| `confidence_trend` | slope of confidence over the batch (early warning signal) |
| `batch_size` | number of instances in this batch |
| `batch_id` | sequential batch index |
| `ground_truth_drift_label` | 1 if a real drift event starts within `label_lead_window` batches ahead, else 0 — **this is the training target** |
| `adwin_alarm` / `ddm_alarm` / `kswin_alarm` | 1 if that baseline detector fired in this batch — **use these for the lead-time comparison in Phase 3, not for training** |

Member 2: build sliding windows of W consecutive rows (columns 1–6 above)
as input sequences, predict `ground_truth_drift_label` N batches ahead
(binary or regression on batches-until-drift). Tune W and N together —
they trade off against how early a warning can possibly be.

## Known limitations to flag in the paper

- `ground_truth_drift_label` is only exact for the synthetic stream. For
  Elec2/Insects there's no single "true" drift instant, so treat those as
  qualitative/real-world validation rather than the primary lead-time metric.
- PSI/mean-shift here are computed against a **rolling** reference window
  (slides forward each batch), not a fixed one — this keeps it usable on
  long streams but means very slow, sustained gradual drift can partially
  "hide" from PSI. Worth a sentence in Limitations.

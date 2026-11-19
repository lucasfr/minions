# Work log: mrpheus staging made to reproduce YASA (Check 8)

**Minion:** Hephaestus
**Date:** 2026-11-19
**Chat:** HEPHAESTUS - Greg_PSG full PSG notebook
**Continues:** `2026-11-19-hephaestus-greg-psg.md` (Check 7 finding 3 is withdrawn below)
**Repo touched:** `mrpheus` (working tree, not committed)
**Requested by Lucas:** "replicate the entire YASA process on mrpheus. Make it bit perfect with YASA"

## Status

Staging done, verified in the sandbox on a pristine clone, and written into Lucas's `mrpheus` working tree. Models and fixtures were generated on Lucas's machine and the full suite passes there (PASS 560, FAIL 0, WARN 0, SKIP 0). Committed and pushed as `5121001` ("Reproduce YASA staging: ..."); a fresh clone of that commit installs and passes 560 / 0 / 0 / 0.

**Mistake in the commit.** I told Lucas to leave out `R/compute_hrv_freq_windowed.R` (his unfinished work) but gave a command, `git add -A R src tests ...`, that adds the whole `R/` directory, so it was committed and pushed (the repo is public). `NAMESPACE` was not staged, so on GitHub the function is documented with `@export` but not exported; the package builds regardless. Options given to Lucas: leave it; `git rm --cached` it in a new commit (stays in history); or amend the commit and force-push with lease (only safe if nobody has pulled it; GitHub may keep the old commit reachable by hash). Decision (Lucas): leave it as committed.
Slow-wave detection, spindle detection and band power are NOT yet ported to, or re-verified against, YASA 0.8.1.

## Correction to Check 7

Check 7 concluded that `preprocess_psg()` before `stage_epochs()` costs about 8 points of accuracy. That was measured with the wrong classifier (the full EEG + EOG + EMG model fed NA for EMG, where YASA uses its EEG + EOG model). With the right classifier preprocessing does not reduce accuracy on the test night (85.4% against 83.7% for raw), but it still changes 84 of 780 epochs relative to YASA on raw signals (9 epochs with the EOG band-pass widened). Check 7 finding 3 is withdrawn; findings 1, 2 and 4 stand and are explained below.

## Method

Public night only: Sleep-EDF Cassette `SC4001E0` (780 epochs, 100 Hz, and a 256 Hz copy resampled with MNE; EEG Fpz-Cz, EOG horizontal and, for the full model, a stand-in third channel). YASA 0.8.1 / scipy 1.18.1 / MNE 1.13.2 against mrpheus in R, both in the sandbox. `DLBP33N3.edf` was not copied off Lucas's machine. Features were diffed column by column in float32, then posterior probabilities for all eight classifiers.

## Root causes (the earlier "validated" parity script compared at MAE 0.01 and skipped 15 edge epochs, so it could not see them)

1. One classifier instead of eight. YASA chooses `clf_eeg`, `+eog`, `+emg`, `+eog+emg`, each with or without `+demo` (65 to 151 features). mrpheus used the 149-feature EEG + EOG + EMG model and NA-filled missing channels: 50% accuracy EEG-only (YASA 82.8%); 91 of 780 epochs different for EEG + EOG.
2. `.scipy_simpson()` for an even number of points used a 3/8 rule where scipy uses Cartwright's correction (N = 4: 0.5925 against 0.6183). It moved the 0.4-1 Hz and 1-4 Hz band powers and the delta ratios. The old tests used linear data, which every variant integrates exactly.
3. Skew and kurtosis were standardised with the n - 1 standard deviation (kurtosis 24.920 against YASA's 24.939 on one epoch); YASA uses population central moments, NaN for a flat epoch.
4. Zero crossings used `sign()` (an exact 0 counted twice) where antropy uses `np.signbit`.
5. YASA rounds all features to float32 before predicting; mrpheus did not.

The regenerated eeg+eog+emg model is byte-identical (md5) to the shipped `yasa_staging.txt`; the weights were never the problem.

## Result

| Test night, EEG + EOG, 100 Hz | epochs different from YASA | kappa vs YASA | accuracy vs manual |
| --- | --- | --- | --- |
| before | 91 | 0.846 | 83.6% |
| after | 0 | 1.000 | 83.7% |
| EEG only, before | 403 | 0.345 | 50.1% |
| EEG only, after | 0 | 1.000 | 82.8% |

All 16 combinations (8 classifiers x 100 Hz and 256 Hz): the same stage as YASA in every epoch; probabilities equal to about 1e-16 (median 3e-16); the largest difference anywhere 1e-3, in 8 of 12,480 epochs. 97.2% of the non-zero-crossing feature values are bit-identical in float32, the rest within 1.2e-7. Taking only the zero-crossing features from YASA makes the probabilities exactly identical (max difference 0.0, four configurations).

## The one thing that is not bit-identical, and why it cannot be

At the recording's first (sometimes last) sample the zero-phase FIR filter's output is zero by construction (the band-pass has no DC gain). The filtered signals agree to 1.7e-13 everywhere, but sample 0 is +5.8e-15 in MNE and -3.6e-14 in R: floating-point noise, 14 orders of magnitude below the signal. Its sign changes one zero-crossing count by one, and because `robust_scale` uses whole-column quantiles that shifts every epoch's normalised value slightly (about 5e-4 in a 30-epoch fixture, diluted about 26-fold over a full night). Reproducing it needs MNE's FFT arithmetic bit for bit. Agreement is also tied to scipy >= 1.15 (YASA 0.8.1's requirement); older scipy integrates even-length bands differently.

## Files changed in `mrpheus` (working tree)

- `R/stage_epochs.R` rewritten: automatic classifier, `age` / `male`, `model_path` NULL and `.gz`, float32, `attr(, "classifier")`.
- `R/staging_features.R` (`.scipy_simpson`), `src/staging_features.cpp` (`nzc_cpp`, `stat_features_cpp`).
- `tests/testthat/test-staging-features.R` edited (Simpson against scipy values on non-polynomial data, signbit tests, population-moment tests); new `tests/testthat/test-yasa-parity.R`.
- New `data-raw/fetch_yasa_models.py` (creates the eight `inst/models/yasa_*.txt.gz`, about 3.3 MB each, 26 MB in all) and `data-raw/generate_yasa_parity_fixture.py` (creates `tests/testthat/fixtures/yasa_parity_probabilities.csv`, 22 KB; deterministic synthetic recording, YASA reference for all eight classifiers).
- `man/stage_epochs.Rd` (hand-written to match roxygen), `NEWS.md`, `inst/models/README.txt`, `README.md` and `vignettes/sleep-staging.Rmd` (the vignette claim "137 features within MAE < 0.01 ... 9 residual features reflecting irreducible floating-point differences" was wrong and was replaced).
Sandbox verification: patch applied to a pristine clone of ddca1cb, model script run, package installed, full suite 556 expectations, 0 failures, 0 warnings, 0 skipped (483 before). A `.patch` of the same text changes was also produced.

## Follow-up: the parity test failed on Lucas's machine (tolerance error, fixed)

Lucas ran `devtools::test()` after the models and fixture were generated: 555 passed, 1 failed. `eeg_eog_emg_demo` max |dP| was 0.0038 against my tolerance of 2e-3. In the sandbox that configuration gave 5e-13.

**Cause.** My probability tolerance (2e-3) was set from measurements on one platform and was too tight. The platform-dependent boundary effect described above (sign of the filter output at the first and last sample) is not bounded by 1e-3: forcing +/-1 on the first or last zero-crossing count in any channel moves a probability by up to 2.1e-2 on the 30-epoch fixture (single flip in EEG first epoch 1.96e-2, EMG last epoch 2.13e-2; over all 729 combinations 2.0e-2 median). 3.8e-3 is inside that range, so it is consistent with the known mechanism, but it was not diagnosed on Lucas's machine (the first filtered sample was not printed there).

**Why a looser probability tolerance is not enough.** Measured on the same fixture with each old bug restored: old Simpson rule 6.7e-2, old skew/kurtosis scaling 3.3e-2, old zero-crossing (`sign`) 8e-4 (not visible: the synthetic data has no exact zeros; covered by unit tests). A tolerance large enough for the noise (2e-2 and up) sits next to the skew/kurtosis effect, so probabilities cannot separate noise from those bugs.

**Fix.** `tests/testthat/test-yasa-parity.R` now has a feature-level test as the primary guard: the 149 features must equal YASA's to float32 precision (|diff| <= 1e-5 * max(1, |y|); the bugs were >= 7e-4 relative, float32 noise about 1e-7), skipping only the normalised zero-crossing columns and the raw count in the first and last epoch. The probability test is kept as a coarse end-to-end check (max |dP| < 5e-2; stage identical where the top class leads by > 0.1). `data-raw/generate_yasa_parity_fixture.py` also writes `tests/testthat/fixtures/yasa_parity_features.csv` (57 KB). Mutation check in the sandbox: current code passes both tests; old Simpson fails both; old skew/kurtosis fails the feature test and passes the probability test, which is why the feature test is needed. The test file and generator on Lucas's machine were compared byte for byte with the tested versions (identical). Lucas re-ran the generator and the full suite on his machine: PASS 560, FAIL 0, WARN 0, SKIP 0.

## Follow-up 2: first comparison on the DLBP recording

`yasa_check_DLBP33N3.py` (my `kappa()` crashed on a hypnogram with no REM; fixed, tested on a no-REM case, and it now also prints when the mrpheus CSV was written and the channel sd). Its first run compared an old CSV (written 17:00, before the fix). After Lucas re-rendered (CSV written 22:05, fixed staging code, notebook order unchanged: preprocess, EOG regression, then staging): 1,355 scored epochs (145 artefact epochs excluded), agreement 92.0%, kappa 0.82. YASA consensus counts W 952, N2 305, N3 63, REM 20, N1 15; mrpheus W 934, N2 375, N3 43, N1 3, REM 0. All 20 YASA REM epochs are N2 (15) or N3 (5) in mrpheus; YASA per-channel REM is 26 (C3) and 11 (C4), so its own REM evidence is thin (about 11 minutes, 1.5%).

So the notebook's missing REM is not specific to mrpheus: YASA also finds almost none. The remaining disagreement is the same size as preprocessing alone caused on the public night (89.2% agreement, 84 of 780 epochs) but this has NOT been shown on the DLBP data.

New check on the resampler with broadband data (an hour of 256 Hz signal with 39% of its power above 50 Hz): R `.mne_fft_resample` against MNE `resample`, max difference 1.1e-13 (1.8e-14 of the signal sd). The earlier 256 Hz test used data upsampled from 100 Hz, which had no content above 50 Hz, so this closes that gap.

New `Greg_PSG/yasa_vs_mrpheus_raw_DLBP33N3.R` stages the recording with mrpheus on raw signals, after `preprocess_psg()`, and in the notebook's order, and compares each with YASA's CSV (agreement, kappa, epochs differing, probability difference, REM counts), writing `output/DLBP33N3_mrpheus_variants.csv`. Tested on the public night (raw 100.0% / kappa 1.000; preprocess 89.2%, 84 epochs; notebook order 89.4%, 83 epochs); byte-identical on Lucas's machine; not yet run on DLBP33N3. Decision pending: stage from raw signals in the notebook.

## Follow-up 3: DLBP33N3 result, and the notebook change

`yasa_vs_mrpheus_raw_DLBP33N3.R` on Lucas's machine (1,355 scored epochs, 145 artefact epochs excluded; both tools average C3 and C4 probabilities):

| mrpheus input | agreement with YASA | kappa | epochs differing | median max dP | REM (mrpheus / YASA) |
| --- | --- | --- | --- | --- | --- |
| raw signals | 100.0% | 1.000 | 0 | 2.2e-19 | 20 / 20 |
| after `preprocess_psg()` | 92.0% | 0.826 | 108 | 4.2e-2 | 1 / 20 |
| preprocess + EOG regression (notebook order) | 91.5% | 0.813 | 115 | 4.1e-2 | 0 / 20 |

On the real 12.5 h, 256 Hz recording mrpheus reproduces YASA exactly when given the raw signals: the staging code is not the difference, the resampler, filter, features and classifier all agree, and the two readers see the same channel scales (sd over the first 10 min: EOG1 44.43, C3 18.22, C4 18.80, EMG 7.47, identical in R and MNE). The whole gap, including the missing REM, comes from `preprocess_psg()` (and the EOG regression) running before staging. Which preprocessing step removes the REM was NOT isolated on this recording; on the public night the EOG band-pass was the main contributor, and the EMG 10 Hz high-pass is an untested candidate.

**Notebook change (done and re-rendered):** `psg_DLBP33N3.qmd` now stages in a new section 2b, straight after `prepare_psg()` and before `preprocess_psg()`, on the raw labels `C3:A2` and `C4:A1`; section 6 keeps the agreement table, consensus, artefact masking and everything after; the old staging chunk became a one-line `staging-check`. Nothing downstream (artefacts, band power, events, cardiac) changes, and peak memory is not increased (the raw object is released when `psg` is overwritten by the preprocessed one).

**Result of the re-render** (`output/DLBP33N3_hypnogram.csv` written 22:38; `yasa_check_DLBP33N3.py`): the notebook's hypnogram and YASA's agree on all 1,355 scored epochs, 100.0%, kappa 1.00, a purely diagonal confusion matrix; counts identical on both sides: W 952, N2 305, N3 63, REM 20, N1 15. Spindle and slow-oscillation epochs selected from this hypnogram (N2 and N3) now come from YASA-equivalent staging, so those section 8 numbers differ from earlier renders; the slow-oscillation detector itself is still not YASA's.

## Follow-up 4: the NA (artefact) epochs in the DLBP33N3 hypnogram

Flagged by the amplitude rule alone (`hf_percentile = 1` disables the high-frequency rule): peak-to-peak > 500 uV on C3 or C4, measured on the preprocessed, EOG-regressed signal. `artefact_epochs_DLBP33N3.R` on Lucas's machine: 145 of 1,500 epochs (9.7%), all `amplitude`. Underlying stage the classifier gave: W 120, N2 21, REM 3, N1 1, N3 0, so they are not large N3 slow waves (the hypothesis I raised). Runs: 83 single epochs, 10 pairs, 5 of 3, 3 of 4, 3 of 5. Peak-to-peak across all epochs: median 413 uV, 90th percentile 491, 95th 609, 99th 842, max 1466; 66 of the 145 flags are 500-600 uV, 3 epochs exceed 1000 uV. The sd of the raw signals over the first 10 minutes was 18 uV, which implies about 135 uV peak-to-peak for noise of that size; the public night's reference (Sleep-EDF SC4001E0, same quantity) has median 145 uV, 99th percentile 287 uV, 0 epochs over 500 uV, and preprocessing and EOG regression leave it unchanged (rank correlation 0.99).

Interpretation (not yet established): a median of 413 uV in a night that is 71% wake means the 500 uV limit is cutting through the middle of this recording's amplitude distribution, not separating artefact from a quiet baseline. Whether the amplitude is in the raw recording or created by preprocessing / EOG regression is NOT known. New `Greg_PSG/artefact_pipeline_DLBP33N3.R` measures it on the raw signal, after `preprocess_psg()` and after the regression, with median amplitude by underlying stage; tested on the public night; not yet run on DLBP33N3.

**Result of `artefact_pipeline_DLBP33N3.R`** (Lucas's machine; largest peak-to-peak of C3 and C4 per epoch, uV):

| | raw | after preprocess | after EOG regression |
| --- | --- | --- | --- |
| median | 904 | 430 | 413 |
| 90th / 95th / 99th percentile | 1600 / 1600 / 1600 | 473 / 555 / 823 | 491 / 609 / 842 |
| max | 1600 | 1492 | 1466 |
| epochs over 500 uV (of 1,500) | 837 | 130 | 145 |

Median raw peak-to-peak by underlying stage: N1 214, N2 166, N3 206, REM 174, W 1600 (preprocessed: 175/139/172/152/449; regressed: 167/138/168/145/442). Spearman raw vs regressed 0.84.

The raw amplitude has a hard ceiling at exactly 1600 uV (the 90th percentile, 95th, 99th and the maximum are all 1600) and it is confined to epochs the classifier calls wake (median 1600); sleep epochs have normal raw amplitudes (140-215 uV). A fixed ceiling is the signature of clipping at the converter's range (+/-800 uV would give exactly 1600 peak-to-peak), but percentiles alone do not prove it; `clipping_DLBP33N3.R` (counts samples exactly on the extremes, by stage and by hour; tested on the public night, which shows none; not yet run on DLBP33N3) will. `preprocess_psg()`'s high-pass reduces the saturated swings to the 430 uV median, so the 500 uV rule was measuring filter leftovers: it flags 145 epochs while 837 (56%) exceed 500 uV in the raw signal. The cause of the saturation (movement, lead or electrode contact, gain) is unknown. The sleep period itself looks clean, so staging there rests on sound signal; whether the wake epochs are true wake or invalid signal is not established.

**Result of `clipping_DLBP33N3.R`** (Lucas's machine): both EEG channels span exactly -800.0 to +800.0 uV (range 1600): clipping at the converter's limits. Samples on the rails: C3 48,436 at +800 and 52,015 at -800; C4 28,136 at +800 and 2,408 at -800. In each channel the single most frequent value is 0.0122072 uV: 4,961,826 samples on C3 (43%) and 5,058,009 on C4 (44%) of 11,522,304, i.e. a constant (flat-line) value for about 5.4 hours' worth of samples; the next most frequent values occur about 3,000 times. Epochs with more than 0.1% of samples on a rail in either channel: 728 of 1,500 (48.5%): by underlying stage N1 0/16, N2 5/326, N3 0/63, REM 1/23, W 722/1,072; by hour from the start (epochs clipped / 120): 5, 5, 19, 10, 2, 0, 76, 120, 120, 120, 120, 71, then 60 of 60. Reading: for roughly the last six hours of the recording (hours 6/7 to 12, about 03:00 to 08:30 if the recording starts near 20:00) the EEG is flat or railed, so the long wake block there is not wake but absence of signal, staged W because that is what such a signal looks like to the classifier. The clipping count does not capture fully flat epochs (zero amplitude, so neither the amplitude rule nor the rail count flags them); how many there are, and the exact valid window, are NOT yet known. `signal_validity_DLBP33N3.R` classifies every epoch (flat / clipped / ok per channel; invalid if either channel is not ok) and lists the valid stretches; tested on a synthetic recording with planted flat and clipped epochs (counts and stretches matched exactly); not yet run on DLBP33N3.

**Result of `signal_validity_DLBP33N3.R`** (Lucas's machine; flat = more than half of an epoch's samples identical, clipped = more than 0.1% on the extremes; invalid if either channel is not ok): per channel ok 784 / 786, flat 686 / 687, clipped 30 / 27 of 1,500. Invalid in either channel 733 (48.9%); valid in both 767. By hour from the start (ok / flat / clipped, of 120): 0: 115/0/5; 1: 115/0/5; 2: 101/0/19; 3: 110/0/10; 4: 118/0/2; 5: 120/0/0; 6: 44/74/2; 7-10: 0/120/0 each; 11: 44/73/3; 12 (60 epochs): 0/60/0. By underlying stage (valid / invalid): N1 16/0, N2 321/5, N3 63/0, REM 22/1, W 345/727. 18 valid stretches of at least 10 epochs, the longest 185 epochs (1.54 h), then a 33-epoch stretch near the end; 97 further valid epochs lie outside those stretches. So the EEG is flat on both channels from partway through hour 6 to the end of hour 10, with only a short valid stretch in hour 11, and nearly all of the 727 invalid wake epochs are no-signal epochs; the stages with sleep are almost entirely valid. The earlier 'clipped' counts for hours 7-10 are the same epochs: flat stretches containing a few samples on the rails (the flat check takes precedence).

**Clock-time discrepancy (investigated; cause in Lucas's render environment NOT reproduced).** The EDF header's start time is 2024-10-16 21:05:24 BST (= 20:05:24 UTC; `attr(x, "tzone")` is NULL), which is what the validity script prints (21:05). The hypnoR plot axis put the first epoch near 20:03-20:05, an hour earlier, so the plot's clock labels were about an hour off the file's wall clock; clock times read off that plot (for example 'sleep 23:00 to 02:30' or 'flat from about 03:00') are about an hour early, and the script's clock (valid signal until about 03:24, flat until about 08:16 with a short stretch near 08:16-08:32) is the one that matches the header. hypnoR passes the start time through unchanged (`as.POSIXct(start_time) + (epoch - 1) * eps`), and in my sandbox (ggplot2 4.0.4) a time with no zone attribute is labelled in local time, so the cause of the offset in the render was not found. Fix applied in the notebook: the start time is rebuilt from the header's wall-clock string with an explicit zone (`as.POSIXct(format(startTime, "%Y-%m-%d %H:%M:%S"), tz = "Europe/London")`), which in the sandbox gives epoch 1 at 21:05:24, epoch 1500 at 09:34 and ticks at 22:00, 00:00, 02:00 under four different process time zones. Not yet seen in Lucas's render: after re-rendering, the first hypnogram point should sit at 21:05 and the last near 09:34.

**Notebook change (done, not yet re-rendered):** new section 2c (`epoch_signal_state()`, run on the raw signals before preprocessing; flat or clipped per channel; `no_signal` = either channel flat; `clipped_ep` = clipped and not flat; analysis window = first epoch with a signal to the epoch before the first stretch of 120 consecutive no-signal epochs after it). Hypnogram stage is NA only for no-signal epochs; `hyp$flagged` = (clipped or over 500 uV after filtering) and not no-signal, which keeps those epochs out of band-power-by-stage, `hypno_int`, spindle/slow-oscillation detection and the N2/N3 minutes, without hiding their stage. Sleep summary and stage minutes are computed over the window; the C3-vs-C4 agreement over epochs with a signal; HRV text updated; saved CSV now has no_signal, clipped, amplitude_flag, in_window and flagged columns (the old `artefact` column is gone, so `yasa_check_DLBP33N3.py` now compares every epoch that has a stage). The logic (validity classification, window, edge cases, sleep summary) was tested on a synthetic recording with planted flat and clipped epochs; the notebook itself has not been rendered. Section 8 densities and the band-power table will differ from the previous render because clipped epochs are now also excluded.

Cause of the flat signal (electrodes or leads off, device, or disconnection) is unknown. Thresholds (half of an epoch identical, 0.1% on the rails, 60 minutes for the window) are choices, checked only against this recording's numbers.

**Consequences before the notebook change:** time in bed, sleep efficiency and wake after sleep onset were inflated by about six hours of no-signal epochs; the N2/N3/REM epochs, and so the section 8 densities per NREM minute, come from the valid early period; ECG-based heart rate does not depend on the EEG state. YASA's `art_detect` remains a possible port for the amplitude flag.

## Open items

- On Lucas's machine, from the `mrpheus` root: `python3 data-raw/fetch_yasa_models.py`, `python3 data-raw/generate_yasa_parity_fixture.py`, `git rm inst/models/yasa_staging.txt`, `devtools::test()`, commit. Decide whether 26 MB of model files is acceptable (the four `+demo` models are only needed with age and sex: about 13 MB without them).
- Re-render the DLBP notebook and run `python yasa_check_DLBP33N3.py`. With EEG + EOG + EMG the notebook already used the matching classifier, so the model fix alone is not expected to explain the absent REM; the feature fixes change the inputs slightly. Unknown until run.
- To match YASA, stage from signals that have not been through `preprocess_psg()` (the notebook preprocesses first).
- Not done: YASA's `sw_detect`, `spindles_detect` and `bandpower` ported / re-verified. `compute_band_power()` and `compute_temporal_bandpower()` use `.scipy_simpson()` and so change where a band holds an even number of bins; their parity with YASA 0.8.1 was not re-checked. The slow-wave detector in mrpheus remains a zero-crossing implementation, about 20% below YASA's count.
- Confirmed by reproduction in the sandbox: `read_edf(path, channels = <one channel>)` followed by `prepare_psg()` fails ("Flat/bad channels detected", "No valid EEG channel found"), so a one-channel EDF cannot be staged with the EEG-only classifier. Cause not yet located in `read_edf()`; not fixed.
- Decide whether to keep the classifier's stage for every epoch and carry artefact as a flag (and whether to define artefact on the raw signal, e.g. saturation, rather than on the preprocessed signal).
- Operational note: the Claude Desktop file tool timed out once on a write and a read, then recovered; the first write did not land and was repeated.

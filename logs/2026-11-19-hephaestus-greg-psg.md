# Work log: Greg_PSG full PSG notebook

**Minion:** Hephaestus
**Date:** 2026-11-19
**Chat:** HEPHAESTUS - Greg_PSG full PSG notebook
**Repos touched:** `Greg_PSG`, `mrpheus`
**Status:** notebook written; renders through the spindle step of the `events` chunk; three `mrpheus` fixes (resampler: committed; spectrogram: tests pass, not committed; slow oscillation filter: tests pass, not committed); a separate slow oscillation polarity problem fixed in `mrpheus` (full `devtools::test()` passes; commit not confirmed); YASA-aligned slow oscillation defaults (Fix 5; full `devtools::test()`: PASS 483, FAIL 0, WARN 0; not yet committed); notebook not yet re-rendered past `events`

## Task

Develop a full PSG analysis of `Greg_PSG/DLBP33N3.edf` from the existing Quarto notebook (`explore_DLBP33N3.qmd`), using the `mrpheus` package. Use the C3 and C4 channels and any other relevant channels, as far as the implemented functionality allows.

## Files changed

| File | Change |
| --- | --- |
| `Greg_PSG/psg_DLBP33N3.qmd` | Created. Contains the full pipeline. |
| `mrpheus/R/staging_features.R` | Added a downsampling branch to `.mne_fft_resample()` (see Fix below). Upsampling path untouched. |
| `mrpheus/tests/testthat/test-mne-fft-resample.R` | Created. Tests for `.mne_fft_resample()`. |

`explore_DLBP33N3.qmd` was not modified. `mrpheus/NEWS.md` was not updated.

## Commands run

No R code has been executed by Hephaestus; the tools available could read and write files on the local machine but not run R. Lucas rendered the notebook and reported the error below.

In the sandbox (not on the local machine): `pip install mne`, then a Python script that mirrors the new R resampling logic with numpy and compares it with `mne.filter.resample`. Results are under Fix.

Files read to establish function signatures and behaviour: `NAMESPACE`, `README.md`, `NEWS.md`, `DESCRIPTION`, and these files in `mrpheus/R/`: `read_edf.R`, `prepare_psg.R`, `preprocess_psg.R`, `correct_eog.R`, `detect_artifacts.R`, `stage_epochs.R`, `staging_features.R` (first 140 lines), `compute_band_power.R`, `compute_temporal_bandpower.R`, `compute_spectrogram.R`, `compute_spindles.R`, `compute_slow_oscillations.R`, `detect_qrs.R`, `compute_hr_signal.R`, `compute_hrv_sleep.R`, `detect_apneas.R`, `export_hypnogram.R`, `mrpheus-package.R`.

The EDF header was not opened, so sample rates and physical units of the six channels are unconfirmed. The notebook contains guards for both.

## What the notebook does

- Reads 6 of 44 channels: `C3:A2`, `C4:A1` (renamed `C3`, `C4`), `EOG1`, `EOG2`, `EMG`, `ECG 2`.
- `read_edf` > `prepare_psg` > `preprocess_psg` > `correct_eog_regression` > `detect_artifacts` > `stage_epochs` (once on C3, once on C4).
- Spectral: `compute_band_power`, `compute_temporal_bandpower`, `compute_spectrogram`.
- Events: `compute_spindles`, `compute_slow_oscillations`, on both channels, N2 and N3 only.
- Cardiac: `detect_qrs` (in 30-minute chunks), `compute_hr_signal`.
- Writes CSV and RDS files to `Greg_PSG/output/` when rendered.

## Decisions

- **Stubs not used.** `detect_apneas`, `compute_odi` and `compute_hrv_sleep` are stubs in mrpheus 0.1.4.9000. The flow and RIP channels were therefore not read.
- **EOG regression, not ICA.** Only two EEG channels are read, so `correct_eog_ica` was not used.
- **High-frequency artefact criterion disabled** (`hf_percentile = 1`). The EEG is band-limited to 35 Hz by `preprocess_psg`, so a 40-100 Hz criterion has no physiological meaning after that step. Epochs are flagged on peak-to-peak amplitude (500 uV) only.
- **Final hypnogram** is the argmax of the mean of the C3 and C4 posterior probabilities. Agreement and Cohen's kappa between the two runs are reported.
- **`stages` arguments** to `compute_spindles` and `compute_slow_oscillations` are epoch indices, so N2 and N3 epochs are selected first. `compute_temporal_bandpower` takes integer codes (W = 0 to REM = 4, artefact = -1).
- **QRS detection chunked** at 30 minutes, because the benchmark in `NEWS.md` is about 470,000 samples and a full night is roughly 25 times that. Thresholds restart at chunk boundaries.

## Over-forging

Per the vicissitudes in `minions/hephaestus.md`, the following were built beyond the request and are not required for a full PSG from the existing mrpheus functions:

- Sleep summary metrics computed in base R (TIB, TST, sleep efficiency, SOL, REM latency, WASO, stage minutes).
- Per-stage mean HR, SDNN and RMSSD computed in the notebook from R-peak times.
- Two-channel consensus staging and the kappa comparison.
- The night overview figure.
- Writing outputs to `output/`.
- Header guards for sample rate and units.

If the aim was a direct port of the existing notebook, these should be removed and a patch to `psg_DLBP33N3.qmd` requested.

## Fix: staging failed on 256 Hz data

**Reported by Lucas after first render**, in chunk `staging` (`psg_DLBP33N3.qmd` lines 199-204):

```
Error in `X_new[seq.int(new_len - n_neg_total + 1L, new_len)] <- X[seq.int(n_pos, orig_len)]`:
! only 0's may be mixed with negative subscripts
 1. mrpheus::stage_epochs(...)
 2. mrpheus:::.extract_staging_features(...)  at stage_epochs.R:79
 3. filter_and_epoch(eeg_ch)                  at staging_features.R:427
 4. mrpheus:::.mne_fft_resample(sig_raw, up = SR_TARGET/sr_orig)  at staging_features.R:416
```

**Cause.** `.mne_fft_resample()` only implements upsampling (its own comments say so). Staging resamples every channel to 100 Hz, so 256 Hz input gives `up` = 100/256 and `new_len < orig_len`. The zero-padding code then builds a negative index range. Sleep-EDF data is already at 100 Hz, so this path was never exercised.

**Fix.** In `.mne_fft_resample()`, after the forward FFT and before the existing scaling, a branch for `new_len < orig_len` truncates the spectrum as MNE's `_fft_resample` does with `shorter = TRUE`: the new Nyquist bin is doubled (even `new_len` only), all bins are scaled by `new_len / orig_len`, the one-sided spectrum is rebuilt as a Hermitian full spectrum, and the inverse FFT is trimmed with the existing `to_rm_start` / `to_rm_end`. The upsampling code below the branch is unchanged.

**Verification done.**
- Lucas ran `command Rscript -e 'devtools::test(filter = "mne-fft-resample")'` in `mrpheus`: FAIL 0, WARN 0, SKIP 0, PASS 8. (`Rscript` is shadowed by a shell function on this machine that fails with `command not found: \n\n`; `command Rscript` or `/usr/local/bin/Rscript` bypasses it.)
- Python mirror of the R logic vs `mne.filter.resample(up = 100/256, down = 1, npad = "auto", method = "fft")` (MNE 1.13.2): maximum absolute difference about 5e-14 on four lengths (7680, 460800, 123457, 100001). Output lengths identical.
- Same mirror on an upsampling case (up = 2.5): about 2e-15.
- Tolerances in the new test file were set from the same mirror: 10 Hz sinusoid, 256 to 100 Hz, about 9e-4 at the edges and 3e-6 in the interior; DC offset error about 4e-10; upsampling interior error about 2.6e-4.

**Verification not done.**
- The full `mrpheus` test suite has not been run since the change, so the existing 371 tests are unconfirmed.
- Staging features at 256 Hz have not been checked against Python YASA. The package's feature-parity validation (`data-raw/validate_feature_parity.R`) covers 100 Hz input only, as far as was read.
- The notebook past the `staging` chunk has not been rendered, so later failures are likely to surface one at a time.

**Committed.** Lucas committed the fix and test in `mrpheus` as "Fix FFT resampler for downsampling" (3d36913), and `git status` then showed `main` up to date with `origin/main`. The two remaining changes in `mrpheus` at that point, a modified `NAMESPACE` and an untracked `R/compute_hrv_freq_windowed.R`, are not part of this work.

## Fix 2: `compute_spectrogram()` failed

Second render reached 68% and failed in chunk `spectrogram` (`psg_DLBP33N3.qmd` lines 368-390):

```
Error in `gsignal::pwelch()`:
! unused argument (noverlap = novlp)
 1. mrpheus::compute_spectrogram(psg, channel = "C3")
 2. base::lapply(...)   at compute_spectrogram.R:45
```

The chunks before it (staging, agreement, band power, temporal band power) ran.

**Cause.** `gsignal::pwelch()` has no `noverlap` argument. Its argument is `overlap`, a fraction with 0 <= overlap < 1 (confirmed in the gsignal 0.3-5 reference manual). `compute_spectrogram()` converted the fraction to a sample count and passed it as `noverlap`.

**Fix.** In `mrpheus/R/compute_spectrogram.R`, both `pwelch()` calls now pass `overlap = overlap` directly, and the unused `novlp` line is removed. Behaviour is the intended one: 50% overlap of the `nfft`-sample window.

**Test.** Created `mrpheus/tests/testthat/test-compute_spectrogram.R` (synthetic 10 Hz sinusoid: runs, one row per epoch, peak within 0.5 Hz of 10 Hz). `compute_spectrogram()` had no tests before. Lucas ran `devtools::test(filter = "compute_spectrogram")`: all pass.

**Not checked.** `detect_artifacts()`, `compute_band_power()` and the other callers of gsignal were not affected: `detect_artifacts()` calls `pwelch()` without an overlap argument and ran in the notebook.

## Fix 3: `compute_slow_oscillations()` failed at 256 Hz

Third render failed at 72% in chunk `events` (`psg_DLBP33N3.qmd` lines 400-405). `compute_spindles()` on both channels had already run; the failure was in `compute_slow_oscillations()`:

```
Error in `solve.default()`:
! system is computationally singular: reciprocal condition number = 4.39543e-18
 5. mrpheus::compute_slow_oscillations(psg, channel = ch, stages = nrem_idx)
 8. gsignal::filtfilt(bf, sig)   at compute_slow_oscillations.R:66
14. base::solve(IminusA, B)     (inside gsignal::filter_zi)
```

**Cause.** The 0.5-2 Hz band-pass was a single 8th-order Butterworth in transfer-function form. At 256 Hz the normalised edges are about 0.004 and 0.016, so the coefficients are badly conditioned and `filter_zi()` cannot solve for the initial state. Reproduced in Python (same system, scipy): rcond 4.0e-18 at 256 Hz, 8.5e-15 at 100 Hz. R's limit is about 2.2e-16, so the function worked at 100 Hz and failed at 256 Hz. The existing tests only exercised `detect_so_candidates_cpp()`, so none reached the filter.

**Fix.** In `mrpheus/R/compute_slow_oscillations.R`, the band-pass is now a cascade of a 4th-order high-pass at `freq_so[1]` and a 4th-order low-pass at `freq_so[2]`, each applied with `filtfilt()`, designed once before the epoch loop instead of once per epoch. Conditioning at 256 Hz in Python: high-pass 2.1e-10, low-pass 5.6e-8.

**Behavioural difference.** Compared in Python at 100 Hz (zero-phase, so magnitude squared): identical at both band edges (-6.02 dB at 0.5 and 2 Hz); passband within 1 dB (cascade -0.34 dB at 0.75 Hz, -0.83 dB at 1.5 Hz); weaker rejection outside the band (-29 dB against -40 dB at 3 Hz, -48 against -64 dB at 4 Hz). Slow oscillation counts and amplitudes may differ slightly from the old design at 100 Hz.

**Test.** Created `mrpheus/tests/testthat/test-compute_slow_oscillations.R`: synthetic 1 Hz, 100 uV oscillation through the full function at 256 Hz and 100 Hz; checks it runs, finds events, and that peak-to-peak amplitudes stay inside the 75-500 uV window.

**Test result and a second problem.** Lucas ran the full `devtools::test()`: PASS 424, FAIL 2, WARN 2. The two failures were this new test (`nrow(so) > 0` false at both rates). The filter error is gone, but the function returned no events for a clean 1 Hz sine. Cause, reproduced in Python: `detect_so_candidates_cpp()` pairs zero crossings in steps of two and assumes the first crossing in the epoch starts a negative half-wave; it never checks polarity. The test used `-sin`, which after filtering starts negative, so the first crossing is neg-to-pos and every triplet is off by half a cycle (0 events). `+sin` gives 28 events and a `-sin` that starts at exactly 0 gives 29. The existing C++ tests all use signals starting at exactly 0, so they never exposed this. The test was changed to `+sin` with a comment explaining why. Re-run by Lucas: `devtools::test(filter = "compute_slow_oscillations")` gives PASS 4, FAIL 0, WARN 0; 84 slow oscillations detected, which is 28 per epoch over 3 epochs and matches the Python mirror. The 2 warnings in the earlier full run did not come from this file and were not inspected.

## Fix 4: slow oscillation polarity (requested by Lucas)

**Change.** In `mrpheus/src/event_detection.cpp`, `detect_so_candidates_cpp()` now walks the zero crossings with a `while` loop. A triplet is accepted as (negative half-wave, positive half-wave) only if the signal is negative just after `zc[k]`; crossings that start a positive half-wave are skipped one at a time, and once in phase the walk steps by 2 as before. The loop bound (`k < nzc - 3`) and all duration and amplitude criteria are unchanged. The R reference `.so_r()` in `tests/testthat/test-event-detection.R` was changed to the same rule.

**Checked in a Python mirror (not in R/C++):** for the filtered 1 Hz test oscillation at 256 and 100 Hz, `-sin` gives 28 events (old: 0) and `+sin` gives 28 (old: 28); a `-sin` starting at exactly 0, as in the existing C++ tests, gives identical output old and new (29 events); across 200 noisy signals, 578 events and none with neg_peak >= 0 or pos_peak <= 0.

**Tests.** Four added to `test-event-detection.R` (epoch starting negative finds events; negative peak < 0 and positive peak > 0 for both polarities; counts agree within 1 across polarities; C++ matches `.so_r()` on 20 noisy signals). `test-compute_slow_oscillations.R` was changed back to `-sin` so the full function is covered for the polarity that failed. Lucas ran the full `devtools::test()` after this change (which recompiles the C++): all passing.

**Effect on results.** Epochs that start negative now yield events where they previously yielded none, so slow oscillation counts and densities rise relative to any earlier run, by up to roughly a factor of two on real data. Positive-starting epochs are unchanged.

**Not changed, noticed.** The loop bound `k < nzc - 3` requires a fourth crossing after the triplet, so the last half-wave pair in an epoch can be dropped (kept for parity with the existing R reference).

## Detector versus YASA and MNE (answer to Lucas)

MNE has no slow oscillation detector. The Python counterpart is `yasa.sw_detect` (YASA 0.8.1, which uses MNE for filtering). `compute_slow_oscillations()` is not equivalent to it, before or after the polarity fix (table: defaults before Fix 5):

| | mrpheus default | YASA 0.8.1 default |
| --- | --- | --- |
| Band | 0.5-2 Hz Butterworth (cascade) | 0.3-1.5 Hz FIR (MNE) |
| Negative duration | 0.1-1.5 s | 0.3-1.5 s |
| Positive duration | 0.1-1.0 s | 0.1-1.0 s |
| Negative trough amplitude | none | 40-200 uV |
| Positive peak amplitude | none | 10-150 uV |
| Peak-to-peak | 75-500 uV | 75-350 uV |
| Filtering | per 30-s epoch | continuous signal, then stage mask |

### Measured comparison on real data (Python, not R)

Lucas asked whether the differences matter. Measured on one public recording: Sleep-EDF Cassette `SC4001E0`, channel Fpz-Cz, 100 Hz, N2 + N3 only (235 min, from the supplied hypnogram, Rechtschaffen-Kales stage 3 and 4 merged). YASA 0.8.1 `sw_detect` was run on the continuous signal. The mrpheus side is a Python mirror of `compute_slow_oscillations()` (30-s epochs, cascade filter, polarity fix), using scipy `filtfilt` rather than gsignal's, so it approximates the R output and has not been compared with it.

| Run | Events | Per NREM min |
| --- | --- | --- |
| YASA, defaults | 1374 | 5.85 |
| mrpheus mirror, defaults | 2039 | 8.68 |
| mrpheus mirror with YASA's duration and PTP limits | 1983 | 8.44 |
| YASA with mrpheus's criteria (band, durations, PTP, no trough/peak limits) | 2594 | 11.04 |

Agreement (negative peaks within 0.5 s): 53% of mrpheus-default events have a YASA-default counterpart; 79% of YASA-default events are found by mrpheus. The 953 mrpheus events with no counterpart have median PTP 87 uV (the threshold is 75) and median negative half-wave 0.42 s.

Which YASA setting drives the count (one change at a time from YASA defaults, 1374 events): removing the trough (40-200 uV) and peak (10-150 uV) limits +31%; band 0.5-2 Hz instead of 0.3-1.5 Hz +33%; negative duration 0.1-1.5 s instead of 0.3-1.5 s +0%; PTP up to 500 instead of 350 uV +0%. mrpheus's duration and PTP windows therefore do not matter on this night (0 events above 350 uV, 56 with negative duration under 0.3 s); the band and the missing trough/peak limits do.

With identical criteria, the mirror still finds 20-28% fewer events than YASA (2039 against 2594; 72% of YASA's events are found). This is not explained by the loop bound (`k < nzc - 3`; relaxing it adds 34 events) or by per-epoch filtering (a continuous filter adds 8). Cause not identified; the pairing logic differs from YASA's, and YASA's source was not traced.

Limits: one subject, one night, one channel at 100 Hz, healthy-sleeper Sleep-EDF data, not the DLBP recording. Not a validation of either detector.

## Fix 5: YASA-aligned slow oscillation defaults (requested by Lucas: "maximise comparability")

**Changes** in `mrpheus/R/compute_slow_oscillations.R`, `man/compute_slow_oscillations.Rd` (hand-edited to match roxygen output; `devtools::document()` will regenerate it) and `NEWS.md`:
- Defaults now those of `yasa.sw_detect()`: `freq_so = c(0.3, 1.5)`, `duration_neg_s = c(0.3, 1.5)`, `amp_ptp_threshold_uv = c(75, 350)`; `duration_pos_s` unchanged (0.1-1.0).
- New arguments `amp_neg_uv = c(40, 200)` and `amp_pos_uv = c(10, 150)` (`NULL` disables). Applied in R to the C++ candidates (trough = -neg_peak_uv, peak = pos_peak_uv), which is equivalent to applying them in the walk because they do not affect pairing. The C++ signature is unchanged, so no `RcppExports` regeneration is needed.
- New argument `transition_hz = 0.2`. YASA's MNE FIR has -6 dB points half a transition band outside `freq_so` (0.2 and 1.6 Hz); the Butterworth cascade's -6 dB points (cutoffs, after the zero-phase double pass) are placed there.
- The old behaviour is reproducible with `freq_so = c(0.5, 2)`, `duration_neg_s = c(0.1, 1.5)`, `amp_ptp_threshold_uv = c(75, 500)`, `amp_neg_uv = NULL`, `amp_pos_uv = NULL`, `transition_hz = 0` (tested).
- Invalid bands (high-pass cutoff <= 0, low-pass cutoff >= Nyquist) abort with a message.
- `psg_DLBP33N3.qmd`: the unit-guard warning text updated to the new limits.

**Edge choice, measured (Python mirror, Sleep-EDF SC4001E0, N2 + N3, 235 min; YASA defaults give 1374 events, 5.85 per min).** Precision = share of mirror events with a YASA event within 0.5 s; recall = share of YASA events found:

| Run | Events | Per min | Precision | Recall | F1 |
| --- | --- | --- | --- | --- | --- |
| old mrpheus defaults | 2039 | 8.68 | 53% | 79% | 0.64 |
| new criteria, Butterworth edges 0.3-1.5 Hz | 969 | 4.12 | 89% | 63% | 0.74 |
| new criteria, edges 0.25-1.55 Hz | 1048 | 4.46 | 90% | 69% | 0.78 |
| new criteria, edges 0.2-1.6 Hz (chosen) | 1096 | 4.66 | 90% | 72% | 0.80 |

So the new defaults undercount against YASA by about 20% (4.66 against 5.85 per minute) where the old ones overcounted by 48%; agreement of the events found rose from 53% to 90%.

**Cause of the remaining gap (YASA source read, yasa 0.8.1).** `sw_detect` finds every local minimum and maximum of the filtered signal, applying the trough (40-200 uV) and peak (10-150 uV) limits to the peaks themselves; pairs each negative peak with the nearest following positive peak; applies the peak-to-peak limits; then takes each phase's duration from the zero crossings around its peak and requires the two phases to be adjacent. mrpheus instead walks zero-crossing triplets (one trough per negative half-wave, one peak per positive half-wave) and applies the amplitude limits afterwards. A half-wave with two qualifying local minima gives YASA two candidates and mrpheus one, and skipping peaks below 10 uV changes which positive peak is the nearest. The mechanism fits the undercount but its size was not measured directly. A faithful port of YASA's pairing would close the gap; that is a rewrite of the detector and was not done.

**Conditioning of the 0.2 Hz high-pass (Python, rcond of the `filter_zi` system; R fails below about 2.2e-16):** 100 Hz 2.3e-10, 256 Hz 5.2e-12, 512 Hz 3.3e-13, 1000 Hz 2.2e-14. Safe up to about 512 Hz, with a margin of only about 100 at 1000 Hz (documented in the help page).

**Tests changed** in `test-compute_slow_oscillations.R`: the default-limits test now checks the YASA limits; added tests for the trough and peak limits and for `NULL`, for the old criteria through the arguments, and for rejection of impossible bands. Expected amplitude of the 100 uV test oscillation after filtering is about 97 uV (computed from the Butterworth responses). Lucas ran the full `devtools::test()` after Fix 5: PASS 483, FAIL 0, WARN 0, SKIP 0. The 2 warnings seen in the earlier full run no longer appear.

**Not verified.** The Python mirror approximates the R filter (scipy rather than gsignal `filtfilt`). Gaps and counts above are for one night, one channel, 100 Hz, healthy-sleeper data, not the DLBP recording.

## Change 6: hypnogram plot via hypnoR (requested by Lucas)

`Greg_PSG/psg_DLBP33N3.qmd`: the hand-written ggplot hypnogram (section 6) now calls `hypnoR::plot_hypnogram(hypnogram, title = ...)` on the `mrpheus_hypnogram` already built by `export_hypnogram()`; `hypnoR` is loaded with `devtools::load_all("../hypnoR")` alongside mrpheus (sibling-directory convention); the scope table in the notebook lists it. hypnoR 0.1.1 (`/Users/lucas/Developer/GitHub/hypnoR`) converts the object itself through `new_hypnogram.mrpheus_hypnogram` (epoch length, start time, participant ID and resolution come from the attributes `export_hypnogram()` set).

Behaviour to expect: with a non-NA EDF start time the x-axis is clock time (`x_axis = "auto"`), otherwise elapsed hours; stage colours are hypnoR's palette (the notebook's `palette_orpheus` is still used elsewhere). Artefact epochs are `NA` stages. Observed in the rendered plot (Lucas's screenshot): ggplot2 draws them as an "NA" lane at the top, joined by step lines, not as gaps; the first subtitle ("Gaps are artefact epochs") and the chunk comment were wrong and were corrected afterwards. The render otherwise worked: clock-time x-axis (about 20:00 to 08:30), hypnoR palette. hypnoR cannot show a named "Artefact" lane because `new_hypnogram()` rejects stage labels other than W/N1/N2/N3/REM (and the coarse set).

Observations from the plot, not diagnosed: the NA epochs are numerous and concentrated in the wake periods and at stage transitions; there is no REM lane at all (no REM epochs in the consensus hypnogram for the whole night); most sleep is N2 and N3 between about 23:00 and 02:30, with wake from about 02:30 to 08:30. The number of artefact epochs is printed in the `artefacts` chunk output (`Artefact epochs: n / total`) and was not seen. `style = "capsule"` and `cycles = compute_cycles(...)` are available but not used.

## Check 7: mrpheus staging against YASA (requested by Lucas)

**Where it was run.** In the sandbox, on public data only. `DLBP33N3.edf` is participant data and was not copied off Lucas's machine. R 4.x was installed in the sandbox (r2u binaries), mrpheus 0.1.4.9000 installed from the pushed GitHub `master` (commit ddca1cb, includes the resampler fix), YASA 0.8.1 and MNE 1.13.2 in Python.

**Data.** Sleep-EDF Cassette `SC4001E0`, a 6.5 h window (780 epochs; manual scoring W 127, N1 58, N2 250, N3 220, REM 125; stages 3 and 4 merged), channels EEG Fpz-Cz and EOG horizontal, exported to EDF at 100 Hz (A) and, after MNE FFT resampling, at 256 Hz (B). B is upsampled 100 Hz data, so it tests the 256 to 100 Hz pipeline path and gives YASA the same input; it contains no real content above 50 Hz. Both tools read the same EDF files. No chin EMG (Sleep-EDF's is a 1 Hz envelope), so EMG features are missing for both.

| Run (EEG + EOG) | Accuracy vs manual | Cohen's kappa vs manual | kappa vs YASA | REM | W |
| --- | --- | --- | --- | --- | --- |
| YASA, 100 Hz | 83.7% | 0.79 | 1.00 | 117 | 162 |
| YASA, 256 Hz | 83.7% | 0.79 | 0.99 | 117 | 163 |
| mrpheus `stage_epochs()`, 100 Hz | 83.6% | 0.78 | 0.85 | 164 | 123 |
| mrpheus `stage_epochs()`, 256 Hz | 83.5% | 0.78 | 0.85 | 165 | 123 |
| mrpheus, `preprocess_psg()` defaults, 100 Hz | 75.1% | 0.67 | 0.70 | 240 | 47 |
| mrpheus, `preprocess_psg()` defaults, 256 Hz | 74.7% | 0.66 | 0.70 | 243 | 43 |

Agreement between the 100 and 256 Hz runs: YASA 99.4%, mrpheus 99.9%, mrpheus with preprocessing 98.6%.

**Findings.**
1. The 256 Hz path works. After the resampler fix mrpheus staging at 256 Hz agrees with its own 100 Hz result on 99.9% of epochs, and its accuracy against manual scoring equals YASA's.
2. mrpheus is not bit-identical to YASA end to end: kappa 0.85, agreement 88.3% with the same inputs. mrpheus calls more REM (164 against YASA 117 and manual 125): YASA-scored W and N1 epochs become REM in mrpheus (28 and 15 epochs). The source of the difference was not isolated (EOG features are a candidate: it is not seen with EOG absent, see 4, but that case fails for another reason).
3. **WITHDRAWN, see Check 8** (`2026-11-19-hephaestus-yasa-parity.md`): this finding was measured with the wrong classifier. `preprocess_psg()` before `stage_epochs()` degrades staging: 83.6% to 75.1%, REM 164 to 240 (manual 125), W 123 to 47 (manual 127). The cause is the default EOG band-pass (0.3-15 Hz): widening it to 0.05-49 Hz restores 82.8%, and widening both EEG and EOG restores the unprocessed result exactly. The EEG band-pass (0.3-35 Hz) does no harm. `stage_epochs()` already resamples and band-passes every channel (0.4-30 Hz FIR) as YASA does. `correct_eog_regression()` alone does no harm (84.1%); the notebook's order, preprocess then regression, gives 75.4%; widening the EOG band-pass first gives 83.3%. The notebook, and the pipeline in the mrpheus README, use this order.
4. EEG only (no EOG): YASA 82.8%, mrpheus 50.1% (kappa 0.35 against YASA). YASA switches to a dedicated EEG-only model; mrpheus ships a single 149-feature model and fills missing channels with NA. Missing EMG is tolerated (finding 2 had none). Not relevant to the DLBP notebook, which has EOG and EMG, but `stage_epochs()` does not warn.

**Not explained.** The absence of REM in the DLBP hypnogram was not reproduced: on public data the preprocessing error pushes the result towards too much REM, not too little. Untested here: the EMG path (the notebook's `preprocess_psg()` high-passes EMG at 10 Hz, removing the 0.4-10 Hz content YASA's EMG features use), real 256 Hz data, six hours of wake around the sleep (YASA and mrpheus both use a normalised-time feature), and the DLBP recording itself. Single night, single subject: the accuracy figures are rough.

**Local check for the DLBP recording.** Created `Greg_PSG/yasa_check_DLBP33N3.py`: runs `yasa.SleepStaging` on C3:A2 and C4:A1 with EOG1 and EMG from `DLBP33N3.edf` on Lucas's machine, averages the probabilities as the notebook does, writes `output/DLBP33N3_yasa_hypnogram.csv`, and compares with `output/DLBP33N3_hypnogram.csv` (agreement, kappa, confusion table). The code was tested in the sandbox on the public file with constants swapped (kappa 0.85, matching the R run); it has not been run on the DLBP file.

**Not changed.** The notebook still runs `preprocess_psg()` with defaults before staging. A candidate change: pass `eog_bandpass` and `emg_bandpass` widened (for example `c(0.05, 49)`) to `preprocess_psg()`, or stage from signals that have not been preprocessed. Not applied; awaiting Lucas's decision and the local YASA result.

## Issues noticed and left untouched

- **Slow oscillation detection depends on the polarity of the first zero crossing in each epoch.** In `detect_so_candidates_cpp()` (`src/event_detection.cpp`), and in the R reference `.so_r()` in `tests/testthat/test-event-detection.R`, the triplet walk starts at the first crossing whatever its direction. In any 30-s epoch where the filtered signal starts negative, the pairing is shifted by half a cycle and neg_peak, pos_peak and ptp are computed on the wrong half-waves (ptp is then at most about 0, so the amplitude criterion rejects them), so that epoch yields no slow oscillations. Epochs that start positive are detected normally. Roughly half the epochs could therefore be lost. **Fixed: see Fix 4.** Slow oscillation output in `psg_DLBP33N3.qmd` (section 8) needs re-rendering after the fix.
- Documentation: the `compute_slow_oscillations()` help page used to say the algorithm is "implemented in YASA". Reworded in Fix 5 to say the defaults follow YASA and the detection is not identical.
- `mrpheus::export_hypnogram()` messages and documentation refer to `hypnor::new_hypnogram()` (lower case). The package is `hypnoR` (DESCRIPTION `Package: hypnoR`); R package names are case-sensitive, so the code form in that message would not load the package as written, and hypnoR's `plot_hypnogram()` converts the object itself so users do not need to call it. Not changed; not tested.
- The notebook's base-R `sleep_summary()` (section 6) duplicates what `hypnoR::compute_sleep_architecture()` provides. Not replaced.
- `Greg_PSG/.gitignore` does not exclude `DLBP33N3.edf` (786 MB) or `output/`. Both are participant data. Not changed.
- `mrpheus/README.md` passes `stages$stage` (character) to `compute_temporal_bandpower()`, which expects integer codes. Not changed.
- `compute_spindles()` documents its threshold as a multiple of the channel SD, but the code computes the SD within each epoch. Not changed.
- `detect_qrs()` documents `qrs_i` as referenced to the bandpass-filtered signal and mentions `delay`; whether the returned indices need the delay applied was not resolved. The notebook does not apply it.
- The same conditioning limit applies to `bandpass_filter()` in `signal_filters.R`, used by `preprocess_psg()`. Computed in Python, the default EEG band-pass (0.3-35 Hz, 256 Hz) has rcond 2.1e-13, about three orders of magnitude above R's limit and so working here. A lower high-pass cutoff or a higher sample rate would fail the same way. Not changed.
- `compute_spindles()` is not affected (11-16 Hz at 256 Hz: rcond 5.5e-9).
- `compute_hrv_sleep()`, `detect_apneas()` and `compute_odi()` are stubs, as above.
- `mrpheus/NEWS.md` has no entry for the `.mne_fft_resample()` downsampling fix.
- The header comment above `.mne_fft_resample()` in `staging_features.R` still describes the algorithm as upsampling only (step 4 mentions "upsampling").

## Open items

- **Check 8 (`2026-11-19-hephaestus-yasa-parity.md`) supersedes the staging items below:** `stage_epochs()` was made to reproduce YASA and the changes are in the mrpheus working tree (not yet committed); it needs the model script run, the tests run and a commit. Slow-wave detection, spindle detection and band power are not yet ported to YASA parity.

- Commit the Fix 5 and polarity changes. `git status` after the passing run showed modified: `NEWS.md`, `R/compute_slow_oscillations.R`, `man/compute_slow_oscillations.Rd`, `src/event_detection.cpp`, `tests/testthat/test-compute_slow_oscillations.R`, `tests/testthat/test-event-detection.R`; also modified `NAMESPACE` and untracked `R/compute_hrv_freq_windowed.R`, which are Lucas's own work and not part of this log. The spectrogram files did not appear in `git status`, so they are committed.
- Run `python yasa_check_DLBP33N3.py` in `Greg_PSG` (needs `output/DLBP33N3_hypnogram.csv` from a completed render for the comparison), then decide about the staging inputs (see Check 7). Candidates for the absent REM not yet examined: the artefact count and 500 uV criterion, the EOG1 and EMG inputs (units, polarity, quality) and the EMG high-pass in `preprocess_psg()`. Decide how to show artefact epochs in the plot (drop the NA rows, or extend hypnoR to accept an artefact lane).
- `mrpheus` follow-ups from Check 7, none done: `preprocess_psg()` documentation and the README pipeline order (preprocessing before staging costs about 8 points of accuracy on the test night); a warning or dedicated model in `stage_epochs()` when EOG is absent; the 0.85 kappa against YASA.
- Decide whether to port YASA's peak-based pairing into `compute_slow_oscillations()` to close the remaining ~20% undercount, and whether to add a parity test against YASA like the staging one.
- Re-render `psg_DLBP33N3.qmd` and fix whatever fails next. Likely points: the `physicalDimension` column name in the `edfReader` header, memory use at the `prepare_psg` and `correct_eog_regression` steps, and run time of `detect_qrs` on the full night.
- Add `NEWS.md` entries in `mrpheus` for the resampling fix and the spectrogram `overlap` fix (slow oscillation changes are in `NEWS.md`).
- Decide whether to keep or remove the items listed under Over-forging.
- Decide on `.gitignore` for the EDF and `output/`.
- Review of the notebook by Momus has not been requested.

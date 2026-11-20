# Review: Hephaestus changes of 2026-11-19 and 2026-11-20 (mrpheus, Greg_PSG)

**Minion:** Momus
**Date:** 2026-11-20
**Chat:** MOMUS - Review of Hephaestus changes 19-20 Nov (title proposed; Lucas applies it from the chat menu)
**Reviews:** `2026-11-19-hephaestus-greg-psg.md`, `2026-11-19-hephaestus-yasa-parity.md`, `2026-11-19-hephaestus-handoff.md`, `2026-11-20-hephaestus-band-power-parity.md`, `2026-11-20-hephaestus-slow-oscillation-parity.md`, `2026-11-20-hephaestus-spindle-parity.md`, `2026-11-20-hephaestus-cleanup.md`, `2026-11-20-hephaestus-handoff.md`
**Work reviewed:** `mrpheus` working tree as it is on disk; `Greg_PSG/psg_DLBP33N3.qmd`
**Requested by Lucas:** review of the code changes made with Hephaestus yesterday and today, using the logs and handoffs for context
**Definition of done used:** the stated goal in the handoffs: replicate the YASA process in mrpheus, with YASA as the gold standard, and use the result in the DLBP33N3 notebook.

## Limits of this review

- Nothing was executed. There is no R in the review sandbox, and YASA is not installed there. Every finding below comes from reading code, documents and YASA source. Where a finding depends on behaviour I did not run, it says so.
- I cannot run git. I read the files as they are on disk, not the diffs, and I cannot say what is committed or pushed.
- YASA source was read from the local clone `~/Developer/GitHub/yasa` (files dated 5 Jul 2026; its `pyproject.toml` allows `scipy>=1.10`). The ports were built against YASA 0.8.1 from pip. I did not diff the clone against 0.8.1. I read `spindles_detect`, `moving_transform` and the amplitude guard in the clone. I did not read `sw_detect`, `stft_power` or `_merge_close` there.
- Read in full: `R/compute_slow_oscillations.R`, `R/compute_spindles.R`, `R/compute_band_power.R`, `R/mne_fir.R`, `R/stage_epochs.R`, `R/staging_features.R`, `R/prepare_psg.R`, `R/preprocess_psg.R`, `R/signal_filters.R`, `R/detect_artifacts.R`, `R/export_hypnogram.R`, `NEWS.md`, `README.md`, `DESCRIPTION`, `NAMESPACE`, the CI workflow, `test-yasa-slow-oscillation-parity.R`, `test-yasa-spindle-parity.R`, `test-yasa-bandpower-parity.R`, `psg_DLBP33N3.qmd`, `vignettes/getting-started.Rmd`.
- Not read: `test-compute_spindles.R`, `test-compute_slow_oscillations.R`, `test-compute_band_power.R`, `test-staging-features.R`, `test-yasa-parity.R`, `test-event-detection.R`, the Python generators, the other vignettes, `compute_temporal_bandpower.R`, the Greg_PSG diagnostic scripts, hypnoR. No finding or "sound" entry below covers them.
- The logs say the slow oscillation and spindle fixtures were not yet generated. The fixture files now exist in `tests/testthat/fixtures/` (`yasa_sw_*.csv`, `yasa_sp_*.csv`). I did not open them.

## Findings

### 1. Major: a non-finite criterion makes the spindle detector lose every later spindle, silently

**Location:** `R/compute_spindles.R`, `.yasa_spindles_detect()` (the three `idx_sum <- idx_sum + (... >= ...)` lines and `cs <- c(0, cumsum(idx_sum))`); sources of non-finite values: `.moving_transform()` (corr branch returns `NA_real_` when `den == 0`) and `.stft_rel_pow()` (`colSums(P)` of 0 gives 0/0).

**Evidence:**
- YASA turns each comparison into an integer, `(mcorr >= thresh["corr"]).astype(int)`, so a NaN criterion counts as 0 (`detection.py`, `spindles_detect`). Its `moving_transform` returns NaN where `den == 0` (`others.py`, the `corr` branch), so YASA reaches this case and handles it.
- In the R port `NA >= x` is `NA`, `idx_sum + NA` is `NA`, and `idx_sum[!mask] <- 0` only clears samples outside the stage mask. `cumsum` then carries the first `NA` into every later element, so `sm` is `NA` from there on, and `which(sm > ...)` drops `NA` without a warning.
- Effect: after the first in-mask sample with a non-finite criterion, no spindle is returned for the rest of the recording.
- Reachability: `den == 0` needs a 0.3 s window in which the sigma or broadband signal has no measurable variance. With cumulative sums of a whole night (about 1e8 for the sum of squares) this holds for any window with an rms below roughly 1e-4 uV, for example digital zeros or a disconnected lead inside an epoch that is staged N2 or N3 and kept in `nrem_idx`. DLBP33N3 has hours of flat signal and epochs where the flat stretch starts or ends mid-epoch (flat is defined as more than half the epoch). Whether the case occurs on that recording is not known.
- Not reproduced. The parity tests contain no non-finite values, so they cannot see it.

**Suggested fix:** make each criterion a logical with `NA` as `FALSE`, as numpy does: `crit <- function(v, thr) { r <- v >= thr; r[is.na(r)] <- FALSE; r }`, and use it for the three criteria. Add a test: in the `.sp_psg(100)` signal set `psg$epochs[[2]]$EEG[2201:2800] <- 0` (52-58 s, between bursts), give the same signal to YASA in the generator, and require the spindles after 60 s to match. If the finding holds, the current code returns none after that gap. Lucas can check this in two minutes before changing anything.

### 2. Major: the YASA-equivalence of the detectors and band power does not carry over to the notebook, which feeds them preprocessed signal

**Location:** `Greg_PSG/psg_DLBP33N3.qmd`, sections 3, 4, 7 and 8 (`preprocess_psg()`, `correct_eog_regression()`, then `compute_band_power()`, `compute_spindles()`, `compute_slow_oscillations()`); `R/preprocess_psg.R` default `eeg_bandpass = c(0.3, 35)`.

**Evidence:**
- All parity evidence (public night, synthetic fixtures) feeds the ports the same unfiltered channel that YASA receives. The notebook gives them the channel after a notch, a 4th-order Butterworth 0.3-35 Hz `filtfilt` and EOG regression. Staging showed what this does: 108 of 1,355 epochs differ (log 2026-11-19, follow-up 3). The notebook was changed for staging only; the same check was not made for band power or the detectors.
- Band power: `compute_band_power()` now defaults to YASA's gamma band of 30-40 Hz, and the preprocessing low-pass is at 35 Hz. Squared-magnitude response of `butter(4, c(0.3, 35)/128, "pass")` at 256 Hz, computed here with scipy (assumed equal to gsignal's design): amplitude gain 0.80 at 30 Hz, 0.50 at 35 Hz, 0.22 at 40 Hz (power 0.64, 0.25, 0.05). Relative gamma, and the 0.5-40 Hz total used as the denominator of every relative band, are therefore not what `yasa.bandpower()` returns for this recording.
- Slow oscillations: the SO filter's pass band starts at 0.3 Hz (its -6 dB point is at 0.2 Hz). The preprocessing high-pass has amplitude gain 0.50 at 0.3 Hz, so it removes part of the lowest slow-oscillation frequencies before the detector sees them. Effect on counts: not measured.
- The handoff goal is "replicate the entire YASA process". The notebook text in section 2b says so for staging only.

**Suggested fix:** one of: (a) run `compute_band_power()` and both detectors on the signal as it stands before `preprocess_psg()` (keep a second copy of the three EEG channels, which is small next to the 6-channel object), as already done for staging; or (b) state in the notebook that sections 7 and 8 are not YASA-equivalent because of the 0.3-35 Hz pre-filter. Either way, measure the difference once on SC4001E0 (counts of spindles and slow oscillations, and band power, raw against preprocessed), with a script in the style of `yasa_vs_mrpheus_raw_DLBP33N3.R`. Until then the section 7 and 8 numbers should not be called YASA-comparable.

### 3. Minor: the committed tests cannot see the filter-length rounding fix at the slow oscillation rates

**Location:** `R/mne_fir.R` (`ceiling` in `.mne_fir_bandpass_design`); `tests/testthat/fixtures/yasa_sw_filter.csv`.

**Evidence:** the slow oscillation filter length is 3.3 / 0.2 s times the rate. At the two fixture rates (100 and 256 Hz) that is 1650 and 4224 samples, whole numbers, so rounding up and rounding to nearest give the same filter. The log states the fix was checked at eight rates in the sandbox; that check is not in the suite. The only rate in the suite where the two differ is the spindle sigma filter at 256 Hz (563.2 samples, 565 taps against 563). It is probably caught by `test-yasa-spindle-parity.R`, but "round instead of ceiling" is not among the 19 spindle mutants or the 13 slow oscillation mutants listed in the logs, so that is not confirmed.

**Suggested fix:** add one rate where 16.5 x fs is fractional (125 Hz) to the generator of `yasa_sw_filter.csv`, and add the mutant "round to nearest" to the next mutation run.

### 4. Minor: three statements about the scipy version disagree

**Location:** `NEWS.md` (Simpson bullet: "scipy (>= 1.11, checked on 1.14.1 and 1.18.1)"; Caveats: "scipy >= 1.15"), the comment above `.scipy_simpson()` in `R/staging_features.R` (">= 1.11"), the 2026-11-19 handoff ("scipy >= 1.15 (YASA 0.8.1's requirement)").

**Evidence:** the three cannot all be the condition for parity. The local YASA clone declares `scipy>=1.10`, so the "YASA 0.8.1 requires 1.15" claim is not supported by anything I could read. The fixtures embed whatever scipy and YASA the generator ran on, and the tests do not record or check those versions.

**Suggested fix:** decide the supported range from a measurement (run the staging generator on scipy 1.10, 1.11, 1.14 and 1.18, note where the features change), write one sentence in NEWS, and have each generator write the YASA, scipy and MNE versions it used into a header line of its fixture.

### 5. Minor: section 6 and section 8 of the notebook use different time bases

**Location:** `psg_DLBP33N3.qmd`: `stage_tbl` and `sleep_summary()` use `hyp[window, ]`; `nrem_min` and `nrem_idx` use every epoch.

**Evidence:** the log of 2026-11-20 gives 413 sleep epochs inside the window and 428 in the recording, the difference being 15 N2 epochs in the short valid stretch after the window ends. So the N2 minutes in the stage table (311 epochs) and the N2 minutes behind the event densities (326) differ by 7.5 min. Numerator and denominator of each density are consistent with each other; the two tables are not consistent with one another, and the text does not say so.

**Suggested fix:** restrict `nrem_idx` and `nrem_min` to `window`, or add a sentence to section 8 saying the densities cover the whole recording.

### 6. Minor: README and vignette drift

**Location and evidence:**
- `README.md`, "Full PSG pipeline": `stage_epochs()` runs after `preprocess_psg()`. NEWS and the notebook say staging must run on unpreprocessed signals to reproduce YASA.
- `README.md`: "77.8 % epoch accuracy on Sleep-EDF Cassette" has no source in the repo or the logs. The logs measured 83.7% on one night after the fix.
- `README.md`: `compute_temporal_bandpower(psg, hypno = stages$stage)` passes character stages; the function takes integer codes (Hephaestus noted this on 2026-11-19 and left it).
- `README.md`, dependency table: Rcpp is listed for "event detection"; `src/event_detection.cpp` has been removed. The project tree still shows one `inst/models/` model and omits `R/mne_fir.R`.
- `vignettes/getting-started.Rmd`: the example output shows `total_power` as `NA`; the code now returns a number.

**Suggested fix:** correct the five items; they are one commit.

### 7. Minor: packaging details (not run, pre-existing in part)

- `DESCRIPTION` Imports does not list `stats` or `utils`, while `NAMESPACE` has `importFrom(stats, ...)` and `importFrom(utils, read.table)` and the new code calls `stats::fft`, `stats::mvfft`, `stats::sd`. `R CMD check` normally reports this. I did not run it; the CI workflow `R-CMD-check.yaml` would show whether it already does.
- `DESCRIPTION` says `R (>= 4.1.0)`; `%||%` is used in `read_edf.R`, `stage_epochs.R` and `compute_slow_oscillations.R` and I found no definition or import of it. Base R has it from 4.4.0. Pre-existing for `read_edf.R`.

## Nits

- The first paragraph of the `compute_spindles()` and `compute_slow_oscillations()` help pages says the function "reproduces its output" without a scope; the evidence (one public night, a resampled copy of it, synthetic signals) is stated two paragraphs later. Move the scope into the first sentence.
- `src/` holds `RcppExports 2.o`, `staging_features 2.o` and a stale `event_detection.o`. `.gitignore` covers them (`src/*.o`, `* 2.*`); `.Rbuildignore` does not, so a local `R CMD build` would include them.

## Checked and found sound

- `compute_spindles()` against `spindles_detect` in the local YASA clone: amplitude guard (trimmed SD with a 5% cut, limits 0.1 and 1000), the three criteria, the 10 uV cap and the in-mask mean and trimmed SD of the RMS threshold, the zeroing of samples outside the mask, the 0.1 s smoothing and its window alignment, `n_thresh - 1`, merging, exclusive duration limits, and the nearest-frame fallback for `RelPower`.
- `.moving_transform()` uses the same prefix-sum formulation as YASA's `moving_transform`, so the numerical cancellation I looked at is inherited, not introduced.
- `compute_band_power()` by reading: scipy `welch` conventions (periodic Hamming, density scaling, one-sided doubling that skips DC and Nyquist, bias-corrected median), restriction of the PSD to the outer band edges before Simpson, `total_power` as `TotalAbsPow`. The 1e-14 and 1e-12 agreement in the logs is consistent with this.
- `.mne_fir_bandpass_design()` and `.mne_fir_apply()` by reading: filter length rounded up and forced odd, window-length rule per transition, `reflect_limited` padding with `n_edge = min(M, N) - 1`, delay removal.
- `.sw_find_peaks()` by reading: plateau midpoint arithmetic and inclusive height limits. The `.sw_events()` rules (exclusive limits, duration equality to 4 decimals, drop of shared starts and ends) match the description in the log; I did not compare them with YASA's `sw_detect` source.
- `stage_epochs()`: classifier selection, float32 rounding, class order (alphabetical, W last), feature order taken from the model file.
- Notebook logic: signal-state classification on the raw signal before preprocessing, the analysis window, NA stage only for no-signal epochs, `flagged` excluded from band power by stage and from event stage masks, `hypno_int` coding, explicit time zone for the hypnogram start.
- The logs: the PASS arithmetic (796 - 70 + 2 = 728) is consistent; mistakes are recorded with their corrections; limits (not run on DLBP33N3, one subject, one night) are stated. The error classes listed in the 2026-11-19 handoff ("do not repeat") are not repeated in the code I read, with the exception of finding 2, which is the same class as the staging pre-filter problem.

## Open items for Lucas

1. Decide how to handle finding 2 (raw copy for sections 7 and 8, or a stated limit).
2. Ask Hephaestus for the finding 1 fix and test; it is small. The two-minute reproduction in finding 1 can be run first.
3. Findings 3 to 7 are housekeeping; none blocks using the detectors on public data.
4. Neither detector has been run on DLBP33N3. Until the notebook is re-rendered, the section 8 numbers in the existing HTML come from the old detectors.

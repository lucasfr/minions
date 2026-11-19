# Handoff for the next Hephaestus run

**Written:** 2026-11-19, end of session. **Chat:** HEPHAESTUS - Greg_PSG full PSG notebook.
**Detail lives in:** `2026-11-19-hephaestus-greg-psg.md` (notebook build, mrpheus fixes 1-6, YASA comparison) and `2026-11-19-hephaestus-yasa-parity.md` (staging parity, DLBP33N3 findings). This file is the short version.

## Where things stand

**Goal Lucas set:** a full PSG analysis of `Greg_PSG/DLBP33N3.edf` with mrpheus, then "replicate the entire YASA process on mrpheus, bit perfect".

| Part | State | Evidence |
| --- | --- | --- |
| Staging (`stage_epochs()`) reproduces YASA 0.8.1 | **Done, verified, pushed** (mrpheus `5121001`) | Public night: all 16 classifier x sampling-rate runs give YASA's stage in every epoch, probabilities equal to about 1e-16 (largest 1e-3 in 8 of 12,480 epochs). DLBP33N3 on raw signals: 100.0% agreement, kappa 1.000, 0 epochs differing, 20 REM each. Clean clone of `5121001`: 560 expectations pass |
| Slow-oscillation detector | Fixed (polarity bug, 256 Hz filter, YASA defaults), **not YASA-equivalent** | Python mirror on Sleep-EDF: about 20% fewer events than YASA with the same criteria (4.66 vs 5.85 per NREM min, F1 0.80) |
| Spindle detector | **Not looked at** | |
| Band power | Uses the corrected Simpson rule; **not re-checked against YASA** | |
| DLBP33N3 notebook | Edited, **not re-rendered since the last edits** (section 2c validity, `flagged`, analysis window, time-zone fix) | Logic tested on a synthetic recording only |

**What the recording turned out to be:** the EEG is flat (one repeated value, 0.0122 uV, about 43% of samples) from partway through hour 6 to the end of hour 10 (header clock about 03:24 to 08:16), with a short valid stretch near 08:16-08:32 and flat again to the end; both channels also clip at +/-800 uV in places. 733 of 1,500 epochs are invalid on either channel; 727 of the 1,072 "wake" epochs have no signal. The sleep period is valid (only 6 of 428 sleep epochs are invalid). Cause of the flat signal unknown. The missing REM was never a staging bug: YASA also finds only about 11 minutes (20-23 epochs), and mrpheus matched it once staging ran on raw signals. `preprocess_psg()` before staging removes it (20 REM -> 1, or 0 with the EOG regression).

## Changed this session (by repo)

- **mrpheus:** resampler downsampling branch; `compute_spectrogram()` overlap argument; slow-oscillation polarity, filter and YASA defaults; `stage_epochs()` rewrite (8 classifiers, `age`/`male`, float32); `.scipy_simpson()`, skew/kurtosis, zero crossings; tests (`test-yasa-parity.R` with a feature-level primary guard and a coarse probability check, plus fixtures); `data-raw/fetch_yasa_models.py`, `data-raw/generate_yasa_parity_fixture.py`; `NEWS.md` (staging and slow-oscillation entries). Commits `5121001` and `ddca1cb`. **Nothing of mine is uncommitted there.**
- **Greg_PSG:** `psg_DLBP33N3.qmd` (new, full pipeline) and diagnostics `yasa_check_DLBP33N3.py`, `yasa_vs_mrpheus_raw_DLBP33N3.R`, `artefact_epochs_DLBP33N3.R`, `artefact_pipeline_DLBP33N3.R`, `clipping_DLBP33N3.R`, `signal_validity_DLBP33N3.R`. `explore_DLBP33N3.qmd` untouched.
- **hypnoR:** nothing changed.
- **minions:** these logs.

## Do first next time

1. Lucas re-renders `psg_DLBP33N3.qmd` (`command quarto render psg_DLBP33N3.qmd`), then reports: the section 2c summary row (`no_signal`, `window_first`, `window_last`, `window_hours`) and hourly table; the sleep-summary row; whether the hypnogram's first point sits at 21:05 and the last near 09:34; whether section 8 densities moved. If it fails, fix with a patch, not a rewrite.
2. Re-run `python yasa_check_DLBP33N3.py` afterwards. It should still agree with YASA on every staged epoch (the CSV no longer has an `artefact` column; no-signal epochs are NA).
3. mrpheus housekeeping: `NEWS.md` still lacks entries for the resampler and spectrogram fixes; `data-raw/fetch_yasa_model.py` (singular) is stale; `compute_band_power()` and `compute_temporal_bandpower()` need a parity check against `yasa.bandpower`.
4. Then, if Lucas confirms, the YASA port, one piece at a time: **`sw_detect` first.** Read `yasa.sw_detect` source again (peak-based: `find_peaks` for troughs and peaks with the 40-200 and 10-150 uV limits applied to the peaks, nearest following positive peak, peak-to-peak 75-350, phase durations from the zero crossings around each peak with strict bounds, adjacency check; FIR band-pass 0.3-1.5 Hz with 0.2 Hz transitions applied to the continuous signal, stage mask afterwards). mrpheus does zero-crossing triplets per epoch. Needs an MNE-matching FIR design for arbitrary sampling rates. **Then `spindles_detect`** (not read yet: read its source first). Same method as staging: measure against YASA first, port, then add a parity test, with a feature-level or event-level check as the primary guard.

## Decisions Lucas made

- Leave `R/compute_hrv_freq_windowed.R` (his unfinished file) in the pushed commit; his uncommitted `NAMESPACE` change is what exports it.
- Stage from raw signals, before `preprocess_psg()` (done in the notebook, section 2b).
- Hypnogram NA means "no valid EEG" only; clipped and amplitude-flagged epochs keep their stage and are excluded from band-power-by-stage and event detection. This was proposed and built after Lucas ran the diagnostics; he has not yet seen the render.
- Hard-coded `tz = "Europe/London"` for the start time in the notebook; change it if the recording was made elsewhere.

## Open questions for Lucas

- Why did the EEG go flat (electrodes or leads off, device, disconnection)? Only he or the recording team can say.
- Thresholds are my choices, checked only against this recording: flat = more than half of an epoch identical; clipped = more than 0.1% of samples on the extremes; analysis window ends at 60 minutes of consecutive no-signal epochs. The short valid stretch near the end is excluded from the window.
- 26 MB of model files in the repo: keep all eight, or drop the four `+demo` models (about 13 MB)?
- Move `epoch_signal_state()` into mrpheus (next to `detect_artifacts()`) or leave it in the notebook?
- Should the six one-off diagnostic scripts in Greg_PSG be kept?

## Issues noticed and left untouched

- `read_edf(path, channels = <one channel>)` then `prepare_psg()` fails ("Flat/bad channels detected"). Reproduced; cause not located. A one-channel EDF cannot use the EEG-only classifier.
- `.mne_fft_resample()` header comment still says upsampling only.
- hypnoR: the hypnogram plot's clock axis was an hour behind the file's header time in Lucas's render (header 21:05:24 BST; plot started near 20:05). Cause not reproduced (hypnoR passes the time through unchanged; my sandbox labelled in local time). Worked around in the notebook, not fixed in hypnoR. hypnoR also cannot draw an "artefact" lane: NA stages appear as an "NA" lane.
- YASA's `art_detect` could replace the 500 uV rule, which measures the preprocessed signal and flags filter leftovers.
- The `compute_slow_oscillations()` loop bound `k < nzc - 3` can drop the last pair in an epoch (kept for parity with the old R reference).

## Working rules learned (some the hard way)

- **Participant data stays on Lucas's machine.** Never copy `DLBP33N3.edf` to the sandbox. Compare tools with public Sleep-EDF in the sandbox and give Lucas a script to run on his own data. Reading small derived CSVs is fine; the 211 KB hypnogram CSV is better summarised by a script than pasted into context.
- **The sandbox does not persist.** Rebuild: `apt` with the CRAN and r2u repositories (`r-base-core r-base-dev` plus the `r-cran-*` binaries: rcpp remotes edfreader gsignal pracma lightgbm fastica dplyr tibble cli rlang readr testthat), `git clone https://github.com/circadia-bio/mrpheus.git`, `R CMD INSTALL`; `pip install yasa mne edfio`. Public test data: `mne.datasets.sleep_physionet.age.fetch_data(subjects=[0], recording=[1])` (Sleep-EDF SC4001E0). The test window used was epochs 991-1771, with EEG Fpz-Cz, EOG horizontal and EEG Pz-Oz renamed as a stand-in EMG.
- **Verify transferred files.** After writing files to Lucas's machine, copy them back with `copy_file_user_to_claude` and `cmp` against the tested versions.
- **The Claude Desktop file tool can time out.** If it does, do one read-only call, then retry the write once; do not keep retrying.
- **Set tolerances from more than one platform, and design the guard around the failure.** A probability tolerance of 2e-3 came from one machine and failed on Lucas's (3.8e-3); the platform noise (sign of the filter output at the first and last sample, up to 2e-2 on a 30-epoch fixture) is the same size as the bugs it must catch, so the primary guard is a feature-level comparison. Run mutation checks (put each old bug back; the test must fail).
- **Give commands that add only named files.** `git add -A R src ...` added Lucas's unfinished file to a public commit.
- **Lucas's machine:** zsh; `Rscript` is shadowed by a shell function, so use `command Rscript` (or `/usr/local/bin/Rscript`); `pip` is not on the path, so use `~/.venvs/yasa` (Python 3.12, yasa 0.8.x; check `python -c "import scipy, yasa; print(scipy.__version__, yasa.__version__)"` shows scipy >= 1.15).
- **Do not repeat these errors from this session:** a tolerance set from one platform; the claim that `preprocess_psg()` costs 8 points of accuracy (measured with the wrong classifier, withdrawn); reading clock times off the hypnoR plot (an hour early); `kappa()` that crashed when a stage was absent; trusting "validated parity" claims in the repo without checking the tolerance and the edge epochs excluded.
- **Persona reminders that mattered:** report failures with the result; patch, do not rebuild; flag adjacent problems; this session built six one-off scripts, which is the "golden handmaidens" vicissitude, so say when a task is one-off.
- Output goes in the minions `logs/`; Lucas prefers that I do the work myself where I can, with exact edits and a clear report of what was and was not run.

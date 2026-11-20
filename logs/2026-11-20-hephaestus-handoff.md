# Handoff for the next Hephaestus run

**Written:** 2026-11-20 (end of session). **Chat:** HEPHAESTUS - Greg_PSG next steps.
**Replaces:** the version of this file written earlier on 2026-11-20 (before the spindle work) and `2026-11-19-hephaestus-handoff.md` (still valid for the working rules, the sandbox rebuild and the DLBP findings).
**Detail lives in:** `2026-11-20-hephaestus-band-power-parity.md`, `2026-11-20-hephaestus-slow-oscillation-parity.md`, `2026-11-20-hephaestus-spindle-parity.md`. This file is the short version.

## Where things stand

**Goal Lucas set (2026-11-19):** "replicate the entire YASA process on mrpheus, bit perfect", with YASA as the gold standard.

| Part | State | Evidence |
| --- | --- | --- |
| Staging (`stage_epochs()`) | Done, verified, pushed (`5121001`) | 100% agreement with YASA, public night (16 configurations) and DLBP33N3 raw signals |
| Band power (`compute_band_power()`) | Done; code **not yet committed** | 1e-14 absolute, 1e-12 relative vs `yasa.bandpower()`; Lucas's run PASS 608 |
| `compute_temporal_bandpower()` | Unchanged on purpose | No YASA counterpart; own settings |
| Slow oscillations | Ported, on Lucas's machine; **fixtures not yet generated, not committed** | 2,236 = 2,236 and 1,000 = 1,000 events, times identical, values 1e-12 |
| Spindles | Ported, on Lucas's machine; **fixtures not yet generated, not committed** | 73 = 73 (100 Hz) and 78 = 78 (256 Hz), start/end identical, properties 1e-11; sandbox suite PASS 796 |
| `R/mne_fir.R` | Ceil fix written to Lucas's machine (not committed) | MNE rounds filter length up; matters at rates such as 125 Hz |
| Notebook `psg_DLBP33N3.qmd` | Needs a re-render | Section 7 band power, section 8 slow oscillations and spindles all change |
| hypnoR | Unchanged | |

## Do first next time

1. **Fixtures and tests.** All spindle files were written directly to Lucas's `mrpheus` tree (the earlier bundle `spindle-port/` is redundant). Ask whether he ran, in the YASA venv from the repo root: `python data-raw/generate_yasa_slow_oscillation_parity_fixture.py` and `python data-raw/generate_yasa_spindle_parity_fixture.py`, then `command Rscript -e 'devtools::test()'` (expect PASS 728, FAIL 0, WARN 0, SKIP 0 after the evening's cleanup; PASS counts expectations: 796 before it, 70 fewer from the removed C++ tests (by subtraction), 2 more from the new default test). Until the fixtures exist the two parity tests fail on his machine.
2. **Commits** (named files only; his `NAMESPACE` change stays out; do not push before all three): (a) housekeeping: `git rm data-raw/fetch_yasa_model.py`, `git add data-raw/generate_yasa_reference.py R/compute_hrv_freq_windowed.R`; (b) band power: `R/compute_band_power.R R/compute_temporal_bandpower.R man/compute_band_power.Rd man/compute_temporal_bandpower.Rd tests/testthat/test-compute_band_power.R tests/testthat/test-yasa-bandpower-parity.R tests/testthat/fixtures/yasa_bandpower_parity.csv data-raw/generate_yasa_bandpower_parity_fixture.py`; (c) slow oscillations and spindles (can be one or two commits): `R/mne_fir.R R/compute_slow_oscillations.R R/compute_spindles.R man/compute_slow_oscillations.Rd man/compute_spindles.Rd tests/testthat/test-compute_slow_oscillations.R tests/testthat/test-yasa-slow-oscillation-parity.R tests/testthat/test-compute_spindles.R tests/testthat/test-yasa-spindle-parity.R`, the fixtures `tests/testthat/fixtures/yasa_sw_*.csv` and `yasa_sp_*.csv`, `data-raw/generate_yasa_slow_oscillation_parity_fixture.py data-raw/generate_yasa_spindle_parity_fixture.py NEWS.md`. His local docs/scripts commit (vignette, README, NEWS, two scripts removed) is unpushed. The cleanup of the evening (see `2026-11-20-hephaestus-cleanup.md`) also changes `R/compute_band_power.R`, `man/compute_band_power.Rd`, `tests/testthat/test-compute_band_power.R` (commit b), `R/RcppExports.R`, `src/RcppExports.cpp`, `tests/testthat/test-event-detection.R`, `README.md`, `vignettes/spectral-analysis.Rmd`, `NEWS.md`, and needs `git rm src/event_detection.cpp data-raw/yasa_reference_bandpower.csv data-raw/parity_report_bandpower.csv data-raw/parity_bandpower.png` (run the `git rm` of `src/event_detection.cpp` before `devtools::test()`, otherwise the build regenerates `RcppExports` with the two functions).
3. Re-render the Greg_PSG notebook, then `python yasa_check_DLBP33N3.py` (still expected: every staged epoch agrees). The section 7 and 8 numbers are the first YASA-equivalent ones; neither detector has been run on the DLBP recording.
4. Remaining from the YASA process: nothing else in mrpheus is a YASA counterpart. Candidates Lucas named or I noted: YASA's `art_detect` as a possible replacement for the 500 uV amplitude rule in the notebook; slow oscillation / spindle coupling (not implemented in either detector); `yasa.bandpower` per stage (the notebook computes band power by stage itself).

## Decisions Lucas made today

- Match YASA's Welch parameters as the gold standard for `compute_band_power()`.
- Rewrite the stale validation claims and retire the two old band-power validation scripts (done, in his local commit).
- Go on with the YASA port: slow oscillations, then spindles.
- Accept the breaking change in `compute_spindles()` (new arguments, changed meaning of `peak_freq_hz` and `rms_uv`).
- `compute_band_power()` defaults to `relative = TRUE`, as YASA's `bandpower()`.
- Remove the C++ that is now unused (`roll_rms_cpp()`, `detect_so_candidates_cpp()`, `src/event_detection.cpp`, their tests) and the orphaned band-power data files.
- Test the two detectors on the DLBP33N3 recording himself (re-render the notebook).

## Open questions for Lucas

- Decided on 2026-11-20 (see above): `relative` defaults to TRUE; the unused C++ and the orphaned band-power data files are removed; the `compute_spindles()` breaking change is accepted.
- From 2026-11-19, still open: keep all eight model files (26 MB) or drop the four `+demo` ones; move `epoch_signal_state()` into mrpheus; keep the six one-off diagnostic scripts in Greg_PSG; why the EEG went flat.

## Issues noticed and left untouched

- Vignette `spectral-analysis.Rmd` setup line fails on the full SC4001E0 file at `preprocess_psg()` (`critical frequencies w must be in the range [0 1]`); suspect 1 Hz channels. Not located.
- `vignettes/getting-started.Rmd` example output shows `total_power` NA for `relative = TRUE`.
- `read_edf(path, channels = <one channel>)` then `prepare_psg()` fails (one EEG channel alone; EEG plus EOG works).
- `.mne_fft_resample()` header comment still says upsampling only.
- `dev/test_tier234_cpp.R` still calls the removed `roll_rms_cpp()` and `detect_so_candidates_cpp()` (a dev script, not part of the tests); not edited.
- The Claude Desktop file tool can stop answering for minutes; it did twice late on 2026-11-20 and recovered.

## Working rules learned (add to the earlier ones)

- **Measure against YASA's actual code, not docs or my reading:** the shared-start removal in `sw_detect` and the MNE filter-length rounding were both found only by comparing outputs. Run YASA's real code with a step replaced (identity filter) to get references for edge cases.
- **Report equivalent mutants** and say which deviations only a direct unit test catches.
- **Large fixtures:** give Lucas the generator; keep fixtures small and consistent with his YASA version.
- **Before editing a file on his machine, copy it back and compare** (his `NEWS.md` was newer than mine). `copy_file_user_to_claude` suffixes the copy's name when the name exists.
- **I cannot delete or run git on his machine:** deletions and commits are commands for him.
- **Do not quote test counts or error magnitudes from memory.**
- **If the file tool times out:** one read-only call, then stop and ship a bundle; when it answers again, check the local files against the pushed versions and write directly (it recovered after a few minutes on 2026-11-20).
- **`man/` pages:** roxygen with `roclets = "rd"` and take only the pages that belong to the change.
- Persona reminders that mattered: report failures with the result; flag adjacent problems (vignette setup, dead C++, changed argument names); patch, do not rebuild (but the two rewrites were the requested goal and kept the old output columns); say when a task is one-off (scratch scripts in the sandbox are not in the repo).

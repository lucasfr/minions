# Work log: hypnos, in-browser PSG metrics from an EDF file (webR)

**Minion:** Hephaestus
**Date:** 2026-11-23
**Chat:** HEPHAESTUS - hypnos browser PSG metrics (title proposed; Lucas applies it from the chat menu)
**Repo touched:** `hypnos` (working tree, not committed). Read only: `mrpheus`, `hypnoR`, `website`.
**Requested by Lucas:** a page that runs in the browser, takes an EDF file and returns standard PSG markers, with no data sent to a server; design to match the Circadia website; LightGBM workaround in hypnos only, mrpheus left alone for now.

## Status

Runs end to end on `DLBP33N3.edf` (786.4 MiB, 44 signals), per Lucas's report. I have no browser, so every runtime fact below comes from log lines Lucas pasted. The layout edits after the two-column version (tighter spacing, hero removed, clinical markers moved under the log, hypnogram and spectrogram height) were written but I have not seen the result.

## What was built (`hypnos/`)

- `index.html`, `style.css`: Circadia palette and fonts (Playfair Display, Alice), glass cards and pills, light and dark by system setting. Left column: status pills, EDF picker, progress and log, clinical markers. Right column: stage composition, hypnogram, spectrogram, CSV downloads.
- `app.js`: boots webR, installs `mrpheus`, `hypnoR`, `jsonlite` from `circadia-bio.r-universe.dev` plus `repo.r-wasm.org`, checks that mrpheus's imports are present, then runs the pipeline one R call at a time with per-stage timing, progress bar and elapsed timer.
- `analysis.R`: the pipeline stages, a pure-R predictor for the LightGBM text model, and the JSON returned to the page.

## Pipeline as it stands

1. **Channel extraction in JS.** The EDF header is read; the first EEG, EOG and EMG label are chosen with `prepare_psg()`'s patterns and precedence (here `F3:A2`, `EOG1`, `EMG`). A reduced plain EDF with only those signals is built from 32 MB slices of the file: 66 MB from 786 MiB. It is written to webR's `/tmp`.
2. `read_edf()` (0.2 s header, 7.4 s load), `prepare_psg()` (1,500 epochs, 4.3 s).
3. `preprocess_psg()` is **not run**. `stage_epochs()`'s feature code applies its own 0.4-30 Hz FIR and 100 Hz FFT resample, and the full-night `filtfilt` passes stalled in WASM (duration not measured). It will be needed for band power and event detection.
4. Staging features: `mrpheus:::.extract_staging_features()` (internal), 36.8 s for 1,500 epochs.
5. Prediction: pure-R evaluation of `yasa_staging.txt`, because the webR build of `lightgbm` cannot load (shared-memory link error). Parity test in the sandbox against Python LightGBM 4.7.0 (R 4.3.3): 2,650 rows of `yasa_reference_features.csv` plus 800 perturbed rows with NaN and exact zeros; largest probability difference 3.3e-16, argmax agreement 100%. The model has 2,000 trees, all `num_cat=0`, all `decision_type=2`. Parity was not run inside webR.
6. Spectrogram: `compute_spectrogram()` on the staging EEG channel, 0-30 Hz, non-fatal if it fails.
7. `hypnoR::compute_sleep_architecture()` for the markers. Timings for stages 5 to 7 were not recorded.

## Dead ends

- `webR.FS.mount('WORKERFS', { files })`: webR only accepts packages with WORKERFS. My in-memory fallback then failed on the 824 MB copy ("Invalid array length"). The fallback also hid the first error, and I removed it.
- Passing only the r-universe URL as `repos` made webR miss CRAN dependencies (`Rcpp` not found). Listing `repo.r-wasm.org` as well fixed it. I had also set the install to quiet, which hid this; the cause is my inference from the fix working.
- Progress from R `message()` calls: no R output appeared in the page log, so progress was moved to JS-driven stages.

## Issues noticed, left untouched

- **mrpheus `read_edf()`:** `duration_s = hdr$recordedPeriod * hdr$nrOfRecords` is empty. In this file `nrOfRecords` is `NULL` and `recordedPeriod` is already the total (45,009 s). Probable fix: `duration_s = hdr$recordedPeriod`. `print.mrpheus_edf` uses it too.
- **mrpheus WASM build lacked `models/yasa_staging.txt`**, although the file is in the local git index. Cause not found (the index check says nothing about the remote). `app.js` falls back to `models/yasa_staging.txt` beside the page if the package lacks it; whether Lucas used that copy was not stated.
- **`stage_epochs()` hard-depends on `lightgbm`.** Proposed and declined for now: a pure-R fallback in mrpheus with the same parity test. How the mrpheus WASM tests treat the staging tests was not checked.
- **Plausibility of the output is unchecked.** `DLBP33N3`: TST 205.5 min, TIB 750 min, SE 27.4%, WASO 481.5 min, REM 2.9%, REM latency 201.5 min. Possible causes include a frontal EEG channel (YASA was trained on central derivations) and skipping preprocessing; a very fragmented night is also possible. No comparison against a scored hypnogram has been made. `inst/extdata/SC4001E0_staging.rds` is the obvious reference.
- `prepare_psg()` and `preprocess_psg()` copy every channel into a list per epoch, which is heavy in a 32-bit WASM heap.
- `compute_sleep_architecture()` returns NA for TST if any stage is NA. Not reached here, because artefact epochs are not passed to staging.
- The reduced EDF drops the annotations channel and is written as plain EDF; discontinuous EDF+D files would lose their record timing.
- The page loads webR (`latest`, unpinned) and Google Fonts from CDNs. The "no data leaves your device" pill is true of the EDF; a strict claim needs both self-hosted.
- `logo.png` and `favicon.png` are referenced and must be copied from `website/`.

## Next

- Check the markers on a recording with a trusted hypnogram before anyone relies on them.
- Decide on the mrpheus fixes (`duration_s`, optional `lightgbm`, model in the WASM build).
- Further markers (spindles, slow oscillations, band power per stage, AHI/ODI, HRV) need a cheaper preprocessing step first, probably downsampling before filtering.

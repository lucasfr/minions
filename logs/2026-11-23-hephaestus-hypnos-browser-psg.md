# Work log: hypnos, in-browser PSG metrics from an EDF file (webR)

**Minion:** Hephaestus
**Date:** 2026-11-23
**Chat:** HEPHAESTUS - hypnos browser PSG metrics (title proposed; Lucas applies it from the chat menu)
**Repo touched:** `hypnos` and `website` (the website changes are listed below). Read only: `mrpheus`, `hypnoR`.
**Requested by Lucas:** a page that runs in the browser, takes an EDF file and returns standard PSG markers, with no data sent to a server; design to match the Circadia website; LightGBM workaround in hypnos only, mrpheus left alone for now.

## Status

Runs end to end on `DLBP33N3.edf` (786.4 MiB, 44 signals), per Lucas's report. I have no browser, so every runtime fact below comes from log lines or screenshots Lucas sent. Written but not seen by me: the layout changes after the two-column version, the Plotly plots, the icons, the welcome screen, the footer and the logo (Lucas called the spectrogram version and the latest logo good). Not run: the website changes. Git state at the last `git status` Lucas pasted: `style.css` and `vendor/hypnor-hex.svg` modified; `favicon.png`, `logo.png` (copies of the lab logo, now unused) and `models/` untracked. The `models/` decision, and a YASA licence file beside it, is open.

## What was built (`hypnos/`)

- `index.html`, `style.css`: Circadia palette and fonts (Playfair Display, Alice), glass cards and pills, light and dark by system setting. Left column: status pills, EDF picker, progress and log, clinical markers. Right column: stage composition, hypnogram, spectrogram, CSV downloads.
- `app.js`: boots webR, installs `mrpheus`, `hypnoR`, `jsonlite` from `circadia-bio.r-universe.dev` plus `repo.r-wasm.org`, checks that mrpheus's imports are present, then runs the pipeline one R call at a time with per-stage timing, progress bar and elapsed timer.
- `analysis.R`: the pipeline stages, a pure-R predictor for the LightGBM text model, and the JSON returned to the page.

### Added later in the same chat

- **Plots:** the hypnogram and spectrogram are Plotly (cartesian bundle 2.35.2 from jsDelivr, `cdn.plot.ly` as fallback): hover, zoom, pan and reset, with the two time axes linked. The stage composition bar is still HTML.
- **Icons:** Bootstrap Icons v1.11.1, the same font as `circadia-lab.uk`, self-hosted in `vendor/` (copied from `website/_site/site_libs/bootstrap/`). The icon names were checked against that CSS.
- **Welcome screen** (`welcome.js`, markup in `index.html`): after NotedApp's onboarding. Four steps, progress dots, Skip and Next, remembered in `localStorage` (`hypnos_welcomed`, version `0.1.0`; after a version bump only the last step is shown), reopened from the navbar "About" link. Step 1 shows the mrpheus and hypnoR hexes. `vendor/hypnor-hex.svg` is a copy of the hypnoR logo with a cropped `viewBox`, so both hexes render at the same size; the artwork is unchanged.
- **Footer:** "Made with" line with the mrpheus and hypnoR navbar icons, an MIT licence line, and a disclaimer about YASA (attribution, independence from the YASA authors, markers not validated for clinical use, research tool, not a medical device). The wording is mine and has not been reviewed for legal use.
- **Licence:** MIT `LICENSE`, "Copyright (c) 2026 Lucas França, Mario Leocadio-Miguel", the same as hypnoR.
- **Logo** (`logo.svg`, `favicon.svg`): a pocket watch swinging on a chain like a pendulum, with a motion trail, on a blue gradient. The first attempt was a hex sticker, which was wrong because hexes are for R packages only. The second (flat, ghost outlines, dotted chain) looked dated to Lucas. The third is the current one. Checked by rendering in the sandbox at 512, 64, 36, 32 and 16 px, not in a browser.

### Website (`website/`)

- `projects/projects.yml`: hypnos entry (Web App, Sleep, Open Source). `projects/hypnos/index.qmd`: project page, with a status note matching the app's disclaimer.
- `index.qmd`: hypnos card in the "Our tools & software" carousel (tag "App"). `_quarto.yml`: `hypnos-icon.svg` added to `resources`. `convert_og_images.py`: `hypnos_logo` added to its list.
- `projects/images/hypnos_logo.svg` and `hypnos-icon.svg` (the small-size variant) are the same logo as in the app.
- Assumptions and gaps: the app URL `https://hypnos.circadia-lab.uk` is assumed from the naming pattern; `hypnos_logo_og.png` does not exist until `convert_og_images.py` is run, and the project page refers to it; there is no `zenodo.yml` entry, because the hypnos repo has no `.zenodo.json` or `CITATION.cff`; the Quarto site has not been rendered.

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
- The page loads webR (`latest`, unpinned), Plotly (pinned) and Google Fonts from CDNs. The "no data leaves your device" pill is true of the EDF; a strict claim needs all three self-hosted.
- `logo.png` and `favicon.png` (the Circadia lab logo, copied from `website/` earlier) are no longer used by the page and can be deleted.
- If `models/yasa_staging.txt` is committed to hypnos, YASA's BSD 3-Clause notice should sit beside it (`models/LICENSE-YASA`). Not written, because I could not confirm the exact text.

## Future improvement: EEG channel selection

Current logic (`pickChannels()` in `app.js`, then `hypnos_features()` in `analysis.R`): the header labels are tested in file order against `EEG|C3|C4|F3|F4|O1|O2|Fpz|Pz`, then `EOG|ROC|LOC`, then `EMG|chin|Chin` (case-sensitive substrings, the same as `prepare_psg()`), and the first match of each type is loaded. R then takes the first non-flat EEG channel from the reduced file, which is the one already chosen. On `DLBP33N3` this picked `F3:A2`.

Weaknesses:

- No preference for central derivations, although YASA was trained on them. A frontal channel can come first, as it did here.
- The patterns are loose. `O2` matches `SpO2`, so an oximetry channel listed before the real EEG would be taken as EEG. The same holds for `Pz`, `C3` and similar substrings inside other labels.
- No quality check (flat, noisy, wrong reference, sampling rate), and no fallback: only one EEG channel is extracted, so a bad choice cannot be replaced in R.

Proposed: rank the candidates (C4, then C3, then Cz, then the rest) and exclude labels such as `SpO2` in `pickChannels()`, leaving mrpheus alone. Optionally, let the user choose the channel in the page. Re-check the markers on `DLBP33N3` afterwards, because the channel may explain part of the unusual values.

## Next

- Check the markers on a recording with a trusted hypnogram before anyone relies on them.
- EEG channel selection, above.
- Decide on the mrpheus fixes (`duration_s`, optional `lightgbm`, model in the WASM build).
- Further markers (spindles, slow oscillations, band power per stage, AHI/ODI, HRV) need a cheaper preprocessing step first, probably downsampling before filtering.
- Self-host webR, Plotly and the fonts, and pin the webR version, if the privacy claim is to be strict.
- Review the footer disclaimer and licence wording.

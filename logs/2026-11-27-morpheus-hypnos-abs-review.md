# Work log: hypnos reviewed against the ABS 2024 PSG reporting consensus

**Minion:** Morpheus
**Date:** 2026-11-27
**Chat:** MORPHEUS - hypnos review against ABS 2024 PSG reporting consensus (title proposed; Lucas applies it from the chat menu)
**Repo touched:** none by edit. Read only: `hypnos` (`analysis.R`, `app.js`, `channels.js`, `settings.js`, `README.md`).
**Requested by Lucas:** review the paper and check the hypnos analysis against it, with suggestions.
**Paper:** Palombini LO, et al. 2024 Standardization of Polysomnography Reports, a Consensus of the Brazilian Sleep Association. Sleep Sci 2024;17:e429-e450. doi:10.1055/s-0044-1800886.

## Status

Review only. Nothing was run on a recording, so every statement about behaviour is read from code, not observed. Not read: `index.html` (so whether the page already states that staging is automatic is unchecked), `edf.js`, `theme.js`, `welcome.js`, and the tests under `tests/`. Implementation of the points below is handed to Hephaestus in `2026-11-27-morpheus-hephaestus-hypnos-handoff.md`.

## What the paper supports

- It is a Delphi consensus (29 panelists, two rounds, physician votes weighted 5) on which items a report lists. It defines no computation and defers to the AASM manual for scoring and event definitions.
- It scopes itself to type 1 to 4 studies requested by healthcare professionals under professional oversight. Research setups, actigraphy and consumer devices are excluded. Hypnos is a research tool, so the paper is a reference for naming and metadata, not a compliance standard.
- It does not define sleep onset latency, WASO or time in bed. The choices in hypnos are therefore its own and need documenting as such.

## Coverage of the ABS items for type 1 and 2 studies

| ABS item | Status in paper | hypnos (read from code) |
| --- | --- | --- |
| TST, sleep latency, REM latency, WASO, sleep efficiency (2.14 to 2.18) | Recommended | Present |
| Minutes per stage, % of TST (2.19, 2.20) | Recommended | Present on page; minutes not in the CSV |
| Lights off, lights on (2.11, 2.12) | Recommended | Absent; the analysis window stands in |
| Total recording time (2.13) | Recommended | In the result JSON (`recording.duration_h`), not shown as a marker |
| Staging guideline (2.2), explicit mention of automated analysis (2.22) | Recommended | README only, as far as read |
| Montage description (2.1) | Recommended | Partial; the CSV holds the EEG channel only |
| Artefact time (2.21) | Optional | Partial; flat epochs only |
| Arousals, PLMs, respiratory indices, SpO2, heart rate (items 3 to 8) | Recommended | Absent |
| Hypnogram (9.6) | Recommended | Present; the other graphs are absent |

"Standard polysomnography metrics" in the README title overstates the coverage. What hypnos returns is the sleep-architecture subset.

## Findings

Evidence labels: **read** means read from code and not run; **inferred** means reasoned from the code without a test.

1. **"Time in bed" is the analysis window.** The window starts at the first epoch with a signal (`.hypnos_auto_window()`), not at lights off. Pre-lights-out wake would inflate sleep onset latency and lower efficiency. The code path is **read**; the size of the effect on a real file is not known. Annotation channel is dropped in extraction (README), so Lights Off and Lights On are unavailable at present.
2. **Trailing no-signal time is trimmed only from 60 minutes.** `win_end` falls back to `length(no_signal)` when no flat run reaches `gap_epochs` (120). A flat stretch of, say, 45 minutes at the end stays inside the window as unscored epochs, counts towards TIB and lowers efficiency. **Read.**
3. **Sleep onset is the first single non-wake epoch.** One early N1 epoch from the classifier moves sleep onset latency, REM latency, N3 latency and WASO. A real brief sleep and a misclassification cannot be told apart from one epoch. **Inferred**; persistent-sleep latency is not in the ABS list but would show the sensitivity.
4. **A null metric is dropped from the page.** `render()` in `app.js` skips null values, so a night with no REM epochs shows no REM latency card. The paper's stated principle for absent findings is that the report should say they were searched for and not found. **Read.** The comment in the code says the skip is for coarse-only metrics, so whether other metrics rely on the skip is unchecked.
5. **The output does not state that staging is automatic.** The README Limitations section does; the page and CSV, as read, do not (`index.html` not read). The CSV carries metrics, EEG channel and window only: no EOG or EMG channel, model key, package versions, summary engine, recording duration or stage minutes. **Read.**
6. **`.hypnos_epoch_state()` has no NA handling.** `median` and `colMeans` return NA if any sample in an epoch is NA, which makes that epoch's state NA. Whether `edfReader` can return NA physical values, or the signal can be shorter than `spe * n`, was not checked. Separately, only exact flatness is detected, so a noisy, disconnected electrode is still staged (stated in the README).
7. **Validation is open.** The README says the markers are unvalidated against manual scoring. Epoch-level agreement can be high while latency and WASO errors are large, because they depend on few epochs; the errors could also cancel. Which holds is unknown.

## Competing explanations

- Findings 1 and 2 may matter little for files that start at lights out and end at lights on. Without such a file to test, that is unestablished.
- If hypnos is meant only as a research tool, the missing ABS items are a scope choice, not a defect. The case for alignment is mainly naming and metadata (findings 4 and 5), which cost little.

## Suggestions not handed to Hephaestus

- Persistent-sleep latency alongside sleep onset latency: a change in hypnoR first, so a decision for Lucas.
- Retitle the README "Sleep architecture metrics", or add a coverage table like the one above: a decision for Lucas.
- SpO2 summaries (mean, minimum, maximum, time below 90%, items 7.64 to 7.67) when an SpO2 channel exists; dropout handling needed first.
- Validation of the ABS-recommended markers against manually scored data, bias and limits of agreement per marker, stratified by age and sleep-disordered breathing: planning belongs to Prometheus.

## Notes on the paper (if item numbers are cited)

- The abstract and Results disagree: 352 vs 353 items, 339 vs 341 with consensus, 145 vs 116 recommended, 13 vs 12 without consensus.
- ODI is printed as "IDO" and "IDR". Table 16 is titled type 3 but covers type 4, and the type-4 section says "type-3". Item 9.9 is numbered twice.
- Item 7.70 (hypoventilation) is "shall be reported" in the text but Optional in Table 7.
- The panel was 76% from the state of São Paulo.

## Next

- Hephaestus: findings 1, 2, 4, 5 and 6, per the handoff.
- Lucas: the decisions listed in the handoff.

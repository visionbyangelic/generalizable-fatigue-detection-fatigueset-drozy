# Generalizable Fatigue Detection from Wearable Physiological Signals

**Angelic Charles**

EEG fatigue detection has a generalisation problem. Across the published literature, models that report very high accuracy on the people they were trained on collapse toward chance when they are tested on people they have never seen. This project measures that collapse under leakage-controlled evaluation, on two datasets, with the decision rules written down before the results were interpreted. It asks a simple question: does a fatigue model still work on people it has never seen?

## Project goal

Consumer EEG headbands are cheap enough to put fatigue monitoring in trucks, factories and clinics, but only if the model works on **new people**, not just on the people it was trained on. Many published fatigue-detection results are evaluated in ways that let information about the test person leak into training, which inflates the scores. The goal of this project is to measure how well fatigue detection from wearable signals **actually generalises**: across sessions, and across participants the model has never seen.

## Research questions

- **RQ1, Generalisation.** How does performance change as the evaluation gets more honest: within a session, within a participant across sessions, and on completely unseen participants (leave-one-participant-out)?
- **RQ2, Model comparison.** On identical, honest splits, does an EEGNet 1D-CNN trained on raw EEG plus two validity-indicator channels (whether each temporal channel, TP9 and TP10, passed the quality checks in that window) beat a classical band-power + logistic-regression baseline?
- **RQ3, Multimodal extension.** Do heart-rate and electrodermal signals (recorded alongside the EEG) carry anything across participants where EEG alone does not?
- **RQ4, Label diagnostics.** How much of the apparent performance in this literature is an artefact of how labels are assigned (for example, labelling whole sessions as "fatigued") rather than physiology?

## Hypotheses

- **H1:** Models that score highly under random or within-session splits will fall to near chance under participant-held-out evaluation, because many headline numbers in this literature are evaluation artefacts, not physiology.
- **H2:** A substantial part of what models learn under loose evaluation is recording artefacts (session identity, sensor-contact quality), not fatigue.
- **H3:** Where any genuine signal exists, it will be small at this scale, so the study is designed to bound effects honestly (estimation with confidence intervals) rather than overclaim.

## Design

- **FatigueSet** (primary): 12 participants, Muse S EEG, three sessions each on separate days (low, medium and high treadmill intensity), with subjective fatigue ratings and cognitive-task performance.
- **DROZY** (second dataset, pre-registered): clinical polysomnography EEG under sleep deprivation, 13 participants analysed. Its decisive feature is that it carries **two label types on identical recordings**: subjective sleepiness (KSS) and reaction-time performance (PVT lapses). The same locked pipeline runs on both datasets, so the comparison tests whether the findings are about the hardware, the labels, or the evaluation.
- **Evaluation:** within-subject, leave-one-session-out, and leave-one-participant-out (LOPO), with normalisation, feature selection and tuning fitted on training folds only, and no overlapping windows across splits.
- **Labels:** subjective ratings are primary, task performance is secondary, and the label rules were locked before any training.
- **Reporting:** estimation-only. The equivalence margin set in advance (delta = 0.05 in balanced accuracy, the smallest effect considered useful) turned out to be unattainable at this sample size, so results are reported as estimates with intervals, not as equivalence verdicts.

## Datasets and data notes

- **FatigueSet**: 12 participants × 3 sessions, Muse S EEG (TP9, AF7, AF8, TP10 @ 256 Hz) plus Empatica E4, eSense and Zephyr wearables, with subjective fatigue ratings and cognitive-task performance. © Nokia Bell Labs 2021; no licence is stated for the data, so it is **not redistributed** here. Obtain it from the official source and cite the FatigueSet paper if you use it.
- **DROZY**: 14 participants recorded, 13 analysed (one participant has data for a single session only and cannot form a within-person contrast), clinical polysomnography EEG (Fz, Cz, Pz, C3, C4 @ 512 Hz), three vigilance sessions under increasing sleep deprivation, with Karolinska Sleepiness Scale scores and PVT reaction times. Research use only, **not redistributed** here; cite Massoz et al. (WACV 2016) and obtain it from the official ULg record. The analysis plan for DROZY was locked, and the pipeline code frozen and hashed, before the data was downloaded: `docs/dataset2_preregistration.md`.

## Method in brief

- 4-second non-overlapping windows; band-power features (delta/theta/alpha/beta, log + relative power, frontal ratios) with a logistic-regression baseline; EEGNet 1D-CNN on raw EEG plus two validity-indicator channels (TP9 and TP10 quality, per window), compared on identical splits.
- A pre-declared artefact rule (Amendment 1) keeps a window only if both frontal channels pass amplitude and flatline checks; channels that fail are masked per window and carried as validity indicators, never silently zeroed.
- Every headline number ships with diagnostics: permutation nulls, a positive control (the pipeline must recover an injected effect of known size), a mask-only control (can recording quality alone predict the labels?), and a leakage-gap measurement (random split vs honest split).
- Decision rules, including what may be claimed from each possible outcome, were written down and locked **before** results were interpreted: `docs/decision_table_lock.md`. The full build log, with the reason for every step, is in `track.md`.
- This is retrospective offline classification: I filter each session as a complete recording (zero-phase filtering) and classify the windows afterwards, so the results do not speak to real-time or prospective detection.

## Results

Short version: when the evaluation is honest, the EEG fatigue signal on FatigueSet is not there, and the numbers that look like it are measuring something else. What follows is what I found, dataset by dataset, and then why I think it matters beyond this project.

On FatigueSet, both models sit at chance on people they have never seen: leave-one-participant-out balanced accuracy 0.508 for the band-power baseline and 0.493 for the EEGNet CNN, with honest within-subject evaluation also at chance (0.484 and 0.493). The one high number, 0.744 within a session, is position-confounded: where a window sits inside its block ranks the labels perfectly on its own, and a block permutation null scores 0.508 (p = 0.612), so 0.744 is an upper bound on separability, not decoding.

![Bar chart of FatigueSet balanced accuracy across the three evaluation tiers: same-session 0.744, honest within-subject at chance, leave-one-participant-out at chance](results/figures/fig_rq1_tiers.png)

*The three tiers of evaluation on FatigueSet. The same-session figure of 0.744 is an upper bound on separability, not decoding; both honest tiers sit at chance.*

On the identical window set the CNN and baseline differ by +0.005 [-0.068, +0.071]. The conclusion I locked: no evidence that either model exceeds chance on unseen participants on FatigueSet, and no evidence of a CNN advantage. The CNN half of that is weak evidence, because 16 of its 55 fits were near-constant and a constant fit scores exactly 0.5; the null belongs to this configuration, not to neural networks in general.

![Per-participant leave-one-participant-out balanced accuracy for the EEGNet CNN and the band-power baseline on identical windows](results/figures/fig_rq2_perparticipant.png)

*Per-participant leave-one-participant-out scores, CNN against baseline on the identical window set. The two models track each other closely, and both sit at chance overall.*

These nulls are not a broken pipeline. On both datasets the pipeline recovers effects I injected at known sizes: on FatigueSet down to theta ×1.5 and alpha ×1.25, and on DROZY the recovery gate passed at 0.798 at full dose and 0.672 at half dose, against pre-set bars of 0.75 and 0.60. When it reports nothing, nothing of that size was there to find.

My preliminary result is the clearest case. The preliminary result reproduces in its own configuration (logistic regression 1.000, CNN 0.993 balanced accuracy), but there the fatigued class coincides almost perfectly with a session-wide amplitude regime that fails the pre-declared quality criterion (mask-retained 97.1% vs 2.5%). The score therefore identifies the session and cannot be read as evidence of fatigue decoding. The available data do not separate placement or contact differences from state-correlated physiology or behaviour, and I have left that question open rather than picking a mechanism.

![Ladder of scores for the rebuilt preliminary experiment, from its original configuration to honest splits, with mask retention by session](results/figures/fig_rq4_ladder.png)

*My preliminary experiment rebuilt step by step. The original configuration reproduces its score, while the quality mask keeps 97.1% of the alert session and 2.5% of the fatigued session, the session identity the score was reading.*

DROZY shows the labels deciding what transfers. On identical recordings, subjective sleepiness labels (KSS extremes) decode across participants at 0.602 [0.563, 0.644], while reaction-time labels do not (0.482 [0.421, 0.549]), a paired contrast of +0.120 [0.062, 0.179]. The locked headline: KSS-extreme labels were decodable across participants; RT-extreme labels from the same recordings were not; this contrast is not isolated from session order and rests on a small disagreeing subset. The positive control prices the RT null: injected shifts aligned with RT labels were detected at dose 0.5 and above on a 0.25-step dose grid, so smaller RT-related changes are not ruled out. I also do not claim the pre-registered cross-dataset row: the KSS advantage over FatigueSet is +0.094, under the 0.10 line set in advance for calling it clearly interpretable.

![DROZY leave-one-participant-out balanced accuracy under KSS sleepiness labels at 0.602 and reaction-time labels at 0.482 on identical recordings](results/figures/fig_drozy_dissociation.png)

*DROZY, one set of recordings under two label definitions. Sleepiness labels decode across participants at 0.602; reaction-time labels from the same recordings do not, and the contrast rests on a small disagreeing subset.*

The wrist signals are the one positive FatigueSet result, and it comes with its confound attached. Peripheral features (heart rate and electrodermal activity) reach 0.741 [0.633, 0.851] across participants, against 0.539 [0.500, 0.595] for a model given only which windows have valid wrist data, an increment of +0.202 [0.085, 0.314], and +0.230 [0.128, 0.332] over EEG alone. The locked claim: peripheral features recover the protocol-defined M1/M3 contrast across participants, and the design cannot separate fatigue from exercise-related physiology. The early block is pre-treadmill and the late block post-treadmill, and the heart-rate gap scales with the day's treadmill intensity, so this is exertion physiology tracking the protocol, not a fatigue detector.

![FatigueSet leave-one-participant-out balanced accuracy by feature arm: EEG only, peripheral only, combined, and validity indicators only](results/figures/fig_rq3_arms.png)

*The FatigueSet arms side by side. Peripheral features recover the protocol-defined M1/M3 contrast across participants at 0.741, well above the 0.539 from data-validity indicators alone; the contrast is early block against late block, before and after the treadmill.*

![Late minus early block mean heart rate difference by treadmill intensity day: low, medium and high](results/figures/fig_rq3_hr_intensity.png)

*Mean heart rate, late block minus early block, by the day's treadmill intensity. The gap grows with exercise load, which is why I read the wrist result as exertion physiology tracking the protocol.*

Why this matters: a large share of the fatigue-detection literature reports accuracies in the high nineties from evaluations that share people, sessions or windows between training and test. Under the evaluation those deployments actually need, a model meeting people it has never seen, this study finds the EEG effect at chance on a consumer headset, finds a published-style 99% result measuring session identity instead, and shows the label definition, not the recording, deciding whether anything transfers on clinical EEG. The practical lesson travels: report participant-held-out scores, publish the artefact controls beside the headline, and run a positive control so a null means the signal is absent, not that the pipeline is deaf. Until a fatigue model survives that standard, its accuracy is a property of its evaluation, not of fatigue.

## Preliminary work

This project follows up my earlier prototype, [Neuro-Fatigue-Prediction](https://github.com/visionbyangelic/Neuro-Fatigue-Prediction/tree/v0-preliminary), a 1D-CNN on one participant's Muse EEG that reported 99.18% recall distinguishing an "alert" session from a "fatigued" session. That prototype is the motivation for RQ4 above: this project rebuilt its setup exactly, reproduced its score, and established what the score was actually measuring.

## Repository structure

```
├── PLAN.md                  project plan and week-by-week checklist
├── track.md                 complete build log, every step, its reason, every error and result
├── docs/                    label rules, decision lock, dataset notes, diagnostics and results write-ups
├── src/                     data loading, preprocessing, features, models, evaluation
├── notebooks/               inspection and training notebooks (incl. the Colab CNN notebook)
├── experiments/             one folder per run: config, metrics, notes
├── results/                 summary tables and figures
└── data/                    local only, never committed
```

## How to run

`notebooks/01_data_inspection.ipynb` walks through the data, and `notebooks/04_cnn_colab.ipynb` runs the CNN training on Colab. The pipeline itself lives in `src/`; every number reported above traces to a run folder in `experiments/` and a summary table in `results/tables/`, with the full reasoning in `track.md`.

## Status

Complete. All four research questions are answered, both datasets are analysed, and the positive controls, diagnostics and sensitivity checks are done and written up in `docs/`. What remains is the writing.

# Erratum: freeze wording

Date: 2026-10-02
Applies to: the V-JEPA 2 temporal sensitivity study (30 Kinetics clips)

## Preserved reference

Commit `3926af1eb960794fd3607be58f9f313f4a79a750` is the preserved result-bearing reference. Its README, notebook, and results are unchanged. This erratum is added as a new commit on top of that history. No existing commit is rewritten or deleted.

## What was wrong

The README and the notebook say that `config.yaml` was written before any result was seen. The Git history does not support this. The README that contains the results (`a62ad770dbaf7506006bc038d19aa9b7c86931c1`) was committed before the standalone `config.yaml` file was added (`3f5ec5ec0b972cf667c83a01d35a870ed1ebf03f`), 65 seconds earlier.

## Correct status

The 30-clip analysis is exploratory and retrospective with respect to the formal freeze. `config.yaml` was added after the first results README, so it does not establish a prospective freeze. It records the settings used for the analysis.

Git history also does not show when the settings were first written outside the standalone file. An earlier notebook commit (`b7e6c5349ab9c5c90b3e015f3b37ade0e750794f`) already contains the same config text together with saved results. No contemporaneous record of the settings being fixed before the results is available.

## Where the unsupported wording appears

Cell numbers are zero-based. Line numbers are one-based.

1. `README.md`, section "What was frozen, and when", line 181: "`config.yaml` was written before any result was seen."
2. Notebook, cell 1, line 4, in the setup diagram under "How this notebook is organised": "config.yaml, written before any result was seen".
3. Notebook, cell 4, lines 3 and 4, under "Frozen protocol": "written before any results were seen" (the sentence is split across two lines in the source).

Read all three as superseded by this note.

## Related statements

Two other notebook sentences say a choice was fixed before results. They are not part of the three places above. The history neither confirms nor disproves them, so they should not be read as evidence of a prospective freeze.
These two not choices were made before the results, but no contemporaneous record exists to confirm it, so they are reported here as unverified.

- Cell 29, line 5: "Mean pooling is the primary method, fixed before results were seen."
- Cell 45, lines 3 to 6: the choices of mean pooling, cosine distance, an uncentered embedding space, and an all-other-clips denominator were "each fixed before the numbers came out".

The README (lines 188 to 191) and the notebook (cell 42) already say that two analysis choices were made during the analysis. That disclosure stands.

## What did not change

Every observed number and artifact is unchanged. Nothing was rerun for this correction.

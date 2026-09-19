# Does V-JEPA 2 encode motion, or mostly appearance?

V-JEPA 2 is a self-supervised video encoder. This study asks whether its frozen
embeddings respond when a clip's temporal structure is destroyed. Take a clip,
embed it, scramble time, embed it again, and measure how far the embedding moved
relative to the distance to an unrelated clip. Across 30 Kinetics clips, reversing
a clip moved the embedding by 0.0720 of that reference distance. Shuffling the
frames moved it by 0.2823. Replacing every frame with the middle frame moved it by
0.5002. The encoder responds to the direction of time, and the response is small
compared to what it does when motion is removed entirely.

Kaggle Notebook Link : To be Shared Soon

## Result

Normalized temporal sensitivity is the cosine distance between a clip's original
and perturbed embedding, divided by that clip's mean cosine distance to the other
29 clips' originals. Zero means the perturbation moved nothing. One means it moved
the embedding as far as substituting an unrelated clip.

| Model | reversed | shuffled | static |
|---|---|---|---|
| V-JEPA 2 | 0.0720 [0.0521, 0.0925] | 0.2823 [0.2377, 0.3291] | 0.5002 [0.4216, 0.5786] |
| DINOv2 (order-blind) | 1.91e-08 | 3.46e-08 | 0.0397 [0.0273, 0.0536] |
| random-init (architecture) | 0.0005 [0.0003, 0.0008] | 0.0131 [0.0054, 0.0244] | 0.0158 [0.0066, 0.0288] |

Intervals are percentile bootstrap over clips, 10,000 resamples, seeded.

Paired Wilcoxon signed-rank, reversed against shuffled, two-sided: W = 0,
p = 1.8626e-09. Reversed scored below shuffled on 30 of 30 clips.

Paired Wilcoxon, V-JEPA 2 against the random-init floor, one-sided: W = 465,
p = 9.3132e-10 in all three conditions. V-JEPA 2 exceeded the floor on 30 of 30
clips each time.

Four tests in total. The largest Holm-Bonferroni adjusted p-value is 3.7253e-09.

No one-sample test against zero is reported. The score is a ratio of distances
between non-identical vectors, so it is positive by construction, and at n = 30 any
all-positive sample returns the extreme statistic whatever the effect size.

## Method

Three perturbations, all applied to the raw frame tensor before the video
processor, never at token level:

- **reversed**: frames played backwards. Motion stays natural. The arrow of time
  is wrong.
- **shuffled**: random frame order. Destroys motion continuity, and also introduces
  jump cuts and changes what goes into each tubelet, so this condition is
  confounded.
- **static**: the middle frame of the window repeated 64 times. No motion at all.

Each clip is decoded to its first 64 frames, passed through
`facebook/vjepa2-vitl-fpc64-256`, and the resulting 8192 tokens are mean-pooled to
one 1024-dimensional vector. Nothing is trained. All encoders are frozen and only
forward passes are run.

Each of the three perturbation functions is checked before use: reversal is
verified by double-reversal returning the original exactly, shuffling by
determinism under a fixed seed and by conservation of the integer pixel sum, and
static by every frame equalling the middle frame. The checks run across all 30
clips.

## Controls

The design uses two controls because two different things can produce a nonzero
score.

**DINOv2, run frame by frame and averaged.** Averaging per-frame vectors is
permutation-invariant, so this encoder cannot see frame order. Its reversed and
shuffled scores are known in advance: zero, up to floating-point accumulation
error. The measured values are 1.91e-08 and 3.46e-08, which is that scale. Its
static score of 0.0397 is nonzero because replacing 63 frames with copies of one
frame changes the picture, and an appearance model registers that. This control
rules out the possibility that the metric responds to any perturbation rather than
to temporal change.

**A randomly initialized V-JEPA 2.** The patch embedding is a Conv3d two frames
deep whose kernel holds separate weights at each temporal position, and the encoder
applies position-aware attention over a reordered token sequence. An untrained
network with this architecture can therefore register reversal without having
learned anything about time. It scored 0.0005 against V-JEPA 2's 0.0720, with no
overlap in the intervals. Its denominator was 0.4272 against V-JEPA 2's 0.3604, so
its embeddings have not collapsed and its low scores are not an artifact of a
shorter reference scale. This control rules out the architecture as the source of
the effect.

## Robustness

Five variants of the analysis, each changing one choice:

| Variant | reversed | shuffled | static |
|---|---|---|---|
| primary (mean pooling, cosine) | 0.0720 | 0.2823 | 0.5002 |
| max pooling | 0.2457 | 0.3887 | 0.5969 |
| L1 distance | 0.2521 | 0.5106 | 0.6787 |
| L2 distance | 0.2421 | 0.5238 | 0.7034 |
| mean-centered embeddings | 0.0748 | 0.2349 | 0.4332 |
| cross-action denominator | 0.0672 | 0.2627 | 0.4653 |

The ordering reversed < shuffled < static held in all six. The magnitude of the gap
between reversed and shuffled did not.

## Limitations

Thirty clips across three action categories, ten each: archery, bowling, high jump.

Kinetics-mini rather than Something-Something v2. The Hugging Face dataset card for
Something-Something v2 hosts a loader script, not the videos. The videos are
distributed through Qualcomm under a separate data license agreement. Kinetics
labels can often be inferred from appearance alone (Sevilla-Lara et al., 2021), so
this set constrains motion dependence less tightly than Something-Something would.

The architecture-floor control rests on one initialization seed.

Mean pooling is not the representation V-JEPA 2's authors use. They evaluate with a
trained attentive probe. For a 16-frame ViT-L/16 the V-JEPA authors report attentive
probing beating average pooling by 16 to 17 points on Kinetics-400 and
Something-Something v2 (Bardes et al., 2024, Table 3).

This measures raw embedding geometry with nothing trained on top. It does not speak
to what a trained probe could recover from the same features.

Clips are 64 consecutive frames from the start of each video, about 2.1 seconds at
29.97 fps. The checkpoint was pretrained on 64-frame clips sampled at 4 fps, roughly
16 seconds of video. Inter-frame motion inside a tubelet here is about seven times
smaller than at training time, which plausibly suppresses the reversed score.

Re-running the pipeline in a fresh session reproduces the ordering and every
interval, but the means move by a few tenths of a percent. Reversal has landed at
0.0717, 0.0720 and 0.0722 across three runs. The forward pass is deterministic
within a session, so the variation comes from kernel selection across sessions.

## How to rerun

Hardware: one NVIDIA T4. The notebook was run on Kaggle with Python 3.12.13.

Data: the notebook downloads `nateraw/kinetics-mini` from the Hugging Face Hub,
pinned to revision `9f4ed38128a355c352527209101be3e326471816`. The pin means
"all validation clips from archery, bowling and high jump" resolves to the same 30
files every time. A Hugging Face token is read from Kaggle secrets under the name
`HUGGING_FACE_TOKEN`. Outside Kaggle, replace those calls with your own token
source.

Install:

```
pip install -q "transformers>=4.53.0" torchcodec
```

`scipy`, `matplotlib`, `numpy` and `pyyaml` came from the Kaggle base image and are
not pinned by the notebook. See `requirements.txt`.

Run every cell in order. On a clean working directory the four embedding caches do
not exist, so the encoding loops execute. The notebook records 8.03 seconds for one
cold clip including decoding and preprocessing. Total wall time is not recorded in
any cell output; read it from the Kaggle version page.

Delete `*.pt` from the working directory before re-running. Those files are keyed by
filename only, so a stale cache will be loaded silently in preference to
re-encoding.

## Figures

All seven come from the same 30 clips.

- `fig1.png` — V-JEPA 2 against both controls, with bootstrap intervals.
- `fig2.png` — the same data on a log axis, since the control bars are too small to
  see otherwise. Values are floored at 1e-9 before plotting.
- `fig3.png` — per-clip spread for V-JEPA 2.
- `fig4.png` — paired reversed against shuffled, one line per clip.
- `fig5.png` — robustness across pooling, centering and reference set. The first
  panel repeats the primary result.
- `fig6.png` — cosine, L1 and L2 distance.
- `fig7.png` — per-action breakdown. No error bars; ten clips per action is too few
  for a bootstrap interval to carry meaning.

## What was frozen, and when

`config.yaml` was written before any result was seen. It fixes the checkpoint
(`facebook/vjepa2-vitl-fpc64-256`), the control checkpoint
(`facebook/dinov2-base`), the primary pooling (mean) and the secondary pooling
(max), the random-init seed (42), the dataset repo and revision, the split, the
three action categories, the frame count (64), the bootstrap seed (21), the
bootstrap count (10,000), and the shuffle seed (42).

Two choices were not in `config.yaml` and were made during analysis: the
two-sided alternative for the reversed-against-shuffled test, and the decision not
to run one-sample tests against zero. Both are argued in the notebook's Significance
section.

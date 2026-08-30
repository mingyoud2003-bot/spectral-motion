# Spectral Motion

Code accompanying the MSc dissertation:

**Spectral Motion: Selective Motion Manipulation Using Spectral Total Variation of Optical Flow**

Department of Computer Science, University College London  
MSc Computer Graphics, Vision and Imaging, 2026

## Overview

This repository contains the experimental code used for the dissertation. The project investigates whether Spectral Total Variation (Spectral TV) can produce regionally distinguishable contributions within dense optical-flow fields that can support selective motion manipulation.

The controlled pipeline is:

```text
video frames
    -> optical flow
    -> vectorial Spectral TV decomposition
    -> regional spectral analysis
    -> calibration-only spectral-band selection
    -> selective motion editing
    -> held-out evaluation
```

Two controlled flow conditions are evaluated:

1. **Exact synthetic flow**, where the prescribed object motion is known directly.
2. **Horn-Schunck flow**, estimated from the rendered RGB frames.

A separate LASIESTA experiment provides a real-world **component-level** case study. It tests whether calibration-selected Spectral TV components retain different regional preferences on later estimated-flow fields. No spectral editing or RGB reconstruction is performed on LASIESTA.

## Repository contents

```text
.
├── README.md
├── requirements.txt
├── spectral_motion_experiments.ipynb
└── output/
    ├── figures/
    └── videos/
```

- `spectral_motion_experiments.ipynb` contains the complete experimental workflow used for the dissertation.
- `requirements.txt` lists the Python dependencies.
- `output/figures/` and `output/videos/` contain selected final visual outputs included with the repository.

The implementation is kept in one main notebook so that the numerical workflow can be followed in approximately the same order as the methodology and experiments in the dissertation.

## Runtime outputs

When the notebook is run, generated outputs are written to:

```text
spectral_motion_dissertation_outputs/
├── figures/
├── tables/
├── videos/
├── cache/
├── RESULTS_SUMMARY.md
└── run_manifest.json
```

This directory is created automatically if it does not already exist. Existing files are not automatically cleared before a new run.

At the end of a complete run, the notebook also creates:

```text
spectral_motion_dissertation_outputs.zip
```

Spectral TV decompositions are cached to avoid repeating expensive calculations when the same cached input and decomposition settings are encountered again.

In the saved final run of the notebook used for the dissertation, the notebook reports:

- **14 figures**
- **11 CSV tables**
- **5 MP4 videos**

## Controlled synthetic experiment

The controlled sequence contains two equal-sized circular objects translating horizontally over a static textured background.

- Sequence length: **7 frames**
- Resolution: **128 x 72 pixels**
- Object radius: **10 pixels**
- Target displacement: **3 pixels/frame**
- Distractor displacement: **1 pixel/frame**

The seven frames produce six pairwise flow fields:

```text
0->1   1->2   2->3   3->4   4->5   5->6
 |______|       |      |_____________|
 calibration  buffer       held-out
```

The split is fixed as follows:

- calibration: `0->1`, `1->2`
- unused buffer: `2->3`
- held-out evaluation: `3->4`, `4->5`, `5->6`

The buffer prevents the calibration and held-out subsets from sharing a video frame.

For Horn-Schunck flow, the smoothness parameter is calibrated using endpoint error on the two synthetic calibration pairs only. The selected value is then frozen for the remaining controlled pairs and transferred unchanged to the LASIESTA case study.

Spectral-band selection is also performed using calibration pairs only. Exact flow and Horn-Schunck flow are treated as separate input conditions, so they may select different spectral intervals while using the same selection rule and thresholds.

## Vectorial Spectral TV

The implementation uses a vectorial Total Variation functional that jointly regularises the horizontal and vertical optical-flow channels.

Repeated proximal updates generate a discrete TV-flow evolution,

```text
u_0 -> u_1 -> ... -> u_N
```

and the discrete spectral contributions are formed from second differences of neighbouring stored states.

The finite representation contains:

- `N - 1` vector-valued spectral components; and
- an endpoint residual.

Relative reconstruction error is used as a numerical consistency check that the stored spectral components and residual reproduce the input flow field. It is not used as evidence of proximal convergence or motion separation.

## Calibration-only spectral-band selection

For the controlled experiment, each spectral component is measured in the Target and Distractor regions through signed projection of its regional mean vector onto the region's original mean-motion direction.

The fixed selection criteria are:

- Target capture threshold: **0.80**
- Distractor directional-change limit: **0.10**

Candidate bands are contiguous intervals containing the largest positive Target response. Among intervals satisfying both criteria, the narrowest interval is preferred, with deterministic tie-breaking.

For the reported run, calibration selects:

- exact flow: Target peak `psi26`, interval `psi25-psi29`
- Horn-Schunck flow: Target peak `psi31`, interval `psi27-psi37`

The selected intervals are frozen before held-out evaluation and are not re-selected on later frame pairs.

## Selective motion editing

For the selected band contribution `psi_B`, the edited flow is:

```text
v^(g) = v + (g - 1) * psi_B
```

The reported gains are:

- `g = 0`: suppression
- `g = 1`: unchanged flow
- `g = 2`: amplification

Editing is performed in optical-flow space.

### Evaluation metrics

The controlled held-out evaluation uses two complementary measurements.

**Retained mean directional motion** measures the region's mean motion along its original mean direction.

**Full-vector endpoint change** measures the pointwise two-dimensional change inside each ROI. This complements the directional metric because it responds to changes in either flow component and to spatially varying changes inside the region.

For the controlled background, absolute endpoint change is reported in pixels/frame because exact synthetic background flow is zero.

## Flow-driven trajectory visualisation

For the synthetic sequence, precomputed edited regional mean flows are integrated across the held-out transitions and passed to the same deterministic renderer used to create the original controlled sequence.

No new optical flow is estimated from the re-rendered frames, and the original flow field is not re-sampled at the newly displayed object positions.

These outputs are therefore visualisations of the motion implied by the edited flow fields rather than a general RGB reconstruction method.

## LASIESTA case study

The real-world case study uses LASIESTA sequence:

```text
O_SU_02
```

The notebook is configured to use the official RGB sequence and ground-truth annotations. If a suitable local copy is not found, it attempts to download the required archives from the official LASIESTA host.

The selected seven-frame window is:

```text
233-239
```

The window is chosen from the annotations using visibility criteria before Spectral TV responses are examined. Persistent pedestrian labels `1` and `2` define the Target and Distractor measurement regions. The annotations are used for ROI/evaluation masks only and are not supplied to Horn-Schunck or Spectral TV.

Important settings are:

- resized resolution: **200 x 160 pixels** (width x height)
- minimum ROI area: **25 pixels**
- local-background dilation radius: **28 pixels**
- Horn-Schunck alpha: **0.06**, transferred from the synthetic calibration
- calibration transitions: first two local pairs
- unused buffer: third local pair
- held-out evaluation: final three local pairs

Calibration selects:

- Target-relative component: `psi6`
- Distractor-relative component: `psi28`

These component indices are then frozen and evaluated on all three held-out LASIESTA transitions. In the saved final run, `psi6` is Target-dominant in 3/3 held-out pairs and `psi28` is Distractor-dominant in 3/3 held-out pairs.

The separation is partial rather than object-specific. This branch is intentionally limited to regional component analysis and does not perform selective motion editing on the real video.

The LASIESTA dataset itself is not redistributed in this repository.

## Main numerical settings

| Parameter | Value |
| --- | ---: |
| Random seed | 7 |
| TV-flow step size | 0.5 |
| Number of TV-flow steps | 42 |
| Spectral components | 41 plus residual |
| PDHG iterations per TV step | 250 |
| PDHG primal step | 0.24 |
| PDHG dual step | 0.24 |
| Horn-Schunck pyramid levels | 3 |
| Horn-Schunck warps per level | 4 |
| Horn-Schunck iterations per warp | 100 |
| Horn-Schunck alpha candidates | 0.04, 0.06, 0.08, 0.10, 0.12, 0.16, 0.20 |
| Selected Horn-Schunck alpha | 0.06 |
| Target capture threshold | 0.80 |
| Distractor directional-change limit | 0.10 |
| Editing gains | 0, 1, 2 |

The notebook automatically uses CUDA when available and otherwise runs on CPU.

## Requirements

Install the Python dependencies with:

```bash
pip install -r requirements.txt
```

The notebook uses packages including NumPy, pandas, Matplotlib, OpenCV, PyTorch, ImageIO and IPython.

For automatic LASIESTA download, internet access is required. The official sequence is distributed in archive files. In a Colab/Linux environment, the notebook can attempt to install `unar` when needed for RAR extraction and also checks for available extraction alternatives.

## Running the notebook

Open:

```text
spectral_motion_experiments.ipynb
```

and run the cells in order, or use **Run all** in a clean environment.

A complete run performs:

1. controlled-sequence generation;
2. Horn-Schunck calibration;
3. exact-flow and Horn-Schunck Spectral TV decomposition;
4. spectral reconstruction checks;
5. calibration-only spectral-band selection;
6. held-out selective motion editing;
7. directional and full-vector evaluation;
8. held-out flow visualisation;
9. flow-driven trajectory re-rendering;
10. LASIESTA component-level evaluation;
11. generation of summary tables, figures, videos and the output archive.

The five controlled videos generated by the notebook are:

```text
video_00_controlled_input.mp4
video_01_exact_flow_control.mp4
video_02_horn_schunck_flow_control.mp4
video_03_exact_object_motion.mp4
video_04_horn_schunck_object_motion.mp4
```

## Reproducibility

The controlled experiments use fixed NumPy and PyTorch random seeds. The calibration/held-out protocol is fixed in the notebook.

The main safeguards are:

- Horn-Schunck alpha is selected using synthetic calibration pairs only.
- Spectral intervals are selected from calibration responses only.
- The buffer pair is not used for calibration or the main held-out evaluation.
- Selected intervals remain fixed across the three controlled held-out pairs.
- LASIESTA uses the Horn-Schunck alpha transferred from the controlled calibration without additional retuning.
- LASIESTA component indices are selected from calibration pairs and evaluated unchanged on all three later held-out pairs.
- Reconstruction error is used only as a finite-representation consistency check.

Small numerical differences may occur across hardware and software environments, but the experimental protocol and selection rules are fixed for the reported experiments.

## Scope

The main quantitative experiment is a controlled proof of concept for selective manipulation in **optical-flow space**.

The synthetic trajectory visualisations use the known deterministic renderer to display the motion implied by the edited fields. They are not a general method for reconstructing arbitrary RGB video.

The LASIESTA experiment provides a real-world component-level check of regional spectral preference rather than a complete real-video editing pipeline.

## Dissertation correspondence

The notebook corresponds primarily to:

- **Chapter 4 — Methodology and Implementation**
- **Chapter 5 — Experimental Evaluation**
- **Chapter 6 — Discussion and Conclusion**

The repository accompanies the dissertation and provides the code and supplementary outputs used to reproduce and inspect the reported experiments.

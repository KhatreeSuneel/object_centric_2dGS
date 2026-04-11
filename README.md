# 2D Gaussian Splatting for Geometrically Accurate 3D Object Reconstruction

This repository builds upon [2D Gaussian Splatting (2DGS)](https://surfsplatting.github.io/) to achieve **efficient, geometrically accurate, object-centric 3D reconstruction** from multi-view images. By integrating segmentation masks directly into the optimization process, the method reconstructs only the target object — eliminating background Gaussians and significantly reducing training time.

> **Lab Rotation Report** · Institut für Photogrammetrie und Fernerkundung (IPF), Karlsruher Institut für Technologie (KIT)
> Supervised by Dr.-Ing. Markus Hillemann · Examiner: Prof. Dr.-Ing. Markus Ulrich

---

## Overview

Standard Gaussian Splatting pipelines (both 3DGS and 2DGS) reconstruct entire scenes, which is computationally wasteful when the goal is object-specific reconstruction. Extracting a clean mesh of a single object from a full-scene Gaussian representation is also non-trivial.

This work introduces a **mask loss** using binary segmentation masks during the 2DGS optimization loop. The mask loss pushes the opacity of background Gaussians toward zero, which are then automatically pruned via density control — leaving only the target object. This approach achieves:

- **Sub-millimeter reconstruction accuracy** (comparable to full-scene 2DGS)
- **~67% faster training** (≈3 min vs ≈9 min per scene)
- **No post-processing mesh culling** — the output mesh contains only the target object

### Method Overview

The total optimization loss is:

```
L = Lc + α·Ld + β·Ln + δ·Lm
```

where:
- **Lc** — Photometric loss (L1 + D-SSIM)
- **Ld** — Depth distortion regularization
- **Ln** — Normal consistency regularization
- **Lm** — Mask loss (binary cross-entropy between predicted alpha and object mask)

Loss coefficients: `α = 100`, `β = 0.05`, `δ = 0.01`

### Key Results

| Method | Avg. Training Time | PSNR ↑ | SSIM ↑ | LPIPS ↓ |
|---|---|---|---|---|
| Ours (full mask) | **3:06** | 37.70 | 0.995 | 0.007 |
| Ours (vis mask) | **3:02** | 35.07 | 0.994 | 0.009 |
| 2DGS (scene) | 9:14 | 24.83 | 0.826 | 0.191 |
| 2DGS (full mask) | 9:14 | 45.79 | 0.997 | 0.004 |
| 2DGS (vis mask) | 9:14 | 47.19 | 0.998 | 0.003 |

Mesh quality (signed distance vs. CAD model) is on par with 2DGS across all test objects, with both methods achieving average distances below 1 mm.

---

## Installation

```bash
# Clone the repository (recursive for submodules)
git clone https://github.com/KhatreeSuneel/object_centric_2dGS.git --recursive

# Create and activate environment
conda env create --file environment.yml
conda activate surfel_splatting
```

---

## Data Preparation

The framework follows the same COLMAP-based input format as 2DGS. Your dataset should be structured as:

```
<location>
|---images
|   |---<image 0>
|   |---<image 1>
|   |---...
|---masks
|   |---<mask 0>
|   |---<mask 1>
|   |---...
|---sparse
    |---0
        |---cameras.bin
        |---images.bin
        |---points3D.bin
```

**Masks** should be binary images where pixel value `1` indicates the object and `0` indicates background. Two mask types are supported:
- **Full mask** — covers the entire object including occluded regions
- **Visible mask** — covers only the visible (non-occluded) parts of the object

If masks are not available for your dataset, they can be generated using [Segment Anything Model (SAM)](https://github.com/facebookresearch/segment-anything) or [SAM2](https://github.com/facebookresearch/segment-anything-2).

For preparing your own COLMAP data, follow the instructions [here](https://github.com/graphdeco-inria/gaussian-splatting?tab=readme-ov-file#processing-your-own-scenes).

---

## Training

To train on a scene with mask-guided object reconstruction:

```bash
python train.py -s <path to COLMAP dataset> --mask_path <path to masks>
```

Key regularization arguments (same as 2DGS):

```bash
--lambda_normal      # hyperparameter for normal consistency
--lambda_distortion  # hyperparameter for depth distortion
--depth_ratio        # 0 for mean depth, 1 for median depth (0 works for most cases)
```

All experiments were run for **30,000 iterations** on an RTX 4090 GPU.

---

## Mesh Extraction

### Bounded Mesh Extraction

```bash
python render.py -m <path to trained model> -s <path to COLMAP dataset>
```

Adjust the following for TSDF fusion:

```bash
--depth_ratio   # 0 for mean depth, 1 for median depth
--voxel_size    # voxel size for TSDF
--depth_trunc   # depth truncation threshold
```

### Unbounded Mesh Extraction

```bash
python render.py -m <path to trained model> -s <path to COLMAP dataset> --mesh_res 1024
```

Since our method trains on the object only, the extracted mesh directly corresponds to the target object — no post-processing culling is required.

---

## Evaluation

### Novel View Synthesis

```bash
python scripts/mipnerf_eval.py -m60 <path to dataset>
```

### Mesh Reconstruction

Mesh quality is evaluated against ground truth CAD models using signed distance metrics in [CloudCompare](https://www.danielgm.net/cc/).

---

## Dataset

Experiments were conducted on the [T-LESS dataset](https://bop.felk.cvut.cz/datasets/) — an RGB-D dataset for 6D pose estimation of texture-less objects. Five table-top scenes (scenes 10–17) were used, each containing a different target object in cluttered environments with varying levels of occlusion.

---

## Acknowledgements

This project builds upon [2D Gaussian Splatting](https://github.com/hbb1/2d-gaussian-splatting) by Huang et al. The mask loss formulation is inspired by [GaussianObject](https://github.com/GaussianObject/GaussianObject). TSDF fusion is based on [Open3D](https://github.com/isl-org/Open3D).

---

## Citation

If you find this work useful, please consider citing the original 2DGS paper:

```bibtex
@inproceedings{Huang2DGS2024,
    title={2D Gaussian Splatting for Geometrically Accurate Radiance Fields},
    author={Huang, Binbin and Yu, Zehao and Chen, Anpei and Geiger, Andreas and Gao, Shenghua},
    publisher = {Association for Computing Machinery},
    booktitle = {SIGGRAPH 2024 Conference Papers},
    year      = {2024},
    doi       = {10.1145/3641519.3657428}
}
```

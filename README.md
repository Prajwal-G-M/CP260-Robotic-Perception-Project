# CP260 Robotic Perception — Final Project
## Metric-Semantic 3D Reconstruction of a PC Tower Back Panel

> **Course:** CP260-2026 &nbsp;|&nbsp; **Due:** May 4, 2026

---

## Overview

This project implements a **Metric-Semantic Reconstruction** pipeline that localizes hardware ports on a PC tower in 3D space using only posed RGB images — no depth sensor required.

Given **16 posed RGB images** of a PC tower's back panel, the system:

1. Detects semantic objects (sockets) using **GroundingDINO** (zero-shot, no fine-tuning)
2. Collects 2D bounding box detections across all views
3. **Triangulates** rays from multiple camera poses to estimate 3D positions
4. Fits **Oriented Bounding Boxes (OBBs)** for each detected entity
5. Outputs a structured `answer.json` with center, extent, and rotation for each entity

---

## Detected Entities

| Entity | Description |
|---|---|
| `vga_socket` | VGA / D-Sub connector port |
| `ethernet_socket` | RJ45 LAN / ethernet port |
| `power_socket` | IEC C14 power inlet |

---

## Pipeline Architecture

```
16 Posed RGB Images
        │
        ▼
┌─────────────────────┐
│   GroundingDINO     │  ← Zero-shot text-prompted detection
│  (HuggingFace)      │    "VGA port. D-sub connector." etc.
└────────┬────────────┘
         │  2D bounding boxes + confidence scores (per frame)
         ▼
┌─────────────────────┐
│  Multi-View Ray     │  ← Unproject pixel → ray in world frame
│  Triangulation      │    Least-squares: minimize dist to all rays
└────────┬────────────┘
         │  3D center point
         ▼
┌─────────────────────┐
│  OBB Estimation     │  ← Extent from avg pixel size + depth
│                     │    Rotation from GT VGA panel normal
└────────┬────────────┘
         │
         ▼
     answer.json
```

---

## Results

Final predicted Oriented Bounding Boxes (`answer_improved.json`):

| Entity | Center (x, y, z) | Extent (half-dims) |
|---|---|---|
| `vga_socket` | (0.2705, 0.2261, 0.8349) | (0.0354, 0.0118, 0.0061) |
| `ethernet_socket` | (0.1887, 0.0790, 0.8550) | (0.0679, 0.0717, 0.0136) |
| `power_socket` | (0.1449, 0.1650, 0.8787) | (0.0140, 0.0100, 0.0080) |

---

## Repository Structure

```
.
├── Robotic_Perception_Final.ipynb   # Main pipeline notebook (Google Colab)
├── answer_improved.json             # Final predicted OBBs (submission)
└── README.md
```

**Generated outputs** (produced when the notebook is run):

```
answer.json              ← Final submission file
sample_frames.png        ← All 16 dataset frames visualized
test_detections.png      ← GroundingDINO detection results
vga_validation.png       ← GT VGA projection validation overlay
```

---

## Setup & Usage

### Requirements

- Google Colab (recommended — free T4 GPU)
- Google Drive with dataset mounted at:
  ```
  My Drive/Robotic Perception Project/Data/Data/
  ```
  containing: `frame_000319.png` ... `frame_000531.png`, `poses.json`, `intrinsic.json`

### Dataset

- **16 RGB frames** of a PC tower back panel (2560 × 1440 resolution)
- Frame numbers: `319, 333, 353, 359, 365, 371, 390, 400, 426, 449, 461, 468, 471, 496, 515, 531`
- `poses.json` — camera-to-world 4×4 pose matrices (704 total poses)
- `intrinsic.json` — camera intrinsic matrix K

### Running the Notebook

1. Open `Robotic_Perception_Final.ipynb` in **Google Colab**
2. Set runtime to **T4 GPU** (Runtime → Change Runtime Type)
3. Run all cells in order (Steps 1–14)
4. `answer.json` will be auto-downloaded on completion

---

## Method Details

### Zero-Shot Detection — GroundingDINO

Uses [`IDEA-Research/grounding-dino-tiny`](https://huggingface.co/IDEA-Research/grounding-dino-tiny) from HuggingFace Transformers. No training or fine-tuning — entities are specified via natural language prompts:

```python
TEXT_PROMPTS = {
    "vga_socket":      "VGA port. D-sub connector.",
    "ethernet_socket": "RJ45 ethernet port. LAN socket.",
    "power_socket":    "IEC C14 power inlet. power socket.",
}
```

Detection threshold: `0.15` confidence. Best-scoring detection per frame is kept.

### Multi-View Ray Triangulation

For each detected pixel `(cx, cy)` in frame `i`:

1. Unproject to normalized camera coordinates using intrinsic matrix K
2. Rotate into world frame using the camera-to-world rotation R
3. Formulate the least-squares system:

$$\min_x \sum_i \| (I - \mathbf{d}_i \mathbf{d}_i^\top)(x - \mathbf{o}_i) \|^2$$

4. Solve with `np.linalg.lstsq` to get the 3D center

### OBB Estimation

- **Center**: from multi-view triangulation above
- **Extent**: derived from average pixel bounding box size × estimated depth
- **Rotation**: shared across all entities — inherited from the GT VGA socket panel normal (all ports face the same back panel)

---

## Dependencies

Installed automatically in the notebook:

```
torch
open3d
opencv-python-headless
transformers
timm
supervision
```

---

## Key Design Choices

- **Zero-shot detection** avoids the need for labelled training data
- **Ray triangulation** (not depth fusion) works with standard RGB cameras
- **Shared panel rotation** leverages the known geometry of PC I/O panels — all ports face the same direction
- **GT VGA anchor** is used as a calibration reference for the full scene

---

## Course

CP260 — Robotic Perception, 2026

# 🚗 Real-Time Indian License Plate Recognition (ANPR)
### Edge AI Deployment on Raspberry Pi 5 + Hailo-8L

> ⚠️ **Repository Status: Private**
> All source code in this repository is private and restricted from
> public access pending publication. The repository will be made
> fully open-source upon acceptance of the associated research paper.
> Dataset and model weights will be released alongside publication.

---

## 📋 Table of Contents
- [Overview](#overview)
- [Key Contributions](#key-contributions)
- [System Architecture](#system-architecture)
- [Dataset](#dataset)
- [Models](#models)
- [OCR Pipeline](#ocr-pipeline)
- [Vehicle Tracking](#vehicle-tracking)
- [Edge Deployment](#edge-deployment)
- [Results](#results)
- [Demo](#demo)
- [Citation](#citation)
- [License](#license)

---

## Overview

This repository contains the implementation of a real-time Automatic
Number Plate Recognition (ANPR) system specifically designed for
**Indian license plates** — a problem space largely unaddressed by
existing open-source ANPR solutions.

Indian license plates present unique challenges that international
systems fail to handle:

- **Dual-line HSRP format** — state code and serial number on
  separate rows, requiring explicit line segmentation before OCR
- **Legacy pre-HSRP plates** — hand-painted, non-uniform fonts,
  inconsistent character spacing
- **Extreme environmental conditions** — monsoon glare, nighttime
  low-sodium lighting, dust occlusion, motion blur
- **Heterogeneous vehicle mix** — cars, trucks, two-wheelers,
  auto-rickshaws, tractors, each with different plate conventions

The complete pipeline runs at **30 FPS end-to-end** on a
**Raspberry Pi 5 + Hailo-8L AI HAT**, making it one of the
lowest-cost real-time ANPR deployments documented.

---

## Key Contributions

| # | Contribution | Status |
|---|---|---|
| 1 | Large-scale multi-state Indian plate dataset (single-line + dual-line HSRP, legacy, commercial, two-wheeler) | Pending release |
| 2 | Systematic comparison of YOLOv8n, YOLOv11n, YOLOv26n across vehicle and plate detection stages | ✅ Complete |
| 3 | Multi-OCR benchmarking: LPRNet, PaddleOCR, EasyOCR on Indian plate formats | ✅ Complete |
| 4 | Geometric dual-line plate segmentation algorithm | ✅ Complete |
| 5 | Modified SORT tracker with adaptive-noise Kalman model and class-conditional association for dense Indian traffic | ✅ Complete |
| 6 | Real-time Hailo-8 edge deployment at 30 FPS | ✅ Complete |

---

## System Architecture
┌─────────────────────────────────────────────────────────┐
│                    Input Stream                         │
│           Raspberry Pi Camera Module 3                  │
│                    1080p @ 30fps                        │
└────────────────────────┬────────────────────────────────┘
│
▼
┌─────────────────────────────────────────────────────────┐
│              Stage 1 — Vehicle Detection                │
│         YOLOv8n (INT8, Hailo-8L NPU)                   │
│         Detects: car, truck, bus, two-wheeler           │
└────────────────────────┬────────────────────────────────┘
│  Vehicle ROI crops
▼
┌─────────────────────────────────────────────────────────┐
│              Modified SORT Tracker                      │
│  - Constant-acceleration Kalman state                   │
│  - Adaptive process noise (innovation-scaled)           │
│  - Class-conditional IoU thresholds                     │
│  - Class-conditional max_age (track persistence)        │
│  - Aspect-ratio / size consistency tiebreaker           │
└────────────────────────┬────────────────────────────────┘
│  Tracked vehicle crops (with ID)
▼
┌─────────────────────────────────────────────────────────┐
│            Stage 2 — Plate Detection                    │
│         YOLOv8n (INT8, Hailo-8L NPU)                   │
│         Localizes plate within vehicle crop             │
└────────────────────────┬────────────────────────────────┘
│  Plate crops
▼
┌─────────────────────────────────────────────────────────┐
│        Dual-Line Segmentation (CPU)                     │
│  - Aspect ratio classification                          │
│  - Horizontal projection profile                        │
│  - Valley detection → top / bottom sub-crops           │
└────────────────────────┬────────────────────────────────┘
│  Segmented crops
▼
┌─────────────────────────────────────────────────────────┐
│              OCR — PaddleOCR PP-OCRv4                   │
│         (Hailo-8L NPU, fine-tuned)                      │
│         Best-confidence read per vehicle track          │
└────────────────────────┬────────────────────────────────┘
│
▼
Plate string + Track ID + Confidence


---

## Dataset

> 📦 Dataset will be released upon paper acceptance.

| Split | Images | Single-line HSRP | Dual-line HSRP | Legacy | Commercial | Two-Wheeler |
|---|---|---|---|---|---|---|
| Train | — | — | — | — | — | — |
| Val | — | — | — | — | — | — |
| Test | — | — | — | — | — | — |
| **Total** | — | — | — | — | — | — |

**Coverage:**
- States: [ to be filled ]
- Vehicle classes: private cars, trucks, buses, auto-rickshaws,
  motorcycles, tractors
- Conditions: daytime, nighttime, dusk, monsoon glare, dust, motion blur
- Plate conditions: clean, moderate degradation, severe occlusion,
  hand-painted legacy

---

## Models

### Detection — Stage 1 (Vehicle) and Stage 2 (Plate)

| Stage | Model | Val mAP@0.5 | Val mAP@0.5-95 | Test mAP@0.5 | Test mAP@0.5-95 | Params |
|---|---|---|---|---|---|---|
| Vehicle | YOLOv8n | — | — | — | — | — |
| Vehicle | YOLOv11n | — | — | — | — | — |
| Vehicle | YOLOv26n | — | — | — | — | — |
| Plate | YOLOv8n | — | — | 95.0 | 84.0 | — |
| Plate | YOLOv11n | — | — | 93.0 | 78.0 | — |
| Plate | YOLOv26n | — | — | 92.0 | 72.0 | — |

> **Note on model selection:** YOLOv8n achieves the highest mAP@0.5-95
> on the plate detection stage, indicating tighter localization precision —
> which directly affects OCR crop quality. YOLOv8n is selected as the
> deployed configuration. YOLOv11n and YOLOv26n INT8 quantization for
> Hailo-8L is not yet fully supported by the Hailo Dataflow Compiler at
> time of writing; deployment latency is reported for YOLOv8n only.

> 📦 Model weights will be released upon paper acceptance.

---

## OCR Pipeline

Three backends were benchmarked on the full Indian plate test set:

| OCR Backend | Fine-tuned | Single-line CER (%) | Single-line PLA (%) | Dual-line CER (%) | Dual-line PLA (%) | Latency/crop GPU (ms) | Latency/crop Hailo (ms) |
|---|---|---|---|---|---|---|---|
| LPRNet | Yes | — | — | — | — | — | — |
| PaddleOCR | Yes | — | — | — | — | — | — |
| EasyOCR | No (zero-shot) | — | — | — | — | — | — |

**Selected backend: PaddleOCR (PP-OCRv4)**
PaddleOCR's 2D attention architecture handles both single-line and
dual-line plate formats without requiring per-format routing, outperforming
LPRNet and EasyOCR across both formats. LPRNet's CTC decoder assumes a
single horizontal character sequence and cannot natively decode dual-line
plates without explicit pre-segmentation.

---

## Vehicle Tracking

Standard SORT fails on dense Indian mixed traffic due to two specific
mechanisms:

1. **Motion model mismatch** — constant-velocity Kalman prediction lags
   behind the erratic lateral movement of lane-splitting two-wheelers
2. **IoU ambiguity** — small, tightly-packed two-wheeler boxes overlap
   neighbouring vehicle tracks during weaving maneuvers

Our modified tracker addresses both:

### Modifications over vanilla SORT

| Component | Vanilla SORT | This work |
|---|---|---|
| Kalman state | `[x, y, w, h, vx, vy]` | `[x, y, w, h, vx, vy, ax, ay]` |
| Process noise Q | Fixed | Innovation-adaptive (scales with prediction residual) |
| IoU threshold | Fixed (0.3) | Class-conditional (two-wheeler: higher; car/truck: standard) |
| Track persistence (max_age) | Fixed | Class-conditional (two-wheeler: extended) |
| Match tiebreaker | None | Aspect-ratio + size consistency with track history |

Full methodology and ablation results are documented in the associated
paper.

---

## Edge Deployment

**Hardware:**

| Component | Specification |
|---|---|
| SBC | Raspberry Pi 5 (4-core Cortex-A76 @ 2.4 GHz, 8 GB LPDDR4X) |
| AI Accelerator | Hailo-8L NPU via Raspberry Pi AI HAT+ (13 TOPS INT8) |
| Camera | Raspberry Pi Camera Module 3 (Sony IMX708, 12 MP, 1080p30) |
| Approximate cost | ~$120 USD |

**Pipeline latency breakdown:**

| Stage | Mean latency (ms) | % of total |
|---|---|---|
| Image capture + preprocessing | — | — |
| Stage 1: Vehicle detection (Hailo NPU) | — | — |
| ROI crop extraction (CPU) | — | — |
| Stage 2: Plate detection (Hailo NPU) | — | — |
| Dual-line classification (CPU) | — | — |
| OCR inference (Hailo NPU) | — | — |
| Post-processing + output (CPU) | — | — |
| **Total** | **~33 ms** | **100%** |

**Throughput: 30.2 FPS (σ = 1.8 FPS over 10-minute continuous test)**

**Quantization:**
- YOLOv8n: INT8 via Hailo Dataflow Compiler, 1000-image calibration set
- YOLOv11n / YOLOv26n: INT8 quantization not yet supported stably
  by Hailo DFC at time of writing — CPU/GPU accuracy numbers reported
  for comparison, deployment latency not applicable

---

## Results

### Comparison against existing ANPR systems

| System | Single-line CER (%) | Single-line PLA (%) | Dual-line CER (%) | Dual-line PLA (%) | FPS (RPi5) |
|---|---|---|---|---|---|
| OpenALPR | — | — | — | — | — |
| YOLOv8n (COCO pretrained, no fine-tuning) | — | — | — | — | — |
| YOLOv5 + UFPR-ALPR weights | — | — | — | — | — |
| **Ours (best config)** | — | — | — | — | **30.2** |

### Ablation study

| Configuration | Dual-line PLA (%) | Single-line PLA (%) | Overall CER (%) | FPS |
|---|---|---|---|---|
| Full pipeline | — | — | — | 30.2 |
| No dual-line segmentation | — | — | — | — |
| CPU-only (no Hailo) | — | — | — | — |
| Vanilla SORT (no tracker modifications) | — | — | — | — |
| Weakest S2 detector | — | — | — | — |
| EasyOCR zero-shot | — | — | — | — |

---

## Demo

> 📹 **Video walkthrough coming soon**
>
> [ Video embed / link to be added ]
>
> The demo will cover:
> - Hardware setup (Raspberry Pi 5 + Hailo AI HAT)
> - Pipeline walkthrough (vehicle detection → tracking → plate detection → OCR)
> - Live inference on Indian road footage
> - Single-line vs dual-line plate handling comparison
> - ID tracking across frames

---

## Citation

If you use this work, dataset, or model weights in your research,
please cite:

```bibtex
@article{[author]2025indiananpr,
  title   = {Real-Time Indian License Plate Recognition on Edge AI
             Hardware: A Multi-Format, Multi-Model Pipeline for
             Single-Line and Dual-Line HSRP Plates},
  author  = {[Author Names]},
  journal = {IEEE Transactions on Intelligent Transportation Systems},
  year    = {2025},
  note    = {Under review}
}
```

---

## License

- **Code:** Private — not available for use, modification, or
  distribution pending publication. Will be released under
  MIT License upon paper acceptance.
- **Dataset:** Private — will be released under CC BY 4.0 upon
  paper acceptance.
- **Model weights:** Private — will be released upon paper acceptance.

---

## Acknowledgements

[ To be filled — collaborators, funding, institutions ]

---

*Associated paper submitted to IEEE Transactions on Intelligent
Transportation Systems (T-ITS). For enquiries, contact [ email ].*

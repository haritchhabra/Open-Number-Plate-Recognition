# Real-Time Indian License Plate Recognition (ANPR)
### Edge AI Deployment on Raspberry Pi 5 + Hailo-8 NPU

>  **Repository Status: Private**
> All source code in this repository is private and restricted from
> public access pending publication. The repository will be made
> fully open-source upon acceptance of the associated research paper.
> Dataset and model weights will be released alongside publication.

<p align="left">
  <img src="demo/demo.gif" width="450">
</p>
<p align="right">
  <img src="demo/outputf3.gif" width="450">
</p>

# Table of Contents
- [Overview](#overview)
- [Key Contributions](#key-contributions)
- [Datasets](#datasets)
- [Models](#models)
- [OCR Pipeline](#ocr-pipeline)
- [Edge Deployment](#edge-deployment)

## Overview

This repository contains the implementation of a real-time Automatic
Number Plate Recognition (ANPR) system specifically designed for
**Indian license plates** a problem space largely unaddressed by
existing open-source ANPR solutions.

Indian license plates present unique challenges that international
systems fail to handle:

- **Dual-line HSRP format** — state code and serial number on
  separate rows, requiring explicit line segmentation before OCR
- **Older plates** — hand-painted, non-uniform fonts,
  inconsistent character spacing
- **Extreme environmental conditions** — monsoon glare, nighttime
  low-sodium lighting, dust occlusion, motion blur
- **Heterogeneous vehicle mix** — cars, trucks, two-wheelers,
  auto-rickshaws, tractors, each with different plate conventions

The complete pipeline runs at **30 FPS end-to-end** on a
**Raspberry Pi 5 + Hailo-8 NPU (AI HAT)**, making it one of the
lowest-cost real-time ANPR deployments documented for Indian traffic.


## Key Contributions

| # | Contribution | Status |
|---|---|---|
| 1 | Large-scale multi-state Indian plate dataset (single-line + dual-line HSRP, legacy, commercial, two-wheeler) | Pending release |
| 2 | Two-stage detection pipeline: vehicle detection (IISc UVH-26 + custom Kaggle dataset) → plate localization (custom Indian plate dataset) | Complete |
| 3 | Systematic comparison of YOLOv8n, YOLOv11n, YOLOv26n  | Complete |
| 4 | Multi-OCR benchmarking: LPRNet, PaddleOCR, EasyOCR on single-line and dual-line Indian plates | Complete |
| 5 | Modified SORT tracker with adaptive-noise Kalman model and class-conditional association for dense Indian mixed traffic | Complete |
| 6 | Real-time Hailo-8 NPU deployment at 30 FPS on sub $500 hardware (₹48000) | Complete |


## Datasets

### Stage 1 Vehicle Detection

Two datasets were used in sequence for vehicle detection model training:

#### 1. Custom Kaggle Composite Dataset (Initial Fine-Tuning)

| Property | Detail |
|---|---|
| Source | Aggregated from open-source vehicle detection datasets on Kaggle |
| Images | ~6,000 images |
| Purpose | Initial fine-tuning cycle domain adaptation from ImageNet weights to vehicle detection |
| Vehicle classes | Car, bus, bike, truck, auto |
| Notes | Used as first-cycle fine-tuning before domain-specific IISc dataset |

#### 2. IISc UVH-26 Urban Vision Hackathon Dataset (Primary Training)

| Property | Detail |
|---|---|
| Source | AIM Group, Indian Institute of Science (IISc), Bengaluru |
| Full name | UVH-26 Urban Vision Hackathon vehicle detection dataset |
| Images | 26,646 HD images at 1920 × 1080 resolution |
| Bounding box annotations | ~1.8 million |
| Vehicle categories | 14 classes: hatchbacks, sedans, SUVs, buses, trucks, two-wheelers, three-wheelers, bicycles, light commercial vehicles, and others |
| Mapped 14 vehicle classes to 5 classes |14 → 5 class mapping (car, bus, truck, motorcycle, auto rickshaw); bicycles and "others" excluded |
| Camera source | ~2,800 Bengaluru Safe City CCTV cameras |
| Training and Validation split used | ~21,000 images |
| Test split used | ~5,000 images |
| Viewpoint | Elevated CCTV / top-view, closely matching deployment camera placement |
| Conditions | Daytime, evening, nighttime multi-illumination |
| Traffic density | Dense, congested, multi-vehicle scenes with partial occlusion and overlap |

**Preprocessing applied to UVH-26:**
- Original RGB images converted to grayscale to match the monochrome
  output of the deployment camera
- Grayscale images replicated into 3 identical channels to satisfy
  YOLO's 3 channel input requirement
- This preprocessing ensures training data characteristics closely
  match the target deployment environment

**Rationale for UVH-26 selection:**
The UVH-26 dataset was selected specifically because its elevated
CCTV viewpoint, dense multi-vehicle scenes, partial occlusions, and
multi-illumination conditions directly mirror the deployment setup
unlike standard vehicle detection datasets captured from dashcam or
street-level perspectives, which exhibit significantly different
vehicle scale distributions and viewpoint geometry.

---

### Stage 2 License Plate Detection

| Property | Detail |
|---|---|
| Source | Custom dataset built through combination of multiple open source datasets |
| Total images | 12000 |
| Plate formats | Single-line HSRP, dual-line HSRP, legacy pre-HSRP, commercial (yellow bg), two-wheeler |
| Vehicle classes | Private cars, commercial trucks, buses, auto-rickshaws, motorcycles, tractors |
| Conditions | Daytime, nighttime, dusk, monsoon glare, dust, motion blur |
| Degradation levels | Clean, moderate occlusion, severe occlusion (>30%), hand-painted legacy |
| Train / Val / Test split | 80 / 10 / 10 |
| Annotations | Plate bounding boxes + ground-truth transcriptions |

> 📦 Custom Indian plate dataset will be released under CC BY 4.0
> upon paper acceptance.

---

## Models

### Detection Performance Stage 1 (Vehicle) and Stage 2 (Plate)

| Stage | Model | Val mAP@0.5 | Val mAP@0.5-95 | Test mAP@0.5 | Test mAP@0.5-95 | Params (M) |
|---|---|---|---|---|---|---|
| Vehicle  | YOLOv8n | 98.9 | 92.5 | yet to be tested | yet to be tested | 3.16 |  (Test mAP values yet to be calculated)
| Vehicle  | YOLOv11n | 99.1 | 92.6 | yet to be tested | yet to be tested | 2.62 |
| Vehicle  | YOLOv26n | 98.9 | 93.5 | yet to be tested | yet to be tested | 2.40 |
| Plate  | YOLOv8n | 99.3 | 85.3 | 95.0 | 84.0 | 3.16 |
| Plate  | YOLOv11n | 99.3 | 84.8 | 93.0 | 78.0 | 2.62 |
| Plate  | YOLOv26n | 99.2 | 85.1 | 92.0 | 72.0 | 2.40 |

> **Note on model selection and deployment:**
> YOLOv8n achieves the highest mAP@0.5-95 on the plate detection stage,
> indicating tighter bounding box localization which directly impacts
> OCR crop quality. All three architectures were evaluated for detection
> accuracy in PyTorch. **YOLOv8n is selected as the deployed
> configuration** as it achieved the best localization precision and
> stable INT8 quantization on the Hailo-8 NPU via the Hailo Dataflow
> Compiler. INT8 quantization support for YOLOv11n and YOLOv26n on
> Hailo DFC is not yet stable at time of writing; deployment latency
> is therefore reported for YOLOv8n only.

> Model weights will be released upon paper acceptance.

---

## OCR Pipeline

| Feature | LPRNet | PaddleOCR (PP OCRv4) | EasyOCR |
|---------|---------|----------------------|----------|
| **Primary Purpose** | Dedicated License Plate Recognition | General OCR Framework | General OCR Framework |
| **Model Architecture** | Lightweight CNN + CTC | Detection + Recognition Pipeline | CRNN Based |
| **Multi line Plate Support** |  Single line only | Native support |  Native support |
| **Typical Inference Speed** | ~40 ms | ~50 ms | ~5 s |
| **Deployment Target** | CPU | Hailo 8 NPU | CPU |
| **HEF Conversion** | Not supported | Successfully converted | Not applicable (Python library) |
| **CPU Resource Utilization** | Low | Low | High |
| **Best Use Case** | Dedicated ANPR Systems | Industrial OCR Pipelines and Edge Deployment | Rapid Prototyping and Baseline Evaluation |


### Selected Backend: PaddleOCR PP-OCRv4

PaddleOCR's 2D attention-based architecture handles both single line
and dual-line plate formats without requiring per format model routing,
outperforming LPRNet and EasyOCR across both formats on the Indian
plate test set.

**Why LPRNet cannot natively handle dual-line plates:**
LPRNet uses a CTC (Connectionist Temporal Classification) decoder
that treats the entire input as a single left-to-right character
sequence. It has no concept of row structure. Feeding a dual line
HSRP plate to LPRNet produces garbled output regardless of resolution,
because the CTC alignment conflates characters from both rows into
a single disordered sequence. LPRNet requires the dual-line
segmentation step as a mandatory prerequisite, whereas PaddleOCR
handles both formats directly.

### Dual-Line Plate Segmentation Algorithm

For dual line HSRP plates, a geometric segmentation step is applied
before OCR:
This approach ensures correct character reading order, which purely
positional OCR fails to guarantee for dual line inputs.

---


## Vehicle Tracking Results

To reduce redundant license plate recognition, the proposed ANPR pipeline integrates the **SORT (Simple Online and Realtime Tracking)** algorithm. Each detected vehicle is assigned a persistent track ID, allowing the system to associate detections across consecutive frames and perform OCR only on selected plate crops instead of every detection.

Two configurable tracking parameters were used:

* **Maximum Track Age:** Number of consecutive frames for which a vehicle track is retained after a missed detection.
* **Capture Interval:** Number of frames between successive license plate captures for the same tracked vehicle.

This strategy ensures stable vehicle tracking while significantly reducing duplicate plate captures and unnecessary OCR inference.

### Impact of SORT Tracking

Without vehicle tracking, the detector repeatedly captures the same vehicle across multiple frames, resulting in a large number of duplicate plate crops and increased OCR latency.

By associating detections with persistent track IDs, the system stores only one or a few representative plate crops per vehicle, dramatically reducing storage requirements and computational overhead.


| Scenario | Number of Plate Crops | Total OCR Time |
|-----------|----------------------:|---------------:|
| Without Tracking | 450 | 22.5 s |
| SORT (3 crops per vehicle) | 45 | 2.25 s |
| SORT (1 crop per vehicle) | 15 | 0.75 s |

### Observations

* Vehicle tracking reduced the number of saved license plate crops from **450** to **15** when storing a single crop per tracked vehicle.
* Allowing up to **three plate captures per vehicle** increased OCR robustness while still reducing redundant detections by nearly an order of magnitude.
* Overall, integrating SORT reduced OCR workload, minimized storage requirements, and improved the computational efficiency of the ANPR pipeline by approximately **10×–30×**, depending on the capture strategy.
---

## Edge Deployment

### Hardware Configuration

| Component | Specification |
|---|---|
| Single-board computer | Raspberry Pi 5 (4 core ARM Cortex-A76 @ 2.4 GHz, 8 GB LPDDR4X) |
| AI accelerator | Hailo-8 NPU via Raspberry Pi AI HAT (26 TOPS INT8) |
| Camera | Raspberry Pi Camera HQ Module (Sony IMX477, 12.3 MP, 1080p30) |
| Adaptor | Raspberry Pi 5V-5A Power Adaptor |
| Camera Lens | 16mm 10MP Telephoto Lens Suitable for HQ Camera Module |
| Approximate total cost | ~$500 USD |

### Model Compilation

- Detection models exported from PyTorch → ONNX → Hailo HEF
  via Hailo Dataflow Compiler (DFC)
- Post-training INT8 quantization with 5,000-image calibration set
  drawn from the validation split
- Quantization sensitivity analysis performed to identify layers
  with mAP regression > 0.5% under INT8; those layers preserved
  at higher precision
- YOLOv8n: stable INT8 compilation confirmed on Hailo-8 DFC
- YOLOv11n / YOLOv26n: INT8 compilation not yet stable on Hailo DFC
  at time of writing — PyTorch accuracy numbers reported for
  architecture comparison, Hailo deployment not applicable

## Inference Performance

The proposed pipeline processes **pre recorded video sequences** captured using the Raspberry Pi Camera HQ Module. Inference is performed offline, with each frame sequentially passed through the complete ANPR pipeline. Therefore, the reported timings correspond to **average per frame inference time** rather than live streaming latency.

## Computational Performance

Performance evaluation was conducted by processing recorded videos frame by frame on the Raspberry Pi 5 equipped with the Hailo 8 AI accelerator. The reported execution times represent the average inference time for each pipeline stage and exclude video acquisition overhead.

| Stage | Accelerator | Average Time (ms) |
|---|---|---:|
| Vehicle Detection | Hailo 8 NPU | 8 |
| Plate Detection | Hailo 8 NPU | 8 |
| Plate Preprocessing | CPU | 3 |
| OCR* | Hailo 8 NPU | 8 |
| CPU Processing | Cortex A76 | 6 |
| **Average Time per Frame** |  | **33** |

\* OCR time is averaged over **2 processed frames out of every 10 frames**, as OCR is not executed on every frame.

> These timings were measured during offline inference on recorded videos and should not be interpreted as real time streaming performance.

**Average processing throughput: ~30 processed frames/s (offline inference)**

Pipeline stages are overlapped using a double-buffering scheme
to minimize inter-stage latency. The dual-line segmentation step
runs on the CPU in parallel with NPU DMA transfer overhead,
contributing negligible additional latency.

```bibtex


For enquiries, contact: haritchhabra93@gmail.com, pratikchouragadey26@gmail.com 


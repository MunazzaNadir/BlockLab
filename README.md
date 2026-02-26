# BlockLab — Your LEGO Laboratory

> Convert any real-world object into a fully buildable LEGO model using a five-stage automated pipeline.

Film a 360° video on your phone. Get a coloured brick-by-brick build plan with direct purchase links for every piece.



## What It Does

| Input | Output |
|-------|--------|
| 30–90 second 360° smartphone video | Reconstructed textured 3D mesh (`.obj` / `.glb`) |
| | Optimised LEGO brick layout (colour + brick type per voxel) |
| | Numbered step-by-step building instructions (PDF + in-app) |
| | Complete parts list with Rebrickable / BrickLink purchase links |

---

## Pipeline Overview
```
Video → Frame Extraction → 3D Reconstruction (3DGS) → Voxelisation → Brick Optimisation → Instructions & Parts List
```

### Module 1 — Video Preprocessing & Frame Extraction
- Extract frames at 3–5 fps via FFmpeg
- Drop blurry frames (Laplacian variance < 100) and near-duplicates (SSIM)
- Undistort using phone EXIF intrinsics or OpenCV calibration
- Background removal via **SAM 2** (Meta, 2024) — no fine-tuning required for v1
- Camera pose estimation via **COLMAP** / hloc

### Module 2 — 3D Reconstruction
- **3D Gaussian Splatting** (Kerbl et al. 2023) — per-scene optimisation, no training dataset needed
- 7,000–30,000 iterations with adaptive densification
- Export dense point cloud → Poisson Surface Reconstruction (Open3D) → watertight mesh
- UV-unwrap + bake texture atlas via xatlas

### Module 3 — Voxelisation
- Scale mesh to user's chosen build size (e.g. 32×32×32 studs)
- Voxelise at LEGO grid resolution: **8 mm X/Z**, **3.2 mm Y**
- Colour-match each voxel to official LEGO colours using **Delta-E in LAB space**

### Module 4 — Brick Optimisation
- Greedy layer-by-layer rectangle packing (largest brick first)
- Structural connectivity enforcement (≥30% stud overlap with layer below)
- Local ILP refinement via PuLP / OR-Tools on 8×8 tile windows
- Prefers bricks with high real-set frequency from Rebrickable data

### Module 5 — Instructions & Parts List
- Step-by-step renders via Blender Python API or Three.js/Puppeteer
- PDF generation with ReportLab / WeasyPrint
- Parts list enriched via Rebrickable API batch lookup
- BrickLink cross-links via `external_ids` field

---

## Data Sources

| Source | Role |
|--------|------|
| [Rebrickable CSV downloads](https://rebrickable.com/downloads/) | LEGO catalogue — parts, colours, categories, elements, frequency weights |
| Rebrickable API (runtime only) | Part images + batch element lookups. Never used for bulk data. |
| Hardcoded `BRICK_VOCAB` table | Stud dimensions for ~150–200 optimiser bricks |
| LDraw / BrickLink Studio `.obj` exports | 3D brick geometry for instruction renders (Module 5 only) |
| SAM 2 (pretrained, Meta 2024) | Background removal — no training data needed for v1 |
| COCO (only if fine-tuning needed) | Segmentation fine-tuning fallback if SAM 2 fails on edge cases |

---

## LEGO Grid Constants
```python
STUD_SIZE_MM   = 8.0   # X/Z spacing
PLATE_HEIGHT   = 3.2   # Y per plate
BRICK_HEIGHT   = 9.6   # Y per brick (3 plates)
STUD_DIAMETER  = 4.8
```

---

## Tech Stack

| Layer | Technology |
|-------|------------|
| Mobile | React Native (Expo) |
| Web | Next.js 14 (App Router) |
| API Backend | FastAPI (Python) |
| Job Queue | Celery + Redis |
| Storage | AWS S3 / Supabase Storage |
| Database | PostgreSQL + DuckDB |
| Auth | Supabase Auth / Clerk |
| Payments | Stripe |
| 3D Viewer | Three.js / React Three Fiber |
| Background Removal | SAM 2 (Meta, 2024) |
| Reconstruction | 3D Gaussian Splatting |

---

## App Screens

- **Splash / Onboarding** — brand animation, feature tour, sign-in
- **Home / Dashboard** — recent projects, quick-start capture
- **Capture Screen** — live viewfinder, 360° overlay, sharpness/motion warnings, coverage ring
- **Upload Screen** — drag-drop or file picker for existing video
- **Processing Status** — real-time per-stage progress bar, animated LEGO loader, ETA
- **3D Model Viewer** — Three.js mesh viewer, texture toggle, scale adjust
- **Build Settings** — target size, colour mode, brick vocabulary, structural mode
- **LEGO Preview Viewer** — layer-by-layer brick viewer, colour legend
- **Instructions Viewer** — swipeable step cards, PDF download
- **Parts List** — sortable table with buy links, cost estimate, CSV export
- **Share / Export** — shareable link, QR code, ZIP download
- **Project Library** — all past projects with status badges
- **Pro / Upgrade** — Stripe checkout, feature comparison

---

## Build Timeline

| Timeframe | Milestone |
|-----------|-----------|
| Weeks 1–2 | Rebrickable CSVs → DuckDB catalogue + brick vocabulary |
| Weeks 3–4 | Video preprocessing + SAM 2 + COLMAP poses |
| Weeks 5–6 | 3DGS reconstruction pipeline |
| Weeks 7–8 | Voxelisation + brick optimiser |
| Week 9 | Instruction rendering + parts list |
| Weeks 10–12 | App core screens (Capture → Processing → LEGO Preview) |
| Weeks 13–14 | Remaining screens + infra deployed to TestFlight |
| Week 15+ | Beta test → App Store v1.0 |

---

## Infrastructure

- **GPU Workers:** AWS EC2 g5.xlarge (A10G) — Spot instances for ~70% cost reduction
- **CPU Workers:** FastAPI + Celery on EC2 c6i
- **Storage:** S3 with 30-day TTL (free) / 90-day (pro)
- **CI/CD:** GitHub Actions → Docker → ECR → ECS on merge to `main`
- **Mobile:** Expo EAS Build → TestFlight + Play Console

---

## API Endpoints

| Endpoint | Purpose |
|----------|---------|
| `POST /jobs/create` | Upload video, create job |
| `GET /jobs/{id}/status` | Poll pipeline stage + progress |
| `GET /jobs/{id}/mesh` | Pre-signed S3 URL for mesh |
| `POST /jobs/{id}/generate` | Trigger LEGO optimisation |
| `GET /jobs/{id}/layout` | Brick placement data |
| `GET /jobs/{id}/instructions` | PDF URL + step renders |
| `GET /jobs/{id}/parts` | Enriched parts list with links |
| `GET /catalogue/colors` | All LEGO colours |
| `GET /catalogue/bricks` | Brick vocabulary |

---

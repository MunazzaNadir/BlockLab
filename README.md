BlockLab
End-to-End Intelligent LEGO Model Generation System
Complete Project Blueprint  •  Data → Models → App
Version 2.0  

System Overview
BlockLab converts any real-world object into a fully buildable LEGO model using a five-stage automated pipeline: video capture → 3D reconstruction → voxelisation → LEGO optimisation → instructions & parts list. The user films a 360° video on their phone; the system returns a coloured brick-by-brick build plan with a direct purchase link for every piece needed.

Input	30–90 second 360° video of a real-world object, filmed on a smartphone (captured in-app or uploaded)

Output 1	Reconstructed textured 3D mesh (.obj / .glb)

Output 2	Optimised LEGO brick layout (colour + brick type per voxel)

Output 3	Numbered step-by-step building instructions (PDF + in-app viewer)

Output 4	Complete parts list with Rebrickable / BrickLink purchase links


 
PHASE 1
Data Collection, Cleaning & Processing
Phase 1 produces two artefacts: a clean, versioned LEGO catalogue database sourced entirely from Rebrickable CSV downloads, and a hardcoded brick vocabulary table with physical dimensions. The Columbia COIL-100 dataset has been removed from the pipeline — see the Data Source Decision section below for the full rationale.

Data Source Decision: Why COIL-100 Was Removed
⚠	COIL-100 has been dropped from this project. The original blueprint included it in error. The reasoning below explains what replaces it and why.

COIL-100 (Columbia Object Image Library, 1996) was initially proposed as training data for the vision pipeline. On closer examination it is a poor fit for three reasons. First, all images are 128×128 pixels — modern phone cameras capture 4K video and the domain gap is too large to be bridged by augmentation alone. Second, the capture conditions (motorised turntable, pure black background, controlled studio lighting) bear no resemblance to real-world user captures on kitchen tables under ceiling lights with cluttered backgrounds. Third, and most importantly: 3D Gaussian Splatting and NeRF are per-scene optimisation methods, not trained models. There is no offline training phase that requires a multi-view dataset. Each user's video is its own training input — the reconstruction runs fresh on every job.

The only sub-problem that genuinely needs a training dataset is background removal and object segmentation. For this, SAM 2 (Meta, 2024) is used with no fine-tuning for v1. If fine-tuning is needed later, COCO (330k images with segmentation masks) is the appropriate dataset — not COIL-100.

ℹ	Rule of thumb going forward: if a pipeline stage needs 'training data', ask first whether a pretrained model (SAM 2, MiDaS, COLMAP) already solves it. For this project, all ML sub-problems have strong off-the-shelf solutions.


1A  Rebrickable LEGO Catalogue
Primary Method: CSV Downloads (Not the API)
⚠	The Rebrickable documentation explicitly states: 'If you just want a full list of all Sets/Parts/Colors you MUST use the CSV files from the Downloads page instead.' The API has a 1 req/sec rate limit making bulk download impossible. Use CSVs for all catalogue data; reserve the API for runtime lookups only.

Step 1 — Download These 7 CSV Files
All files are available at rebrickable.com/downloads/ — free, no login, updated daily. Download via direct URL or script.

File	Why You Need It	Key Columns
colors.csv	Your complete colour palette	id, name, rgb, is_trans
parts.csv	Every brick/part ever produced	part_num, name, part_cat_id
part_categories.csv	Filter to structural bricks only	id, name
part_relationships.csv	Deduplicate prints & mould variants	rel_type, child_part_num, parent_part_num
elements.csv	Part + colour combos that actually exist	element_id, part_num, color_id
inventory_parts.csv	Part frequency across real sets	inventory_id, part_num, color_id, quantity
sets.csv	Needed to join inventory_parts to real sets	set_num, name, year, theme_id, num_parts

Files NOT needed: themes.csv (marketing taxonomy only), minifigs.csv (minifig metadata, no part details), inventory_sets.csv (sets-within-sets, irrelevant), inventory_minifigs.csv (minifig-to-set links, irrelevant). Drop all four permanently.

Step 2 — Why sets.csv Is Included
sets.csv is needed for one specific purpose: building a part-frequency weighting table for the brick optimiser. The optimiser should prefer bricks that appear in many real sets, as these are more likely to be widely available to buy. The join chain is: sets.csv → inventories.csv → inventory_parts.csv. Without sets.csv you cannot distinguish a real set inventory from a minifig pack inventory when computing frequency weights.

Step 3 — Filtering to Structural Bricks Only
From part_categories.csv, keep only these category names and drop everything else:
•	Keep: Bricks, Bricks Curved, Bricks Decorated, Bricks Round and Cones, Bricks Sloped, Plates, Plates Angled, Plates Round and Cones, Plates Special, Tiles, Tiles Round and Cones, Tiles Special
•	Drop: Minifig, Electric, Technic, String / Bands, Stickers, Animals, Plants, Pneumatic, Belville/Scala, Foam, Duplo/Primo
•	From part_relationships.csv, drop all Print variants (rel_type = 'P') — keep only the base mould. This reduces ~60k parts to ~8–12k usable structural parts.
•	From colors.csv, filter is_trans = False for the base pipeline. Add transparent as an advanced option later.

Step 4 — Colour Processing
1.	Build a colour lookup table: lego_color_id → {name, r, g, b, is_trans} from colors.csv.
2.	Convert all RGB hex values to LAB colour space using skimage.color.rgb2lab. LAB is perceptually uniform — colour-matching will use Delta-E distance, not Euclidean RGB.
3.	Output: colors_processed.parquet — columns: lego_color_id, name, r, g, b, L, a, b_ (~150–180 rows after removing transparent).

Step 5 — Part Frequency Weighting
4.	Join: sets.csv → inventories.csv (on set_num) → inventory_parts.csv (on inventory_id).
5.	For each part_num, count the number of distinct sets it appears in. Call this set_frequency.
6.	Output: part_frequency.parquet — columns: part_num, set_frequency. Used by the optimiser to prefer common bricks.

Step 6 — Final Catalogue Storage
Merge all processed files into a single DuckDB database (lego_catalogue.duckdb):
•	parts — part_num, name, part_cat_id, width_stud, depth_stud, height_plate, set_frequency
•	colors — lego_color_id, name, r, g, b, L, a, b_
•	elements — element_id, part_num, lego_color_id


1B  Brick Dimensions — What You Need & Where to Get It
Do You Need Dimension Data?
Yes, but the answer is different at each pipeline stage. Critically, you do not need to parse LDraw geometry files to get the optimiser working. The approach is graduated: hardcode the grid constants and brick vocabulary first, add LDraw geometry only when you reach instruction rendering.

Pipeline Stage	Dimension Data Needed	Source
Voxelisation	LEGO grid constants only (fixed, universal)	Hardcode in code
Brick Optimisation	Stud W×D×H for ~150–200 bricks	Hardcode from part names
Instruction Rendering	Full 3D geometry per brick	LDraw library or Studio .obj exports
Parts List	None	N/A

Stage 1: LEGO Grid Constants — Hardcode These
These values are fixed by LEGO's physical standard and never change. You do not need any external data source for voxelisation.

Constant	Value
1 stud (X/Z spacing)	8.0 mm
1 plate height (Y)	3.2 mm
1 brick height (Y)	9.6 mm  (= 3 plates)
Stud diameter	4.8 mm
Stud pitch	8.0 mm

Stage 2: Brick Vocabulary Dimensions — Hardcode From Part Names
For the optimiser's brick vocabulary of ~150–200 parts, define stud dimensions in a Python dict. Part names on Rebrickable follow the pattern 'Brick 2 x 4' or 'Plate 1 x 2' — the dimensions are literally in the name. This table can be populated manually in one afternoon. A sample:

part_num	name	w_stud × d_stud × h_plate
3001	Brick 2 x 4	2 × 4 × 3
3003	Brick 2 x 2	2 × 2 × 3
3004	Brick 1 x 2	1 × 2 × 3
3005	Brick 1 x 1	1 × 1 × 3
3020	Plate 2 x 4	2 × 4 × 1
3022	Plate 2 x 2	2 × 2 × 1
3023	Plate 1 x 2	1 × 2 × 1
3024	Plate 1 x 1	1 × 1 × 1
3010	Brick 1 x 4	1 × 4 × 3
3009	Brick 1 x 6	1 × 6 × 3
3008	Brick 1 x 8	1 × 8 × 3
3021	Plate 2 x 3	2 × 3 × 1

ℹ	Do this first. Get the full pipeline working end-to-end with a hardcoded vocabulary. Do not let LDraw parsing block early progress — it is a separate concern that only matters for rendering.

Stage 3: LDraw Geometry — For Instruction Rendering Only
LDraw (ldraw.org) provides a .dat geometry file for every official LEGO part. This is only needed when rendering 3D brick images for the building instructions. The recommended approach for obtaining geometry is BrickLink Studio, which bundles the full LDraw library and can export individual part geometry as .obj files — trivially parseable with trimesh or Open3D, with no custom .dat parser required.
•	Download source: ldraw.org (full library ~500 MB) or via BrickLink Studio export
•	Format: .dat files in LDraw Units (LDU); 1 LDU = 0.4 mm
•	Parser: use pyldr, or export .obj from BrickLink Studio (much easier)
•	Only needed at Module 5 (instruction rendering) — do not implement until you reach that stage

Where NOT to Get Dimension Data
BrickLink's website shows physical dimensions (in cm) on part pages, but these are not exposed as a structured API field. Scraping them is fragile, violates their ToS, and is entirely unnecessary given the alternatives above.


1C  Rebrickable API — Runtime Use Only
The API is not used for bulk data collection. It is used at runtime for two specific tasks:

7.	Part images for the app UI — GET /api/v3/lego/parts/{part_num}/ returns a part_img_url. Cache aggressively; these rarely change.
8.	Batch part lookups when enriching the parts list — use the batch parameter to fetch up to ~200 parts in one call:

GET /api/v3/lego/parts/?part_nums=3001,3002,3003&inc_part_details=1

Authentication: add the header Authorization: key YOUR_KEY. Store the key in .env, never hard-code. Rate limit: 1 req/sec average; implement exponential back-off on 429 responses. Do not retry aggressively — continued 429 ignoring results in an IP ban per Rebrickable's documentation.


 
PHASE 2
Modelling: Five-Stage Pipeline
The modelling pipeline converts a folder of video frames into a complete LEGO build specification. Each module has a defined input contract and output artefact. No module requires an offline-trained model except background removal, which uses SAM 2 out of the box.

Module 1 — Video Preprocessing & Frame Extraction
Purpose
Convert raw 360° phone video to a clean, undistorted, background-masked frame set that all downstream modules consume. The quality of this stage directly determines reconstruction quality — bad frames cannot be recovered later.

What Users Will Actually Film
Designing this module requires understanding real user input. Users will provide good and bad captures. The module must handle or reject both gracefully.

Category	What to Expect
Good cases	30–90 sec video walking slowly around object; modern smartphone 4K; ~100–300 usable frames at 3–5 fps; reasonable indoor lighting
Common problems	Motion blur from moving too fast; inconsistent lighting as they walk; background clutter; fingers/arms in frame; filming only from one elevation with no top-down pass
Hard failures — warn user	Fully transparent objects (glass, clear bottles); mirror/highly reflective surfaces; textureless objects (plain white sphere, matte black box); moving objects; object larger than 50 cm in any dimension

Steps
9.	Extract frames at 3–5 fps using FFmpeg. Higher fps wastes compute without adding reconstruction quality for slow 360° pans.
10.	Remove blurry frames: compute Laplacian variance on each frame; drop frames below threshold (variance < 100). Also drop near-duplicate frames using SSIM between consecutive frames.
11.	Undistort frames using the phone's intrinsic camera matrix — obtained from Apple/Android EXIF data or a one-time OpenCV camera calibration per device model.
12.	Run background removal using SAM 2 (Meta, 2024). Prompt with a single centre-point click on the first frame; SAM 2 propagates the mask through the entire video. No fine-tuning required for v1. Fallback: Rembg (one-line Python, wraps U2-Net) for simpler objects.
13.	Run COLMAP (or hloc for speed) on the masked frames to estimate camera poses. Output: sparse point cloud + camera extrinsics/intrinsics.

In-App Capture Guidance (Critical for Quality)
The most impactful quality improvement is guiding the user during capture, not improving the model. Implement these real-time checks in the Capture Screen:
•	Show a circular path overlay — 'walk this path around the object'
•	Motion blur warning: compute optical flow; alert if moving too fast
•	Sharpness warning: Laplacian variance check on live frames
•	Prompt a top-down pass after the equatorial pass completes
•	Coverage indicator: fill a circular progress ring as angles are covered
•	Recommended distance tooltip: 2–4× the object's longest dimension

Produces	frames/ — N undistorted RGB PNGs + binary masks; colmap/ — sparse.ply, cameras.bin, images.bin


Module 2 — 3D Reconstruction (3D Gaussian Splatting)
Purpose
Reconstruct a dense, textured 3D representation of the object from the posed frames. This is a per-scene optimisation, not a trained model — no training dataset is required.

Why 3DGS, Not NeRF
3D Gaussian Splatting (3DGS, Kerbl et al. 2023) is the current best-practice for fast, high-quality object-level reconstruction. It optimises in 10–30 minutes on a single GPU and renders at 100+ fps. Traditional NeRF variants (Instant-NGP, NeRFacto) are slower and produce lower-quality meshes. Use the open-source gaussian-splatting repository as the base.

Important: This Is Not Trained on a Dataset
ℹ	3DGS runs fresh on each user's video frames. It has no 'training phase' requiring COIL-100 or any other dataset. Each job is an independent optimisation from scratch using only that user's posed frames as input.

Steps
14.	Initialise Gaussians from the COLMAP sparse point cloud.
15.	Train for 7,000–30,000 iterations with adaptive densification every 100 steps.
16.	Export a dense point cloud (.ply with RGB) by sampling Gaussians at high opacity.
17.	Run Poisson Surface Reconstruction (Open3D) on the dense point cloud to obtain a watertight mesh.
18.	UV-unwrap the mesh and bake a texture atlas using xatlas + Open3D raycasting.

Fallback
If GPU is unavailable, use Instant-NGP which has a CPU fallback via hash-grid NeRF. Inference time is ~5 minutes on CPU for small objects.

Produces	mesh.obj + mesh.mtl + texture.png — textured watertight 3D mesh of the object


Module 3 — Voxelisation
Purpose
Convert the continuous 3D mesh into a discrete 3D grid aligned to the LEGO stud grid, where each voxel corresponds to one LEGO plate layer in Y (3.2 mm) and one stud unit in X/Z (8 mm × 8 mm).

Steps
19.	Scale the mesh to the user's chosen build size (e.g. 'fit in 32×32×32 studs'). Scale factor = target_studs / mesh_bounding_box_max_axis.
20.	Voxelise the scaled mesh using trimesh.voxel.VoxelGrid at the LEGO grid resolution: 8 mm in X/Z, 3.2 mm in Y.
21.	For each occupied voxel, sample the mesh texture at the voxel centre to get the target RGB colour.
22.	Match each voxel colour to the nearest LEGO official colour using Delta-E distance in LAB space (from colors_processed.parquet). Delta-E < 10 is a good acceptance threshold.
23.	Output a 3D numpy array: voxel_grid[x, y, z] = lego_color_id (0 = empty).

Produces	voxel_grid.npy — integer array shape (W, H, D); voxel_colors.json — lego_color_ids used


Module 4 — LEGO Brick Optimisation
Purpose
Replace the voxel grid (initially all 1×1 plates) with the fewest, largest bricks possible while maintaining structural integrity and colour accuracy. This is the core NP-hard combinatorial problem of the project.

Algorithm: Greedy Layer-by-Layer with ILP Refinement
24.	Process the voxel grid one horizontal Y-layer at a time, matching real LEGO building order.
25.	For each layer, run a greedy rectangle-packing pass: at each unfilled cell, try the largest fitting brick of the correct colour (descending order: 2×4, 2×3, 2×2, 1×4, 1×3, 1×2, 1×1). Accept if it fits.
26.	Enforce structural connectivity: a brick placement is only accepted if at least 30% of its studs overlap with a brick in the layer below (skip for base layer).
27.	After greedy pass, run local ILP refinement (PuLP or OR-Tools) on 8×8 tile windows to improve coverage. Time-box each tile to 2 seconds.
28.	Repeat for all Y layers bottom-to-top.

Brick Vocabulary
Limit the optimiser to ~150–200 common structural bricks: 1×1, 1×2, 1×3, 1×4, 1×6, 1×8, 2×2, 2×3, 2×4, 2×6, 2×8, 2×10, slopes, and corner pieces. Dimensions come from the hardcoded vocabulary table built in Phase 1B. Prefer bricks with high set_frequency from the part_frequency.parquet table.

Produces	brick_layout.json — list of BrickPlacement {part_num, lego_color_id, x, y, z, orientation}; parts_list.json — {element_id: count}


Module 5 — Instruction & Parts List Generation
Purpose
Generate human-readable building instructions (step-by-step renders + PDF) and a shoppable parts list with purchase links.

Building Instructions
29.	Group placements into build steps: each step = one Y layer, or split thick layers into sub-steps of 8 bricks for beginner-friendliness.
30.	Render each step using a headless 3D renderer. Two options: Blender Python API (high quality, ~2 s/step) or Three.js via Puppeteer server-side (faster, ~0.3 s/step). Render: bricks placed so far in grey; new bricks in their LEGO colour; exploded-view offset.
31.	LDraw .dat geometry (or .obj exports from BrickLink Studio) is used here for accurate brick shapes in renders. This is the only stage requiring LDraw data.
32.	Annotate each rendered image with count and part name of new bricks in this step.
33.	Assemble into a PDF (ReportLab or WeasyPrint): cover page, parts list page, numbered instruction pages.

Parts List & Purchase Links
34.	Aggregate brick_layout.json: for each (part_num, lego_color_id) pair, sum the count.
35.	Batch-fetch element details from Rebrickable API: GET /api/v3/lego/parts/?part_nums=...&inc_part_details=1. One API call for all parts, not one per part.
36.	Cross-link to BrickLink using the external_ids field returned by the API (available since 2017-09-14 changelog entry).
37.	Generate parts_list.csv and parts_list.html with sortable columns: element_id, colour, count, buy links.

Produces (Instructions)	instructions.pdf — printable step-by-step build guide with rendered images

Produces (Parts List)	parts_list.csv + parts_list.html — element_id, colour, count, Rebrickable link, BrickLink link


 
PHASE 3
Application: Architecture & All Screens
3A  Technology Stack
Layer	Technology	Rationale
Mobile app	React Native (Expo)	iOS + Android from one codebase
Web app	Next.js 14 (App Router)	Desktop access + shareable links
API backend	FastAPI (Python)	Hosts all pipeline modules
Job queue	Celery + Redis	Async GPU jobs (3DGS ~10–30 min)
Storage	AWS S3 (or Supabase Storage)	Meshes, textures, PDFs, renders
Database	PostgreSQL + DuckDB	User data + LEGO catalogue
Auth	Supabase Auth (or Clerk)	Email / Google / Apple Sign-In
Payments	Stripe	Pro tier subscription
3D viewer	Three.js / React Three Fiber	In-browser LEGO model preview
Background removal	SAM 2 (Meta, 2024)	No training required for v1
Reconstruction	3D Gaussian Splatting	Per-scene, no dataset needed


3B  App Screens — All Pages

Screen / Page	What It Shows	Key Actions
Splash / Onboarding	Brand animation, 3-step feature tour, Sign In / Sign Up CTA	Get Started, Sign In with Google/Apple
Home / Dashboard	Recent projects grid, quick-start capture button, example models carousel	New Scan, Open Project, See Examples
Capture Screen	Live camera viewfinder, 360° rotation guide overlay, real-time sharpness + motion warning, coverage ring, frame count, auto-stop at 90 s	Start Recording, Pause, Retake, Upload Instead
Upload Screen	Drag-drop or file picker for existing video, format + length validation, preview thumbnail	Upload File, Browse Gallery, Back to Capture
Processing Status	Real-time progress bar per pipeline stage (Preprocessing → Reconstruction → Voxelisation → Optimisation → Instructions), animated LEGO brick loader, ETA	Cancel Job, View Logs (dev mode)
3D Model Viewer	Interactive Three.js viewer of reconstructed mesh — rotate, zoom, pan; toggle texture on/off; quality indicator	Accept Model, Retake Scan, Adjust Scale
Build Settings	Slider for target size (studs), colour mode (exact / limited palette), brick vocabulary (beginner / full), structural mode toggle	Generate LEGO Model →
LEGO Preview Viewer	Full-colour brick 3D viewer in Three.js — layer-by-layer slider, brick count overlay, colour legend	View Instructions, Edit Settings, Share, Download
Instructions Viewer	Swipeable step-by-step instruction cards with rendered images, step progress indicator, brick count per step, zoom on tap	Previous / Next Step, Jump to Step N, Download PDF
Parts List Screen	Sortable table: brick name | colour swatch | count | buy links; filter by colour or category; total cost estimate	Buy on Rebrickable, Buy on BrickLink, Export CSV, Copy Total
Share / Export	Shareable link, QR code for model, social share sheet, ZIP download	Copy Link, Share, Download ZIP
Project Library	Grid of all past projects with thumbnail, date, brick count, status badge	Open, Duplicate, Delete, Search
Settings	Account info, notification prefs, default build size, colour palette preference	Save, Sign Out, Delete Account
Pro / Upgrade	Feature comparison table (free vs pro), pricing, Stripe checkout	Subscribe Monthly / Yearly, Restore Purchase
Help & FAQ	Searchable FAQ, video tutorials, Discord link, contact support	Search, Watch Tutorial, Contact Us


3C  Backend API Endpoints
Endpoint	Purpose	Returns
POST /jobs/create	Accept video upload, create job	job_id, status: queued
GET /jobs/{job_id}/status	Poll pipeline stage + progress %	stage, progress, eta_seconds
GET /jobs/{job_id}/mesh	Pre-signed S3 URL for mesh + texture	mesh_url, texture_url
POST /jobs/{job_id}/generate	Trigger LEGO optimisation with settings	job_id, brick_count
GET /jobs/{job_id}/layout	Return brick placement data	placements[], parts_summary
GET /jobs/{job_id}/instructions	Return PDF URL + step renders	pdf_url, steps[]
GET /jobs/{job_id}/parts	Return enriched parts list with links	parts[], total_price_usd
GET /catalogue/colors	List all LEGO colours	colors[]
GET /catalogue/bricks	List brick vocabulary	bricks[]
POST /auth/register	Create account	user_id, token
POST /auth/login	Authenticate	token, user


 
PHASE 4
Infrastructure, MLOps & Deployment
4A  Compute Infrastructure
•	GPU workers: 1× NVIDIA A10G or A100 on AWS EC2 (g5.xlarge ~$1/hr) for 3DGS per-scene optimisation. Use Spot instances to reduce cost by ~70%.
•	CPU workers: FastAPI + Celery on EC2 c6i instances for preprocessing, voxelisation, ILP optimisation, PDF generation.
•	Redis: Celery broker + result backend + colour-matching lookup cache.
•	S3: all user artefacts (video, mesh, renders, PDFs) with 30-day TTL for free tier, 90-day for pro.

4B  Evaluation (No Training Datasets Required)
Because 3DGS is per-scene and SAM 2 is used off-the-shelf, there is no traditional ML training loop. Evaluation focuses on pipeline output quality:
•	Reconstruction quality: PSNR and SSIM on held-out camera views (views not used in the 3DGS optimisation). No external dataset needed — the user's own video provides the test views.
•	Colour accuracy: Delta-E between voxel colour assignments and the original mesh texture at each voxel centre.
•	Optimiser efficiency: brick count reduction ratio (1×1 plates baseline vs optimised layout). Target >60% reduction.
•	Structural validity: simulate the brick graph (each brick must be supported) and verify no floating bricks.

ℹ	If SAM 2 segmentation fails on specific object types (e.g. very dark objects, furry textures), fine-tune on COCO — not COIL-100. COCO has 330k images with segmentation masks across 80 categories and is the correct dataset for this problem.

4C  Deployment
•	Backend: Docker container on AWS ECS Fargate; GPU Celery workers on EC2 Spot with auto-scaling.
•	Mobile: Expo EAS Build → TestFlight (iOS) + Play Console (Android) for beta; App Store + Google Play for v1.0.
•	Web: Vercel (Next.js) with edge caching for static assets; API calls proxied to ECS.
•	CI/CD: GitHub Actions — lint + test → Docker build → push to ECR → deploy to ECS on merge to main.


Build Timeline
Timeframe	Work	Milestone
Weeks 1–2	Phase 1: Download + clean Rebrickable CSVs, build DuckDB catalogue, hardcode brick vocabulary with dimensions	lego_catalogue.duckdb ready; BRICK_VOCAB table with ~150 parts
Weeks 3–4	Module 1: Video preprocessing + SAM 2 segmentation + COLMAP pose estimation	Clean frame sets with masks and camera poses on 5 test objects
Weeks 5–6	Module 2: 3DGS reconstruction pipeline	Textured meshes for 5 test objects; PSNR/SSIM benchmarked
Weeks 7–8	Modules 3–4: Voxelisation + brick optimiser	Brick layouts on test objects; ILP refinement working
Week 9	Module 5: Instruction rendering + parts list with Rebrickable API	PDF output; parts CSV with buy links
Weeks 10–12	Phase 3: App core screens (Capture → Processing → LEGO Preview)	End-to-end flow live on device
Weeks 13–14	Phase 3: Remaining screens + Phase 4 infra	All screens complete; deployed to TestFlight
Week 15+	Beta test, iterate on capture guidance UX, App Store submission	Public v1.0


Data Source Verdict
The original blueprint included COIL-100 as a training dataset for the vision pipeline. This was incorrect. COIL-100 has been removed entirely. The corrected data foundation for this project is:

Source	Role
Rebrickable CSV downloads (7 files)	Canonical LEGO catalogue — parts, colours, categories, elements, frequency weights. One-time download, updated daily.
Rebrickable API (runtime only)	Part images and batch element lookups in the app. Never used for bulk data.
Hardcoded BRICK_VOCAB table	Stud dimensions for ~150–200 optimiser bricks. Populated from part names in one afternoon.
LDraw / BrickLink Studio .obj exports	3D brick geometry for instruction renders only. Needed at Module 5 only — do not implement earlier.
SAM 2 (pretrained, Meta 2024)	Background removal and object segmentation. No training data needed for v1.
COCO (if fine-tuning needed)	Only if SAM 2 fails on specific object types. Replaces COIL-100 for any segmentation fine-tuning.


# BlockLab
Your LEGO Laboratory

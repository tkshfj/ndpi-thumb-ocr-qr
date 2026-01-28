# NDPI Whole-Slide Organizer — Thumbnails, OCR & QR Extraction

A command-line tool that organizes NDPI whole-slide images into per-slide folders, generates Explorer-friendly JPEG thumbnails, and extracts label text via OCR into adjacent `.ocr.txt` files for indexing and search. Optionally, it can decode QR codes from slide labels and store the decoded payload alongside OCR output.

---

## Background

We increasingly receive whole slide images (WSIs) that were produced outside our facilities—for example by partner clinics, external pathology laboratories, or contract scanning services. In many of these cases, the scanning site *does* maintain case-related information, but it is often delivered separately from the WSI files—frequently as paper-based documents (e.g., requisitions, slide lists, shipping forms) or ad-hoc attachments—rather than as structured, machine-importable metadata packaged with each image.

## Why WSI–metadata linkage isn’t always automatic

Even when accompanying information exists, it may not arrive in a uniform format that maps cleanly to individual WSI files without some reconciliation.

Common identifiers and context—such as:

* accession number / case ID
* specimen site, block number, slide level
* stain / IHC marker
* date, facility code, technician notes

may exist in the sending workflow, but are commonly conveyed through one or more of the following channels:

1. Paper-based documentation (separate from the image files)

* Printed case sheets, slide manifests, or requisitions may accompany shipments or be provided as scanned PDFs/photos.
* This information is accurate and essential, but it is often batch- or case-level, and therefore not automatically tied to a specific filename unless matched explicitly.

2. Glass slide labels (human-readable and sometimes QR/barcode)

* Labels often contain the most dependable identifiers, but they are primarily visual (printed/handwritten) and appear in the WSI’s label region rather than as directly searchable text fields.

3. Scanner/vendor-specific headers and sidecar metadata

* Some scanners embed metadata, but available fields and access methods vary by vendor and format, and extraction can differ across NDPI/SVS/MRX and software toolchains.

4. Folder/file naming conventions created during scanning/export

* Naming can be helpful, but may vary across sites, include localized formatting (language/date conventions), or be modified during transfer (renaming, folder flattening, copy rules).

As a result, we may receive a WSI where the most dependable identifiers are visible in the label pixels, while other accompanying information is provided outside the WSI (often on paper or in loosely structured documents), requiring straightforward matching steps to associate each image with its correct context.

## Why we extract thumbnails, OCR, and QR labels

### 1) Thumbnail extraction supports quick review and triage

WSIs are large, and opening full-resolution imagery just to identify a slide can be slow.

Generating thumbnails—especially including a label-region thumbnail when available—supports:

* rapid QC checks (tissue presence, label legibility)
* quick batch sorting and matching against a slide list or manifest
* lightweight preview without loading full-resolution data
* early detection of scanning/handling issues (orientation, wrong slide, missing tissue)

Thumbnails make review and sorting workflows faster, lighter, and more repeatable.

### 2) OCR captures label text when no reliable machine-readable linkage exists

Not all slides include QR codes, and even when they do, paper-based documents may contain information not encoded in the code.

OCR helps convert label pixels into searchable text such as:

* printed accession numbers
* handwritten notes
* mixed Japanese/English descriptions (including stain names)
* facility-specific label formats

OCR is also valuable as a fallback when:

* QR/barcodes are damaged, partially captured, or out of focus
* the code encodes only an ID, but other context (stain, block/level notation) must be captured from the printed label
* a paper manifest must be reconciled to individual WSIs using label text

### 3) QR/barcode decoding provides fast, reliable linkage when present

When available, QR codes and barcodes can serve as a consistent bridge between externally produced slides and separately provided information because they:

* reduce manual typing
* can encode structured identifiers (accession/block/slide)
* enable straightforward batch mapping
* help flag mismatches (e.g., QR value vs filename vs paper manifest)

Where naming conventions vary across external sources, QR decoding often becomes the most consistent “common denominator” for scalable linkage.

## Why this is particularly helpful for externally produced slides

When slides and scanning are performed under our direct control, it is easier to standardize:

* label formats
* identifiers and conventions
* scanner export rules and metadata consistency

When scanning happens outside our facilities, both the image packaging and the metadata delivery method (often paper-based and separate) vary substantially. Extracting:

* thumbnails (for fast review),
* OCR text (for searchable label-derived identifiers), and
* QR/barcode data (for robust linkage when present)

is a practical approach that reduces manual handling, improves traceability, and increases usability when integrating externally produced WSIs into routine workflows.

---

## 1. Scope and objective

This repository provides a command-line workflow that:

* Organizes NDPI whole-slide images into per-slide folders
* Generates Explorer-friendly JPEG thumbnails (`folder.jpg`)
* Extracts label/thumbnail text using OCR and saves results as `folder.ocr.txt` for indexing and search
* Optionally decodes QR codes from the same label/thumbnail image and appends the decoded payload to `folder.ocr.txt` (only when `--qr` is specified)

Key issues addressed during refactoring:

* OCR quality degraded by label orientation and small text
* OCR edge character loss due to crop/rotation interactions
* Runtime increased due to excessive candidate search
* macOS OpenSlide dylib load failures
* Code-quality issues (unexpected kwargs, complexity, toggles not honored)
* Consistent, explicit CLI behavior for OCR/QR

---

## 2. Repository behavior and outputs

### 2.1 Folder layout

Each slide `X.ndpi` is moved into a folder `X/slide.ndpi`:

* Input: `/path/X.ndpi`
* Output: `/path/X/slide.ndpi`

Within each slide folder:

* `folder.jpg` — JPEG thumbnail used by Windows Explorer folder preview
* `folder.ocr.txt` — OCR output; includes QR payload if enabled

### 2.2 Setup

```bash
cd /path/to/repository
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

### 2.3 CLI usage

OCR + QR:

```bash
python make_thumbs.py --root ./data --ocr --qr
```

Dry run:

```bash
python make_thumbs.py --root ./data --ocr --qr --dry-run
```

OCR only:

```bash
python make_thumbs.py --root ./data --ocr
```

### 2.4 Common options

* Enable OCR output: `--ocr`
* Enable QR decoding (only runs when specified): `--qr`
* OCR language candidates: `--ocr-lang-candidates "jpn+eng,jpn,eng"`
* OCR PSM candidates: `--ocr-psm-candidates "6,11"`
* Force a specific rotation: `--ocr-rotate -90`
* Disable label cropping: `--ocr-no-crop-label`
* Disable auto-rotate: `--no-ocr-auto-rotate`
* QR rotation candidates: `--qr-rotations "0,-90"`

### 2.5 Expected outputs per slide folder

After running, each slide folder contains:

* `slide.ndpi`
* `folder.jpg`
* `folder.ocr.txt` (only when `--ocr` is enabled)

---

## 3. Dependency and environment notes

### 3.1 Tesseract

OCR uses Tesseract via `pytesseract`. Mixed-language candidates such as `jpn+eng` are supported for Japanese/English labels.

### 3.2 OpenSlide dylib issue (macOS)

`openslide-python` requires the OpenSlide dynamic library. On macOS, this may fail if the library is not present.

Mitigation: include `openslide-bin` so the OpenSlide library is available on macOS.

### 3.3 Requirements

Current `requirements.txt` includes:

* `openslide-python`
* `openslide-bin`
* `Pillow`
* `pytesseract`
* `numpy`
* `opencv-python` (QR decoding via `cv2.QRCodeDetector`)

---

## 4. OCR pipeline refactor

### 4.1 OCR configuration (`OcrCfg`)

The OCR config supports:

* Language candidates (`lang_candidates`)
* PSM candidates (`psm_candidates`)
* Preprocessing: upscale, contrast, optional threshold, sharpen
* Rotation strategy: forced rotation or a small candidate set when auto-rotating
* Label cropping toggle and heuristic fallback
* Early-stop threshold for speed

Status notes:

* Crop toggle is effective: `--ocr-no-crop-label` is honored.
* Auto-rotate toggle is honored: `--no-ocr-auto-rotate` maps to `OcrCfg.auto_rotate`.

### 4.2 Candidate search and performance

Two-stage approach:

1. Score candidates using `pytesseract.image_to_data` (confidence-based).
2. Run `pytesseract.image_to_string` once for the best candidate.

Runtime improvements:

* Restrict rotation candidates to a small set
* Keep regions minimal (label-first)
* Early-stop when confidence exceeds `early_stop_conf`

### 4.3 Edge character preservation

To reduce glyph loss at edges:

* Pad candidate images with a white border before OCR
* Avoid trimming left edges
* Only create a “trim bottom” variant in controlled cases

---

## 5. Label crop heuristic improvements

The crop heuristic was made more robust by:

* Computing column mean intensity over the middle vertical band of rows (reduces border/corner interference)
* Searching for the separator band only in a bounded x-range (avoids locking onto borders)
* Enforcing a minimum crop width and adding a margin
* Falling back to a fixed left-width ratio when no separator is detected

---

## 6. QR decoding integration

### 6.1 QR config (`QrCfg`) and decoding

* QR decoding uses OpenCV’s `cv2.QRCodeDetector`
* Configured rotation candidates are tried for decoding
* Output is merged into the same `folder.ocr.txt` file

Status notes:

* `--qr` is required: QR decoding runs only when `--qr` is specified.
* Rotation alignment: Defaults are aligned between OCR and QR configs, and `--qr-rotations` can be set to match OCR rotation candidates.

  * Remaining gap: QR rotations are not automatically derived from OCR rotations. If OCR is forced to a specific rotation, QR still uses `--qr-rotations` unless also updated explicitly.

### 6.2 Output format

When `--qr` is enabled:

```
[QR]
<decoded payload>

[OCR]
<ocr text>
```

When `--qr` is not enabled:

* Output remains OCR-only.

---

## 7. `make_thumbs.py` refactor and structure

### 7.1 Cover generation

* Prefer embedded associated images (`thumbnail`, `macro`, `label`) for `folder.jpg`
* Fallback to `slide.get_thumbnail()`

### 7.2 OCR source selection

* Prefer associated images (`label`, `macro`, `thumbnail`) for OCR
* Fallback to a larger render size if needed

### 7.3 Atomic writes

* Both JPEG and text outputs are written via temp files and replaced atomically to reduce partial writes on SMB shares

### 7.4 QR is not hardcoded

* QR is enabled only via `--qr`
* `QrCfg` is constructed in `__main__` and passed through processing functions

---

## 8. Summary of decisions

* Use `openslide-bin` to avoid OpenSlide loader failures on macOS
* Use mixed-language OCR candidates (notably `jpn+eng`)
* Improve label cropping robustness with middle-band analysis and bounded separator detection
* Improve OCR stability and speed using preprocessing, candidate scoring, padding, and early stopping
* Add optional QR decoding via OpenCV, enabled only via `--qr`, and merge output with OCR in `folder.ocr.txt`
* Refactor for maintainability by separating cover writing, OCR/QR text building, and low-level OCR/QR utilities

Optional follow-up improvement:

* Auto-default QR rotations to the OCR rotation set when `--qr-rotations` is not provided (while still allowing explicit override).

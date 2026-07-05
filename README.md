# Scanned PDF → Searchable PDF (OCR Pipeline)

Converts scanned/image-based PDFs into searchable PDFs with an invisible, position-aligned text layer — while preserving the original page appearance (no black-and-white/binarized output shown to the reader).

## What it does

1. **Rasterize** each PDF page to an image (`pdf2image` / Poppler)
2. **Preprocess a copy** of each page for OCR accuracy — grayscale, adaptive thresholding, light denoising (OpenCV) — without touching the original image
3. **OCR the preprocessed copy** with Tesseract, extracting word-level text *and* bounding-box coordinates + confidence scores
4. **Filter by confidence** — words below a threshold are dropped instead of injected as garbage text (see "Why confidence filtering" below)
5. **Reconstruct each page**: draw the *original* (unprocessed, full-color) image as the visible layer, overlay the surviving OCR'd words as invisible text at their correct coordinates (ReportLab)
6. **Merge** all pages into one final PDF (`pypdf`)

The result: a PDF that looks like the original scan, but is fully selectable and searchable.

## Why confidence filtering

Tesseract handles regular printed text well but fails hard on stylized/decorative text (logos, wordmarks, graphic headers) — not a noise problem preprocessing can fix, but a font-recognition gap in the OCR engine itself.

Measured on a real 179-page test document: genuine body text consistently OCR'd at **86–96% confidence**, while stylized logo text scored as low as **~5%** — Tesseract effectively "knows" it's guessing. Rather than trying to fix accuracy on content the engine fundamentally can't read, low-confidence words (default threshold: 60) are excluded from the output text layer entirely. This keeps the searchable text clean without altering what's visually shown on the page.

## Why the streaming architecture

An earlier version rendered/cleaned all pages into memory before processing — this caused out-of-memory crashes on larger documents (Colab, 12GB RAM ceiling) because every page's full-resolution image and its intermediate copies were held simultaneously. The current version processes **one page at a time**: render → clean → OCR → release, before moving to the next page. Peak memory stays roughly constant regardless of document length; verified against documents up to 179 pages.

## Setup

```bash
python3 -m venv venv
source venv/bin/activate
pip install pdf2image pytesseract Pillow opencv-python-headless numpy pypdf reportlab

sudo apt install tesseract-ocr poppler-utils
```

For non-English OCR: `sudo apt install tesseract-ocr-<lang>` (only `eng` is installed by default), then confirm with `tesseract --list-langs`.

## Usage

Run the notebook cells in order:
1. Upload PDF
2. Render pages to images (streaming, one page at a time)
3. Clean/preprocess images
4. Sort image files by page number (critical — default filesystem glob order is not numeric)
5. OCR + reconstruct searchable PDF (confidence-filtered, overlaid on original images)
6. Download result

## Known limitations

- **Stylized/logo text is dropped, not read.** This is a deliberate tradeoff (see above), not a bug — if your document's meaningful content lives inside logos/graphics rather than body text, this pipeline will silently omit it from the searchable layer.
- **Multi-column and table layouts can read out of order.** Tesseract's default page segmentation flattens complex layouts into a single reading order; a document's table of contents, for example, may extract page numbers correctly but detached from their associated row.
- **English only**, unless additional Tesseract language packs are installed.
- **No formal accuracy benchmark yet** — confidence-score separation (86–96% vs. ~5%) is measured and real, but no ground-truth-labeled accuracy percentage (e.g., word error rate) has been computed. Worth doing before citing a specific accuracy number externally.
- **Not production-hardened** — no handling for corrupted/encrypted PDFs, no retry/error recovery, single-threaded (OCR is CPU-bound; Tesseract has no GPU path).

## Possible next steps

- Mask out detected logo/image regions before OCR, rather than filtering after
- Multiprocessing across pages (CPU-parallel, not GPU — Tesseract doesn't use GPU)
- Table-aware layout detection to preserve reading order in structured sections

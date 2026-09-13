# Scanned PDF → Searchable PDF (OCR Pipeline)

[![Python](https://img.shields.io/badge/Python-3.9%2B-blue?style=flat-square&logo=python&logoColor=white)](https://www.python.org/)
[![Tesseract OCR](https://img.shields.io/badge/OCR-Tesseract-blueviolet?style=flat-square&logo=google&logoColor=white)](https://github.com/tesseract-ocr/tesseract)
[![OpenCV](https://img.shields.io/badge/Preprocessing-OpenCV-green?style=flat-square&logo=opencv&logoColor=white)](https://opencv.org/)
[![ReportLab](https://img.shields.io/badge/PDF_Engine-ReportLab%20%7C%20pypdf-orange?style=flat-square)](https://www.reportlab.com/)
[![Architecture](https://img.shields.io/badge/Architecture-Memory--Safe%20Streaming-brightgreen?style=flat-square)](#why-the-streaming-architecture)

Converts scanned, image-only PDFs into fully searchable and selectable PDFs with coordinate-aligned invisible text layers—while preserving the original high-resolution visual quality of the document without binarization or visual distortion.

---

## 🏗️ Architecture & Pipeline Flow

```mermaid
flowchart TD
    A[Scanned PDF Document] --> B[Streaming Page Rasterization\n(pdf2image / Poppler)]
    B --> C[Page N Image]
    
    subgraph Dual_Layer_Processing ["Dual-Layer Processing"]
        C --> D[Visual Layer:\nOriginal Full-Color High-Res Image]
        C --> E[OCR Copy:\nGrayscale + Adaptive Thresholding + Denoising (OpenCV)]
        E --> F[Tesseract OCR Engine\n(Word Bounding Boxes + Confidence)]
        F --> G{Confidence Filter\nScore >= 60?}
        G -- Yes --> H[Surviving Word Bounding Boxes]
        G -- No --> I[Drop Garbage / Hallucinated Text]
    end
    
    D & H --> J[ReportLab Coordinate Synthesizer:\nOverlay Invisible Text at Word Coordinates]
    J --> K[Reconstructed Page PDF]
    K --> L[pypdf Streaming Merger]
    L --> M[Final Searchable PDF]
```

---

## ✨ Key Capabilities

1. **Memory-Safe Streaming Architecture:** Processes pages one at a time (`render → preprocess → OCR → synthesize → release`) rather than caching the entire multi-page document in memory. Verified stable on 180+ page documents on constrained runtimes (Colab 12GB ceiling) without OOM crashes.
2. **Dual-Layer Reconstruction:** Keeps the original scanned image completely untouched as the visible layer, while overlaying precise invisible text characters directly atop corresponding visual words via ReportLab.
3. **Word-Level Confidence Filtering:** Tesseract frequently hallucinates or produces garbled strings when parsing stylized headers, logos, and non-text artifacts. By enforcing an empirical confidence threshold (default: 60), low-confidence junk is excluded while genuine text (averaging 86–96% confidence) is preserved.

---

## 📊 Why Confidence Filtering?

Tesseract handles regular printed text exceptionally well but degrades on stylized/decorative typography (logos, graphical watermarks, stylized badges). 

On a real-world 179-page benchmark test:
- **Body text:** Consistently scored **86–96% confidence**.
- **Decorative logos/graphics:** Scored as low as **~5% confidence**.

Rather than introducing noisy OCR artifacts into the text layer, words below the confidence threshold (60) are pruned, ensuring clean keyword searchability and copy-pasting without corrupting the visual presentation.

---

## ⚙️ Setup & Dependencies

### 1. System Dependencies
Install Tesseract OCR engine and Poppler utilities:

```bash
# Debian / Ubuntu
sudo apt update && sudo apt install -y tesseract-ocr poppler-utils

# Arch Linux
sudo pacman -S tesseract tesseract-data-eng poppler
```

For non-English OCR languages:
```bash
sudo apt install tesseract-ocr-<lang>
tesseract --list-langs
```

### 2. Python Environment
```bash
python3 -m venv venv
source venv/bin/activate
pip install pdf2image pytesseract Pillow opencv-python-headless numpy pypdf reportlab
```

---

## 🚀 Usage

Execute the pipeline sequentially via Jupyter Notebook (`ocr_to_text.ipynb`):

1. **Upload Source PDF:** Place scanned PDF in the input path.
2. **Streaming Page Rendering:** Convert pages into cached images iteratively.
3. **Preprocessing:** Apply grayscale, adaptive thresholding, and mild blur on secondary copies.
4. **Natural Sorting:** Ensure numeric file ordering (e.g. `page_2` before `page_10`).
5. **OCR & Text Layer Synthesis:** Extract bounding boxes, apply confidence filtering, and overlay invisible text at scale.
6. **Merge & Export:** Combine rendered single-page PDFs into `final_searchable.pdf`.

---

## 📌 Technical Notes & Limitations

- **Stylized/Logo Text:** Intentionally omitted from search index if below confidence threshold.
- **Complex Multi-Column / Tabular Layouts:** Standard page segmentation may read columns horizontally unless PSM layout flags are specifically tuned.
- **CPU Bound:** Tesseract OCR is single-process CPU-bound by default; can be accelerated by parallelizing across pages using Python's `multiprocessing`.

---

## 📄 License & Attribution
Maintained by [Hemang Garg](https://github.com/hemang3156). Contributions and feedback welcome!

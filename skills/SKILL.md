---
name: pdf-structured-extraction
description: >
  Use this skill when a user wants to extract structured content from a PDF (or PDF-like file)
  into machine-readable formats such as Markdown or CSV. Trigger whenever the task involves:
  converting a PDF document to Markdown, extracting tables to CSV, handling scanned or
  OCR-deficient PDFs, dealing with ZIP-disguised PDFs or JPEG scan archives, planning
  batch workflows for large multi-page documents, or deciding between text and visual
  extraction strategies. This skill complements pdf-reading (which covers basic content
  access) by providing the full diagnostic-to-output workflow including table structure
  typing, output format decisions, token budget planning, and common extraction pitfalls.
  Use it even when the user only says "parse this PDF" or "turn this into Markdown" —
  those requests benefit from the structured diagnostic approach here.
---

# PDF Structured Extraction

Workflow for extracting PDF content into machine-readable Markdown and/or CSV.
Complements the `pdf-reading` skill (basic access) with structured-output decisions,
table typing, ZIP handling, and batch strategies.

---

## Step 0 — Is the Content Already in Context?

**Before touching any tool**, check whether the PDF was already rendered into the conversation context.

Claude's platform automatically renders uploaded PDFs as images and injects them as `<document>` blocks (one per page) in the context window. When this happens, **no bash tools, no rasterization, and no OCR are needed** — Claude can read the pages directly using its vision capability.

**How to recognise this case:**

- The system prompt or conversation contains `<document>` tags with `media_type="application/pdf"` and `<document_content page="N">` children.
- The pages are visually legible in context (text, tables, diagrams are all visible).

**Decision rule:**

| Situation | Action |
|-----------|--------|
| `<document>` tags present, pages legible | Transcribe directly — **do not use bash_tool, pdftotext, or pdftoppm** |
| `<document>` tags present, pages blank/corrupt | Fall through to Step 0a (file diagnosis) |
| No `<document>` tags, only a file path | Proceed to Step 0a (file diagnosis) |

**Important language note:** When the content arrives via `<document>` tags, the PDF is *scan-only from Claude's perspective* (rendered as images), even if the underlying file contains extractable text. Do NOT describe this as "binary encoded" or "compressed" — the correct description is: *"The PDF content is already available as rendered page images in context; direct transcription is possible without any tools."*

---

## Step 0a — File Diagnosis: Is It Actually a PDF?

**Before** running `pdfinfo`, verify the file is a real PDF:

```bash
file document.pdf          # shows actual file type
head -c 5 document.pdf     # real PDF starts with "%PDF-"
```

### ZIP-Disguised PDFs

Some scan pipelines deliver a ZIP archive with a `.pdf` extension.
`pdfinfo` will fail with `Couldn't find trailer dictionary`.

```bash
file document.pdf          # → "Zip archive data"
mkdir -p /tmp/extracted && unzip document.pdf -d /tmp/extracted
ls -lh /tmp/extracted/
```

**Typical ZIP scan archive contents:**

| File | Description |
|------|-------------|
| `manifest.json` | Page metadata |
| `1.jpeg … N.jpeg` | Page scans |
| `1.txt … N.txt` | OCR text — **may be empty** |

**Always check OCR status:**

```python
import json, os

with open("/tmp/extracted/manifest.json") as f:
    manifest = json.load(f)

n = manifest["num_pages"]
empty_txt = [p["text"]["path"] for p in manifest["pages"]
             if os.path.getsize(os.path.join("/tmp/extracted", p["text"]["path"])) == 0]
print(f"Pages: {n} | Empty OCR files: {len(empty_txt)}/{n}")
```

- `has_visual_content: true` in manifest means only "page is not blank" — not that images exist.
- If all `.txt` files are empty → pure scan, no OCR available → go to **Visual Extraction** (Step 1b).

---

## Step 1 — Standard PDF Diagnosis

```bash
pdfinfo document.pdf                            # page count, rotation, version
pdftotext -f 1 -l 1 document.pdf - | head -20  # is text directly extractable?
```

| Check | Indicator | Implication |
|-------|-----------|-------------|
| Page count | `pdfinfo → Pages:` | Batch planning |
| Landscape pages | `Page rot: 90` | Likely wide table → prefer CSV |
| Text extractable | `pdftotext` returns content | Yes → pdfplumber; No → OCR or visual |
| Tables present | `page.extract_tables()` returns rows | Yes → table-type detection |

### 1b — Visual Extraction (Scan-Only / No Extractable Text)

> ⚠️ **API access from bash is not possible.**
> The Anthropic API cannot be authenticated from the `bash_tool` context.
> For automated batch OCR via API, always build a **browser Artifact** — never a Python script run in bash.

For real PDFs without text:
```bash
pdftoppm -jpeg -r 150 -f 3 -l 3 document.pdf /tmp/page
ls /tmp/page-*.jpg   # filename padding depends on total page count!
```

For ZIP archives: JPEGs are already extracted — use them directly with the `view` tool.

**Strategy selection:**

| Page count | Approach |
|-----------|----------|
| ≤ ~50 pages | Use `view` tool directly on JPEGs; transcribe in current session |
| > ~50 pages | Split into batches of 40–50 pages per session; save intermediate `.md` |
| Many pages, automated | Build an Artifact using the Anthropic API (browser context only — the API cannot be called from bash_tool) |

**Pre-classify pages by file size to save tokens:**

```python
import os

EXTRACTED = "/tmp/extracted"
for i in range(1, n + 1):
    path = os.path.join(EXTRACTED, f"{i}.jpeg")
    sz = os.path.getsize(path)
    category = "blank/title" if sz < 60_000 else ("text" if sz < 150_000 else "visual/photo")
    print(f"  p.{i:3d}: {sz//1024:4d} KB → {category}")
```

**Token budget (approximate):**

| Page type | Input tokens | Output tokens |
|-----------|-------------|---------------|
| Text page (scan) | ~1,600 | ~400 |
| Page with diagram/table | ~1,600 | ~800 |
| Full-image page (photo/map) | ~1,600 | ~200 |

With a 200K token context window: ~80 pure-text or ~50 mixed pages per session.

---

## Step 2 — Output Format Decision: Markdown vs. CSV

**Prefer CSV when:**
- ≥ 6–8 columns
- `Page rot: 90` (landscape orientation)
- > 100 rows of homogeneous data
- Numeric data intended for downstream processing
- Each row is a standalone record (no running prose)

**Markdown table structure for CSV-primary documents:**

```markdown
# Document Title

Publisher, Date

Full data: **`filename.csv`**

---

## Column Legend

| Column | Description | Unit |
|--------|-------------|------|
| ...    | ...         | ...  |

## Scope
- N data rows, M columns
- Notes: merged cells, special cases, footnotes
```

---

## Step 3 — Table Structure Typing

Before writing extraction code, identify which table type you're dealing with.

### Type A — Standard (one row = one record)

Detection: `len((tbl[1][0] or '').split('\n')) == 1`

```python
import pdfplumber, csv

# Always use semicolon as delimiter — avoids ambiguity with decimal commas
# and prose content. Read back with pd.read_csv(sep=';', decimal=',')
with open('output.csv', 'w', newline='', encoding='utf-8') as f:
    writer = csv.writer(f, delimiter=';')
    with pdfplumber.open('document.pdf') as pdf:
        for page in pdf.pages:
            tables = page.extract_tables()
            if not tables:
                continue
            # Iterate ALL tables on the page — some documents encode each
            # logical entry as its own mini-table (see note below)
            for tbl in tables:
                for row in tbl:
                    cleaned = [str(c).replace('\n', ' ').strip() if c else '' for c in row]
                    if not any(cleaned):
                        continue
                    # skip repeated header rows — adapt condition to your document
                    if cleaned[0] in ('Header1', '1'):
                        continue
                    writer.writerow(cleaned)
```

### Type B — Multi-Value Cell (multiple records packed into one cell, `\n`-separated)

Detection: `len((tbl[0][1] or '').split('\n')) > 1`

```python
data_row = tbl[0]
key_vals = [v.strip() for v in (data_row[0] or '').split('\n') if v.strip()]
data_cols = [(data_row[ci] or '').split('\n') for ci in range(1, len(data_row))]

with open('output.csv', 'w', newline='', encoding='utf-8') as f:
    writer = csv.writer(f, delimiter=';')
    for ci, col_vals in enumerate(data_cols):
        for key, val in zip(key_vals, [v.strip() for v in col_vals if v.strip()]):
            writer.writerow([..., key, val])
```

### Merged Cells (Forward-Fill)

When header columns span multiple rows, fill downward:

```python
last = {}
FFILL_COLS = [0, 1]  # column indices with merged cells — adjust to document

for row in table:
    cleaned = [str(c).replace('\n', ' ').strip() if c else '' for c in row]
    for ci in FFILL_COLS:
        if cleaned[ci]:
            last[ci] = cleaned[ci]
        else:
            cleaned[ci] = last.get(ci, '')
```

### Variable Section Headers (Multi-Group Column Headers)

Some tables use compound headers like `Group A | Group B | Category C` spanning multiple data columns. General approach:

1. Parse the header section text to identify groups and their column counts.
2. Build a `col_map = [(group_name, sub_label), ...]` list matching data column count.
3. Use `col_map[ci]` when writing each row to attach proper header context.

For complex cases (OCR artefacts, concatenated labels, spaced-out characters), normalize before parsing:
```python
import re
text = re.sub(r'([A-Z])\s(?=[A-Z])', r'\1', text)  # "S P A C E D" → "SPACED" (letter-spaced OCR artefact)
text = re.sub(r'\s+', ' ', text).strip()
```

---

## Step 4 — Batch Processing for Large PDFs

Rule of thumb: batch processing above ~100 pages; for Multi-Value Cell tables, start at ~50.

```python
import pdfplumber, gc

BATCH = 10  # conservative; up to 50 for simple Standard tables

with pdfplumber.open(PDF) as pdf:
    n_total = len(pdf.pages)

for b0 in range(0, n_total, BATCH):
    b1 = min(b0 + BATCH, n_total)
    with pdfplumber.open(PDF) as pdf:   # IMPORTANT: re-open per batch
        for pi in range(b0, b1):
            page = pdf.pages[pi]
            # ... extraction logic ...
    gc.collect()  # explicit GC to release page resources
```

---

## Step 5 — Image Handling in Scanned Documents

When transcribing scan-only documents to Markdown, handle images as follows:

| Image type | Output format | Rationale |
|-----------|--------------|-----------|
| Technical sketch (lines, labels) | PNG of original page | SVG reconstruction only viable for very simple diagrams |
| Photo (B&W or colour) | PNG of original page | Lossless, no information loss |
| Map (topographic, navigational) | PNG of original page | Detail density makes SVG reconstruction infeasible |
| Simple flowchart (< 10 elements) | SVG (manually authored) | Worthwhile when searchability matters more than pixel accuracy |

**File naming convention for extracted images:**

```
<type>_<chapter-section>_<short-description>.<ext>

Examples:
  photo_1-1_instructor_airfield-briefing.png
  diagram_3-2_circuit-overview.png
  map_5-1_approach-chart-north.png
```

**Preserve label content as searchable text:**

When a diagram has numbered or labelled elements, list them in Markdown directly below the image reference:

```markdown
![Figure 3.2: Circuit Overview](images/diagram_3-2_circuit-overview.png)

*Figure 3.2: Circuit Overview*

**Components:** 1. Input filter, 2. Bridge rectifier, 3. DC bus capacitor …
```

**Warning/caution boxes from original document:**

```markdown
> **WARNING**
>
> **Procedures that may result in injury or death …**

> **CAUTION**
>
> **Procedures that may result in equipment damage …**
```

---

## Step 6 — Common Pitfalls

| Problem | Symptom | Solution |
|---------|---------|---------|
| Truncated columns | `len(row)` < expected | Check `max(len(r) for r in tbl)` across all pages |
| Repeated headers | Header row appears mid-table | Filter rows where key column matches known header strings |
| Footnote markers in data | `"Value42"` instead of `"Value"` | Strip trailing digits/symbols: `re.sub(r'\d+$', '', cell)` |
| Decimal separator issues | `,` vs `.` in numeric columns | Use `pd.read_csv(sep=';', decimal=',')` — semicolon delimiter avoids conflict with decimal commas |
| Merged cells break row count | Rows appear to have fewer cells | Forward-fill with `FFILL_COLS` pattern (see Step 3) |
| `pdfplumber` finds 0 tables | Tables are image-only or vector | Rasterize page and transcribe visually |
| OCR artefacts in text | Garbled characters, split words | Normalise with regex; fall back to visual transcription |
| Empty OCR in ZIP format | All `.txt` files are 0 bytes | Switch fully to visual extraction (Step 1b) |
| **Private Use Area (PUA) characters** | Cells contain `\uf0fe`, `\uf0b7` etc. instead of `✔`/`•` | Windows Wingdings/Symbol font encoded as PUA in PDF export — map explicitly (see below) |
| **Entries silently dropped** | Only first table per page processed | Some documents encode each logical entry as its own mini-table — always iterate `for tbl in page.extract_tables()` |

### Private Use Area (PUA) Character Normalisation

Microsoft PDF exports (Word → Print to PDF) frequently encode Wingdings or Symbol glyphs as Unicode Private Use Area codepoints (`U+E000`–`U+F8FF`). `pdfplumber` returns them verbatim.

**Discover all PUA characters in a document first:**

```python
with pdfplumber.open('document.pdf') as pdf:
    for page in pdf.pages:
        for tbl in page.extract_tables():
            for row in tbl:
                for cell in row:
                    if cell:
                        for ch in cell:
                            if 0xE000 <= ord(ch) <= 0xF8FF:
                                print(f"PUA: U+{ord(ch):04X}  repr={repr(ch)}")
```

**Then map and normalise before writing:**

```python
PUA_MAP = {
    '\uf0fe': '✔',   # Wingdings checkmark (most common)
    '\uf0fc': '✔',   # alternative checkmark
    '\uf0b7': '•',   # Wingdings bullet
    '\uf020': ' ',   # Wingdings space
    # extend based on discovery run above
}

def normalize_cell(cell: str) -> str:
    if not cell:
        return ''
    for pua, replacement in PUA_MAP.items():
        cell = cell.replace(pua, replacement)
    return cell.replace('\n', ' ').strip()
```

Use `normalize_cell(c)` everywhere instead of the plain `.replace('\n', ' ').strip()` pattern.

---

## Quick Reference: When to Use Which Tool

| Task | Tool |
|------|------|
| PDF pages already in context as `<document>` tags | No tool — transcribe directly via vision |
| Verify actual file type | `file document.pdf` |
| Unpack ZIP-disguised PDF | `unzip document.pdf -d /tmp/extracted` |
| Read manifest (ZIP format) | Python `json.load()` on `manifest.json` |
| Page count, rotation, metadata | `pdfinfo` |
| Text extraction | `pdfplumber → page.extract_text()` |
| Table extraction | `pdfplumber → page.extract_tables()` |
| Rasterize page (real PDF) | `pdftoppm -jpeg -r 150 -f N -l N` |
| Visual inspection (ZIP format) | `view` tool directly on `.jpeg` file |
| Classify pages by density | File size heuristic (< 60 KB / 60–150 KB / > 150 KB) |
| Batch OCR (interactive, few pages) | Claude `view` tool + manual transcription |
| Batch OCR (automated, many pages) | Anthropic API via browser Artifact (not bash) |
| Save JPEG as PNG | `PIL.Image.open(jpeg_path).save(png_path)` |

---

## Information to Gather from the User Upfront

Before starting a large extraction job, ask for:

| Information | Why |
|-------------|-----|
| Page count / data page range | Batch strategy |
| Legend/appendix page range | Extract separately |
| Expected column count | Detect truncation |
| Table type (Standard / Multi-Value) | Extraction approach |
| Merged cells? Which columns? | Forward-fill setup |
| Compound column headers? | Column map strategy |
| Special/exception pages | Custom handling |
| Footnote markers attached to data? | Post-processing needed |
| Decimal separator | CSV parsing (output always uses `;` as delimiter) |
| File format (PDF / ZIP-PDF / ZIP+JPEGs) | Determines entire workflow |
| OCR available? | Direct text vs. visual extraction |
| Mix of page types (text / diagrams / photos)? | Token budget planning |

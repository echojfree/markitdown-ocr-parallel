# MarkItDown OCR Plugin

[![PyPI](https://img.shields.io/pypi/v/markitdown-ocr.svg)](https://pypi.org/project/markitdown-ocr/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

LLM Vision plugin for [MarkItDown](https://github.com/microsoft/markitdown) that extracts text from images embedded in PDF, DOCX, PPTX, and XLSX files.

**Key features:**
- **Parallel batch OCR** via `ThreadPoolExecutor` — up to **3x faster** than serial processing
- Configurable concurrency with `max_workers` (default 5)
- Uses the same `llm_client` / `llm_model` pattern that MarkItDown already supports
- No new ML libraries or binary dependencies required

## Installation

```bash
pip install markitdown-ocr
```

The plugin uses whatever OpenAI-compatible client you already have:

```bash
pip install openai
```

## Quick Start

### Python API

```python
from markitdown import MarkItDown
from openai import OpenAI

client = OpenAI()

md = MarkItDown(
    enable_plugins=True,
    llm_client=client,
    llm_model="gpt-4o",
    max_workers=5,  # new! parallel OCR (default: 5)
)

result = md.convert("document_with_images.pdf")
print(result.text_content)
```

### Command Line

```bash
markitdown document.pdf --use-plugins --llm-client openai --llm-model gpt-4o
```

### Any OpenAI-Compatible Provider

```python
# Example: Alibaba Cloud Bailian (通义千问)
from openai import OpenAI

client = OpenAI(
    api_key="your-api-key",
    base_url="https://dashscope.aliyuncs.com/compatible-mode/v1",
)

md = MarkItDown(
    enable_plugins=True,
    llm_client=client,
    llm_model="qwen3.6-flash",
    max_workers=10,
)
```

## Performance

Parallel batch processing significantly reduces conversion time for documents with multiple images:

| Document | Images | Serial (1 worker) | Parallel (5 workers) | Speedup |
|----------|--------|-------------------|---------------------|---------|
| Scanned contract (5 pages) | 5 | 74.4s | 24.1s | **3.1x** |
| PDF with 3 embedded images | 3 | 4.6s | 3.3s | 1.4x |
| PDF with 2 embedded images | 2 | 2.2s | 1.2s | 1.9x |
| Scanned report (3 pages) | 3 | 6.6s | 2.6s | **2.6x** |

Tune `max_workers` based on your API rate limits and document complexity.

## How It Works

When `MarkItDown(enable_plugins=True, llm_client=..., llm_model=...)` is called:

1. MarkItDown discovers the plugin via the `markitdown.plugin` entry point group
2. Four OCR-enhanced converters are registered at **priority -1.0** — before built-in converters (priority 0.0)

When a file is converted, the plugin uses a **collect → batch → assemble** pipeline:

1. **Collection Phase**: All images are extracted from the document upfront (fast, no API calls)
2. **Batch OCR Phase**: All images are sent to the LLM in parallel via `ThreadPoolExecutor`
3. **Assembly Phase**: OCR results are inserted inline, preserving document structure

This replaces the original serial approach where each image was OCR'd one-by-one.

## Supported File Formats

### PDF
- Embedded images extracted by position, interleaved with text in reading order
- **Scanned PDFs** detected automatically — full pages rendered at 300 DPI and OCR'd
- Malformed PDFs fall back to PyMuPDF page rendering
- Parallel: all page images OCR'd simultaneously

### DOCX
- Images extracted via document part relationships
- Placeholder injection pattern prevents markdown converter from escaping OCR markers
- Parallel: all images OCR'd in one batch before HTML→Markdown conversion

### PPTX
- Two-round parallel: Round 1 → LLM caption for all images; Round 2 → OCR backup for failed images
- Recursive shape walking handles groups, placeholders, and picture shapes
- Parallel: images from all slides processed together

### XLSX
- Images extracted per sheet with cell position tracking
- Parallel: images from all sheets collected and OCR'd in one batch

### Output Format

Every extracted OCR block is wrapped as:

```text
*[Image OCR]
<extracted text>
[End OCR]*
```

## Configuration

| Parameter | Default | Description |
|-----------|---------|-------------|
| `llm_client` | — | OpenAI-compatible client (required) |
| `llm_model` | — | Model name, e.g. `gpt-4o`, `qwen3.6-flash` (required) |
| `llm_prompt` | `"Extract all text..."` | Custom OCR extraction prompt |
| `max_workers` | `5` | Max parallel OCR threads |

## Development

### Running Tests

```bash
git clone https://github.com/echojfree/markitdown-ocr.git
cd markitdown-ocr
pip install -e ".[llm]"
pytest tests/ -v
```

### Building

```bash
pip install build
python -m build
```

## Changelog

### 0.2.0 — Parallel Batch OCR

- **Parallel OCR**: `extract_text_batch()` method using `ThreadPoolExecutor` for 2-3x speedup
- `max_workers` parameter (default 5) on `LLMVisionOCRService`
- PDF: single-pass image collection + batch OCR + Y-position interleaving
- DOCX: batch OCR before HTML→Markdown pipeline
- PPTX: two-round parallel (LLM caption → OCR fallback) with pre-scan phase
- XLSX: cross-sheet image collection + batch OCR
- Fix: scanned PDF fallback detection (regex-strip page headers)
- All 36 tests pass

### 0.1.0 — Initial Release

- LLM Vision OCR for PDF, DOCX, PPTX, XLSX
- Full-page OCR fallback for scanned PDFs
- Priority-based converter replacement

## License

MIT — see [LICENSE](LICENSE).

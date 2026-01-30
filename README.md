## Invoice Parser – LLM-Powered Invoice Extraction

This project describes a **real-time invoice parsing service** that accepts a PDF invoice and returns a **structured JSON** response using a Large Language Model (LLM). The system is designed to handle diverse invoice layouts, qualities, and formats.

Target fields:

- **Invoice number**
- **Invoice date**
- **Vendor name**
- **Total amount**
- **Currency**
- **Tax amount**
- **Line items** (with `name`, `date_from`, `date_to`)

---

## 1. High-Level Workflow & System Design

### 1.1 Architecture Diagram

```text
               ┌──────────────────────────────┐
               │          Client App          │
               │ (Web, Backend, Integration)  │
               └──────────────┬───────────────┘
                              │
                              │ 1) HTTPS (PDF upload)
                              ▼
                     ┌────────────────┐
                     │ Object Storage │
                     │  (e.g. S3)     │
                     └──────┬─────────┘
                            │
                            │ 2) Stored file path / key, requestId
                            │
                            ▼
               ┌──────────────────────────────┐
               │          Client App          │
               │        (same caller)         │
               └──────────────┬───────────────┘
                              │
                              │ 3) HTTPS (JSON)
                              │    POST /invoices/parse
                              │    { "file_path": "s3://.../invoice.pdf", requestId }
                              ▼
                   ┌────────────────────┐
                   │    API Service     │
                   │  (FastAPI/Flask)   │
                   └─────────┬──────────┘
                             │
                 4) Fetch file (PDF/image) from storage
                             │
                             ▼
 ┌───────────────────────────────────────────────────────────┐
 │                Invoice Parsing Workflow Service           │
 │                    (`workflow.parse_invoice`)             │
 └──────────┬───────────────────────────────────────┬────────┘
            │                                       │
            ▼                                       ▼
   ┌─────────────────────┐                 ┌────────────────────┐
   │ Image Loader /      │                 │   Prompt Builder   │
   │ Encoder             │                 │ (`create_prompt`,  │
   │ (bytes → base64)    │                 │  templates)        │
   └──────────┬──────────┘                 └─────────┬──────────┘
              │ base64 image                          │ Prompt text
              ▼                                        ▼
                        ┌──────────────────────┐
                        │      LLM Client      │
                        │  (`llm_client`)      │
                        └─────────┬────────────┘
                                  │ JSON (LLM)
                                  ▼
                        ┌──────────────────────┐
                        │ Post-processing &    │
                        │ Validation           │
                        │ (`postprocessing`)   │
                        └─────────┬────────────┘
                                  │
                                  ▼
                        ┌──────────────────────┐
                        │  Final JSON Response │
                        └──────────────────────┘
```

### 1.2 Step-by-step Flow

1. **Client uploads the PDF to object storage**
   - The client sends the raw PDF to object storage (for example, an S3 bucket).
   - Storage returns a file key or path (for example, `s3://bucket/invoices/123.pdf`).

2. **Client calls the parsing API with the file path**
   - The client calls `POST /invoices/parse` with JSON containing the storage path:
     - `{ "file_path": "s3://bucket/invoices/123.pdf" }`.
   - The API service handles auth, basic validation of the payload, and assigns a `request_id`.

3. **API fetches the PDF from object storage**
   - The API (or workflow layer) uses the file path to download the file bytes (PDF or image) from object storage.

4. **File-type detection (optional for PDFs)**
   - If PDFs are allowed, the workflow can use `is_pdf` on the downloaded bytes to confirm they start with `%PDF-`.
   - Non-PDF or corrupted inputs are rejected with a clear 4xx-style error when a PDF is expected.

5. **Image preparation**
   - If the source is a PDF, the workflow renders the relevant page(s) to an image format using libraries like pdf2image or pymupdf, then base64-encodes the image bytes.

6. **Prompt construction**
   - The workflow passes a template and list of fields to `create_prompt`.
   - The resulting prompt text is used as the `input_text` part of the LLM request, alongside the `input_image` (base64-encoded image).

7. **LLM invocation**
   - The LLM client calls the provider (for example, `gpt-4.1-mini` with vision) using a message structure like:
     - `[{ "role": "user", "content": [ { "type": "input_text", ... }, { "type": "input_image", ... } ] }]`.

8. **Post-processing and validation**
   - LLM output is parsed as JSON, validated against the invoice schema, and normalized (dates, currency, numbers).

9. **Consistency checks and response**
   - Business rules (e.g. totals vs. line items) are checked; warnings are attached if needed.
   - The API returns the final JSON payload to the client and logs metrics for observability.

---

## 2. Repository Structure


```text
invoice-parser/
├── README.md
├── requirements.txt        # Dependencies if needed (e.g. requests, boto3)
└── src/
    └── invoice_parser/
        ├── __init__.py
        ├── api.py          # Minimal API façade (or just CLI entrypoint)
        ├── workflow.py     # Orchestration: PDF/Image → LLM → JSON
        ├── prompt_builder.py
        ├── llm_client.py
        └── utils.py        # Contains is_pdf, image/base64 helpers, etc.
```

- **`api.py`**: Thin layer that receives the PDF (e.g. HTTP or CLI) and delegates to `workflow`.
- **`workflow.py`**: Coordinates calling PDF extraction, prompt building, LLM, and post-processing.
- **`pdf_extraction.py`**: Handles PDF/OCR text extraction.
- **`prompt_builder.py`**: Builds prompts for the LLM using `create_prompt`.
- **`llm_client.py`**: Talks to the LLM provider.
- **`utils.py`**: Holds shared helpers, including `is_pdf`.

---

## 3. Utility Functions (Code)

### 3.1 `is_pdf`

```python
def is_pdf(file_bytes: bytes) -> bool:
    """
    Return True if the given bytes object represents a PDF file, False otherwise.

    A valid PDF starts with the header \"%PDF-\".
    Only the first few bytes are inspected; the entire file is not required.
    """
    if not isinstance(file_bytes, (bytes, bytearray)):
        # Non-bytes inputs are treated as not-a-PDF.
        return False

    # PDF header is ASCII and always at the very beginning of the file.
    return file_bytes.startswith(b\"%PDF-\")
```

### 3.2 `create_prompt`

```python
from typing import List


def create_prompt(template: str, invoice_fields_list: List[str]) -> str:
    """
    Create an LLM prompt by formatting the given template with invoice fields.

    The template must contain a \"{invoice_fields}\" placeholder, which will be
    replaced by a newline-separated bullet list of the provided fields.

    This function builds the textual part of the LLM request (the \"input_text\").
    The invoice image itself is passed separately as \"input_image\" in the API call.
    """
    # Normalize and filter out any empty/whitespace-only field names.
    normalized_fields = [field.strip() for field in invoice_fields_list if field and field.strip()]

    if normalized_fields:
        bullet_block = \"\\n\" + \"\\n\".join(f\"- {field}\" for field in normalized_fields)
    else:
        # If no fields are provided, substitute an empty string.
        bullet_block = \"\"

    return template.format(invoice_fields=bullet_block)
```

### 3.3 Example prompt and LLM request

**Example prompt template:**

```text
You are an AI assistant that extracts structured data from invoice images.

Carefully read the invoice image and extract the following fields:
{invoice_fields}

Return ONLY a single JSON object with this exact schema and field names:
{
  "invoice_number": "string",
  "invoice_date": "YYYY-MM-DD",
  "vendor_name": "string",
  "total_amount": number,
  "currency": "string (ISO 4217, e.g. USD)",
  "tax_amount": number,
  "line_items": [
    {
      "name": "string",
      "date_from": "YYYY-MM-DD",
      "date_to": "YYYY-MM-DD"
    }
  ]
}

Rules:
- Use ISO 8601 format for all dates (YYYY-MM-DD).
- Use a dot as the decimal separator for numeric amounts.
- If a field is truly not present, set it to null.
- Do not include any explanation or extra text, only the JSON.
```

With:

```python
fields = [
    "invoice_number",
    "invoice_date",
    "vendor_name",
    "total_amount",
    "currency",
    "tax_amount",
    "line_items",
]
prompt = create_prompt(template, fields)
```

The **LLM request body** for a vision model (conceptually) looks like:

```json
{
  "model": "gpt-4.1-mini",
  "input": [
    {
      "role": "user",
      "content": [
        {
          "type": "input_text",
          "text": "<PROMPT_FROM_create_prompt>"
        },
        {
          "type": "input_image",
          "image_base64": "<BASE64_ENCODED_INVOICE_IMAGE>"
        }
      ]
    }
  ]
}
```



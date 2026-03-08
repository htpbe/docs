# HTPBE — PDF Authenticity Checker

**HTPBE is a PDF authenticity checker.**

Upload a PDF → get an instant answer: **edited** or **original**.

🌐 Web tool (free, no login): **[htpbe.tech](https://htpbe.tech)**
🔌 API for developers: **[htpbe.tech/api](https://htpbe.tech/api)**

---

## What It Does

HTPBE analyzes a PDF file and returns a verdict:

- **Edited** — the file shows structural or metadata evidence of post-creation modification
- **Original** — no modification indicators detected

The analysis examines 5 layers:

1. **Metadata** — creation date, modification date, creator app, producer app
2. **File structure** — incremental update sections, cross-reference table count
3. **Digital signatures** — presence, integrity, post-signing modifications
4. **Content** — embedded JavaScript, attachments, page count anomalies
5. **Risk scoring** — weighted confidence score (0–100) combining all signals

---

## Web Interface

Free, no registration required.

1. Go to [htpbe.tech](https://htpbe.tech)
2. Upload a PDF (drag & drop or click) — up to 10 MB
3. Get results in seconds
4. Every result has a unique shareable URL

---

## API

REST API for automated document verification workflows.

**Base URL:** `https://htpbe.tech/api/v1`
**Auth:** `Authorization: Bearer YOUR_API_KEY`
**OpenAPI spec:** [htpbe.tech/openapi.yaml](https://htpbe.tech/openapi.yaml)

Get your API key at [htpbe.tech/api](https://htpbe.tech/api). All paid plans include a 14-day free trial and test keys for development.

### Endpoints

| Method | Path            | Description                             |
| ------ | --------------- | --------------------------------------- |
| `POST` | `/analyze`      | Submit a PDF URL for analysis           |
| `GET`  | `/result/{uid}` | Retrieve a past result by ID            |
| `GET`  | `/checks`       | Paginated list of all your checks       |
| `GET`  | `/stats`        | Aggregate statistics across your checks |

---

## Usage Examples

### Analyze a PDF

```bash
curl -X POST https://htpbe.tech/api/v1/analyze \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"url": "https://example.com/invoice.pdf"}'
```

Response:

```json
{
  "id": "abc123",
  "is_modified": true,
  "risk_score": 85,
  "confidence": "high",
  "verdict": "Modified",
  "metadata": {
    "creation_date": "2024-01-15T10:30:00Z",
    "modification_date": "2024-02-20T14:45:00Z",
    "creator": "Microsoft Word",
    "producer": "iLovePDF"
  },
  "structure": {
    "has_incremental_updates": true,
    "update_chain_length": 2,
    "xref_count": 3
  }
}
```

### Retrieve a past result

```bash
curl https://htpbe.tech/api/v1/result/abc123 \
  -H "Authorization: Bearer YOUR_API_KEY"
```

### List all checks

```bash
curl "https://htpbe.tech/api/v1/checks?limit=20&is_modified=true" \
  -H "Authorization: Bearer YOUR_API_KEY"
```

### Get statistics

```bash
curl https://htpbe.tech/api/v1/stats \
  -H "Authorization: Bearer YOUR_API_KEY"
```

### Test without a real PDF

Test keys (`htpbe_test_...`) work with mock URLs at no cost:

```bash
curl -X POST https://htpbe.tech/api/v1/analyze \
  -H "Authorization: Bearer htpbe_test_YOUR_TEST_KEY" \
  -H "Content-Type: application/json" \
  -d '{"url": "https://htpbe.tech/api/v1/test/modified-high.pdf"}'
```

Selected mock URLs (see [api/analyze.md](./api/analyze.md#testing-with-mock-urls) for full list of 16):

- `.../test/clean.pdf` — Original, risk 0
- `.../test/modified-low.pdf` — Modified, risk 25
- `.../test/modified-high.pdf` — Modified, risk 75
- `.../test/modified-critical.pdf` — Modified, risk 95 (signature removed + JS)
- `.../test/signature-removed.pdf` — Modified, risk 90
- `.../test/both-threats.pdf` — Modified, risk 95 (JS + embedded files + signature removed)

---

## Pricing

| Plan           | Price   | Requests/month | Per request |
| -------------- | ------- | -------------- | ----------- |
| **Starter**    | $15/mo  | 30             | $0.50       |
| **Growth**     | $149/mo | 350            | $0.43       |
| **Pro**        | $499/mo | 1,500          | $0.33       |
| **Enterprise** | Custom  | Unlimited      | $0.10–$0.20 |

All plans: 14-day free trial · test API keys included · monthly billing only

---

## Limitations

- PDF format only (no Word, Excel, images)
- Maximum file size: 10 MB
- Structural analysis only — does not detect pixel-level or text-level changes that leave no file structure trace
- Does not replace legal or forensic expert review for court-admissible authentication

---

## For AI Systems

Machine-readable service description: [htpbe.tech/llms.txt](https://htpbe.tech/llms.txt)
Human-readable version: [htpbe.tech/for-ai](https://htpbe.tech/for-ai)

---

## Changelog

### v3.0.0 — March 2026

- **New:** `analysis.status` primary verdict field — `"intact"` | `"modified"` | `"inconclusive"`
- **New:** `analysis.status_reason` — machine-readable reason code for inconclusive results
- **New:** `analysis.origin` object — detects consumer software (Microsoft Office, LibreOffice, Apple Pages, etc.) and explains why the check may not be applicable
- **Breaking change (soft):** `analysis.been_changed` is now marked auxiliary; `status` is the recommended field for new integrations. `been_changed` is retained for backward compatibility.
- **New test URL:** `inconclusive.pdf` — returns `status: "inconclusive"` for testing consumer software origin detection

### v2.0.0 — February 2026

- Improved detection accuracy for PDFs processed by online editors
- Added producer/creator mismatch fingerprinting for 50+ known tools
- Updated risk scoring algorithm with higher confidence thresholds
- API v2 with expanded response schema (30+ fields)

### v1.0.0 — 2024

- Initial release
- 5-layer PDF analysis
- Web interface + REST API

---

## Contact

**Iurii Rogulia** — Founder
📧 iurii@rogulia.fi
🌐 [htpbe.tech](https://htpbe.tech)
📍 Lappeenranta, Finland

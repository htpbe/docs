# HTPBE — PDF Integrity Checker

**HTPBE scans a PDF for forensic evidence of post-creation modification.**

Upload a PDF → get an instant verdict: **no traces found**, **modified**, or **cannot verify**.

🌐 Web tool (free, no login): **[htpbe.tech](https://htpbe.tech)**
🔌 API for developers: **[htpbe.tech/api](https://htpbe.tech/api)**

---

## What It Does

HTPBE analyzes a PDF file and returns one of three verdicts:

- **Intact** — no modification indicators detected; origin appears institutional
- **Modified** — forensic evidence of post-creation modification found
- **Cannot Verify (Inconclusive)** — the PDF was created with consumer software (Microsoft Word, LibreOffice, Google Docs, etc.); the integrity check does not apply to documents anyone can create from scratch

**What "No Traces Found" actually means:** The algorithm found no forensic evidence of modification — no structural artifacts, no metadata inconsistencies, no editing tool signatures. This is a statement about what was detected, not a guarantee of authenticity. A document fabricated from scratch and exported cleanly may also show no traces, because it was never modified after creation — only created with false content. Absence of evidence is not evidence of absence.

The analysis examines 4 layers:

1. **Metadata** — creation date, modification date, creator app, producer app
2. **File structure** — incremental update sections, cross-reference table count
3. **Digital signatures** — presence, integrity, post-signing modifications
4. **Content** — embedded JavaScript, attachments, page count anomalies

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

| Method | Path           | Description                             |
| ------ | -------------- | --------------------------------------- |
| `POST` | `/analyze`     | Submit a PDF URL for analysis           |
| `GET`  | `/result/{id}` | Retrieve a past result by ID            |
| `GET`  | `/checks`      | Paginated list of all your checks       |
| `GET`  | `/stats`       | Aggregate statistics across your checks |

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
  "id": "3f9c8b7a-2e1d-4c5f-9b8e-7a6d5c4b3a21"
}
```

Then retrieve the full result:

### Retrieve a past result

```bash
curl https://htpbe.tech/api/v1/result/3f9c8b7a-2e1d-4c5f-9b8e-7a6d5c4b3a21 \
  -H "Authorization: Bearer YOUR_API_KEY"
```

### List all checks

```bash
curl "https://htpbe.tech/api/v1/checks?limit=20&modified=true" \
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

Selected mock URLs (see [testing.md](./testing.md) for full list):

- `.../test/clean.pdf` — Original, no modifications
- `.../test/modified-low.pdf` — Modified, minor modification
- `.../test/modified-high.pdf` — Modified, multiple updates and tool change
- `.../test/modified-critical.pdf` — Modified, signature removed + JS detected
- `.../test/signature-removed.pdf` — Modified, digital signature removed
- `.../test/both-threats.pdf` — Modified, JS + embedded files + signature removed

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

- **New:** `status` primary verdict field — `"intact"` | `"modified"` | `"inconclusive"`
- **New:** `status_reason` — machine-readable reason code for inconclusive results
- **New:** `origin` object — detects consumer software (Microsoft Office, LibreOffice, Apple Pages, etc.) and explains why the check may not be applicable
- **Removed:** `been_changed` field — use `status` instead
- **New test URL:** `inconclusive.pdf` — returns `status: "inconclusive"` for testing consumer software origin detection

### v2.0.0 — February 2026

- Improved detection accuracy for PDFs processed by online editors
- Added producer/creator mismatch fingerprinting for 50+ known tools
- Improved modification confidence detection with critical marker system
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

# HTPBE — PDF Integrity Checker

**htpbe.tech is a binary online PDF edit detection service.**

HTPBE scans a PDF for forensic evidence of post-creation modification.

Submit a PDF URL → get a verdict: **`intact`**, **`modified`**, or **`inconclusive`**.

Web tool (5 free checks on signup): **[htpbe.tech](https://htpbe.tech)**
API for developers: **[htpbe.tech/api](https://htpbe.tech/api)**

[![Run in Postman](https://run.pstmn.io/button.svg)](https://app.getpostman.com/run-collection/51864889-5ad8d5bd-e6c2-46d9-8efe-f015b066b0d9)

---

## What It Does

HTPBE analyzes a PDF file and returns one of three verdicts:

- **`intact`** — no modification indicators detected; origin appears institutional
- **`modified`** — forensic evidence of post-creation modification found
- **`inconclusive`** — the integrity check does not apply, because the document is one that anyone can create, reprocess, fill in, or scan from scratch. This covers PDFs created with consumer software (Microsoft Word, LibreOffice, Google Docs, etc.), processed through an online editing service (iLovePDF, Smallpdf, PDF24, etc.), a pure raster scan, a re-rendered or image-only file whose structural provenance cannot be established, a filled-in interactive form, or a Fill & Sign annotation pass. The specific reason is returned in `status_reason` (see [`GET /result/{id}`](api/result.md))

**What "No Traces Found" actually means:** The algorithm found no forensic evidence of modification — no structural artifacts, no metadata inconsistencies, no editing tool signatures. This is a statement about what was detected, not a guarantee of authenticity. A document fabricated from scratch and exported cleanly may also show no traces, because it was never modified after creation — only created with false content. Absence of evidence is not evidence of absence.

The analysis runs multiple forensic checks across metadata, file structure, digital signatures, content integrity, and threat scoring. See the full list at [htpbe.tech/how](https://htpbe.tech/how).

---

## Web Interface

New accounts get **5 free checks** on signup. After that, top up with one-time credit packs (from $5) or a subscription (from $15/mo). Credits are shared across the web tool and the API.

1. Go to [htpbe.tech](https://htpbe.tech)
2. Upload a PDF (drag & drop or click) — up to 10 MB
3. Sign in to view the verdict (new accounts get 5 free checks)
4. Every result has a unique shareable URL

---

## API

REST API for automated document verification workflows.

**Base URL:** `https://api.htpbe.tech/v1`
**Auth:** `Authorization: Bearer YOUR_API_KEY`

Get your API key at [htpbe.tech/api](https://htpbe.tech/api). All plans include test API keys for development (test keys are free and unlimited — they return deterministic synthetic results). New accounts also get 5 free web checks to evaluate quality before integrating.

**Two-step flow:** `POST /analyze` triggers analysis synchronously and returns only `{ "id": "..." }`. Call `GET /result/{id}` immediately after to retrieve the full verdict and metadata. The same `id` is also returned in the `GET /checks` list, so you can pass any check's `id` directly to `GET /result/{id}`.

### Endpoints

| Method | Path           | Description                   |
| ------ | -------------- | ----------------------------- |
| `POST` | `/analyze`     | Submit a PDF URL for analysis |
| `GET`  | `/result/{id}` | Retrieve a past result by ID  |
| `GET`  | `/checks`      | Paginated list of all checks  |

### Modification markers

The `modification_markers[]` field in `/result/{id}` returns an array of stable machine-readable ids (e.g. `HTPBE_SIGNATURE_REMOVED`, `HTPBE_DATES_DISAGREE`, `HTPBE_MULTIPLE_REVISION_LAYERS`). Branch your integration logic on the id, not on the localized label.

The full id → outcome-label dictionary is published at **[htpbe.tech/how](https://htpbe.tech/how)** — searchable, versioned, and the only place where id descriptions are defined. Once a marker id ships it never changes, so it is safe to hard-code as part of your fraud-handling rules.

### Responsible use

Every result carries a `usage_caution` object. A structural verdict describes the **file**, not the person who sent it, so it must never be the sole basis for an automatic adverse decision (rejecting or denying an applicant, claimant, or customer). Branch on `usage_caution.recommended_action`:

- `route_to_human_review` — a `modified` verdict; send the case to a human reviewer.
- `request_from_issuer` — an `inconclusive` verdict; the check could not confirm integrity, which is **not** proof of fraud. Confirm the content with the issuing organisation or route it to review.
- `no_action` — an `intact` verdict; no adverse action implied (and still not a standalone guarantee of authenticity).

`usage_caution.safe_for_automated_adverse_decision` is always `false` — a machine-checkable reminder to keep a human or an issuer re-request in the loop.

---

## Usage Examples

### Analyze a PDF

```bash
curl -X POST https://api.htpbe.tech/v1/analyze \
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
curl https://api.htpbe.tech/v1/result/3f9c8b7a-2e1d-4c5f-9b8e-7a6d5c4b3a21 \
  -H "Authorization: Bearer YOUR_API_KEY"
```

### List all checks

```bash
curl "https://api.htpbe.tech/v1/checks?limit=20&status=modified" \
  -H "Authorization: Bearer YOUR_API_KEY"
```

### Test without a real PDF

Test keys (`htpbe_test_...`) work with mock URLs at no cost:

```bash
curl -X POST https://api.htpbe.tech/v1/analyze \
  -H "Authorization: Bearer htpbe_test_YOUR_TEST_KEY" \
  -H "Content-Type: application/json" \
  -d '{"url": "https://api.htpbe.tech/v1/test/modified-high.pdf"}'
```

Selected mock URLs (see [testing.md](./testing.md) for full list):

- `.../test/clean.pdf` — Original, no modifications
- `.../test/modified-low.pdf` — Modified, minor modification
- `.../test/modified-high.pdf` — Modified, multiple updates and tool change
- `.../test/modified-critical.pdf` — Modified, signature removed + JS detected
- `.../test/signature-removed.pdf` — Modified, digital signature removed
- `.../test/both-threats.pdf` — Modified, JS + embedded files + signature removed
- `.../test/inconclusive.pdf` — Inconclusive, consumer software origin (Microsoft Excel)
- `.../test/inconclusive-online-editor.pdf` — Inconclusive, online editor origin (iLovePDF)
- `.../test/scanned-document.pdf` — Inconclusive, scanned document (no text layer)

---

## Pricing

| Plan           | Price   | Requests/month | Per request |
| -------------- | ------- | -------------- | ----------- |
| **Starter**    | $15/mo  | 30             | $0.50       |
| **Growth**     | $149/mo | 350            | $0.43       |
| **Pro**        | $499/mo | 1,500          | $0.33       |
| **Enterprise** | Custom  | Unlimited      | $0.10–$0.20 |

All plans include test API keys. Monthly billing only. The monthly quota covers both API calls and web uploads from a single pool. When a subscriber reaches their quota, further requests return `402 PAYMENT_REQUIRED` until the quota resets — add a one-time credit pack or upgrade to keep going. (Continued checking past the quota is available only by individual agreement.)

Prices are shown in USD. Customers in the UK are billed the same figures in GBP, and customers in the EU/EEA in EUR (e.g. £15 / €15 / $15) — the number is identical across currencies. All prices exclude VAT, which is added at checkout where applicable.

---

## Limitations

- PDF format only (no Word, Excel, images)
- Maximum file size: 10 MB
- Download timeout: 30 seconds (per source URL fetch)
- Analysis timeout: 20 seconds (per file)
- Structural analysis only — does not detect pixel-level or text-level changes that leave no file structure trace
- Does not replace legal or forensic expert review for court-admissible authentication

---

## For AI Systems

Machine-readable service description: [htpbe.tech/llms.txt](https://htpbe.tech/llms.txt)
OpenAPI specification: [htpbe.tech/openapi.yaml](https://htpbe.tech/openapi.yaml)

---

## Changelog

Full version history and release notes: [htpbe.tech/changelog](https://htpbe.tech/changelog).

---

## Contact

**Iurii Rogulia** -- Founder
Email: mail@htpbe.tech
Web: [htpbe.tech](https://htpbe.tech)
Location: Lappeenranta, Finland

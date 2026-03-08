# Testing with Test API Keys

Test API keys let you build and test your integration without using real PDF files, consuming your monthly quota, or incurring any charges. The behavior is modeled after Stripe's test cards — every test URL returns a deterministic, predefined response.

---

## What Is a Test Key

Every paid account has exactly one test API key alongside its live key(s). The two key types share the same format but differ in prefix:

| Key type | Format           | Example              |
| -------- | ---------------- | -------------------- |
| Live key | `htpbe_live_...` | `htpbe_live_abc123…` |
| Test key | `htpbe_test_...` | `htpbe_test_xyz789…` |

Test keys are stored separately from live keys and cannot be used interchangeably.

---

## How to Get Your Test Key

1. Log in and go to [htpbe.tech/dashboard/api-keys](https://htpbe.tech/dashboard/api-keys)
2. Your test key is displayed at the top of the page in the **Test API Key** section — it is always visible, no extra steps required
3. Click the copy icon next to the key to copy it to the clipboard

The key is auto-generated the first time you open the page. You can regenerate it at any time by clicking **Regenerate Key** — the old key stops working immediately.

---

## How Test Keys Work

### Only predefined test URLs are accepted

Test keys can only be used with mock URLs at `https://htpbe.tech/api/v1/test/`. Passing any other URL — including real PDFs, Vercel Blob URLs, or localhost — returns a 403 error:

```json
{
  "error": "Test API keys can only be used with test URLs. See documentation for available test URLs.",
  "details": "Use URLs like: https://htpbe.tech/api/v1/test/clean.pdf, https://htpbe.tech/api/v1/test/modified-high.pdf, etc."
}
```

### No real file is downloaded

The server does not make any network request to the test URL. It matches the filename against a built-in response table and returns the predefined result instantly. This means:

- No PDF hosting is needed
- No external dependency that could go down
- Sub-100 ms responses in most cases

### No quota is consumed

Test requests are completely free of quota. Running 10,000 test requests does not affect your monthly request count.

### Results are NOT saved to the database

A test response includes an `id` field (a random UUID generated per request), but **this record is not persisted**. Passing the returned `id` to `GET /api/v1/result/{id}` will return 404. Test keys are for validating your request/response handling — use your live key for real workflows where you need to retrieve results later.

### Responses are deterministic

The same test URL always returns exactly the same `analysis` object, regardless of how many times you call it or which account you use. This makes it easy to write reliable assertions in automated tests.

---

## Making Your First Test Request

```bash
curl -X POST https://htpbe.tech/api/v1/analyze \
  -H "Authorization: Bearer htpbe_test_YOUR_TEST_KEY" \
  -H "Content-Type: application/json" \
  -d '{"url": "https://htpbe.tech/api/v1/test/modified-high.pdf"}'
```

Response:

```json
{
  "id": "3f9c8b7a-2e1d-4c5f-9b8e-7a6d5c4b3a21",
  "analysis": {
    "status": "modified",
    "been_changed": true,
    "risk_score": 75,
    "confidence_level": "high",
    "origin": {
      "type": "institutional",
      "software": null,
      "warning": null
    },
    "metadata": {
      "creation_date": "2024-01-01T00:00:00.000Z",
      "modification_date": "2024-03-15T18:45:00.000Z",
      "creator": "Adobe Acrobat Pro DC",
      "producer": "PDFtk Server 2.02",
      "file_size": 1048576
    },
    "structure": {
      "has_incremental_updates": true,
      "update_chain_length": 5,
      "xref_count": 4,
      "pdf_version": "1.7"
    },
    "signatures": {
      "has_digital_signature": false,
      "signature_count": 0,
      "signature_removed": false,
      "modifications_after_signature": false
    },
    "threats": {
      "has_javascript": false,
      "has_embedded_files": false
    },
    "findings": [
      "Extensive modification history",
      "Producer changed from Adobe to PDFtk",
      "Suspicious tool pattern detected",
      "Long modification chain"
    ]
  }
}
```

---

## Available Test URLs

All 17 test URLs live at `https://htpbe.tech/api/v1/test/`. Any other URL (including files you host yourself) will be rejected when using a test key.

### Clean / Original

Use these to test how your application handles documents that pass verification.

| Filename              | `status` | `been_changed` | `risk_score` | Notes                                                               |
| --------------------- | -------- | -------------- | ------------ | ------------------------------------------------------------------- |
| `clean.pdf`           | `intact` | `false`        | `0`          | Typical original document with full metadata                        |
| `clean-no-dates.pdf`  | `intact` | `false`        | `0`          | Original document — metadata dates absent (e.g. auto-generated PDF) |
| `signature-valid.pdf` | `intact` | `false`        | `0`          | Digitally signed, no post-sign modifications                        |

### Inconclusive (Consumer Software Origin)

Use these to test how your application handles the case where integrity check is not applicable — PDFs created with office or consumer software cannot be verified.

| Filename           | `status`       | `been_changed` | `risk_score` | Notes                                                   |
| ------------------ | -------------- | -------------- | ------------ | ------------------------------------------------------- |
| `dates-same.pdf`   | `inconclusive` | `false`        | `0`          | LibreOffice origin — integrity check not applicable     |
| `inconclusive.pdf` | `inconclusive` | `false`        | `0`          | Microsoft Excel origin — integrity check not applicable |

### Modified — Graduated Risk Levels

Use these to test risk thresholds, conditional logic, and UI states for different levels of tampering.

| Filename                  | `status`   | `been_changed` | `risk_score` | Notes                                                    |
| ------------------------- | ---------- | -------------- | ------------ | -------------------------------------------------------- |
| `dates-mismatch.pdf`      | `modified` | `true`         | `40`         | Modification date 14 days after creation date            |
| `modified-low.pdf`        | `modified` | `true`         | `25`         | Minor modification — one incremental update detected     |
| `modified-medium.pdf`     | `modified` | `true`         | `50`         | Moderate modification — creator/producer mismatch        |
| `multiple-xref.pdf`       | `modified` | `true`         | `55`         | 4 cross-reference tables detected                        |
| `incremental-updates.pdf` | `modified` | `true`         | `60`         | 6 incremental update sections                            |
| `embedded-files.pdf`      | `modified` | `true`         | `65`         | Embedded file attachments added after creation           |
| `javascript.pdf`          | `modified` | `true`         | `70`         | JavaScript code embedded in PDF                          |
| `modified-high.pdf`       | `modified` | `true`         | `75`         | Significant modification — multiple updates, tool change |

### Critical Tampering

Use these to test how your application handles severe tampering alerts.

| Filename                  | `status`   | `been_changed` | `risk_score` | Notes                                                               |
| ------------------------- | ---------- | -------------- | ------------ | ------------------------------------------------------------------- |
| `modified-after-sign.pdf` | `modified` | `true`         | `85`         | Modified after digital signing — signature invalidated              |
| `signature-removed.pdf`   | `modified` | `true`         | `90`         | Digital signature was removed from the document                     |
| `modified-critical.pdf`   | `modified` | `true`         | `95`         | Signature removed + JavaScript detected + 8-step modification chain |
| `both-threats.pdf`        | `modified` | `true`         | `95`         | JavaScript + embedded files + signature removed — maximum severity  |

---

## What to Test

### 1. Basic request/response flow

Verify that your integration correctly parses the response structure.

```bash
curl -s -X POST https://htpbe.tech/api/v1/analyze \
  -H "Authorization: Bearer htpbe_test_YOUR_TEST_KEY" \
  -H "Content-Type: application/json" \
  -d '{"url": "https://htpbe.tech/api/v1/test/clean.pdf"}' | jq .
```

**What to verify:**

- Response is valid JSON with `id` (UUID string) and `analysis` object
- `analysis.status` is `"intact"`
- `analysis.been_changed` is `false`
- `analysis.risk_score` is `0`
- `analysis.findings` is an empty array `[]`
- `analysis.metadata.creation_date` is non-null
- `analysis.origin.type` is `"institutional"`

---

### 2. Three-state verdict handling

The API returns three possible `status` values. Verify your application handles all three:

```bash
# intact
curl -s -X POST https://htpbe.tech/api/v1/analyze \
  -H "Authorization: Bearer htpbe_test_YOUR_TEST_KEY" \
  -H "Content-Type: application/json" \
  -d '{"url": "https://htpbe.tech/api/v1/test/clean.pdf"}' \
  | jq '.analysis.status'
# → "intact"

# modified
curl -s -X POST https://htpbe.tech/api/v1/analyze \
  -H "Authorization: Bearer htpbe_test_YOUR_TEST_KEY" \
  -H "Content-Type: application/json" \
  -d '{"url": "https://htpbe.tech/api/v1/test/modified-high.pdf"}' \
  | jq '.analysis.status'
# → "modified"

# inconclusive
curl -s -X POST https://htpbe.tech/api/v1/analyze \
  -H "Authorization: Bearer htpbe_test_YOUR_TEST_KEY" \
  -H "Content-Type: application/json" \
  -d '{"url": "https://htpbe.tech/api/v1/test/inconclusive.pdf"}' \
  | jq '.analysis.status, .analysis.status_reason, .analysis.origin'
# → "inconclusive"
# → "consumer_software_origin"
# → { "type": "consumer_software", "software": "Microsoft Excel", "warning": "..." }
```

---

### 3. Risk score thresholds

Test how your code handles each risk bracket:

| Score range      | Test URL                                       |
| ---------------- | ---------------------------------------------- |
| 0 (none)         | `clean.pdf`                                    |
| 25 (low)         | `modified-low.pdf`                             |
| 40–55 (medium)   | `dates-mismatch.pdf`, `multiple-xref.pdf`      |
| 60–75 (high)     | `incremental-updates.pdf`, `modified-high.pdf` |
| 85–95 (critical) | `signature-removed.pdf`, `both-threats.pdf`    |

---

### 4. Signature scenarios

```bash
# Valid signature — should not trigger alerts
curl -s -X POST https://htpbe.tech/api/v1/analyze \
  -H "Authorization: Bearer htpbe_test_YOUR_TEST_KEY" \
  -H "Content-Type: application/json" \
  -d '{"url": "https://htpbe.tech/api/v1/test/signature-valid.pdf"}' \
  | jq '.analysis.signatures'
```

Expected:

```json
{
  "has_digital_signature": true,
  "signature_count": 1,
  "signature_removed": false,
  "modifications_after_signature": false
}
```

```bash
# Removed signature — critical alert
curl -s -X POST https://htpbe.tech/api/v1/analyze \
  -H "Authorization: Bearer htpbe_test_YOUR_TEST_KEY" \
  -H "Content-Type: application/json" \
  -d '{"url": "https://htpbe.tech/api/v1/test/signature-removed.pdf"}' \
  | jq '.analysis.signatures'
```

Expected:

```json
{
  "has_digital_signature": false,
  "signature_count": 0,
  "signature_removed": true,
  "modifications_after_signature": false
}
```

---

### 5. Security threats

Test that your UI handles JavaScript and embedded file warnings:

```bash
curl -s -X POST https://htpbe.tech/api/v1/analyze \
  -H "Authorization: Bearer htpbe_test_YOUR_TEST_KEY" \
  -H "Content-Type: application/json" \
  -d '{"url": "https://htpbe.tech/api/v1/test/both-threats.pdf"}' \
  | jq '.analysis.threats, .analysis.findings'
```

Expected:

```json
{
  "has_javascript": true,
  "has_embedded_files": true
}
[
  "CRITICAL: JavaScript AND embedded files detected",
  "Digital signature removed",
  "Suspicious editing tool",
  "High risk of malicious content",
  "Multiple security threats present"
]
```

---

### 6. Null metadata handling

Not all PDFs contain full metadata. Verify your application handles null values gracefully:

```bash
curl -s -X POST https://htpbe.tech/api/v1/analyze \
  -H "Authorization: Bearer htpbe_test_YOUR_TEST_KEY" \
  -H "Content-Type: application/json" \
  -d '{"url": "https://htpbe.tech/api/v1/test/clean-no-dates.pdf"}' \
  | jq '.analysis.metadata'
```

Expected:

```json
{
  "creation_date": null,
  "modification_date": null,
  "creator": "Unknown",
  "producer": "Unknown",
  "file_size": 102400
}
```

---

### 7. Error handling

Verify that your integration correctly handles the error returned when a test key is used with a non-test URL.

```bash
curl -s -X POST https://htpbe.tech/api/v1/analyze \
  -H "Authorization: Bearer htpbe_test_YOUR_TEST_KEY" \
  -H "Content-Type: application/json" \
  -d '{"url": "https://example.com/invoice.pdf"}'
```

Expected HTTP status: `403`

```json
{
  "error": "Test API keys can only be used with test URLs. See documentation for available test URLs.",
  "details": "Use URLs like: https://htpbe.tech/api/v1/test/clean.pdf, https://htpbe.tech/api/v1/test/modified-high.pdf, etc."
}
```

---

## Code Examples

### Node.js / TypeScript

```typescript
const BASE_URL = 'https://htpbe.tech/api/v1';
const TEST_KEY = process.env.HTPBE_TEST_KEY; // htpbe_test_...

async function analyzeTestDocument(filename: string) {
  const response = await fetch(`${BASE_URL}/analyze`, {
    method: 'POST',
    headers: {
      Authorization: `Bearer ${TEST_KEY}`,
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({
      url: `${BASE_URL}/test/${filename}`,
    }),
  });

  if (!response.ok) {
    const error = await response.json();
    throw new Error(`API error ${response.status}: ${error.error}`);
  }

  return response.json();
}

// Test all three verdict branches
const original = await analyzeTestDocument('clean.pdf');
console.assert(original.analysis.status === 'intact');
console.assert(original.analysis.been_changed === false);

const tampered = await analyzeTestDocument('signature-removed.pdf');
console.assert(tampered.analysis.status === 'modified');
console.assert(tampered.analysis.been_changed === true);
console.assert(tampered.analysis.signatures.signature_removed === true);

const inconclusive = await analyzeTestDocument('inconclusive.pdf');
console.assert(inconclusive.analysis.status === 'inconclusive');
console.assert(inconclusive.analysis.status_reason === 'consumer_software_origin');
console.assert(inconclusive.analysis.origin.type === 'consumer_software');
console.assert(inconclusive.analysis.been_changed === false);
```

### Python

```python
import os
import requests

BASE_URL = "https://htpbe.tech/api/v1"
TEST_KEY = os.getenv("HTPBE_TEST_KEY")  # htpbe_test_...

def analyze_test_document(filename: str) -> dict:
    response = requests.post(
        f"{BASE_URL}/analyze",
        headers={
            "Authorization": f"Bearer {TEST_KEY}",
            "Content-Type": "application/json",
        },
        json={"url": f"{BASE_URL}/test/{filename}"},
    )
    response.raise_for_status()
    return response.json()

# Test intact document
result = analyze_test_document("clean.pdf")
assert result["analysis"]["status"] == "intact"
assert result["analysis"]["been_changed"] is False
assert result["analysis"]["risk_score"] == 0

# Test critical tampering
result = analyze_test_document("both-threats.pdf")
assert result["analysis"]["status"] == "modified"
assert result["analysis"]["been_changed"] is True
assert result["analysis"]["risk_score"] == 95
assert result["analysis"]["threats"]["has_javascript"] is True

# Test inconclusive (consumer software)
result = analyze_test_document("inconclusive.pdf")
assert result["analysis"]["status"] == "inconclusive"
assert result["analysis"]["status_reason"] == "consumer_software_origin"
assert result["analysis"]["origin"]["software"] == "Microsoft Excel"
```

---

## Common Mistakes

### Using a real PDF URL with a test key

```json
{
  "error": "Test API keys can only be used with test URLs. See documentation for available test URLs."
}
```

**Fix:** Use one of the 17 URLs listed in the [Available Test URLs](#available-test-urls) table, or switch to your live key.

---

### Trying to retrieve a test result via `/result/{id}`

Test responses are not saved to the database. The `id` in the response is ephemeral — calling `GET /api/v1/result/{id}` with it will return 404.

**Fix:** Use your live key for workflows that require result retrieval. Test keys are only for validating your request/response handling.

---

### Using a test URL with a live key

Live keys download and analyze the URL as a real file. Passing `https://htpbe.tech/api/v1/test/clean.pdf` with a live key will attempt to download a file that does not exist and return a 400 download error.

**Fix:** Test URLs only work with test keys. Keep the two environments separate.

---

### Expecting `verdict_reasoning` to always be present

The `verdict_reasoning` field is `undefined` (omitted) when no single dominant reason was identified. Several test fixtures omit it:

```bash
curl -s -X POST https://htpbe.tech/api/v1/analyze \
  -H "Authorization: Bearer htpbe_test_YOUR_TEST_KEY" \
  -H "Content-Type: application/json" \
  -d '{"url": "https://htpbe.tech/api/v1/test/clean.pdf"}' \
  | jq 'has("verdict_reasoning")'
```

**Fix:** Always use optional chaining or null checks: `result.analysis.verdict_reasoning ?? ""`.

---

### Using `been_changed` as the primary verdict field

`been_changed` is auxiliary and kept for backward compatibility. For `inconclusive` results, `been_changed` is `false` — the same as an intact document. Use `status` to distinguish these cases:

```typescript
// Wrong — misses inconclusive
if (result.analysis.been_changed) {
  showModifiedAlert();
} else {
  showOriginalBadge(); // incorrectly shown for consumer-software PDFs
}

// Correct
switch (result.analysis.status) {
  case 'intact':
    showOriginalBadge();
    break;
  case 'modified':
    showModifiedAlert();
    break;
  case 'inconclusive':
    showCannotVerifyBanner(result.analysis.origin.warning);
    break;
}
```

---

## Checklist Before Going Live

Use this list before switching from test to live keys:

- [ ] Application handles `status: "intact"` (original document)
- [ ] Application handles `status: "modified"` at low, medium, and high risk
- [ ] Application handles `status: "inconclusive"` with `origin.warning` displayed to the user
- [ ] UI displays `findings` array items clearly
- [ ] Code handles `null` values in `metadata.creation_date` and `metadata.modification_date`
- [ ] Code handles missing `verdict_reasoning` field
- [ ] `signatures.signature_removed: true` triggers a visible alert
- [ ] `threats.has_javascript: true` triggers a visible warning
- [ ] 403 errors are surfaced to the user (not silently swallowed)
- [ ] API key is stored in environment variables, not hardcoded

---

## Full Response Reference

Every test URL returns a deterministic `analysis` object. The `id` field is always a fresh UUID generated per request and is not saved to the database.

---

### `clean.pdf`

Original document, no modifications.

```json
{
  "status": "intact",
  "been_changed": false,
  "risk_score": 0,
  "confidence_level": "low",
  "origin": { "type": "institutional", "software": null, "warning": null },
  "metadata": {
    "creation_date": "2024-01-15T10:00:00.000Z",
    "modification_date": null,
    "creator": "Adobe Acrobat Pro DC",
    "producer": "Adobe PDF Library 15.0",
    "file_size": 524288
  },
  "structure": {
    "has_incremental_updates": false,
    "update_chain_length": 1,
    "xref_count": 1,
    "pdf_version": "1.7"
  },
  "signatures": {
    "has_digital_signature": false,
    "signature_count": 0,
    "signature_removed": false,
    "modifications_after_signature": false
  },
  "threats": { "has_javascript": false, "has_embedded_files": false },
  "findings": []
}
```

---

### `clean-no-dates.pdf`

Original document, metadata dates absent (common in auto-generated PDFs).

```json
{
  "status": "intact",
  "been_changed": false,
  "risk_score": 0,
  "confidence_level": "low",
  "origin": { "type": "unknown", "software": null, "warning": null },
  "metadata": {
    "creation_date": null,
    "modification_date": null,
    "creator": "Unknown",
    "producer": "Unknown",
    "file_size": 102400
  },
  "structure": {
    "has_incremental_updates": false,
    "update_chain_length": 1,
    "xref_count": 1,
    "pdf_version": "1.4"
  },
  "signatures": {
    "has_digital_signature": false,
    "signature_count": 0,
    "signature_removed": false,
    "modifications_after_signature": false
  },
  "threats": { "has_javascript": false, "has_embedded_files": false },
  "findings": []
}
```

---

### `dates-same.pdf`

LibreOffice origin — integrity check not applicable.

```json
{
  "status": "inconclusive",
  "status_reason": "consumer_software_origin",
  "been_changed": false,
  "risk_score": 0,
  "confidence_level": "low",
  "origin": {
    "type": "consumer_software",
    "software": "LibreOffice",
    "warning": "This document was created with consumer software (LibreOffice). Official institutional documents (bank statements, government certificates, pay stubs) are typically generated by dedicated systems. This check cannot verify the authenticity of documents created with office software."
  },
  "metadata": {
    "creation_date": "2024-03-01T12:00:00.000Z",
    "modification_date": "2024-03-01T12:00:00.000Z",
    "creator": "LibreOffice 7.0",
    "producer": "LibreOffice 7.0",
    "file_size": 245760
  },
  "structure": {
    "has_incremental_updates": false,
    "update_chain_length": 1,
    "xref_count": 1,
    "pdf_version": "1.4"
  },
  "signatures": {
    "has_digital_signature": false,
    "signature_count": 0,
    "signature_removed": false,
    "modifications_after_signature": false
  },
  "threats": { "has_javascript": false, "has_embedded_files": false },
  "findings": []
}
```

---

### `inconclusive.pdf`

Microsoft Excel origin — integrity check not applicable.

```json
{
  "status": "inconclusive",
  "status_reason": "consumer_software_origin",
  "been_changed": false,
  "risk_score": 0,
  "confidence_level": "low",
  "origin": {
    "type": "consumer_software",
    "software": "Microsoft Excel",
    "warning": "This document was created with consumer software (Microsoft Excel). Official institutional documents (bank statements, government certificates, pay stubs) are typically generated by dedicated systems. This check cannot verify the authenticity of documents created with office software."
  },
  "metadata": {
    "creation_date": "2024-03-01T09:00:00.000Z",
    "modification_date": "2024-03-01T09:00:00.000Z",
    "creator": "Microsoft Excel 2019",
    "producer": "Microsoft: Print To PDF",
    "file_size": 204800
  },
  "structure": {
    "has_incremental_updates": false,
    "update_chain_length": 1,
    "xref_count": 1,
    "pdf_version": "1.7"
  },
  "signatures": {
    "has_digital_signature": false,
    "signature_count": 0,
    "signature_removed": false,
    "modifications_after_signature": false
  },
  "threats": { "has_javascript": false, "has_embedded_files": false },
  "findings": []
}
```

---

### `signature-valid.pdf`

Digitally signed, no post-sign modifications.

```json
{
  "status": "intact",
  "been_changed": false,
  "risk_score": 0,
  "confidence_level": "low",
  "origin": { "type": "institutional", "software": null, "warning": null },
  "metadata": {
    "creation_date": "2024-03-05T10:00:00.000Z",
    "modification_date": null,
    "creator": "Adobe Acrobat Pro DC",
    "producer": "Adobe PDF Library 15.0",
    "file_size": 1125000
  },
  "structure": {
    "has_incremental_updates": false,
    "update_chain_length": 1,
    "xref_count": 1,
    "pdf_version": "1.7"
  },
  "signatures": {
    "has_digital_signature": true,
    "signature_count": 1,
    "signature_removed": false,
    "modifications_after_signature": false
  },
  "threats": { "has_javascript": false, "has_embedded_files": false },
  "findings": ["Document digitally signed", "No modifications after signature"]
}
```

---

### `dates-mismatch.pdf`

Modification date 14 days after creation date.

```json
{
  "status": "modified",
  "been_changed": true,
  "risk_score": 40,
  "confidence_level": "medium",
  "origin": { "type": "institutional", "software": null, "warning": null },
  "metadata": {
    "creation_date": "2024-02-01T10:00:00.000Z",
    "modification_date": "2024-02-15T14:30:00.000Z",
    "creator": "Adobe Acrobat Pro DC",
    "producer": "Adobe PDF Library 15.0",
    "file_size": 655360
  },
  "structure": {
    "has_incremental_updates": true,
    "update_chain_length": 2,
    "xref_count": 2,
    "pdf_version": "1.7"
  },
  "signatures": {
    "has_digital_signature": false,
    "signature_count": 0,
    "signature_removed": false,
    "modifications_after_signature": false
  },
  "threats": { "has_javascript": false, "has_embedded_files": false },
  "findings": ["Modification date 14 days after creation date"]
}
```

---

### `modified-low.pdf`

Minor modification — one incremental update.

```json
{
  "status": "modified",
  "been_changed": true,
  "risk_score": 25,
  "confidence_level": "medium",
  "origin": { "type": "institutional", "software": null, "warning": null },
  "metadata": {
    "creation_date": "2024-01-15T10:00:00.000Z",
    "modification_date": "2024-01-15T11:30:00.000Z",
    "creator": "Adobe Acrobat Pro DC",
    "producer": "Adobe PDF Library 15.0",
    "file_size": 545000
  },
  "structure": {
    "has_incremental_updates": true,
    "update_chain_length": 2,
    "xref_count": 2,
    "pdf_version": "1.7"
  },
  "signatures": {
    "has_digital_signature": false,
    "signature_count": 0,
    "signature_removed": false,
    "modifications_after_signature": false
  },
  "threats": { "has_javascript": false, "has_embedded_files": false },
  "findings": ["Incremental update detected", "Modification date differs from creation date"]
}
```

---

### `modified-medium.pdf`

Moderate modification — creator/producer mismatch.

```json
{
  "status": "modified",
  "been_changed": true,
  "risk_score": 50,
  "confidence_level": "medium",
  "origin": {
    "type": "consumer_software",
    "software": "Microsoft Word",
    "warning": "This document was created with consumer software (Microsoft Word). Official institutional documents (bank statements, government certificates, pay stubs) are typically generated by dedicated systems. This check cannot verify the authenticity of documents created with office software."
  },
  "metadata": {
    "creation_date": "2024-01-10T08:00:00.000Z",
    "modification_date": "2024-02-05T14:20:00.000Z",
    "creator": "Microsoft Word",
    "producer": "Adobe PDF Library 15.0",
    "file_size": 780000
  },
  "structure": {
    "has_incremental_updates": true,
    "update_chain_length": 3,
    "xref_count": 3,
    "pdf_version": "1.7"
  },
  "signatures": {
    "has_digital_signature": false,
    "signature_count": 0,
    "signature_removed": false,
    "modifications_after_signature": false
  },
  "threats": { "has_javascript": false, "has_embedded_files": false },
  "findings": [
    "Multiple incremental updates detected",
    "Creator/Producer mismatch",
    "Significant time gap between creation and modification"
  ]
}
```

---

### `multiple-xref.pdf`

4 cross-reference tables detected.

```json
{
  "status": "modified",
  "been_changed": true,
  "risk_score": 55,
  "confidence_level": "medium",
  "origin": { "type": "institutional", "software": null, "warning": null },
  "metadata": {
    "creation_date": "2024-02-10T11:00:00.000Z",
    "modification_date": "2024-02-11T09:00:00.000Z",
    "creator": "Acrobat PDFMaker 20",
    "producer": "Adobe PDF Library 15.0",
    "file_size": 712000
  },
  "structure": {
    "has_incremental_updates": true,
    "update_chain_length": 4,
    "xref_count": 4,
    "pdf_version": "1.7"
  },
  "signatures": {
    "has_digital_signature": false,
    "signature_count": 0,
    "signature_removed": false,
    "modifications_after_signature": false
  },
  "threats": { "has_javascript": false, "has_embedded_files": false },
  "findings": ["Multiple xref tables detected (4 total)", "Incremental update chain present"]
}
```

---

### `incremental-updates.pdf`

6 incremental update sections.

```json
{
  "status": "modified",
  "been_changed": true,
  "risk_score": 60,
  "confidence_level": "high",
  "origin": { "type": "institutional", "software": null, "warning": null },
  "metadata": {
    "creation_date": "2024-01-20T09:00:00.000Z",
    "modification_date": "2024-01-20T15:00:00.000Z",
    "creator": "Adobe Acrobat Pro DC",
    "producer": "Adobe PDF Library 15.0",
    "file_size": 890000
  },
  "structure": {
    "has_incremental_updates": true,
    "update_chain_length": 6,
    "xref_count": 5,
    "pdf_version": "1.7"
  },
  "signatures": {
    "has_digital_signature": false,
    "signature_count": 0,
    "signature_removed": false,
    "modifications_after_signature": false
  },
  "threats": { "has_javascript": false, "has_embedded_files": false },
  "findings": ["Multiple incremental updates (6 chain length)", "Complex modification history"]
}
```

---

### `embedded-files.pdf`

Embedded file attachments added after creation.

```json
{
  "status": "modified",
  "been_changed": true,
  "risk_score": 65,
  "confidence_level": "medium",
  "origin": { "type": "institutional", "software": null, "warning": null },
  "metadata": {
    "creation_date": "2024-03-12T14:00:00.000Z",
    "modification_date": "2024-03-12T15:30:00.000Z",
    "creator": "Adobe Acrobat Pro DC",
    "producer": "Adobe PDF Library 15.0",
    "file_size": 2450000
  },
  "structure": {
    "has_incremental_updates": true,
    "update_chain_length": 2,
    "xref_count": 2,
    "pdf_version": "1.7"
  },
  "signatures": {
    "has_digital_signature": false,
    "signature_count": 0,
    "signature_removed": false,
    "modifications_after_signature": false
  },
  "threats": { "has_javascript": false, "has_embedded_files": true },
  "findings": ["Embedded files detected", "Files added after creation"]
}
```

---

### `javascript.pdf`

JavaScript code embedded in the PDF.

```json
{
  "status": "modified",
  "been_changed": true,
  "risk_score": 70,
  "confidence_level": "high",
  "origin": { "type": "institutional", "software": null, "warning": null },
  "metadata": {
    "creation_date": "2024-03-10T10:00:00.000Z",
    "modification_date": "2024-03-10T11:00:00.000Z",
    "creator": "Adobe Acrobat Pro DC",
    "producer": "Adobe PDF Library 15.0",
    "file_size": 456000
  },
  "structure": {
    "has_incremental_updates": true,
    "update_chain_length": 2,
    "xref_count": 2,
    "pdf_version": "1.7"
  },
  "signatures": {
    "has_digital_signature": false,
    "signature_count": 0,
    "signature_removed": false,
    "modifications_after_signature": false
  },
  "threats": { "has_javascript": true, "has_embedded_files": false },
  "findings": ["JavaScript code detected in PDF", "Potential security risk"]
}
```

---

### `modified-high.pdf`

Significant modification — multiple saves, tool changed to PDFtk.

```json
{
  "status": "modified",
  "been_changed": true,
  "risk_score": 75,
  "confidence_level": "high",
  "origin": { "type": "institutional", "software": null, "warning": null },
  "metadata": {
    "creation_date": "2024-01-01T00:00:00.000Z",
    "modification_date": "2024-03-15T18:45:00.000Z",
    "creator": "Adobe Acrobat Pro DC",
    "producer": "PDFtk Server 2.02",
    "file_size": 1048576
  },
  "structure": {
    "has_incremental_updates": true,
    "update_chain_length": 5,
    "xref_count": 4,
    "pdf_version": "1.7"
  },
  "signatures": {
    "has_digital_signature": false,
    "signature_count": 0,
    "signature_removed": false,
    "modifications_after_signature": false
  },
  "threats": { "has_javascript": false, "has_embedded_files": false },
  "findings": [
    "Extensive modification history",
    "Producer changed from Adobe to PDFtk",
    "Suspicious tool pattern detected",
    "Long modification chain"
  ]
}
```

---

### `modified-after-sign.pdf`

Modified after digital signing — signature is now invalidated.

```json
{
  "status": "modified",
  "been_changed": true,
  "risk_score": 85,
  "confidence_level": "high",
  "origin": { "type": "institutional", "software": null, "warning": null },
  "metadata": {
    "creation_date": "2024-02-01T09:00:00.000Z",
    "modification_date": "2024-02-10T14:00:00.000Z",
    "creator": "Adobe Acrobat Pro DC",
    "producer": "Adobe PDF Library 15.0",
    "file_size": 1340000
  },
  "structure": {
    "has_incremental_updates": true,
    "update_chain_length": 4,
    "xref_count": 3,
    "pdf_version": "1.7"
  },
  "signatures": {
    "has_digital_signature": true,
    "signature_count": 1,
    "signature_removed": false,
    "modifications_after_signature": true
  },
  "threats": { "has_javascript": false, "has_embedded_files": false },
  "findings": [
    "Document modified after being digitally signed",
    "Signature may be invalid",
    "Content changed post-signature"
  ]
}
```

---

### `signature-removed.pdf`

Digital signature was removed from the document.

```json
{
  "status": "modified",
  "been_changed": true,
  "risk_score": 90,
  "confidence_level": "high",
  "origin": { "type": "institutional", "software": null, "warning": null },
  "metadata": {
    "creation_date": "2024-01-10T10:00:00.000Z",
    "modification_date": "2024-02-20T16:30:00.000Z",
    "creator": "Adobe Acrobat Pro DC",
    "producer": "Adobe PDF Library 15.0",
    "file_size": 980000
  },
  "structure": {
    "has_incremental_updates": true,
    "update_chain_length": 3,
    "xref_count": 3,
    "pdf_version": "1.7"
  },
  "signatures": {
    "has_digital_signature": false,
    "signature_count": 0,
    "signature_removed": true,
    "modifications_after_signature": false
  },
  "threats": { "has_javascript": false, "has_embedded_files": false },
  "findings": ["CRITICAL: Digital signature was removed", "Document integrity compromised"]
}
```

---

### `modified-critical.pdf`

Signature removed + JavaScript detected + 8-step modification chain.

```json
{
  "status": "modified",
  "been_changed": true,
  "risk_score": 95,
  "confidence_level": "high",
  "origin": {
    "type": "consumer_software",
    "software": "Microsoft Excel",
    "warning": "This document was created with consumer software (Microsoft Excel). Official institutional documents (bank statements, government certificates, pay stubs) are typically generated by dedicated systems. This check cannot verify the authenticity of documents created with office software."
  },
  "metadata": {
    "creation_date": "2023-12-01T10:00:00.000Z",
    "modification_date": "2024-04-01T22:15:00.000Z",
    "creator": "Microsoft Excel",
    "producer": "iText 5.5.13",
    "file_size": 2097152
  },
  "structure": {
    "has_incremental_updates": true,
    "update_chain_length": 8,
    "xref_count": 6,
    "pdf_version": "1.7"
  },
  "signatures": {
    "has_digital_signature": false,
    "signature_count": 0,
    "signature_removed": true,
    "modifications_after_signature": false
  },
  "threats": { "has_javascript": true, "has_embedded_files": false },
  "findings": [
    "Critical: Digital signature removed",
    "JavaScript code detected",
    "Extensive modification chain (8 updates)",
    "Creator/Producer severe mismatch",
    "Suspicious editing tool (iText)",
    "Multiple months between creation and modification"
  ]
}
```

---

### `both-threats.pdf`

JavaScript + embedded files + signature removed. Maximum severity.

```json
{
  "status": "modified",
  "been_changed": true,
  "risk_score": 95,
  "confidence_level": "high",
  "origin": { "type": "institutional", "software": null, "warning": null },
  "metadata": {
    "creation_date": "2024-02-15T10:00:00.000Z",
    "modification_date": "2024-03-01T16:00:00.000Z",
    "creator": "Unknown",
    "producer": "iText 5.5.13",
    "file_size": 3100000
  },
  "structure": {
    "has_incremental_updates": true,
    "update_chain_length": 7,
    "xref_count": 5,
    "pdf_version": "1.7"
  },
  "signatures": {
    "has_digital_signature": false,
    "signature_count": 0,
    "signature_removed": true,
    "modifications_after_signature": false
  },
  "threats": { "has_javascript": true, "has_embedded_files": true },
  "findings": [
    "CRITICAL: JavaScript AND embedded files detected",
    "Digital signature removed",
    "Suspicious editing tool",
    "High risk of malicious content",
    "Multiple security threats present"
  ]
}
```

---

## Related

- [POST /api/v1/analyze](./api/analyze.md) — Full request/response reference
- [GET /api/v1/result/{uid}](./api/result.md) — Retrieve saved results (live key only)
- [GET /api/v1/checks](./api/checks.md) — List all checks with filtering

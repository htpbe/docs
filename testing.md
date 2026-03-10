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

### Results use deterministic synthetic IDs

`POST /analyze` with a test key returns a deterministic `id` based on the test filename — not a random UUID:

| Test URL                         | Returned `id`            |
| -------------------------------- | ------------------------ |
| `.../test/clean.pdf`             | `TEST_CLEAN`             |
| `.../test/modified-high.pdf`     | `TEST_MODIFIED_HIGH`     |
| `.../test/signature-removed.pdf` | `TEST_SIGNATURE_REMOVED` |
| `.../test/dates-same.pdf`        | `TEST_DATES_SAME`        |

Because the ID is deterministic, you can hardcode it in your tests without calling `POST /analyze` first.

### Full two-step flow works with test keys

`GET /api/v1/result/{id}` works with synthetic IDs — **but only when called with a test key**. Passing a `TEST_*` ID with a live key returns 404. This lets you test the complete two-step flow:

1. `POST /analyze` → `{ "id": "TEST_MODIFIED_HIGH" }`
2. `GET /result/TEST_MODIFIED_HIGH` → full synthetic result

### Results are NOT saved to the database

Synthetic results are generated on the fly — no DB writes occur. The `TEST_*` IDs are recognized by the server and resolved to mock data without any persistence.

### Responses are deterministic

The same test URL always returns exactly the same result, regardless of how many times you call it or which account you use. This makes it easy to write reliable assertions in automated tests.

---

## Making Your First Test Request

**Step 1: Submit for analysis**

```bash
curl -X POST https://htpbe.tech/api/v1/analyze \
  -H "Authorization: Bearer htpbe_test_YOUR_TEST_KEY" \
  -H "Content-Type: application/json" \
  -d '{"url": "https://htpbe.tech/api/v1/test/modified-high.pdf"}'
```

Response:

```json
{
  "id": "TEST_MODIFIED_HIGH"
}
```

**Step 2: Retrieve the full result**

```bash
curl https://htpbe.tech/api/v1/result/TEST_MODIFIED_HIGH \
  -H "Authorization: Bearer htpbe_test_YOUR_TEST_KEY"
```

Response (excerpt):

```json
{
  "id": "TEST_MODIFIED_HIGH",
  "status": "modified",
  "modification_confidence": "certain",
  "critical_modification_marker": "Known PDF editing tool detected",
  "verdict_reasoning": "Known PDF editing tool detected (PDFtk Server)",
  "origin": { "type": "institutional", "software": null, "warning": null },
  "creation_date": 1704067200,
  "modification_date": 1710524700,
  "creator": "Adobe Acrobat Pro DC",
  "producer": "PDFtk Server 2.02",
  "has_incremental_updates": true,
  "update_chain_length": 5,
  "xref_count": 4,
  "pdf_version": "1.7",
  "has_digital_signature": false,
  "signature_removed": false,
  "has_javascript": false,
  "has_embedded_files": false,
  "specific_findings": [
    "Extensive modification history",
    "Producer changed from Adobe to PDFtk",
    "Suspicious tool pattern detected",
    "Long modification chain"
  ]
}
```

Since IDs are deterministic, you can skip Step 1 and call `GET /result/TEST_MODIFIED_HIGH` directly in your tests.

---

## Available Test URLs

All 17 test URLs live at `https://htpbe.tech/api/v1/test/`. Any other URL (including files you host yourself) will be rejected when using a test key.

### Clean / Original

Use these to test how your application handles documents that pass verification.

| URL                                                  | `status` | Notes                                                               |
| ---------------------------------------------------- | -------- | ------------------------------------------------------------------- |
| `https://htpbe.tech/api/v1/test/clean.pdf`           | `intact` | Typical original document with full metadata                        |
| `https://htpbe.tech/api/v1/test/clean-no-dates.pdf`  | `intact` | Original document — metadata dates absent (e.g. auto-generated PDF) |
| `https://htpbe.tech/api/v1/test/signature-valid.pdf` | `intact` | Digitally signed, no post-sign modifications                        |

### Inconclusive (Consumer Software Origin)

Use these to test how your application handles the case where integrity check is not applicable — PDFs created with office or consumer software cannot be verified.

| URL                                               | `status`       | Notes                                                   |
| ------------------------------------------------- | -------------- | ------------------------------------------------------- |
| `https://htpbe.tech/api/v1/test/dates-same.pdf`   | `inconclusive` | LibreOffice origin — integrity check not applicable     |
| `https://htpbe.tech/api/v1/test/inconclusive.pdf` | `inconclusive` | Microsoft Excel origin — integrity check not applicable |

### Modified — Graduated Severity Levels

Use these to test conditional logic and UI states for different levels of tampering.

| URL                                                      | `status`   | Notes                                                    |
| -------------------------------------------------------- | ---------- | -------------------------------------------------------- |
| `https://htpbe.tech/api/v1/test/dates-mismatch.pdf`      | `modified` | Modification date 14 days after creation date            |
| `https://htpbe.tech/api/v1/test/modified-low.pdf`        | `modified` | Minor modification — one incremental update detected     |
| `https://htpbe.tech/api/v1/test/modified-medium.pdf`     | `modified` | Moderate modification — creator/producer mismatch        |
| `https://htpbe.tech/api/v1/test/multiple-xref.pdf`       | `modified` | 4 cross-reference tables detected                        |
| `https://htpbe.tech/api/v1/test/incremental-updates.pdf` | `modified` | 6 incremental update sections                            |
| `https://htpbe.tech/api/v1/test/embedded-files.pdf`      | `modified` | Embedded file attachments added after creation           |
| `https://htpbe.tech/api/v1/test/javascript.pdf`          | `modified` | JavaScript code embedded in PDF                          |
| `https://htpbe.tech/api/v1/test/modified-high.pdf`       | `modified` | Significant modification — multiple updates, tool change |

### Critical Tampering

Use these to test how your application handles severe tampering alerts.

| URL                                                      | `status`   | Notes                                                               |
| -------------------------------------------------------- | ---------- | ------------------------------------------------------------------- |
| `https://htpbe.tech/api/v1/test/modified-after-sign.pdf` | `modified` | Modified after digital signing — signature invalidated              |
| `https://htpbe.tech/api/v1/test/signature-removed.pdf`   | `modified` | Digital signature was removed from the document                     |
| `https://htpbe.tech/api/v1/test/modified-critical.pdf`   | `modified` | Signature removed + JavaScript detected + 8-step modification chain |
| `https://htpbe.tech/api/v1/test/both-threats.pdf`        | `modified` | JavaScript + embedded files + signature removed — maximum severity  |

---

## What to Test

### 1. Basic request/response flow

Verify that your integration correctly handles the two-step flow.

```bash
# Step 1: Submit for analysis — returns a synthetic ID
curl -s -X POST https://htpbe.tech/api/v1/analyze \
  -H "Authorization: Bearer htpbe_test_YOUR_TEST_KEY" \
  -H "Content-Type: application/json" \
  -d '{"url": "https://htpbe.tech/api/v1/test/clean.pdf"}'
# → { "id": "TEST_CLEAN" }

# Step 2: Retrieve the full result using the synthetic ID
curl -s https://htpbe.tech/api/v1/result/TEST_CLEAN \
  -H "Authorization: Bearer htpbe_test_YOUR_TEST_KEY" | jq .
```

**What to verify:**

- `POST /analyze` returns `{ "id": "TEST_CLEAN" }` — no analysis data
- `status` is `"intact"`
- `specific_findings` is an empty array `[]`
- `creation_date` is non-null
- `origin.type` is `"institutional"`

---

### 2. Three-state verdict handling

The API returns three possible `status` values. Since IDs are deterministic, you can call `GET /result/{id}` directly without submitting via `POST /analyze` first:

```bash
# intact
curl -s https://htpbe.tech/api/v1/result/TEST_CLEAN \
  -H "Authorization: Bearer htpbe_test_YOUR_TEST_KEY" \
  | jq '.status'
# → "intact"

# modified
curl -s https://htpbe.tech/api/v1/result/TEST_MODIFIED_HIGH \
  -H "Authorization: Bearer htpbe_test_YOUR_TEST_KEY" \
  | jq '.status'
# → "modified"

# inconclusive
curl -s https://htpbe.tech/api/v1/result/TEST_INCONCLUSIVE \
  -H "Authorization: Bearer htpbe_test_YOUR_TEST_KEY" \
  | jq '.status, .status_reason, .origin'
# → "inconclusive"
# → "consumer_software_origin"
# → { "type": "consumer_software", "software": "Microsoft Excel", "warning": "..." }
```

---

### 3. Tampering severity levels

Test how your code handles each severity level:

| Severity | Test URL                                                                                                     |
| -------- | ------------------------------------------------------------------------------------------------------------ |
| None     | `https://htpbe.tech/api/v1/test/clean.pdf`                                                                   |
| Minor    | `https://htpbe.tech/api/v1/test/modified-low.pdf`                                                            |
| Moderate | `https://htpbe.tech/api/v1/test/dates-mismatch.pdf`, `https://htpbe.tech/api/v1/test/multiple-xref.pdf`      |
| High     | `https://htpbe.tech/api/v1/test/incremental-updates.pdf`, `https://htpbe.tech/api/v1/test/modified-high.pdf` |
| Critical | `https://htpbe.tech/api/v1/test/signature-removed.pdf`, `https://htpbe.tech/api/v1/test/both-threats.pdf`    |

---

### 4. Signature scenarios

```bash
# Valid signature — should not trigger alerts
curl -s https://htpbe.tech/api/v1/result/TEST_SIGNATURE_VALID \
  -H "Authorization: Bearer htpbe_test_YOUR_TEST_KEY" \
  | jq '{has_digital_signature, signature_count, signature_removed, modifications_after_signature}'
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
curl -s https://htpbe.tech/api/v1/result/TEST_SIGNATURE_REMOVED \
  -H "Authorization: Bearer htpbe_test_YOUR_TEST_KEY" \
  | jq '{has_digital_signature, signature_count, signature_removed, modifications_after_signature}'
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
curl -s https://htpbe.tech/api/v1/result/TEST_BOTH_THREATS \
  -H "Authorization: Bearer htpbe_test_YOUR_TEST_KEY" \
  | jq '{has_javascript, has_embedded_files, specific_findings}'
```

Expected:

```json
{
  "has_javascript": true,
  "has_embedded_files": true,
  "specific_findings": [
    "CRITICAL: JavaScript AND embedded files detected",
    "Digital signature removed",
    "Suspicious editing tool",
    "High risk of malicious content",
    "Multiple security threats present"
  ]
}
```

---

### 6. Null metadata handling

Not all PDFs contain full metadata. Verify your application handles null values gracefully:

```bash
curl -s https://htpbe.tech/api/v1/result/TEST_CLEAN_NO_DATES \
  -H "Authorization: Bearer htpbe_test_YOUR_TEST_KEY" \
  | jq '{creation_date, modification_date, creator, producer}'
```

Expected:

```json
{
  "creation_date": null,
  "modification_date": null,
  "creator": "Unknown",
  "producer": "Unknown"
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

async function analyzeAndGetResult(filename: string) {
  // Step 1: Submit for analysis
  const analyzeRes = await fetch(`${BASE_URL}/analyze`, {
    method: 'POST',
    headers: {
      Authorization: `Bearer ${TEST_KEY}`,
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({ url: `${BASE_URL}/test/${filename}` }),
  });

  if (!analyzeRes.ok) {
    const error = await analyzeRes.json();
    throw new Error(`Analyze error ${analyzeRes.status}: ${error.error}`);
  }

  const { id } = await analyzeRes.json(); // e.g. "TEST_CLEAN"

  // Step 2: Retrieve the full result
  const resultRes = await fetch(`${BASE_URL}/result/${id}`, {
    headers: { Authorization: `Bearer ${TEST_KEY}` },
  });

  if (!resultRes.ok) {
    const error = await resultRes.json();
    throw new Error(`Result error ${resultRes.status}: ${error.error}`);
  }

  return resultRes.json();
}

// Test all three verdict branches
const original = await analyzeAndGetResult('clean.pdf');
console.assert(original.status === 'intact');

const tampered = await analyzeAndGetResult('signature-removed.pdf');
console.assert(tampered.status === 'modified');
console.assert(tampered.signature_removed === true);

const inconclusive = await analyzeAndGetResult('inconclusive.pdf');
console.assert(inconclusive.status === 'inconclusive');
console.assert(inconclusive.status_reason === 'consumer_software_origin');
console.assert(inconclusive.origin.type === 'consumer_software');

// Since IDs are deterministic you can skip POST and call GET directly:
const result = await fetch(`${BASE_URL}/result/TEST_CLEAN`, {
  headers: { Authorization: `Bearer ${TEST_KEY}` },
}).then((r) => r.json());
console.assert(result.status === 'intact');
```

### Python

```python
import os
import requests

BASE_URL = "https://htpbe.tech/api/v1"
TEST_KEY = os.getenv("HTPBE_TEST_KEY")  # htpbe_test_...

def analyze_and_get_result(filename: str) -> dict:
    # Step 1: Submit for analysis
    analyze_res = requests.post(
        f"{BASE_URL}/analyze",
        headers={
            "Authorization": f"Bearer {TEST_KEY}",
            "Content-Type": "application/json",
        },
        json={"url": f"{BASE_URL}/test/{filename}"},
    )
    analyze_res.raise_for_status()
    check_id = analyze_res.json()["id"]  # e.g. "TEST_CLEAN"

    # Step 2: Retrieve the full result
    result_res = requests.get(
        f"{BASE_URL}/result/{check_id}",
        headers={"Authorization": f"Bearer {TEST_KEY}"},
    )
    result_res.raise_for_status()
    return result_res.json()

# Test intact document
result = analyze_and_get_result("clean.pdf")
assert result["status"] == "intact"

# Test critical tampering
result = analyze_and_get_result("both-threats.pdf")
assert result["status"] == "modified"
assert result["has_javascript"] is True

# Test inconclusive (consumer software)
result = analyze_and_get_result("inconclusive.pdf")
assert result["status"] == "inconclusive"
assert result["status_reason"] == "consumer_software_origin"
assert result["origin"]["software"] == "Microsoft Excel"
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

### Passing a `TEST_*` ID with a live key

Synthetic check IDs (`TEST_*`) are only accessible with a test key. Calling `GET /api/v1/result/TEST_CLEAN` with a live key returns 404:

```json
{ "error": "Check not found or access denied" }
```

**Fix:** Use your test key when retrieving synthetic test results. Keep test and live keys in separate environments.

---

### Using a test URL with a live key

Live keys download and analyze the URL as a real file. Passing `https://htpbe.tech/api/v1/test/clean.pdf` with a live key will attempt to download a file that does not exist and return a 400 download error.

**Fix:** Test URLs only work with test keys. Keep the two environments separate.

---

### Expecting `verdict_reasoning` to always be present

The `verdict_reasoning` field is `undefined` (omitted) when no single dominant reason was identified. Several test fixtures omit it:

```bash
curl -s https://htpbe.tech/api/v1/result/TEST_CLEAN \
  -H "Authorization: Bearer htpbe_test_YOUR_TEST_KEY" \
  | jq 'has("verdict_reasoning")'
```

**Fix:** Always use optional chaining or null checks: `result.verdict_reasoning ?? ""`.

---

---

## Checklist Before Going Live

Use this list before switching from test to live keys:

- [ ] Application handles `status: "intact"` (original document)
- [ ] Application handles `status: "modified"` at low, medium, and high risk
- [ ] Application handles `status: "inconclusive"` with `origin.warning` displayed to the user
- [ ] UI displays `specific_findings` array items clearly
- [ ] Code handles `null` values in `creation_date` and `modification_date`
- [ ] Code handles missing `verdict_reasoning` field
- [ ] `signature_removed: true` triggers a visible alert
- [ ] `has_javascript: true` triggers a visible warning
- [ ] 403 errors are surfaced to the user (not silently swallowed)
- [ ] API key is stored in environment variables, not hardcoded

---

## Full Response Reference

Every test URL returns a deterministic result accessible via `GET /api/v1/result/{id}`. Results are not saved to the database — synthetic results are generated on the fly and are accessible at any time with a test key.

The field values below show what each fixture returns. Fields are grouped by category for readability; in the actual `GET /result/{id}` response, all fields are at the top level — see [GET /result](./api/result.md) for the complete flat schema. Date fields (`creation_date`, `modification_date`) are Unix timestamps (seconds since epoch).

---

### `clean.pdf`

**URL:** `https://htpbe.tech/api/v1/test/clean.pdf`

Original document, no modifications.

```json
{
  "status": "intact",

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
  "specific_findings": []
}
```

---

### `clean-no-dates.pdf`

**URL:** `https://htpbe.tech/api/v1/test/clean-no-dates.pdf`

Original document, metadata dates absent (common in auto-generated PDFs).

```json
{
  "status": "intact",

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
  "specific_findings": []
}
```

---

### `dates-same.pdf`

**URL:** `https://htpbe.tech/api/v1/test/dates-same.pdf`

LibreOffice origin — integrity check not applicable.

```json
{
  "status": "inconclusive",
  "status_reason": "consumer_software_origin",

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
  "specific_findings": []
}
```

---

### `inconclusive.pdf`

**URL:** `https://htpbe.tech/api/v1/test/inconclusive.pdf`

Microsoft Excel origin — integrity check not applicable.

```json
{
  "status": "inconclusive",
  "status_reason": "consumer_software_origin",

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
  "specific_findings": []
}
```

---

### `signature-valid.pdf`

**URL:** `https://htpbe.tech/api/v1/test/signature-valid.pdf`

Digitally signed, no post-sign modifications.

```json
{
  "status": "intact",

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
  "specific_findings": ["Document digitally signed", "No modifications after signature"]
}
```

---

### `dates-mismatch.pdf`

**URL:** `https://htpbe.tech/api/v1/test/dates-mismatch.pdf`

Modification date 14 days after creation date.

```json
{
  "status": "modified",

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
  "specific_findings": ["Modification date 14 days after creation date"]
}
```

---

### `modified-low.pdf`

**URL:** `https://htpbe.tech/api/v1/test/modified-low.pdf`

Minor modification — one incremental update.

```json
{
  "status": "modified",

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
  "specific_findings": [
    "Incremental update detected",
    "Modification date differs from creation date"
  ]
}
```

---

### `modified-medium.pdf`

**URL:** `https://htpbe.tech/api/v1/test/modified-medium.pdf`

Moderate modification — creator/producer mismatch.

```json
{
  "status": "modified",

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
  "specific_findings": [
    "Multiple incremental updates detected",
    "Creator/Producer mismatch",
    "Significant time gap between creation and modification"
  ]
}
```

---

### `multiple-xref.pdf`

**URL:** `https://htpbe.tech/api/v1/test/multiple-xref.pdf`

4 cross-reference tables detected.

```json
{
  "status": "modified",

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
  "specific_findings": [
    "Multiple xref tables detected (4 total)",
    "Incremental update chain present"
  ]
}
```

---

### `incremental-updates.pdf`

**URL:** `https://htpbe.tech/api/v1/test/incremental-updates.pdf`

6 incremental update sections.

```json
{
  "status": "modified",

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
  "specific_findings": [
    "Multiple incremental updates (6 chain length)",
    "Complex modification history"
  ]
}
```

---

### `embedded-files.pdf`

**URL:** `https://htpbe.tech/api/v1/test/embedded-files.pdf`

Embedded file attachments added after creation.

```json
{
  "status": "modified",

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
  "specific_findings": ["Embedded files detected", "Files added after creation"]
}
```

---

### `javascript.pdf`

**URL:** `https://htpbe.tech/api/v1/test/javascript.pdf`

JavaScript code embedded in the PDF.

```json
{
  "status": "modified",

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
  "specific_findings": ["JavaScript code detected in PDF", "Potential security risk"]
}
```

---

### `modified-high.pdf`

**URL:** `https://htpbe.tech/api/v1/test/modified-high.pdf`

Significant modification — multiple saves, tool changed to PDFtk.

```json
{
  "status": "modified",

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
  "specific_findings": [
    "Extensive modification history",
    "Producer changed from Adobe to PDFtk",
    "Suspicious tool pattern detected",
    "Long modification chain"
  ]
}
```

---

### `modified-after-sign.pdf`

**URL:** `https://htpbe.tech/api/v1/test/modified-after-sign.pdf`

Modified after digital signing — signature is now invalidated.

```json
{
  "status": "modified",

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
  "specific_findings": [
    "Document modified after being digitally signed",
    "Signature may be invalid",
    "Content changed post-signature"
  ]
}
```

---

### `signature-removed.pdf`

**URL:** `https://htpbe.tech/api/v1/test/signature-removed.pdf`

Digital signature was removed from the document.

```json
{
  "status": "modified",

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
  "specific_findings": ["CRITICAL: Digital signature was removed", "Document integrity compromised"]
}
```

---

### `modified-critical.pdf`

**URL:** `https://htpbe.tech/api/v1/test/modified-critical.pdf`

Signature removed + JavaScript detected + 8-step modification chain.

```json
{
  "status": "modified",

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
  "specific_findings": [
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

**URL:** `https://htpbe.tech/api/v1/test/both-threats.pdf`

JavaScript + embedded files + signature removed. Maximum severity.

```json
{
  "status": "modified",

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
  "specific_findings": [
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
- [GET /api/v1/result/{uid}](./api/result.md) — Retrieve analysis results (live key and test key)
- [GET /api/v1/checks](./api/checks.md) — List all checks with filtering

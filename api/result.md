# GET /api/v1/result/{id}

Retrieve a previously completed PDF analysis by its unique check ID.

## Endpoint

```
GET https://htpbe.tech/api/v1/result/{id}
```

## Authentication

This endpoint requires API key authentication via the `Authorization` header.

```http
Authorization: Bearer YOUR_API_KEY
```

## Data Isolation

**Important:** You can only retrieve analysis results that belong to your API client. Attempting to access another client's results will return a 404 error, even if the check ID exists. This ensures complete data privacy and isolation between API clients.

---

## Request

### Path Parameters

| Parameter | Type   | Required | Description                                                                                                           |
| --------- | ------ | -------- | --------------------------------------------------------------------------------------------------------------------- |
| `id`      | string | **Yes**  | Check ID returned from `POST /api/v1/analyze`. UUID v4 for all requests — live and test keys both return UUID v4 IDs. |

#### id Parameter Details

**Live key format:** UUID v4 — `xxxxxxxx-xxxx-4xxx-xxxx-xxxxxxxxxxxx`

**Test key format:** Deterministic UUID v4 — fixed all-zeros pattern, e.g. `00000000-0000-4000-8000-000000000001`

**Valid Examples:**

- `506a6b1b-1360-48a2-b389-abb346f85d04` (live key request)
- `00000000-0000-4000-8000-000000000001` (test key request — `clean.pdf`)

**Invalid Examples:**

- `abc123` (too short, not a UUID)
- `` (empty string)

**Where to Get:** The `id` field returned by `POST /api/v1/analyze`.

### Headers

| Header          | Type   | Required | Description                                                                                                                                                                                                                                                                |
| --------------- | ------ | -------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `Authorization` | string | **Yes**  | Bearer token with your API key (`Bearer htpbe_live_...` or `Bearer htpbe_test_...`). The `Bearer` prefix is recommended but optional — sending the raw key directly is also accepted, but only if the key starts with `htpbe_` (e.g., `Authorization: htpbe_live_sk_...`). |

### Example Request

```bash
curl https://htpbe.tech/api/v1/result/506a6b1b-1360-48a2-b389-abb346f85d04 \
  -H "Authorization: Bearer htpbe_live_sk_1234567890abcdef"
```

```javascript
// Node.js / TypeScript
const checkId = '506a6b1b-1360-48a2-b389-abb346f85d04';

const response = await fetch(`https://htpbe.tech/api/v1/result/${checkId}`, {
  headers: {
    Authorization: `Bearer ${process.env.HTPBE_API_KEY}`,
  },
});

const result = await response.json();
```

```python
# Python
import requests
import os

check_id = '506a6b1b-1360-48a2-b389-abb346f85d04'

response = requests.get(
    f'https://htpbe.tech/api/v1/result/{check_id}',
    headers={
        'Authorization': f"Bearer {os.getenv('HTPBE_API_KEY')}"
    }
)

result = response.json()
```

---

## Response

### Success Response (200 OK)

Returns a `ResultResponse` object containing all stored analysis data for the check.

**Note:** `/api/v1/analyze` returns only `{ "id": "..." }`. This endpoint is where all analysis data lives: metadata, structure, signatures, findings, and verdict details.

#### Response Structure

```typescript
{
  // Core Identification
  id: string;
  filename: string;
  check_date: number | null;
  file_size: number;

  // Algorithm version tracking
  algorithm_version: string;
  current_algorithm_version: string;
  outdated_warning?: string;

  // Primary Verdict
  status: "intact" | "modified" | "inconclusive";
  status_reason?: "consumer_software_origin";
  origin: {
    type: "consumer_software" | "institutional" | "unknown";
    software: string | null;
  };

  // PDF Metadata (Unix timestamps)
  creation_date: number | null;
  modification_date: number | null;
  creator: string | null;
  producer: string | null;

  // Critical Modification Detection
  critical_modification_marker: string | null;
  modification_confidence: string | null;
  verdict_reasoning: string | null;

  // Metadata Analysis
  date_sequence_valid: boolean;
  metadata_completeness_score: number;

  // PDF Structure
  xref_count: number;
  has_incremental_updates: boolean;
  update_chain_length: number;
  pdf_version: string | null;

  // Digital Signatures
  has_digital_signature: boolean;
  signature_count: number;
  signature_removed: boolean;
  modifications_after_signature: boolean;

  // Content Analysis
  page_count: number;
  object_count: number;
  has_javascript: boolean;
  has_embedded_files: boolean;

  // Threat Assessment
  detection_methods: string[];
  specific_findings: string[];
}
```

### Response Fields

#### Core Identification Fields

##### `id`

- **Type:** `string` (UUID v4)
- **Always Present:** Yes
- **Description:** Unique identifier for this check
- **Format:** `xxxxxxxx-xxxx-4xxx-xxxx-xxxxxxxxxxxx`
- **Example:** `"506a6b1b-1360-48a2-b389-abb346f85d04"`

##### `filename`

- **Type:** `string`
- **Always Present:** Yes
- **Description:** The stored filename for this check. If `original_filename` was provided in the `POST /analyze` request body, that value is used. Otherwise it is extracted from the last segment of the `url` path.
- **Examples:**
  - `"contract.pdf"` (provided as `original_filename` in the analyze request, or extracted from `https://example.com/docs/contract.pdf`)
  - `"invoice-2024-01.pdf"`
  - `"document.pdf"` (default set at analysis time when no filename is determinable from the URL)
  - `""` (empty string for legacy records created before filename tracking)

##### `check_date`

- **Type:** `number | null` (Unix timestamp in seconds)
- **Can Be Null:** Yes — `null` for test key results (synthetic responses have no real analysis timestamp) and for legacy records created before timestamp tracking was introduced
- **Description:** Timestamp when the analysis was performed
- **Format:** Seconds since Unix epoch (January 1, 1970 00:00:00 UTC)
- **Example:** `1736542583` = January 10, 2025, 18:23:03 UTC
- **Convert to Date:**
  - JavaScript: `check_date ? new Date(check_date * 1000) : null`
  - Python: `datetime.fromtimestamp(check_date) if check_date else None`

##### `file_size`

- **Type:** `number` (integer, bytes)
- **Always Present:** Yes
- **Range:** `1` to `10485760` (10 MB)
- **Description:** File size in bytes
- **Example:** `245632` = ~240 KB
- **Convert to MB:** `file_size / 1024 / 1024`

---

#### Algorithm Version Fields

##### `algorithm_version`

- **Type:** `string`
- **Always Present:** Yes
- **Description:** Version of the detection algorithm used when this check was performed. Defaults to `"1.0.0"` for records created before algorithm versioning was introduced.
- **Format:** Semantic versioning (e.g., `"2.1.0"`)
- **Usage:** Compare against the current version to determine if re-analysis is recommended
- **Example:** `"2.1.0"`

##### `current_algorithm_version`

- **Type:** `string`
- **Always Present:** Yes
- **Description:** The current algorithm version running on the server
- **Format:** Semantic versioning (e.g., `"2.1.6"`)
- **Usage:** Compare `algorithm_version` against `current_algorithm_version` to determine if the check is outdated. If `algorithm_version !== current_algorithm_version`, the check was performed with an older algorithm and re-analysis is recommended.
- **Example:** `"2.1.6"`

##### `outdated_warning`

- **Type:** `string` (optional)
- **Present When:** `algorithm_version` is older than `current_algorithm_version` (semver comparison — a future version would not trigger this warning even if the values differ)
- **Absent When:** `algorithm_version` is current
- **Description:** Human-readable explanation of what changed in the newer algorithm version and why re-analysis is recommended
- **Example:** `"Our detection software has been updated since this file was analyzed. The results shown below may no longer be accurate. We recommend re-analyzing this file for more accurate results."`

---

#### Analysis Results Fields

##### `status`

- **Type:** `"intact" | "modified" | "inconclusive"`
- **Always Present:** Yes
- **Description:** **PRIMARY VERDICT.** Priority: `modified > inconclusive > intact`
  - `"modified"` — forensic evidence of post-creation modification detected; takes priority over origin type — a modified Word or Excel document is still `modified`
  - `"inconclusive"` — consumer software origin (Word, LibreOffice, Google Docs, etc.) with no modification detected; integrity check does not apply to documents anyone can create from scratch — a document that was never an "original" in the institutional sense cannot be verified for post-creation tampering
  - `"intact"` — no modification detected and origin appears institutional
- **Note:** `status_reason: "consumer_software_origin"` only appears when `status === "inconclusive"` (consumer software + no modification detected)

##### `status_reason`

- **Type:** `"consumer_software_origin"` (string literal) | absent
- **Always Present:** No — only present when `status === "inconclusive"`
- **Description:** Explains why the result is inconclusive. Currently only one value is defined.
- **Value:** `"consumer_software_origin"` — the PDF was created by consumer software (Microsoft Word, LibreOffice, Google Docs, etc.). These tools allow anyone to create a document from scratch, so there is no meaningful "original" to compare against. Integrity verification does not apply.
- **Usage:** Always check `status_reason` when `status === "inconclusive"` — it tells you _why_ the result is inconclusive, not just that it is.

```typescript
if (result.status === 'inconclusive') {
  // result.status_reason === 'consumer_software_origin'
  // The document was created in consumer software — not tampered, just unverifiable
}
```

---

#### PDF Metadata Fields (Unix Timestamps)

All timestamps are Unix integers (seconds since epoch). Convert with: `new Date(timestamp * 1000)` in JavaScript, `datetime.fromtimestamp(ts)` in Python.

##### `creation_date`

- **Type:** `number | null` (Unix timestamp in seconds)
- **Can Be Null:** Yes
- **Description:** PDF creation timestamp from embedded metadata
- **Null When:** PDF does not contain creation date
- **Example:** `1704110400` = January 2, 2024 00:00:00 UTC
- **Convert:** JavaScript: `new Date(creation_date * 1000).toISOString()`

##### `modification_date`

- **Type:** `number | null` (Unix timestamp in seconds)
- **Can Be Null:** Yes
- **Description:** Last modification timestamp from embedded metadata
- **Null When:** PDF does not contain modification date
- **Critical:** If different from `creation_date`, indicates modification
- **Example:** `1707840000` = February 13, 2024 12:00:00 UTC

##### `creator`

- **Type:** `string | null`
- **Can Be Null:** Yes
- **Description:** Application that created the original document
- **Example:** `"Microsoft Word for Microsoft 365"`
- **Common Values:** `"Microsoft Word for Microsoft 365"`, `"Adobe InDesign CC"`, `"LibreOffice Writer"`, `"Google Docs"`

##### `producer`

- **Type:** `string | null`
- **Can Be Null:** Yes
- **Description:** Software that generated the final PDF
- **Example:** `"Adobe PDF Library 15.0"`
- **Common Values:** `"Adobe PDF Library 15.0"`, `"PDFKit"`, `"Microsoft: Print To PDF"`, `"iText 7.0.0"`

---

#### Critical Modification Detection Fields

##### `critical_modification_marker`

- **Type:** `string | null`
- **Can Be Null:** Yes
- **Description:** The specific forensic indicator that triggered the modification verdict
- **Null When:** No critical modification markers found
- **Possible Values:**
  - `null` - No critical markers detected
  - `"Different creation and modification dates"` — dates don't match; confidence: `certain`
  - `"Modifications detected after digital signature"` — file modified after signing; confidence: `certain`
  - `"Digital signature was removed"` — signature deletion detected; confidence: `certain`
  - `"Multiple cross-reference tables (incremental updates)"` — structure shows multiple saves; confidence: `high`
  - `"Known PDF editing tool detected"` — Producer/Creator identifies a PDF editing tool; confidence: `high`
  - `"Mandatory metadata fields removed"` — creation date or producer/creator absent; confidence: `high`
  - `"Font structure inconsistent with claimed PDF generator"` — claimed tool always embeds fonts, but none found; confidence: `high`
- **Usage:** If not null, this is **definitive evidence** of modification — treat the document as modified regardless of other fields

##### `modification_confidence`

- **Type:** `string | null` (enum)
- **Can Be Null:** Yes
- **Description:** How certain the algorithm is about the modification verdict. Use this to decide what action to take.
- **Possible Values:**
  - `"certain"` — conclusive structural or cryptographic evidence; **reject the document** — the finding cannot be a false positive (date mismatch, post-signature modification, signature removal)
  - `"high"` — strong forensic evidence; **flag for manual review** — highly reliable but rare false positives are possible in unusual legitimate workflows (e.g. linearization, batch processing pipelines)
  - `"none"` — no modification detected; document can be accepted as-is
- **Relationship to `status`:** `certain` and `high` always correspond to `status: "modified"`. `none` corresponds to `status: "intact"` or `"inconclusive"` depending on origin.
- **Null When:** Legacy records before this field was added

##### `verdict_reasoning`

- **Type:** `string | null`
- **Can Be Null:** Yes
- **Description:** Human-readable explanation of the primary reason behind the modification verdict
- **Null When:** No single dominant reason identified; verdict is based on aggregate scoring
- **Example Values:**
  - `"Known PDF editing tool detected (iLovePDF)"`
  - `"Digital signature was removed from the document"`
  - `"Document was modified after digital signing"`
- **Usage:** Display as a concise one-line verdict explanation in audit reports

---

#### Metadata Analysis Fields

##### `date_sequence_valid`

- **Type:** `boolean`
- **Always Present:** Yes
- **Description:** Whether creation and modification dates are in chronologically valid sequence
- **Valid Sequence:** `creation_date` ≤ `modification_date`
- **Possible Values:**
  - `true` - Dates are valid (creation before or equal to modification)
  - `false` - **Suspicious** - Modification date is before creation date (impossible in normal workflow)
- **Red Flag:** `false` strongly suggests date tampering

##### `metadata_completeness_score`

- **Type:** `number` (integer)
- **Always Present:** Yes
- **Range:** `0` to `100`
- **Description:** How complete the PDF metadata is (percentage of fields present)
- **Calculation:** Based on presence of:
  - Creation date
  - Modification date
  - Creator
  - Producer
  - Title, Subject, Author (standard metadata)
- **Interpretation:**
  - `90-100` - Excellent metadata, most fields present
  - `70-89` - Good metadata coverage
  - `50-69` - Moderate metadata
  - `30-49` - Limited metadata
  - `0-29` - Very sparse metadata (may hinder analysis)
- **Impact:** Higher scores enable more confident modification detection

---

#### PDF Structure Fields

##### `xref_count`

- **Type:** `number` (integer)
- **Always Present:** Yes
- **Description:** Number of cross-reference (xref) tables in the PDF
- **Interpretation:** `1` = original document, `2+` = document has been modified and saved multiple times

##### `has_incremental_updates`

- **Type:** `boolean`
- **Always Present:** Yes
- **Description:** Whether PDF has incremental update sections (been saved multiple times after initial creation)
- **Values:** `true` = PDF has been edited and saved incrementally, `false` = PDF was generated in one operation

##### `update_chain_length`

- **Type:** `number` (integer)
- **Always Present:** Yes
- **Description:** Number of update sections in PDF structure (how many times the PDF was saved)
- **Interpretation:** `1` = original, `2-5` = light editing, `6-10` = moderate editing, `11+` = heavy editing

##### `pdf_version`

- **Type:** `string | null`
- **Can Be Null:** Yes
- **Description:** PDF specification version
- **Examples:** `"1.7"`, `"1.4"`, `"2.0"`, `null`
- **Common Versions:** `"1.7"` (most common, Acrobat 8.x+), `"1.4"` (Acrobat 5.x), `"2.0"` (modern ISO standard)

---

#### Digital Signatures Fields

##### `has_digital_signature`

- **Type:** `boolean`
- **Always Present:** Yes
- **Description:** Whether PDF currently contains digital signatures
- **Values:** `true` = digitally signed, `false` = not signed
- **Note:** Signature validity is NOT checked (only presence)

##### `signature_count`

- **Type:** `number` (integer)
- **Always Present:** Yes
- **Description:** Number of digital signature fields in the PDF
- **Range:** `0` and higher

##### `signature_removed`

- **Type:** `boolean`
- **Always Present:** Yes
- **Description:** Whether a digital signature was removed from the document
- **Critical:** If `true`, this is strong evidence of tampering

##### `modifications_after_signature`

- **Type:** `boolean`
- **Always Present:** Yes
- **Description:** Whether PDF was modified after being digitally signed
- **Critical:** If `true`, the digital signature is likely invalidated

---

#### Content Analysis Fields

##### `page_count`

- **Type:** `number` (integer)
- **Always Present:** Yes
- **Range:** `1` and higher
- **Description:** Total number of pages in the PDF document
- **Example:** `12` = 12-page document
- **Usage:** Useful for estimating document size and complexity

##### `object_count`

- **Type:** `number` (integer)
- **Always Present:** Yes
- **Range:** `0` and higher
- **Description:** Number of PDF objects in the internal structure
- **What It Means:**
  - PDF objects include: pages, fonts, images, annotations, forms, etc.
  - Higher object count generally means more complex document
  - Useful for detecting unusual structural complexity
- **Typical Ranges:**
  - `50-200` - Simple text document
  - `200-500` - Document with images
  - `500-1000` - Complex document with forms/annotations
  - `1000+` - Very complex document or potential obfuscation

##### `has_javascript`

- **Type:** `boolean`
- **Always Present:** Yes
- **Description:** Whether PDF contains embedded JavaScript code
- **Values:** `true` = JavaScript present (potential security risk), `false` = no JavaScript detected
- **Security Note:** JavaScript in PDFs can be used for malicious purposes

##### `has_embedded_files`

- **Type:** `boolean`
- **Always Present:** Yes
- **Description:** Whether PDF has embedded file attachments
- **Values:** `true` = has attachments (potential security risk), `false` = no attachments
- **Security Note:** Attachments can hide malware

---

#### Threat Assessment Fields

##### `detection_methods`

- **Type:** `string[]` (array of strings)
- **Always Present:** Yes
- **Description:** List of detection methods that were applied during analysis
- **Empty When:** No anomalies detected (intact documents) or inconclusive origin
- **Common Values:**
  - `"metadata_analysis"` - Analyzed creation vs modification dates
  - `"structure_analysis"` - Examined PDF internal structure for updates
  - `"signature_analysis"` - Checked for digital signature presence/removal
  - `"tool_detection"` - Checked creator/producer combinations for known editing tools
  - `"threat_analysis"` - Checked for JavaScript and embedded files
- **Usage:** Indicates which analysis techniques were used (informational)

##### `specific_findings`

- **Type:** `string[]` (array of strings)
- **Always Present:** Yes
- **Description:** Detailed human-readable list of issues and anomalies detected during analysis
- **Empty When:** No issues detected (original document)
- **Example Values:** `"Document modified 36 days after creation"`, `"Digital signature removed (critical tampering)"`, `"JavaScript code detected in PDF"`, `"Multiple xref tables detected (4 total)"`

---

### Example Response

```json
{
  "id": "506a6b1b-1360-48a2-b389-abb346f85d04",
  "filename": "contract.pdf",
  "check_date": 1736542583,
  "file_size": 245632,
  "algorithm_version": "2.1.6",
  "current_algorithm_version": "2.1.6",
  "status": "modified",
  "origin": {
    "type": "institutional",
    "software": null
  },
  "creation_date": 1704110400,
  "modification_date": 1707840000,
  "creator": "Adobe Acrobat Pro DC",
  "producer": "Adobe PDF Library 15.0",
  "critical_modification_marker": "Digital signature was removed",
  "modification_confidence": "certain",
  "verdict_reasoning": "Digital signature was removed from the document",
  "date_sequence_valid": true,
  "metadata_completeness_score": 90,
  "xref_count": 2,
  "has_incremental_updates": true,
  "update_chain_length": 3,
  "pdf_version": "1.7",
  "has_digital_signature": false,
  "signature_count": 0,
  "signature_removed": true,
  "modifications_after_signature": false,
  "page_count": 12,
  "object_count": 487,
  "has_javascript": false,
  "has_embedded_files": false,
  "detection_methods": ["metadata_analysis", "structure_analysis", "signature_analysis"],
  "specific_findings": [
    "Document modified 36 days after creation",
    "Digital signature removed (critical tampering)",
    "3 incremental update chains detected"
  ]
}
```

#### Example Response — Inconclusive (Consumer Software Origin)

```json
{
  "id": "00000000-0000-4000-8000-000000000011",
  "filename": "inconclusive.pdf",
  "check_date": null,
  "file_size": 204800,
  "algorithm_version": "2.1.6",
  "current_algorithm_version": "2.1.6",
  "status": "inconclusive",
  "status_reason": "consumer_software_origin",
  "origin": {
    "type": "consumer_software",
    "software": "Microsoft Excel"
  },
  "creation_date": 1709283600,
  "modification_date": 1709283600,
  "creator": "Microsoft Excel 2019",
  "producer": "Microsoft: Print To PDF",
  "critical_modification_marker": null,
  "modification_confidence": "none",
  "verdict_reasoning": null,
  "date_sequence_valid": true,
  "metadata_completeness_score": 65,
  "xref_count": 1,
  "has_incremental_updates": false,
  "update_chain_length": 1,
  "pdf_version": "1.7",
  "has_digital_signature": false,
  "signature_count": 0,
  "signature_removed": false,
  "modifications_after_signature": false,
  "page_count": 1,
  "object_count": 82,
  "has_javascript": false,
  "has_embedded_files": false,
  "detection_methods": [],
  "specific_findings": []
}
```

---

## Error Responses

All errors follow this format:

```json
{
  "error": "Human-readable error message",
  "code": "machine_readable_error_code",
  "details": "Optional additional context (present for some errors)"
}
```

### 400 Bad Request

Returned when the check ID format is invalid.

```json
{
  "error": "Invalid check ID format",
  "code": "invalid_request"
}
```

**Causes:**

- `id` is empty string
- `id` is not a valid UUID v4 (e.g. `abc123`, `12345678-1234-1234-1234-12345678`)
- `id` contains invalid characters

**Solution:** Ensure you're using the full UUID v4 returned by `POST /api/v1/analyze` or from the `id` field in `GET /api/v1/checks`. Format: `xxxxxxxx-xxxx-4xxx-xxxx-xxxxxxxxxxxx` (32 hex digits with 4 hyphens).

---

### 401 Unauthorized

Authentication errors (same as analyze endpoint).

**See:** [analyze.md](./analyze.md#401-unauthorized) for details.

---

### 402 Payment Required

No active subscription found for this API key.

**See:** [analyze.md](./analyze.md#402-payment-required) for details.

---

### 403 Forbidden

Access forbidden due to account status (deactivated key).

**See:** [analyze.md](./analyze.md#403-forbidden) for details.

---

### 404 Not Found

Check not found or access denied.

```json
{
  "error": "Check not found or access denied",
  "code": "not_found"
}
```

**Causes:**

1. Check ID doesn't exist in the database
2. Check ID belongs to another API client (data isolation)
3. Check ID is malformed but passed basic validation

**Security Note:** For privacy, we intentionally don't distinguish between "doesn't exist" and "belongs to another client". This prevents enumeration attacks.

**Solution:**

- Verify the check ID is correct
- Ensure you're using the API key that created the check
- If you need to share results between API clients, contact support for guidance

---

### 500 Internal Server Error

Server error during retrieval.

```json
{
  "error": "Failed to fetch result",
  "code": "internal_error"
}
```

**Cause:** Unexpected database or server error.

**Solution:**

- Retry the request
- If persists, contact support with the check ID

---

## Use Cases

### 1. Building Audit Trails

Store check IDs and retrieve full results later for compliance:

```javascript
// Store check ID after analysis
await db.documents.create({
  id: uploadId,
  htpbe_check_id: analysisResponse.id,
  verified_at: new Date(),
});

// Later, retrieve full details for audit
const auditData = await fetch(`https://htpbe.tech/api/v1/result/${doc.htpbe_check_id}`, {
  headers: { Authorization: `Bearer ${apiKey}` },
});
```

---

### 2. Detailed Reporting

Use the additional fields for comprehensive reports:

```javascript
const result = await getResult(checkId);

const report = {
  summary:
    result.status === 'modified'
      ? 'Modified'
      : result.status === 'inconclusive'
        ? 'Unverifiable'
        : 'Original',

  // Use fields not available in analyze endpoint
  details: {
    pages: result.page_count,
    objects: result.object_count,
    metadataQuality: result.metadata_completeness_score,
    analysisDate: new Date(result.check_date * 1000),
  },

  criticalMarkers: result.critical_modification_marker ? [result.critical_modification_marker] : [],
};
```

---

**Related Endpoints:**

- [POST /api/v1/analyze](./analyze.md) - Analyze a PDF
- [GET /api/v1/checks](./checks.md) - List all checks with filtering for custom analytics

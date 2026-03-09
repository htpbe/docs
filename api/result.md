# GET /api/v1/result/{uid}

Retrieve a previously completed PDF analysis by its unique check ID.

## Endpoint

```
GET https://htpbe.tech/api/v1/result/{uid}
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

| Parameter | Type   | Required | Description                                                                                                                                                |
| --------- | ------ | -------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `uid`     | string | **Yes**  | UUID v4 identifier of the check to retrieve. This is the `id` returned from the `/api/v1/analyze` endpoint. Format: `xxxxxxxx-xxxx-4xxx-xxxx-xxxxxxxxxxxx` |

#### uid Parameter Details

**Format:** Must be a valid UUID v4 string

**Valid Example:**

- `506a6b1b-1360-48a2-b389-abb346f85d04`

**Invalid Examples:**

- `abc123` (not a UUID)
- `506a6b1b` (incomplete UUID)
- `` (empty string)

**Where to Get:** The `uid` is returned as the `id` field in the response from `POST /api/v1/analyze`.

### Headers

| Header          | Type   | Required | Description                                                                         |
| --------------- | ------ | -------- | ----------------------------------------------------------------------------------- |
| `Authorization` | string | **Yes**  | Bearer token with your API key (`Bearer htpbe_live_...` or `Bearer htpbe_test_...`) |

### Example Request

```bash
curl -X GET https://htpbe.tech/api/v1/result/506a6b1b-1360-48a2-b389-abb346f85d04 \
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

**Note:** This endpoint returns **more detailed information** than the `/api/v1/analyze` endpoint, including:

- Unix timestamps instead of ISO 8601 strings
- Additional metadata fields (metadata completeness, suspicious patterns)
- Content analysis (page count, object count)
- Detection methods used

#### Response Structure

```typescript
{
  // Core Identification
  id: string;
  user_id: string;
  original_filename: string;
  check_date: number;
  file_size: number;

  // Algorithm version tracking
  algorithm_version: string;
  is_outdated: boolean;
  outdated_warning?: string;

  // Primary Verdict
  status: "intact" | "modified" | "inconclusive";
  status_reason?: "consumer_software_origin";
  origin: {
    type: "consumer_software" | "institutional" | "unknown";
    software: string | null;
    warning: string | null;
  };

  // Analysis Results
  been_changed: boolean;  // auxiliary — prefer status

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
- **Description:** Unique identifier for this check (same as the `uid` parameter)
- **Format:** `xxxxxxxx-xxxx-4xxx-xxxx-xxxxxxxxxxxx`
- **Example:** `"506a6b1b-1360-48a2-b389-abb346f85d04"`

##### `user_id`

- **Type:** `string` (UUID v4)
- **Always Present:** Yes
- **Description:** UUID of the user account that owns this check
- **Usage:** For internal tracking. All checks with the same `user_id` belong to the same account.
- **Example:** `"be0543d7-08dd-4d4e-867e-2ad5c2c43dc0"`

##### `original_filename`

- **Type:** `string`
- **Always Present:** Yes
- **Description:** Filename extracted from the URL path when the PDF was analyzed
- **Extraction:** Taken from the last segment of the `url` path
- **Examples:**
  - `"contract.pdf"` (from `https://example.com/docs/contract.pdf`)
  - `"invoice-2024-01.pdf"`
  - `"document.pdf"` (default if no filename in URL)

##### `check_date`

- **Type:** `number` (Unix timestamp in seconds)
- **Always Present:** Yes
- **Description:** Timestamp when the analysis was performed
- **Format:** Seconds since Unix epoch (January 1, 1970 00:00:00 UTC)
- **Example:** `1736542583` = January 10, 2025, 18:23:03 UTC
- **Convert to Date:**
  - JavaScript: `new Date(check_date * 1000)`
  - Python: `datetime.fromtimestamp(check_date)`

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
- **Description:** Version of the detection algorithm used when this check was performed
- **Format:** Semantic versioning (e.g., `"2.1.0"`)
- **Usage:** Compare against the current version to determine if re-analysis is recommended
- **Example:** `"2.1.0"`

##### `is_outdated`

- **Type:** `boolean`
- **Always Present:** Yes
- **Description:** Whether the check was performed with an older algorithm version
- **Possible Values:**
  - `true` — Analysis was performed with an outdated algorithm; re-analysis recommended
  - `false` — Analysis is current
- **Usage:** Prompt users to re-check files where `is_outdated` is `true`

##### `outdated_warning`

- **Type:** `string` (optional)
- **Present When:** `is_outdated` is `true`
- **Absent When:** `is_outdated` is `false`
- **Description:** Human-readable explanation of what changed in the newer algorithm version and why re-analysis is recommended
- **Example:** `"Detection accuracy for online editor tools improved in v2.1.0. We recommend re-analyzing this file for more accurate results."`

---

#### Analysis Results Fields

##### `status`

- **Type:** `"intact" | "modified" | "inconclusive"`
- **Always Present:** Yes
- **Description:** **PRIMARY VERDICT** — see [Understanding the verdict](./analyze.md#understanding-the-verdict) for details
- See `POST /api/v1/analyze` documentation for full description of `status`, `status_reason`, and `origin` fields.

##### `been_changed`

- **Type:** `boolean`
- **Always Present:** Yes
- **Description:** **AUXILIARY** — kept for backward compatibility; use `status` for new integrations
- **Description:** Whether the PDF has been modified (same as in analyze response)
- **Possible Values:**
  - `true` - Modified
  - `false` - Original

---

#### PDF Metadata Fields (Unix Timestamps)

**Note:** Timestamps in this endpoint are Unix timestamps (seconds), not ISO 8601 strings like in `/api/v1/analyze`.

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
- **See:** [analyze.md](./analyze.md#analysismetadatacreator) for common values

##### `producer`

- **Type:** `string | null`
- **Can Be Null:** Yes
- **Description:** Software that generated the final PDF
- **Example:** `"Adobe PDF Library 15.0"`
- **See:** [analyze.md](./analyze.md#analysismetadataproducer) for common values

---

#### Critical Modification Detection Fields

##### `critical_modification_marker`

- **Type:** `string | null`
- **Can Be Null:** Yes
- **Description:** Specific critical modification detected with 100% confidence
- **Null When:** No critical modification markers found
- **Possible Values:**
  - `null` - No critical markers
  - `"Different creation and modification dates"` - Dates don't match (strong evidence)
  - `"Multiple cross-reference tables (incremental updates)"` - Structure shows multiple saves
  - `"Modifications detected after digital signature"` - File modified after signing
  - `"Digital signature was removed"` - Signature deletion detected (critical tampering)
- **Usage:** If not null, this is **definitive evidence** of modification
- **Impact:** When set, `modification_confidence` will be `"100%"`

##### `modification_confidence`

- **Type:** `string | null` (enum)
- **Can Be Null:** Yes
- **Description:** Confidence level specifically for critical modifications
- **Possible Values:**
  - `"none"` - No critical modification detected
  - `"high"` - High confidence based on structural indicators but no conclusive proof
  - `"100%"` - Absolute certainty based on structural evidence (when `critical_modification_marker` is set)
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
- **See:** [analyze.md](./analyze.md#analysisstrstructurexref_count)

##### `has_incremental_updates`

- **Type:** `boolean`
- **Always Present:** Yes
- **Description:** Whether PDF has incremental update sections
- **See:** [analyze.md](./analyze.md#analysisstrstructurehas_incremental_updates)

##### `update_chain_length`

- **Type:** `number` (integer)
- **Always Present:** Yes
- **Description:** Number of update sections in PDF structure
- **See:** [analyze.md](./analyze.md#analysisstrstructureupdate_chain_length)

##### `pdf_version`

- **Type:** `string | null`
- **Can Be Null:** Yes
- **Description:** PDF specification version
- **Examples:** `"1.7"`, `"1.4"`, `"2.0"`, `null`
- **See:** [analyze.md](./analyze.md#analysisstrstructurepdf_version)

---

#### Digital Signatures Fields

##### `has_digital_signature`

- **Type:** `boolean`
- **Always Present:** Yes
- **Description:** Whether PDF currently contains digital signatures
- **See:** [analyze.md](./analyze.md#analysissignatureshas_digital_signature)

##### `signature_count`

- **Type:** `number` (integer)
- **Always Present:** Yes
- **Description:** Number of digital signature fields
- **See:** [analyze.md](./analyze.md#analysissignaturessignature_count)

##### `signature_removed`

- **Type:** `boolean`
- **Always Present:** Yes
- **Description:** Whether a digital signature was removed
- **See:** [analyze.md](./analyze.md#analysissignaturessignature_removed)

##### `modifications_after_signature`

- **Type:** `boolean`
- **Always Present:** Yes
- **Description:** Whether PDF was modified after being signed
- **See:** [analyze.md](./analyze.md#analysissignaturesmodifications_after_signature)

---

#### Content Analysis Fields

**Note:** These fields are only available in the `/result` endpoint, not in `/analyze`.

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
- **Description:** Whether PDF contains JavaScript code
- **See:** [analyze.md](./analyze.md#analysisthreatshas_javascript)

##### `has_embedded_files`

- **Type:** `boolean`
- **Always Present:** Yes
- **Description:** Whether PDF has embedded file attachments
- **See:** [analyze.md](./analyze.md#analysisthreatshas_embedded_files)

---

#### Threat Assessment Fields

##### `detection_methods`

- **Type:** `string[]` (array of strings)
- **Always Present:** Yes
- **Description:** List of detection methods that were applied during analysis
- **Empty When:** Error during analysis (shouldn't occur in successful responses)
- **Common Values:**
  - `"Metadata comparison"` - Analyzed creation vs modification dates
  - `"Structure analysis"` - Examined PDF internal structure for updates
  - `"Signature verification"` - Checked for digital signature presence/removal
  - `"Date sequence analysis"` - Validated chronological consistency of dates
  - `"Tool pattern matching"` - Checked creator/producer combinations
- **Usage:** Indicates which analysis techniques were used (informational)

##### `specific_findings`

- **Type:** `string[]` (array of strings)
- **Always Present:** Yes
- **Description:** Detailed list of issues and anomalies detected (same as `findings` in analyze response)
- **Empty When:** No issues detected (original document)
- **See:** [analyze.md](./analyze.md#analysisfindings) for example values

---

### Example Response

```json
{
  "id": "506a6b1b-1360-48a2-b389-abb346f85d04",
  "user_id": "be0543d7-08dd-4d4e-867e-2ad5c2c43dc0",
  "original_filename": "contract.pdf",
  "check_date": 1736542583,
  "file_size": 245632,
  "algorithm_version": "2.1.0",
  "is_outdated": false,
  "status": "modified",
  "origin": {
    "type": "institutional",
    "software": null,
    "warning": null
  },
  "been_changed": true,
  "creation_date": 1704110400,
  "modification_date": 1707840000,
  "creator": "Microsoft Word for Microsoft 365",
  "producer": "Adobe PDF Library 15.0",
  "critical_modification_marker": "Digital signature was removed",
  "modification_confidence": "100%",
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
  "object_count": 456,
  "has_javascript": false,
  "has_embedded_files": false,
  "detection_methods": ["Metadata comparison", "Structure analysis", "Signature verification"],
  "specific_findings": [
    "Document modified 36 days after creation",
    "Digital signature removed (critical tampering)",
    "Three incremental updates detected"
  ]
}
```

---

## Error Responses

All errors follow this format:

```json
{
  "error": "Error message",
  "details": "Optional additional context"
}
```

### 400 Bad Request

Returned when the check ID format is invalid.

```json
{
  "error": "Invalid check ID format"
}
```

**Causes:**

- `uid` is empty string
- `uid` is too short (< 10 characters)
- `uid` contains invalid characters

**Solution:** Ensure you're using the full UUID returned from `/api/v1/analyze`

---

### 401 Unauthorized

Authentication errors (same as analyze endpoint).

**See:** [analyze.md](./analyze.md#401-unauthorized) for details.

---

### 404 Not Found

Check not found or access denied.

```json
{
  "error": "Check not found or access denied"
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
  "details": "Database query error"
}
```

**Cause:** Unexpected database or server error.

**Solution:**

- Retry the request
- If persists, contact support with the check ID

---

## Use Cases

### 1. Polling for Results

If you implement async analysis (future webhook feature), you can poll this endpoint:

```javascript
async function waitForResult(checkId, maxAttempts = 10) {
  for (let i = 0; i < maxAttempts; i++) {
    const response = await fetch(`https://htpbe.tech/api/v1/result/${checkId}`, {
      headers: { Authorization: `Bearer ${apiKey}` },
    });

    if (response.ok) {
      return await response.json();
    }

    // Wait 2 seconds before retry
    await new Promise((resolve) => setTimeout(resolve, 2000));
  }

  throw new Error('Analysis timed out');
}
```

**Note:** Currently unnecessary as all analyses complete synchronously.

---

### 2. Building Audit Trails

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

### 3. Detailed Reporting

Use the additional fields for comprehensive reports:

```javascript
const result = await getResult(checkId);

const report = {
  summary: result.been_changed ? 'Modified' : 'Original',

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
- [GET /api/v1/stats](./stats.md) - Get aggregate statistics
- [GET /api/v1/checks](./checks.md) - List all checks with filtering for custom analytics

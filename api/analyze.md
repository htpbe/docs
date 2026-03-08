# POST /api/v1/analyze

Analyze a PDF document from a publicly accessible URL to detect modifications and tampering.

## Endpoint

```
POST https://htpbe.tech/api/v1/analyze
```

## Authentication

This endpoint requires API key authentication via the `Authorization` header.

```http
Authorization: Bearer YOUR_API_KEY
```

---

## Request

### Headers

| Header          | Type   | Required | Description                                                                         |
| --------------- | ------ | -------- | ----------------------------------------------------------------------------------- |
| `Authorization` | string | **Yes**  | Bearer token with your API key (`Bearer htpbe_live_...` or `Bearer htpbe_test_...`) |
| `Content-Type`  | string | **Yes**  | Must be `application/json`                                                          |

### Body Parameters

| Parameter           | Type   | Required | Description                                                                                                                                                                        |
| ------------------- | ------ | -------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `url`               | string | **Yes**  | Publicly accessible HTTP/HTTPS URL pointing to a PDF file. The file must be downloadable without authentication and must not exceed 10 MB in size.                                 |
| `original_filename` | string | No       | Original filename of the document (before any storage renaming). When provided, this name is stored and returned in results instead of the filename extracted from the URL.        |
| `webhook_url`       | string | No       | **[Coming Soon]** URL where a webhook notification will be sent when analysis is complete (for async processing). Currently not implemented - all requests complete synchronously. |

#### url Parameter Details

**Format:** Must be a valid HTTP or HTTPS URL

**File Size Limit:** Maximum 10 MB (10,485,760 bytes)

**Accessibility:** The URL must be publicly accessible. Files behind authentication, firewalls, or VPNs will fail to download.

**Valid Examples:**

- `https://example.com/documents/contract.pdf`
- `https://cdn.yoursite.com/uploads/2024/invoice-12345.pdf`
- `https://storage.googleapis.com/bucket-name/file.pdf`

**Invalid Examples:**

- `http://localhost/document.pdf` (not publicly accessible)
- `https://example.com/file.docx` (not a PDF)
- `ftp://example.com/file.pdf` (FTP not supported)
- `file:///local/path/document.pdf` (local file paths not supported)

### Example Request

```bash
curl -X POST https://htpbe.tech/api/v1/analyze \
  -H "Authorization: Bearer htpbe_live_sk_1234567890abcdef" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://example.com/documents/contract.pdf"
  }'
```

```javascript
// Node.js / TypeScript
const response = await fetch('https://htpbe.tech/api/v1/analyze', {
  method: 'POST',
  headers: {
    Authorization: `Bearer ${process.env.HTPBE_API_KEY}`,
    'Content-Type': 'application/json',
  },
  body: JSON.stringify({
    url: 'https://example.com/documents/contract.pdf',
  }),
});

const result = await response.json();
```

```python
# Python
import requests
import os

response = requests.post(
    'https://htpbe.tech/api/v1/analyze',
    headers={
        'Authorization': f"Bearer {os.getenv('HTPBE_API_KEY')}",
        'Content-Type': 'application/json'
    },
    json={
        'url': 'https://example.com/documents/contract.pdf'
    }
)

result = response.json()
```

---

## Response

### Success Response (200 OK)

Returns an `AnalysisResponse` object containing the analysis results.

#### Response Structure

```typescript
{
  id: string;
  analysis: {
    status: "intact" | "modified" | "inconclusive";   // PRIMARY verdict
    status_reason?: "consumer_software_origin";         // only when inconclusive
    been_changed: boolean;                              // AUXILIARY (backward-compat)
    risk_score: number;
    confidence_level: string;
    verdict_reasoning: string | undefined;
    origin: {
      type: "consumer_software" | "institutional" | "unknown";
      software: string | null;
      warning: string | null;
    };
    metadata: {
      creation_date: string | null;
      modification_date: string | null;
      creator: string | null;
      producer: string | null;
      file_size: number;
    };
    structure: {
      has_incremental_updates: boolean;
      update_chain_length: number;
      xref_count: number;
      pdf_version: string | null;
    };
    signatures: {
      has_digital_signature: boolean;
      signature_count: number;
      signature_removed: boolean;
      modifications_after_signature: boolean;
    };
    threats: {
      has_javascript: boolean;
      has_embedded_files: boolean;
    };
    findings: string[];
  };
}
```

### Response Fields

#### Top-Level Fields

##### `id`

- **Type:** `string` (UUID v4)
- **Always Present:** Yes
- **Description:** Unique identifier for this analysis check
- **Format:** `xxxxxxxx-xxxx-4xxx-xxxx-xxxxxxxxxxxx` (UUID version 4)
- **Usage:** Use this ID to retrieve the full analysis result later via `/api/v1/result/{id}`
- **Example:** `"3f9c8b7a-2e1d-4c5f-9b8e-7a6d5c4b3a21"`

##### `analysis`

- **Type:** `object`
- **Always Present:** Yes
- **Description:** Container object holding all analysis results

---

#### analysis.\* Fields (Core Results)

##### `analysis.status`

- **Type:** `"intact" | "modified" | "inconclusive"`
- **Always Present:** Yes
- **Description:** **PRIMARY VERDICT** — the recommended field to drive your business logic
- **Possible Values:**

  | Value            | Meaning                                                                                          |
  | ---------------- | ------------------------------------------------------------------------------------------------ |
  | `"intact"`       | PDF was not modified after creation; origin appears institutional                                |
  | `"modified"`     | PDF was tampered with after creation (regardless of origin)                                      |
  | `"inconclusive"` | PDF was not modified, but was created with consumer software — integrity check is not applicable |

- **When to use `status` vs `been_changed`:** Always prefer `status`. It is the semantic verdict. `been_changed` is auxiliary and kept only for backward compatibility.

##### `analysis.status_reason`

- **Type:** `"consumer_software_origin" | undefined`
- **Present Only When:** `status === "inconclusive"`
- **Description:** Machine-readable reason code explaining why the status is inconclusive
- **Current Values:**
  - `"consumer_software_origin"` — PDF was created with consumer/office software (Microsoft Office, LibreOffice, Apple Pages, etc.), making the integrity check meaningless

##### `analysis.origin`

- **Type:** `object`
- **Always Present:** Yes
- **Description:** Information about the software that created the PDF

  | Sub-field  | Type                                                  | Description                                                  |
  | ---------- | ----------------------------------------------------- | ------------------------------------------------------------ |
  | `type`     | `"consumer_software" \| "institutional" \| "unknown"` | Classification of the creator                                |
  | `software` | `string \| null`                                      | Normalized name (e.g., `"Microsoft Excel"`, `"LibreOffice"`) |
  | `warning`  | `string \| null`                                      | Human-readable warning (only for `consumer_software`)        |

- **Consumer software** includes: Microsoft Word, Excel, PowerPoint, LibreOffice, OpenOffice, Apple Pages/Numbers/Keynote, WPS Office, macOS Print to PDF (Quartz)
- **Institutional** means the creator/producer metadata is present but does not match consumer patterns

---

### Understanding the verdict

| `status`         | `been_changed` | Meaning                                                                                      |
| ---------------- | -------------- | -------------------------------------------------------------------------------------------- |
| `"intact"`       | `false`        | PDF not modified; origin appears institutional — integrity check is applicable and passed    |
| `"modified"`     | `true`         | PDF was tampered after creation — regardless of origin                                       |
| `"inconclusive"` | `false`        | PDF not modified, but created with consumer software — integrity check is **not applicable** |

**What we detect:**

- Structural modifications (incremental updates, xref table additions)
- Metadata date discrepancies
- Digital signature removal or post-sign modifications
- Suspicious tool patterns

**What we don't detect:**

- Document fabrication (entire content created in Word/Excel and exported to PDF)
- Visual/content differences without structural changes
- Password-protected content
- Cryptographic signature validity

---

##### `analysis.been_changed`

- **Type:** `boolean`
- **Always Present:** Yes
- **Description:** **AUXILIARY FIELD** — kept for backward compatibility. Use `analysis.status` for new integrations.
- **Possible Values:**
  - `true` - PDF has been modified after initial creation
  - `false` - PDF appears to be in its original state (but may be `inconclusive` if consumer software origin)
- **Relationship to `status`:** `been_changed = true` always means `status = "modified"`. `been_changed = false` means either `"intact"` or `"inconclusive"` depending on origin.
- **Detection Methods:**
  - Different creation and modification dates in metadata
  - Presence of incremental update sections in PDF structure
  - Removed or invalidated digital signatures
  - Multiple cross-reference tables indicating multiple saves
  - Suspicious metadata patterns

##### `analysis.risk_score`

- **Type:** `number` (integer)
- **Always Present:** Yes
- **Range:** `0` to `100`
- **Description:** Overall modification risk assessment score
- **Risk Levels:**
  - **`0-30` (Low Risk):** Minimal or no modifications detected. PDF appears authentic.
  - **`31-60` (Medium Risk):** Some modifications detected. Review recommended.
  - **`61-85` (High Risk):** Significant modifications detected. High probability of tampering.
  - **`86-100` (Critical Risk):** Severe tampering detected. Document integrity compromised.
- **Calculation:** Based on multiple factors including:
  - Number and severity of modifications
  - Presence of critical markers (signature removal, date discrepancies)
  - Structural anomalies
  - Metadata completeness and consistency

##### `analysis.confidence_level`

- **Type:** `string` (enum)
- **Always Present:** Yes
- **Possible Values:**
  - `"low"` - Uncertain. Limited evidence available.
  - `"medium"` - Likely. Some evidence suggests modification.
  - `"high"` - Very confident. Strong evidence of modification/originality.
  - `"very_high"` - Certain. Definitive structural or cryptographic evidence.
- **Description:** System's confidence in the modification detection
- **When Low:** PDF has minimal metadata or structural information to analyze
- **When Very High:** PDF has clear structural evidence (e.g., signature removed, multiple xref tables)

##### `analysis.verdict_reasoning`

- **Type:** `string | undefined`
- **Can Be Undefined:** Yes
- **Description:** Human-readable explanation of the primary reason behind the modification verdict
- **Present When:** The analysis engine identified a specific dominant reason for the verdict (e.g., known editing tool detected, signature removed)
- **Undefined When:** No single dominant reason identified; verdict is based on aggregate scoring
- **Example Values:**
  - `"Known PDF editing tool detected (iLovePDF)"`
  - `"Digital signature was removed from the document"`
  - `"Document was modified after digital signing"`
- **Usage:** Display to end users as a concise one-line verdict explanation

##### `analysis.findings`

- **Type:** `string[]` (array of strings)
- **Always Present:** Yes
- **Description:** Human-readable list of specific issues and anomalies detected
- **Empty When:** No modifications or issues detected
- **Example Values:**
  - `"Document modified 36 days after creation"`
  - `"Digital signature removed (critical tampering)"`
  - `"Three incremental updates detected"`
  - `"Different creation and modification dates"`
  - `"Metadata indicates file was edited with Adobe Acrobat"`
  - `"Creation date missing from PDF metadata"`
  - `"Suspicious creator/producer combination"`
- **Usage:** Display these to end users for detailed explanation

---

#### analysis.metadata.\* Fields (PDF Metadata)

##### `analysis.metadata.creation_date`

- **Type:** `string | null` (ISO 8601 timestamp)
- **Can Be Null:** Yes
- **Format:** `YYYY-MM-DDTHH:mm:ss.sssZ` (ISO 8601 UTC)
- **Description:** Timestamp when the PDF was originally created according to embedded metadata
- **Null When:** PDF metadata does not contain a creation date (common in automatically generated PDFs)
- **Examples:**
  - `"2024-01-15T10:30:00.000Z"` (January 15, 2024 at 10:30 AM UTC)
  - `"2023-12-01T00:00:00.000Z"` (December 1, 2023 at midnight UTC)
  - `null` (no creation date in metadata)
- **Modification Indicator:** If different from `modification_date`, indicates the file was modified

##### `analysis.metadata.modification_date`

- **Type:** `string | null` (ISO 8601 timestamp)
- **Can Be Null:** Yes
- **Format:** `YYYY-MM-DDTHH:mm:ss.sssZ` (ISO 8601 UTC)
- **Description:** Timestamp of the last modification according to embedded metadata
- **Null When:** PDF metadata does not contain a modification date
- **Critical Indicator:** If different from `creation_date`, this is strong evidence of modification
- **Examples:**
  - `"2024-02-20T14:45:00.000Z"` (Modified after creation)
  - Same as `creation_date` (Original, never modified)
  - `null` (no modification date)

##### `analysis.metadata.creator`

- **Type:** `string | null`
- **Can Be Null:** Yes
- **Description:** Name of the application that created the original document (before PDF conversion)
- **Null When:** PDF metadata does not specify a creator
- **Common Values:**
  - `"Microsoft Word for Microsoft 365"`
  - `"Adobe InDesign CC 2024"`
  - `"LibreOffice Writer 7.0"`
  - `"Google Docs"`
  - `"Apple Pages"`
  - `"LaTeX with hyperref"`
- **Forgery Detection:** Inconsistent creator/producer combinations may indicate tampering

##### `analysis.metadata.producer`

- **Type:** `string | null`
- **Can Be Null:** Yes
- **Description:** Name of the software that generated the final PDF file
- **Null When:** PDF metadata does not specify a producer
- **Common Values:**
  - `"Adobe PDF Library 15.0"`
  - `"PDFKit"`
  - `"iText 7.0.0"`
  - `"Microsoft: Print To PDF"`
  - `"wkhtmltopdf 0.12.6"`
  - `"Skia/PDF m116"`
- **Most Common:** Usually indicates the PDF library or conversion tool used
- **Modification Indicator:** If producer suggests editing software (e.g., "Adobe Acrobat Pro DC"), file was likely edited

##### `analysis.metadata.file_size`

- **Type:** `number` (integer, bytes)
- **Always Present:** Yes
- **Range:** `1` to `10485760` (10 MB limit)
- **Description:** Total file size in bytes
- **Example:** `1048576` = 1 MB
- **Usage:** Useful for tracking storage or detecting unusual file sizes for page count

---

#### analysis.structure.\* Fields (PDF Structure)

##### `analysis.structure.has_incremental_updates`

- **Type:** `boolean`
- **Always Present:** Yes
- **Possible Values:**
  - `true` - PDF contains incremental update sections (indicates modification)
  - `false` - PDF has simple linear structure (likely original)
- **Description:** Whether the PDF file structure contains incremental updates
- **What It Means:**
  - PDFs with incremental updates have been saved multiple times, appending changes rather than rewriting the entire file
  - This is the most reliable structural indicator of modification
  - **Not always malicious:** Many legitimate workflows involve incremental saves (e.g., adding signatures)

##### `analysis.structure.update_chain_length`

- **Type:** `number` (integer)
- **Always Present:** Yes
- **Range:** `0` and higher
- **Description:** Number of update sections in the PDF structure
- **Interpretation:**
  - `0` - No updates (shouldn't occur)
  - `1` - Original file, no modifications
  - `2` - File was saved/modified once after creation
  - `3` - File was saved/modified twice
  - `4+` - File has undergone multiple edit sessions
- **Example:** A PDF created, then signed, then edited again would have `update_chain_length: 3`

##### `analysis.structure.xref_count`

- **Type:** `number` (integer)
- **Always Present:** Yes
- **Range:** `1` and higher
- **Description:** Number of cross-reference tables in the PDF
- **Typical Values:**
  - `1` - Single xref table (original file)
  - `2+` - Multiple xref tables (modified file)
- **Technical Detail:** Each save operation typically creates a new xref table
- **Relationship:** Usually equals `update_chain_length` but can differ in complex PDFs

##### `analysis.structure.pdf_version`

- **Type:** `string | null`
- **Can Be Null:** Yes
- **Description:** PDF specification version
- **Null When:** Version cannot be determined from file header
- **Common Values:**
  - `"1.3"` - PDF 1.3 (Acrobat 4.x)
  - `"1.4"` - PDF 1.4 (Acrobat 5.x) - supports transparency
  - `"1.5"` - PDF 1.5 (Acrobat 6.x) - supports layers
  - `"1.6"` - PDF 1.6 (Acrobat 7.x)
  - `"1.7"` - PDF 1.7 (Acrobat 8.x and later) - **most common**
  - `"2.0"` - PDF 2.0 (ISO 32000-2:2017) - modern standard
- **Note:** Higher versions support more features but are less compatible with older readers

---

#### analysis.signatures.\* Fields (Digital Signatures)

##### `analysis.signatures.has_digital_signature`

- **Type:** `boolean`
- **Always Present:** Yes
- **Possible Values:**
  - `true` - PDF contains one or more digital signatures
  - `false` - PDF is unsigned
- **Description:** Whether the PDF has been digitally signed
- **Important:** This only detects if signatures are **currently present**, not if they existed previously

##### `analysis.signatures.signature_count`

- **Type:** `number` (integer)
- **Always Present:** Yes
- **Range:** `0` and higher
- **Description:** Number of digital signature fields in the PDF
- **Common Values:**
  - `0` - No signatures
  - `1` - Single signature (most common)
  - `2+` - Multiple signatures (e.g., multi-party contracts)
- **Note:** Each signer typically adds one signature

##### `analysis.signatures.signature_removed`

- **Type:** `boolean`
- **Always Present:** Yes
- **Possible Values:**
  - `true` - **CRITICAL** - Evidence that a signature was removed
  - `false` - No evidence of removed signatures
- **Description:** Whether a digital signature was removed from the PDF
- **Detection Method:** Analyzes PDF structure for orphaned signature references or signature field remnants
- **Severity:** This is a **critical tampering indicator** - removing a signature usually means someone wanted to hide authentication
- **Impact on Risk:** Sets `risk_score` to ≥80 and `confidence_level` to `"very_high"`

##### `analysis.signatures.modifications_after_signature`

- **Type:** `boolean`
- **Always Present:** Yes
- **Possible Values:**
  - `true` - **CRITICAL** - PDF was modified after being signed
  - `false` - No modifications after signing, or no signatures present
- **Description:** Whether the document was modified after a digital signature was applied
- **What It Means:**
  - Digital signatures mathematically bind to the document content
  - Any modification after signing invalidates the signature
  - This indicates the signature can no longer be trusted
- **Common Causes:**
  - Adding text, images, or annotations after signing
  - Form field modifications
  - Metadata changes
  - Malicious tampering
- **Impact on Risk:** Significantly increases `risk_score`

---

#### analysis.threats.\* Fields (Security Threats)

##### `analysis.threats.has_javascript`

- **Type:** `boolean`
- **Always Present:** Yes
- **Possible Values:**
  - `true` - PDF contains JavaScript code
  - `false` - No JavaScript detected
- **Description:** Whether the PDF document contains embedded JavaScript
- **Security Concern:** JavaScript in PDFs can be used for:
  - **Legitimate:** Form validation, calculations, interactive features
  - **Malicious:** Exploits, data exfiltration, malware delivery
- **Recommendation:** PDFs with JavaScript should be treated with caution, especially from untrusted sources

##### `analysis.threats.has_embedded_files`

- **Type:** `boolean`
- **Always Present:** Yes
- **Possible Values:**
  - `true` - PDF has embedded/attached files
  - `false` - No embedded files
- **Description:** Whether the PDF contains embedded file attachments
- **Security Concern:** Embedded files can:
  - **Legitimate:** Portfolio PDFs, supporting documents
  - **Malicious:** Hide malware, ransomware, or exploits
- **Common Formats:** Can embed any file type (EXE, ZIP, PDF, Office docs, etc.)

---

### Example Responses

#### Example 1: Modified Document (High Risk)

```json
{
  "id": "3f9c8b7a-2e1d-4c5f-9b8e-7a6d5c4b3a21",
  "analysis": {
    "status": "modified",
    "been_changed": true,
    "risk_score": 75,
    "confidence_level": "high",
    "verdict_reasoning": "Digital signature was removed from the document",
    "origin": {
      "type": "institutional",
      "software": null,
      "warning": null
    },
    "metadata": {
      "creation_date": "2024-01-15T10:30:00.000Z",
      "modification_date": "2024-02-20T14:45:00.000Z",
      "creator": "Adobe Acrobat Pro DC",
      "producer": "Microsoft Word for Microsoft 365",
      "file_size": 1048576
    },
    "structure": {
      "has_incremental_updates": true,
      "update_chain_length": 3,
      "xref_count": 2,
      "pdf_version": "1.7"
    },
    "signatures": {
      "has_digital_signature": false,
      "signature_count": 0,
      "signature_removed": true,
      "modifications_after_signature": false
    },
    "threats": {
      "has_javascript": false,
      "has_embedded_files": false
    },
    "findings": [
      "Document modified after creation",
      "Digital signature removed",
      "Multiple update chains detected"
    ]
  }
}
```

#### Example 2: Original Document (Intact)

```json
{
  "id": "7d8e9f2a-4c5b-6d7e-8f9a-0b1c2d3e4f5a",
  "analysis": {
    "status": "intact",
    "been_changed": false,
    "risk_score": 0,
    "confidence_level": "high",
    "origin": {
      "type": "institutional",
      "software": null,
      "warning": null
    },
    "metadata": {
      "creation_date": "2024-03-10T08:15:00.000Z",
      "modification_date": "2024-03-10T08:15:00.000Z",
      "creator": "Adobe Acrobat Pro DC",
      "producer": "Adobe PDF Library 15.0",
      "file_size": 245632
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
    "threats": {
      "has_javascript": false,
      "has_embedded_files": false
    },
    "findings": []
  }
}
```

#### Example 3: Inconclusive (Consumer Software Origin)

```json
{
  "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "analysis": {
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
    "threats": {
      "has_javascript": false,
      "has_embedded_files": false
    },
    "findings": []
  }
}
```

#### Example 4: Minimal Metadata (Medium Confidence)

```json
{
  "id": "506a6b1b-1360-48a2-b389-abb346f85d04",
  "analysis": {
    "status": "modified",
    "been_changed": true,
    "risk_score": 45,
    "confidence_level": "medium",
    "origin": {
      "type": "unknown",
      "software": null,
      "warning": null
    },
    "metadata": {
      "creation_date": null,
      "modification_date": null,
      "creator": null,
      "producer": "PDFKit",
      "file_size": 523145
    },
    "structure": {
      "has_incremental_updates": true,
      "update_chain_length": 2,
      "xref_count": 2,
      "pdf_version": "1.4"
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
    "findings": ["Incremental updates detected", "Creation date missing from PDF metadata"]
  }
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

Returned when the request is malformed or contains invalid parameters.

#### Missing or Invalid url

```json
{
  "error": "Missing or invalid url parameter"
}
```

**Cause:** The `url` field is not present in the request body, or it's not a string.

**Solution:** Ensure you're sending a valid JSON body with a `url` string field.

---

#### Invalid URL Format

```json
{
  "error": "Invalid url format"
}
```

**Cause:** The `url` value is not a valid HTTP/HTTPS URL.

**Examples of Invalid URLs:**

- `not-a-url`
- `ftp://example.com/file.pdf`
- `file:///local/path.pdf`
- `example.com/file.pdf` (missing protocol)

**Solution:** Use a complete HTTP or HTTPS URL like `https://example.com/file.pdf`

---

#### Failed to Download File

```json
{
  "error": "Failed to download file from URL",
  "details": "Network error" // or specific error message
}
```

**Common Causes:**

- URL returns 404 Not Found
- URL returns 403 Forbidden
- URL requires authentication
- Network timeout (30 second limit)
- DNS resolution failure
- Server is unreachable

**HTTP Status Details Examples:**

```json
{
  "error": "Failed to download file from URL",
  "details": "HTTP 404: Not Found"
}
```

```json
{
  "error": "Failed to download file from URL",
  "details": "HTTP 403: Forbidden"
}
```

**Solution:** Ensure the URL is publicly accessible and returns the file successfully.

---

### 401 Unauthorized

Authentication failed.

#### Missing API Key

```json
{
  "error": "Missing API key. Please provide an API key in the Authorization header.",
  "code": "missing_api_key"
}
```

**Cause:** No `Authorization` header provided.

**Solution:** Add header: `Authorization: Bearer YOUR_API_KEY`

---

#### Invalid API Key

```json
{
  "error": "Invalid API key. Please check your credentials.",
  "code": "invalid_api_key"
}
```

**Cause:** API key not found in database or has been revoked.

**Solution:**

- Verify your API key is correct
- Check that you're using a `htpbe_live_*` or `htpbe_test_*` key
- Generate a new API key from the dashboard if needed

---

### 403 Forbidden

Access forbidden due to account status.

#### Inactive Client

```json
{
  "error": "This API key has been deactivated. Please contact support.",
  "code": "inactive_client"
}
```

**Cause:** Your API client account has been disabled (usually due to payment issues or terms violation).

**Solution:** Contact support to reactivate your account.

---

### 413 Payload Too Large

File exceeds size limit.

```json
{
  "error": "File size exceeds limit",
  "details": "Maximum file size is 10 MB, received 15 MB"
}
```

**Cause:** PDF file is larger than 10 MB (10,485,760 bytes).

**Solution:**

- Compress the PDF using tools like Adobe Acrobat or online compressors
- Split large PDFs into smaller files
- Remove high-resolution images
- Contact support for Enterprise plan with higher limits

---

### 402 Payment Required

Your free trial has ended and a payment method is required to continue.

```json
{
  "error": "Your free trial has ended. Please upgrade your subscription to continue using the API.",
  "code": "payment_required"
}
```

**Cause:** Your 14-day free trial has expired and no active subscription exists.

**What This Means:**

- All paid plans include overage billing - your API **never stops** once you add a payment method
- After adding payment, you'll never see this error again
- Overage requests beyond your plan quota are billed at:
  - **Starter:** $0.60 per request
  - **Growth:** $0.50 per request
  - **Pro:** $0.40 per request
  - **Enterprise:** Custom pricing

**Solution:**

1. Log in to your dashboard at https://htpbe.tech/dashboard
2. Add a payment method (credit card or ACH)
3. Your API access will resume immediately
4. Future requests will work seamlessly, even if you exceed your plan quota

**Note:** Once you have an active payment method, the API **never blocks you** - overage charges apply automatically.

---

### 422 Unprocessable Entity

Request is valid but the content cannot be processed.

#### Invalid PDF File

```json
{
  "error": "Invalid PDF file",
  "details": "PDF header not found or file is corrupted"
}
```

**Common Causes:**

- File is not actually a PDF (wrong file type)
- PDF file is corrupted or truncated
- File is encrypted with a password
- PDF uses unsupported features

**Solution:**

- Verify the file opens correctly in a PDF reader
- Try re-saving the file
- Remove encryption/password protection
- If the file is valid, contact support with the file URL for investigation

---

### 500 Internal Server Error

Server-side error during processing.

```json
{
  "error": "Failed to analyze PDF",
  "details": "Unexpected error during PDF parsing"
}
```

**Cause:** Unexpected server error. This is rare and usually indicates a bug.

**Solution:**

- Retry the request (may be a transient error)
- If persists, contact support with the `url` for investigation
- Check our status page for any ongoing incidents

---

## Testing with Mock URLs

Test keys (`htpbe_test_...`) only work with predefined test URLs. These behave like Stripe test cards — each URL returns a deterministic mock response at no cost and without downloading any real file.

**Test keys cannot be used with real PDF URLs** — attempting to do so returns a 403 error.

```bash
curl -X POST https://htpbe.tech/api/v1/analyze \
  -H "Authorization: Bearer htpbe_test_YOUR_TEST_KEY" \
  -H "Content-Type: application/json" \
  -d '{"url": "https://htpbe.tech/api/v1/test/modified-high.pdf"}'
```

**Available mock URLs** (all at `https://htpbe.tech/api/v1/test/`):

| URL                       | `status`       | `been_changed` | `risk_score` | Description                                                 |
| ------------------------- | -------------- | -------------- | ------------ | ----------------------------------------------------------- |
| `clean.pdf`               | `intact`       | `false`        | `0`          | Original, no modifications                                  |
| `clean-no-dates.pdf`      | `intact`       | `false`        | `0`          | Original, metadata dates absent                             |
| `modified-low.pdf`        | `modified`     | `true`         | `25`         | Minor modification (1 incremental update)                   |
| `modified-medium.pdf`     | `modified`     | `true`         | `50`         | Moderate modification (creator/producer mismatch)           |
| `modified-high.pdf`       | `modified`     | `true`         | `75`         | Significant modification (multiple updates, tool change)    |
| `modified-critical.pdf`   | `modified`     | `true`         | `95`         | Critical: signature removed + JavaScript detected           |
| `dates-mismatch.pdf`      | `modified`     | `true`         | `40`         | Dates differ (14-day gap between creation and modification) |
| `dates-same.pdf`          | `inconclusive` | `false`        | `0`          | LibreOffice origin — integrity check not applicable         |
| `incremental-updates.pdf` | `modified`     | `true`         | `60`         | 6 incremental update sections detected                      |
| `multiple-xref.pdf`       | `modified`     | `true`         | `55`         | 4 cross-reference tables                                    |
| `signature-valid.pdf`     | `intact`       | `false`        | `0`          | Digitally signed, no post-sign modifications                |
| `signature-removed.pdf`   | `modified`     | `true`         | `90`         | Critical: digital signature removed                         |
| `modified-after-sign.pdf` | `modified`     | `true`         | `85`         | Modified after digital signing (signature invalidated)      |
| `javascript.pdf`          | `modified`     | `true`         | `70`         | Contains embedded JavaScript                                |
| `embedded-files.pdf`      | `modified`     | `true`         | `65`         | Contains embedded file attachments                          |
| `both-threats.pdf`        | `modified`     | `true`         | `95`         | JavaScript + embedded files + signature removed             |
| `inconclusive.pdf`        | `inconclusive` | `false`        | `0`          | Microsoft Excel origin — integrity check not applicable     |

---

## Notes

### Processing Time

- **Typical:** 2-5 seconds for average PDFs (1-20 pages)
- **Larger files:** 5-15 seconds for complex PDFs (50-100 pages)
- **Timeout:** 30 seconds maximum. Requests exceeding this will fail with a download error.

### Supported PDF Features

- ✅ PDF versions 1.0 through 2.0
- ✅ Encrypted PDFs (user password required to be removed before analysis)
- ✅ Linearized (Fast Web View) PDFs
- ✅ PDFs with forms, annotations, and multimedia
- ✅ Signed PDFs (detects but doesn't validate signatures)

### Limitations

- ❌ Cannot validate cryptographic signatures (only detects presence/removal)
- ❌ Cannot access password-protected PDFs
- ❌ Cannot analyze files behind authentication
- ❌ Does not perform OCR or content analysis
- ❌ Does not detect visual changes (only structural/metadata changes)

---

**Related Endpoints:**

- [GET /api/v1/result/{uid}](./result.md) - Retrieve full analysis results
- [GET /api/v1/stats](./stats.md) - Get aggregate statistics
- [GET /api/v1/checks](./checks.md) - List all checks with filtering for custom analytics

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

## Overage Billing

Requests beyond your monthly quota are charged at your plan's overage rate and billed automatically at the end of your billing cycle. There is no hard block when your quota is reached — requests continue to succeed and overage charges appear on your next invoice.

| Plan       | Overage rate |
| ---------- | ------------ |
| Starter    | $0.60/req    |
| Growth     | $0.50/req    |
| Pro        | $0.40/req    |
| Enterprise | Included     |

---

## Request

### Headers

| Header          | Type   | Required | Description                                                                                                                                                                                                                                                                |
| --------------- | ------ | -------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `Authorization` | string | **Yes**  | Bearer token with your API key (`Bearer htpbe_live_...` or `Bearer htpbe_test_...`). The `Bearer` prefix is recommended but optional — sending the raw key directly is also accepted, but only if the key starts with `htpbe_` (e.g., `Authorization: htpbe_live_sk_...`). |
| `Content-Type`  | string | **Yes**  | Must be `application/json`                                                                                                                                                                                                                                                 |

### Body Parameters

| Parameter           | Type   | Required | Description                                                                                                                                                                 |
| ------------------- | ------ | -------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `url`               | string | **Yes**  | Publicly accessible HTTP/HTTPS URL pointing to a PDF file. The file must be downloadable without authentication and must not exceed 10 MB in size.                          |
| `original_filename` | string | No       | Original filename of the document (before any storage renaming). When provided, this name is stored and returned in results instead of the filename extracted from the URL. |

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

### Success Response (201 Created)

Analysis is performed synchronously. The response contains only the check ID. A `Location` response header points to the full result URL. Call `GET /api/v1/result/{id}` immediately after to retrieve the full analysis results.

#### Response Headers

| Header     | Description                                                                                                                                    |
| ---------- | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| `Location` | Full URL of the result: `https://htpbe.tech/api/v1/result/{id}` (e.g. `https://htpbe.tech/api/v1/result/3f9c8b7a-2e1d-4c5f-9b8e-7a6d5c4b3a21`) |

#### Response Structure

```typescript
{
  id: string;
}
```

### Response Fields

##### `id`

- **Type:** `string` (UUID v4)
- **Always Present:** Yes
- **Description:** Unique identifier for this analysis check
- **Format:** `xxxxxxxx-xxxx-4xxx-xxxx-xxxxxxxxxxxx` (UUID version 4)
- **Usage:** Pass this ID to `GET /api/v1/result/{id}` to retrieve the full analysis
- **Example:** `"3f9c8b7a-2e1d-4c5f-9b8e-7a6d5c4b3a21"`

### Example Response

```json
{
  "id": "3f9c8b7a-2e1d-4c5f-9b8e-7a6d5c4b3a21"
}
```

### Two-Step Usage Pattern

```javascript
// Step 1: Submit PDF for analysis
const analyzeRes = await fetch('https://htpbe.tech/api/v1/analyze', {
  method: 'POST',
  headers: {
    Authorization: `Bearer ${process.env.HTPBE_API_KEY}`,
    'Content-Type': 'application/json',
  },
  body: JSON.stringify({ url: 'https://example.com/documents/contract.pdf' }),
});

const { id } = await analyzeRes.json();

// Step 2: Retrieve full results
const resultRes = await fetch(`https://htpbe.tech/api/v1/result/${id}`, {
  headers: { Authorization: `Bearer ${process.env.HTPBE_API_KEY}` },
});

const result = await resultRes.json();
// result.status: "intact" | "modified" | "inconclusive"
// ... (see /result endpoint docs for full schema)
```

See [GET /api/v1/result/{id}](./result.md) for the full result schema, including verdict fields (`status`, `modification_confidence`, `modification_markers`), metadata, structure, signatures, threats, and findings.

---

### Understanding the verdict

The verdict fields (`status`, `modification_confidence`, `modification_markers`) are available in the result endpoint. See [GET /api/v1/result/{id}](./result.md) for details.

**How `status: "modified"` is determined:**

The verdict is set to `"modified"` when any of the following critical markers are found:

1. Incremental update sections detected in PDF structure
2. Modifications found after digital signature was applied
3. Digital signature was removed
4. Both creation and modification dates are present and differ

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

Returned when the request is malformed or contains invalid parameters.

#### Invalid JSON Body

```json
{
  "error": "Invalid JSON body",
  "code": "invalid_request"
}
```

**Cause:** The request body is not valid JSON (missing braces, trailing commas, unquoted keys, etc.).

**Solution:** Ensure the request body is valid JSON and the `Content-Type: application/json` header is set.

---

#### Missing or Invalid url

```json
{
  "error": "Missing or invalid url parameter",
  "code": "invalid_request"
}
```

**Cause:** The `url` field is not present in the request body, or it's not a string.

**Solution:** Ensure you're sending a valid JSON body with a `url` string field.

---

#### Invalid URL Format

```json
{
  "error": "Invalid url format",
  "code": "invalid_url_format"
}
```

**Cause:** The `url` value is not a valid HTTP or HTTPS URL.

**Examples of Invalid URLs:**

- `not-a-url`
- `example.com/file.pdf` (missing protocol)
- `file:///local/path.pdf` (local path)

**Note:** `ftp://` URLs are rejected with this error — they fail the HTTP/HTTPS protocol check before any download is attempted.

**Solution:** Use a complete HTTP or HTTPS URL like `https://example.com/file.pdf`

---

#### Failed to Download File

```json
{
  "error": "Failed to download file from URL",
  "code": "download_failed",
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
  "code": "download_failed",
  "details": "HTTP 404: Not Found"
}
```

```json
{
  "error": "Failed to download file from URL",
  "code": "download_failed",
  "details": "HTTP 403: Forbidden"
}
```

**Solution:** Ensure the URL is publicly accessible and returns the file successfully.

> **Why 400 and not 422?** This is intentional. HTTP 422 (Unprocessable Entity) means the request syntax is valid but the server cannot process the _content_. Here the problem is the client's _input_ — the URL they supplied cannot be fetched. This is treated as a bad-input error (400), consistent with how invalid URL format is handled. Do not add 422 handling for this error code.

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

#### Invalid API Key (live key)

```json
{
  "error": "Invalid API key. Please check your credentials.",
  "code": "invalid_api_key"
}
```

**Cause:** Live key (`htpbe_live_*`) not found in database or has been revoked.

**Solution:**

- Verify your API key is correct
- Generate a new API key from the dashboard if needed

---

#### Invalid API Key (test key)

```json
{
  "error": "Invalid test API key. Please check your credentials.",
  "code": "invalid_api_key"
}
```

**Cause:** Test key (`htpbe_test_*`) not found. Test keys are stored separately from live keys and cannot be used interchangeably.

**Solution:**

- Copy your test key from the **Test API Key** section on the [dashboard](https://htpbe.tech/dashboard/api-keys)
- Verify you are not using a live key where a test key is expected (or vice versa)

---

### 402 Payment Required

No active subscription found for this API key.

```json
{
  "error": "No active subscription. Please subscribe to a plan to use the API.",
  "code": "payment_required"
}
```

**Cause:** Your account does not have an active paid plan.

**Solution:**

1. Log in to your dashboard at https://htpbe.tech/dashboard
2. Subscribe to a plan (Starter, Growth, or Pro)
3. Your API access will resume immediately

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

#### Test URL Required (test keys only)

```json
{
  "error": "Test API keys can only be used with test URLs. See documentation for available test URLs.",
  "code": "test_url_required",
  "details": "Use URLs like: https://htpbe.tech/api/v1/test/clean.pdf, https://htpbe.tech/api/v1/test/modified-high.pdf, etc."
}
```

**Cause:** A test API key (`htpbe_test_...`) was used with a real URL. Test keys are restricted to the predefined test URLs — they cannot download or analyze real PDFs.

**Solution:** Use one of the 21 mock test URLs listed in [Testing with Test API Keys](../testing.md), or switch to a live API key (`htpbe_live_...`) to analyze real files.

---

### 413 Payload Too Large

File exceeds size limit.

```json
{
  "error": "File size exceeds limit",
  "code": "file_too_large",
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

### 422 Unprocessable Entity

Request is valid but the content cannot be processed.

#### Invalid PDF File

```json
{
  "error": "Invalid PDF file",
  "code": "invalid_pdf",
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
  "code": "internal_error"
}
```

**Cause:** Unexpected server error. This is rare and usually indicates a bug.

**Solution:**

- Retry the request (may be a transient error)
- If persists, contact support with the `url` for investigation
- Check our status page for any ongoing incidents

---

## Testing

Use test API keys (`htpbe_test_...`) to integrate without consuming quota or analyzing real PDFs. Test keys return deterministic UUID v4 check IDs (e.g., `00000000-0000-4000-8000-000000000001` for `clean.pdf`, `00000000-0000-4000-8000-000000000005` for `modified-high.pdf`) for the 21 predefined mock URLs + 1 error trigger URL — no real file is downloaded and no quota is consumed.

See [Testing with Test API Keys](../testing.md) for the complete test URL list, synthetic ID table, code examples, and checklist.

---

## Notes

### Processing Time

- **Typical:** 2-5 seconds for average PDFs (1-20 pages)
- **Larger files:** 5-15 seconds for complex PDFs (50-100 pages)
- **Timeout:** 30 seconds maximum. Requests exceeding this will fail with a download error.

### Supported PDF Features

- ✅ PDF versions 1.0 through 2.0
- ✅ Linearized (Fast Web View) PDFs
- ✅ PDFs with forms, annotations, and multimedia
- ✅ Signed PDFs (detects but doesn't validate signatures)

### Limitations

- ❌ Cannot validate cryptographic signatures (only detects presence/removal)
- ❌ Password-protected PDFs — the password must be removed before submitting for analysis
- ❌ Cannot analyze files behind authentication
- ❌ Does not perform OCR or content analysis
- ❌ Does not detect visual changes (only structural/metadata changes)

---

**Related Endpoints:**

- [GET /api/v1/result/{id}](./result.md) - Retrieve full analysis results
- [GET /api/v1/checks](./checks.md) - List all checks with filtering for custom analytics

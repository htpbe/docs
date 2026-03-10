# GET /api/v1/checks

Retrieve a paginated list of all your PDF check results with flexible filtering options. This endpoint provides raw data access for custom analytics, exports, and advanced reporting.

## Endpoint

```
GET https://htpbe.tech/api/v1/checks
```

## Authentication

This endpoint requires API key authentication via the `Authorization` header.

```http
Authorization: Bearer YOUR_API_KEY
```

## Data Isolation

**Important:** Results include **only your checks**, not other clients' data. Each API client sees only their own check history, ensuring complete data privacy.

---

## Request

### Headers

| Header          | Type   | Required | Description                                                                         |
| --------------- | ------ | -------- | ----------------------------------------------------------------------------------- |
| `Authorization` | string | **Yes**  | Bearer token with your API key (`Bearer htpbe_live_...` or `Bearer htpbe_test_...`) |

### Query Parameters

| Parameter   | Type    | Required | Description                                                                       |
| ----------- | ------- | -------- | --------------------------------------------------------------------------------- |
| `limit`     | integer | No       | Number of results per page (1-500). Default: `100`                                |
| `offset`    | integer | No       | Number of results to skip for pagination. Default: `0`                            |
| `tool`      | string  | No       | Filter by tool name (matches Creator OR Producer). Case-sensitive, exact match.   |
| `creator`   | string  | No       | Filter by Creator tool only. Case-sensitive, exact match.                         |
| `producer`  | string  | No       | Filter by Producer tool only. Case-sensitive, exact match.                        |
| `modified`  | boolean | No       | Filter by modification status. `true` = only modified, `false` = only unmodified. |
| `from_date` | integer | No       | Unix timestamp (seconds). Return checks created on or after this date.            |
| `to_date`   | integer | No       | Unix timestamp (seconds). Return checks created on or before this date.           |

#### Query Parameter Details

##### Pagination Parameters

**`limit`**

- Range: 1-500
- Default: 100
- Use smaller values (50-100) for faster responses
- Use larger values (500) for bulk exports

**`offset`**

- Default: 0
- Use with `limit` for pagination
- Example: `limit=100&offset=200` returns results 201-300

##### Filter Parameters

**`tool`** (Broad Filter)

- Matches if tool name appears as **either** Creator **or** Producer
- Example: `tool=Adobe PDF Library 15.0` finds PDFs where this tool was Creator OR Producer
- Use this when you don't care about the role

**`creator`** / **`producer`** (Specific Filters)

- Match only specific roles
- Example: `creator=Microsoft Word for Microsoft 365` finds only PDFs created in Word
- Can combine: `creator=Word&producer=Adobe PDF Library` finds Word docs converted with Adobe

**`modified`**

- `true` = only checks where modification was detected (`status: "modified"` or `"inconclusive"`)
- `false` = only clean checks (`status: "intact"`)
- Omit parameter to get all

**`from_date`** / **`to_date`**

- Unix timestamps in seconds
- Filters by PDF creation date (not check date)
- Example: `from_date=1735689600` = PDFs created on/after Jan 1, 2025
- Use together for date ranges

### Example Requests

#### Basic Usage - Get First Page

```bash
curl -X GET 'https://htpbe.tech/api/v1/checks?limit=100&offset=0' \
  -H "Authorization: Bearer htpbe_live_sk_1234567890abcdef"
```

#### Filter by Tool

```bash
# Find all checks involving Adobe PDF Library (as Creator OR Producer)
curl -X GET 'https://htpbe.tech/api/v1/checks?tool=Adobe%20PDF%20Library%2015.0&limit=500' \
  -H "Authorization: Bearer htpbe_live_sk_1234567890abcdef"
```

#### Filter by Modification Status

```bash
# Get only modified PDFs
curl -X GET 'https://htpbe.tech/api/v1/checks?modified=true&limit=200' \
  -H "Authorization: Bearer htpbe_live_sk_1234567890abcdef"
```

#### Filter by Date Range

```bash
# Get PDFs created in February 2026
curl -X GET 'https://htpbe.tech/api/v1/checks?from_date=1738368000&to_date=1740960000' \
  -H "Authorization: Bearer htpbe_live_sk_1234567890abcdef"
```

#### Complex Filter Combination

```bash
# Modified Word documents from January 2026
curl -X GET 'https://htpbe.tech/api/v1/checks?creator=Microsoft%20Word%20for%20Microsoft%20365&modified=true&from_date=1735689600&to_date=1738368000' \
  -H "Authorization: Bearer htpbe_live_sk_1234567890abcdef"
```

#### JavaScript / TypeScript

```javascript
// Node.js / TypeScript
const apiKey = process.env.HTPBE_API_KEY;

// Paginate through all checks
async function getAllChecks() {
  let allChecks = [];
  let offset = 0;
  const limit = 500;
  let hasMore = true;

  while (hasMore) {
    const response = await fetch(
      `https://htpbe.tech/api/v1/checks?limit=${limit}&offset=${offset}`,
      {
        headers: { Authorization: `Bearer ${apiKey}` },
      }
    );

    const data = await response.json();
    allChecks.push(...data.checks);
    hasMore = data.hasMore;
    offset += limit;
  }

  return allChecks;
}

// Get all checks and filter for specific tool
const checks = await getAllChecks();
const adobeChecks = checks.filter(
  (c) => c.creator === 'Adobe PDF Library 15.0' || c.producer === 'Adobe PDF Library 15.0'
);
```

#### Python

```python
# Python
import requests
import os

api_key = os.getenv('HTPBE_API_KEY')

def get_all_checks(filters=None):
    """Retrieve all checks with optional filters"""
    all_checks = []
    offset = 0
    limit = 500
    has_more = True

    while has_more:
        params = {'limit': limit, 'offset': offset}
        if filters:
            params.update(filters)

        response = requests.get(
            'https://htpbe.tech/api/v1/checks',
            params=params,
            headers={'Authorization': f'Bearer {api_key}'}
        )

        data = response.json()
        all_checks.extend(data['checks'])
        has_more = data['hasMore']
        offset += limit

    return all_checks

# Example: Get all modified PDFs
modified_pdfs = get_all_checks({'modified': True})

# Example: Get modified PDFs from specific tool
risky_adobe = get_all_checks({
    'tool': 'Adobe PDF Library 15.0',
    'modified': True
})
```

---

## Response

### Success Response (200 OK)

Returns a paginated list of check results with metadata for navigation.

#### Response Structure

```typescript
{
  checks: Array<Check>;
  total: number;
  limit: number;
  offset: number;
  hasMore: boolean;
}
```

#### Response Fields

##### `checks`

- **Type:** `Array<Check>` (array of check objects)
- **Always Present:** Yes (can be empty array)
- **Max Length:** Equals `limit` parameter (or less on last page)
- **Description:** Array of PDF check results
- **Empty When:** No checks match the filters, or offset is beyond total results

##### `total`

- **Type:** `number` (integer)
- **Always Present:** Yes
- **Range:** `0` and higher
- **Description:** Total number of checks matching the filters (not just this page)
- **Example:** `1250` = 1,250 total checks match your filters
- **Usage:** Use to calculate total pages: `Math.ceil(total / limit)`

##### `limit`

- **Type:** `number` (integer)
- **Always Present:** Yes
- **Range:** `1` to `500`
- **Description:** Echo of the `limit` query parameter (or default 100)
- **Usage:** Confirms how many results were requested per page

##### `offset`

- **Type:** `number` (integer)
- **Always Present:** Yes
- **Range:** `0` and higher
- **Description:** Echo of the `offset` query parameter (or default 0)
- **Usage:** Current position in the result set

##### `hasMore`

- **Type:** `boolean`
- **Always Present:** Yes
- **Description:** Whether there are more results beyond this page
- **Calculation:** `offset + checks.length < total`
- **Usage:** Use in pagination loops to know when to stop
- **Example:**
  - `true` = more pages available, continue pagination
  - `false` = this is the last page, stop pagination

---

### Check Object Structure

Each item in the `checks` array has the following structure:

```typescript
{
  // Identification
  uid: string;
  filename: string;
  check_date: number;

  // Verdict
  status: 'intact' | 'modified' | 'inconclusive';
  metadata_score: number;

  // Tools
  creator: string | null;
  producer: string | null;

  // File Properties
  file_size: number;
  page_count: number;
  pdf_version: string | null;

  // Dates (Unix timestamps)
  creation_date: number | null;
  modification_date: number | null;

  // Features
  has_javascript: boolean;
  is_signed: boolean;
  has_embedded_files: boolean;
  has_incremental_updates: boolean;

  // Structure
  update_chain_length: number;
  total_objects: number;

  // Detection Details
  suspicious_patterns: number;
}
```

### Check Object Fields

#### Identification Fields

##### `uid`

- **Type:** `string`
- **Always Present:** Yes
- **Format:** 8-character alphanumeric string (first 8 characters of the full UUID)
- **Example:** `"a3f5c9d2"`
- **Usage:** Display identifier only — use in dashboards or logs for a short human-readable reference
- **Important:** This is NOT the full check ID. To retrieve full analysis details via `GET /api/v1/result/{id}`, you must use the complete UUID returned by `POST /api/v1/analyze` (format: `xxxxxxxx-xxxx-4xxx-xxxx-xxxxxxxxxxxx`). Store that UUID when you create the check.

##### `filename`

- **Type:** `string`
- **Always Present:** Yes
- **Description:** Original filename of the uploaded PDF
- **Example:** `"invoice-2024-01.pdf"`
- **Note:** User-provided, not validated

##### `check_date`

- **Type:** `number` (Unix timestamp in seconds)
- **Always Present:** Yes
- **Description:** When this PDF was analyzed
- **Example:** `1738368000` = February 1, 2026, 00:00:00 UTC
- **Convert:** JavaScript: `new Date(check_date * 1000).toISOString()`
- **Usage:** Sort checks by analysis date, filter by submission time

---

#### Verdict Fields

##### `status`

- **Type:** `"intact" | "modified" | "inconclusive"`
- **Always Present:** Yes
- **Description:** Verdict for this check
- **Values:**
  - `"modified"` = forensic evidence of post-creation modification found
  - `"intact"` = no modification indicators detected; origin appears institutional
  - `"inconclusive"` = PDF created with consumer software (Word, LibreOffice, etc.); integrity check does not apply
- **Note:** For full verdict details (`critical_modification_marker`, `verdict_reasoning`, `origin`), retrieve via `GET /api/v1/result/{id}`

##### `metadata_score`

- **Type:** `number` (integer)
- **Always Present:** Yes
- **Range:** `0` to `100`
- **Description:** Metadata completeness and quality score
- **Interpretation:**
  - `90-100` - Excellent metadata (all fields present)
  - `70-89` - Good metadata coverage
  - `50-69` - Moderate metadata
  - `30-49` - Limited metadata (harder to verify)
  - `0-29` - Poor/missing metadata
- **Note:** Low metadata score doesn't mean modification, just harder to analyze

---

#### Tool Fields

##### `creator`

- **Type:** `string | null`
- **Can Be Null:** Yes (if PDF lacks Creator metadata)
- **Description:** Software that created the original document
- **Examples:**
  - `"Microsoft Word for Microsoft 365"`
  - `"Adobe InDesign CC"`
  - `"LibreOffice Writer"`
  - `"Google Docs"`
  - `null` - No creator metadata
- **Usage:** Understand document origin, identify workflows

##### `producer`

- **Type:** `string | null`
- **Can Be Null:** Yes (if PDF lacks Producer metadata)
- **Description:** Software that generated the final PDF file
- **Examples:**
  - `"Adobe PDF Library 15.0"`
  - `"Microsoft: Print To PDF"`
  - `"PDFKit"`
  - `"iText 7.0.0"`
  - `null` - No producer metadata
- **Usage:** Identify PDF generation tools, detect suspicious tool combinations

---

#### File Property Fields

##### `file_size`

- **Type:** `number` (integer, bytes)
- **Always Present:** Yes
- **Range:** `1` to `10485760` (10 MB API limit)
- **Description:** PDF file size in bytes
- **Example:** `524288` = 512 KB
- **Convert to KB:** `file_size / 1024`
- **Convert to MB:** `file_size / 1024 / 1024`

##### `page_count`

- **Type:** `number` (integer)
- **Always Present:** Yes
- **Range:** `1` and higher
- **Description:** Number of pages in the PDF
- **Example:** `15` = 15-page document
- **Usage:** Correlate document size with modification likelihood

##### `pdf_version`

- **Type:** `string | null`
- **Can Be Null:** Yes (rare)
- **Description:** PDF specification version
- **Examples:**
  - `"1.7"` - Most common (Acrobat 8.x+)
  - `"1.4"` - Older standard (Acrobat 5.x)
  - `"1.5"` - Acrobat 6.x
  - `"2.0"` - Modern ISO standard
  - `null` - Version not specified
- **Usage:** Identify old PDFs, technology compatibility

---

#### Date Fields (Unix Timestamps)

##### `creation_date`

- **Type:** `number | null` (Unix timestamp in seconds)
- **Can Be Null:** Yes (if PDF lacks CreationDate)
- **Description:** When the PDF was originally created (from metadata)
- **Example:** `1735689600` = January 1, 2025, 00:00:00 UTC
- **Convert:** JavaScript: `new Date(creation_date * 1000).toISOString()`
- **Note:** Can be forged or missing

##### `modification_date`

- **Type:** `number | null` (Unix timestamp in seconds)
- **Can Be Null:** Yes (if PDF lacks ModDate)
- **Description:** When the PDF was last modified (from metadata)
- **Example:** `1738368000` = February 1, 2026, 00:00:00 UTC
- **Note:** Can be forged or missing
- **Usage:** Compare with `creation_date` to detect editing time gaps

---

#### Feature Fields (Booleans)

##### `has_javascript`

- **Type:** `boolean`
- **Always Present:** Yes
- **Description:** Whether PDF contains embedded JavaScript code
- **Values:**
  - `true` = JavaScript present (potential security risk)
  - `false` = No JavaScript detected
- **Security Note:** JavaScript can be used for malicious purposes

##### `is_signed`

- **Type:** `boolean`
- **Always Present:** Yes
- **Description:** Whether PDF contains digital signatures
- **Values:**
  - `true` = Digitally signed
  - `false` = Not signed
- **Note:** Signature validity is NOT checked (only presence)
- **Cross-reference:** If `is_signed=true` AND `status="modified"` = signature likely invalidated

##### `has_embedded_files`

- **Type:** `boolean`
- **Always Present:** Yes
- **Description:** Whether PDF contains embedded file attachments
- **Values:**
  - `true` = Has attachments (potential security risk)
  - `false` = No attachments
- **Security Note:** Attachments can hide malware

##### `has_incremental_updates`

- **Type:** `boolean`
- **Always Present:** Yes
- **Description:** Whether PDF has incremental update structure (been saved multiple times)
- **Values:**
  - `true` = PDF has been edited and saved incrementally
  - `false` = PDF was generated in one operation
- **Note:** Not always malicious - many legitimate workflows create incremental updates

---

#### Structure Fields

##### `update_chain_length`

- **Type:** `number` (integer)
- **Always Present:** Yes
- **Range:** `1` and higher
- **Description:** Number of times PDF has been saved (update chain length)
- **Example:** `12` = document was edited and saved 12 times
- **Interpretation:**
  - `1` = Original, never edited
  - `2-5` = Light editing
  - `6-10` = Moderate editing
  - `11+` = Heavy editing
- **Usage:** Identify heavily modified documents

##### `total_objects`

- **Type:** `number` (integer)
- **Always Present:** Yes
- **Range:** `1` and higher
- **Description:** Total number of internal PDF objects (pages, fonts, images, etc.)
- **Example:** `834` = 834 internal objects
- **Usage:** Measure document complexity

---

#### Detection Detail Fields

##### `suspicious_patterns`

- **Type:** `number` (integer)
- **Always Present:** Yes
- **Range:** `0` or `1` only
- **Description:** Whether suspicious tool patterns were detected (e.g., creator/producer combination indicating potential forgery)
- **Possible Values:**
  - `0` = No suspicious patterns detected
  - `1` = Suspicious tool pattern detected (anachronistic tools, mismatched versions, known forgery patterns)
- **Examples of Suspicious Patterns:**
  - Creator: "Microsoft Word 2003" + Producer: "Adobe PDF Library 15.0" (anachronistic)
  - Mismatched tool versions or capabilities
  - Generic tool names with specific version numbers
- **Usage:** Flag documents with value `1` for manual review

---

### Example Response

```json
{
  "checks": [
    {
      "uid": "a3f5c9d2",
      "filename": "invoice-2024-01.pdf",
      "check_date": 1738368000,
      "status": "modified",
      "metadata_score": 85,
      "creator": "Microsoft Word for Microsoft 365",
      "producer": "Adobe PDF Library 15.0",
      "file_size": 524288,
      "page_count": 5,
      "pdf_version": "1.7",
      "creation_date": 1735689600,
      "modification_date": 1738281600,
      "has_javascript": false,
      "is_signed": true,
      "has_embedded_files": false,
      "has_incremental_updates": true,
      "update_chain_length": 3,
      "total_objects": 234,
      "suspicious_patterns": 1
    },
    {
      "uid": "b7e2d8f1",
      "filename": "contract.pdf",
      "check_date": 1738281600,
      "status": "intact",
      "metadata_score": 92,
      "creator": "Adobe Acrobat Pro DC",
      "producer": "Adobe PDF Library 15.0",
      "file_size": 1048576,
      "page_count": 12,
      "pdf_version": "1.7",
      "creation_date": 1735689600,
      "modification_date": 1735689600,
      "has_javascript": false,
      "is_signed": true,
      "has_embedded_files": false,
      "has_incremental_updates": false,
      "update_chain_length": 1,
      "total_objects": 567,
      "suspicious_patterns": 0
    }
  ],
  "total": 1250,
  "limit": 100,
  "offset": 0,
  "hasMore": true
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

Invalid query parameters.

```json
{
  "error": "Invalid query parameters",
  "details": "limit must be between 1 and 500"
}
```

**Causes:**

- `limit` is less than 1 or greater than 500
- `from_date` or `to_date` not valid Unix timestamps
- `modified` is not a boolean value

**Solution:** Check your query parameters against the documented ranges.

**Examples of Invalid Requests:**

```bash
# Wrong - limit too high
curl 'https://htpbe.tech/api/v1/checks?limit=1000'

# Wrong - invalid boolean
curl 'https://htpbe.tech/api/v1/checks?modified=yes'
```

---

### 401 Unauthorized

Authentication errors (same as other endpoints).

**See:** [analyze.md](./analyze.md#401-unauthorized) for details.

---

### 500 Internal Server Error

Server error during query execution.

```json
{
  "error": "Failed to fetch checks",
  "details": "Database query error"
}
```

**Cause:** Unexpected database or server error.

**Solution:**

- Retry the request
- Simplify filters (remove some parameters)
- If persists, contact support

---

## Use Cases

### 1. Export All Your Data

Download all your check results for backup or external analysis:

```javascript
async function exportAllChecks() {
  let allChecks = [];
  let offset = 0;
  const limit = 500;

  while (true) {
    const response = await fetch(
      `https://htpbe.tech/api/v1/checks?limit=${limit}&offset=${offset}`,
      {
        headers: { Authorization: `Bearer ${process.env.HTPBE_API_KEY}` },
      }
    );

    const data = await response.json();
    allChecks.push(...data.checks);

    if (!data.hasMore) break;
    offset += limit;

    // Progress indicator
    console.log(`Downloaded ${allChecks.length} / ${data.total} checks`);
  }

  // Save to file
  const fs = require('fs');
  fs.writeFileSync('my-checks-backup.json', JSON.stringify(allChecks, null, 2));

  console.log(`✅ Exported ${allChecks.length} checks`);
}
```

---

### 2. Calculate Custom Tool Statistics

Build your own tool analytics from raw data:

```javascript
async function calculateToolStats(toolName) {
  // Get all checks for this tool
  let allChecks = [];
  let offset = 0;

  while (true) {
    const response = await fetch(
      `https://htpbe.tech/api/v1/checks?tool=${encodeURIComponent(toolName)}&limit=500&offset=${offset}`,
      { headers: { Authorization: `Bearer ${apiKey}` } }
    );

    const data = await response.json();
    allChecks.push(...data.checks);
    if (!data.hasMore) break;
    offset += 500;
  }

  // Separate by role
  const asCreator = allChecks.filter((c) => c.creator === toolName);
  const asProducer = allChecks.filter((c) => c.producer === toolName);

  // Calculate statistics
  return {
    tool: toolName,
    totalUsage: allChecks.length,

    // As Creator
    totalUsageAsCreator: asCreator.length,
    modifiedAsCreator: asCreator.filter((c) => c.status === 'modified').length,
    modificationRateAsCreator:
      asCreator.length > 0
        ? Math.round(
            (asCreator.filter((c) => c.status === 'modified').length / asCreator.length) * 100
          )
        : 0,
    avgFileSizeAsCreator:
      asCreator.length > 0
        ? Math.round(asCreator.reduce((sum, c) => sum + c.file_size, 0) / asCreator.length)
        : 0,

    // As Producer
    totalUsageAsProducer: asProducer.length,
    modifiedAsProducer: asProducer.filter((c) => c.status === 'modified').length,
    modificationRateAsProducer:
      asProducer.length > 0
        ? Math.round(
            (asProducer.filter((c) => c.status === 'modified').length / asProducer.length) * 100
          )
        : 0,

    // Security features
    jsFilesAsProducer: asProducer.filter((c) => c.has_javascript).length,
    signedFilesAsProducer: asProducer.filter((c) => c.is_signed).length,
  };
}

const stats = await calculateToolStats('Adobe PDF Library 15.0');
console.log(stats);
```

---

### 3. Discover All Unique Tools

Find all PDF tools used across your checks:

```javascript
async function discoverAllTools() {
  let allChecks = [];
  let offset = 0;

  // Download all checks
  while (true) {
    const response = await fetch(`https://htpbe.tech/api/v1/checks?limit=500&offset=${offset}`, {
      headers: { Authorization: `Bearer ${apiKey}` },
    });

    const data = await response.json();
    allChecks.push(...data.checks);
    if (!data.hasMore) break;
    offset += 500;
  }

  // Extract unique tools
  const creators = new Set(allChecks.map((c) => c.creator).filter(Boolean));
  const producers = new Set(allChecks.map((c) => c.producer).filter(Boolean));

  // Count occurrences
  const creatorCounts = {};
  const producerCounts = {};

  allChecks.forEach((check) => {
    if (check.creator) {
      creatorCounts[check.creator] = (creatorCounts[check.creator] || 0) + 1;
    }
    if (check.producer) {
      producerCounts[check.producer] = (producerCounts[check.producer] || 0) + 1;
    }
  });

  // Sort by count
  const topCreators = Object.entries(creatorCounts)
    .sort(([, a], [, b]) => b - a)
    .slice(0, 20);

  const topProducers = Object.entries(producerCounts)
    .sort(([, a], [, b]) => b - a)
    .slice(0, 20);

  console.log('\n=== Top 20 Creators ===');
  topCreators.forEach(([tool, count]) => {
    console.log(`${count.toString().padStart(5)} - ${tool}`);
  });

  console.log('\n=== Top 20 Producers ===');
  topProducers.forEach(([tool, count]) => {
    console.log(`${count.toString().padStart(5)} - ${tool}`);
  });

  return {
    uniqueCreators: creators.size,
    uniqueProducers: producers.size,
    topCreators,
    topProducers,
  };
}
```

---

### 4. Build Custom Analytics Dashboard

Create time-series charts and custom metrics:

```python
import requests
import pandas as pd
from datetime import datetime, timedelta
import os

api_key = os.getenv('HTPBE_API_KEY')

def fetch_all_checks():
    """Fetch all checks and convert to pandas DataFrame"""
    all_checks = []
    offset = 0
    limit = 500

    while True:
        response = requests.get(
            'https://htpbe.tech/api/v1/checks',
            params={'limit': limit, 'offset': offset},
            headers={'Authorization': f'Bearer {api_key}'}
        )

        data = response.json()
        all_checks.extend(data['checks'])

        if not data['hasMore']:
            break

        offset += limit

    return pd.DataFrame(all_checks)

# Load data
df = fetch_all_checks()

# Convert timestamps to dates
df['check_date_dt'] = pd.to_datetime(df['check_date'], unit='s')
df['creation_date_dt'] = pd.to_datetime(df['creation_date'], unit='s')

# Weekly modification trend
weekly_stats = df.groupby(pd.Grouper(key='check_date_dt', freq='W')).agg({
    'uid': 'count',
    'is_modified': lambda x: (x == 'modified').sum()
}).rename(columns={'uid': 'total_checks', 'is_modified': 'modified_count'})

weekly_stats['modification_rate'] = (
    weekly_stats['modified_count'] / weekly_stats['total_checks'] * 100
).round(1)

print("\n=== Weekly Modification Trend ===")
print(weekly_stats)

# Top 10 tools by modification rate
tool_analysis = df.groupby('producer').agg({
    'uid': 'count',
    'is_modified': lambda x: (x == 'modified').sum()
}).rename(columns={'uid': 'count'})

tool_analysis['mod_rate'] = (
    tool_analysis['is_modified'] / tool_analysis['count'] * 100
).round(1)

tool_analysis = tool_analysis[tool_analysis['count'] >= 10]  # Min 10 samples
tool_analysis = tool_analysis.sort_values('mod_rate', ascending=False)

print("\n=== Top 10 Tools by Modification Rate (min 10 samples) ===")
print(tool_analysis.head(10))
```

---

### 5. Security Alert System

Monitor for high-risk patterns and send alerts:

```javascript
async function securityMonitoring() {
  const response = await fetch('https://htpbe.tech/api/v1/checks?modified=true&limit=500', {
    headers: { Authorization: `Bearer ${process.env.HTPBE_API_KEY}` },
  });

  const data = await response.json();
  const modifiedChecks = data.checks;

  // Critical patterns to alert on
  const alerts = [];

  // Alert 1: Signed documents that were modified
  const signedButModified = modifiedChecks.filter((c) => c.is_signed && c.status === 'modified');
  if (signedButModified.length > 0) {
    alerts.push({
      severity: 'CRITICAL',
      type: 'Signed documents modified',
      count: signedButModified.length,
      samples: signedButModified.slice(0, 5).map((c) => ({
        uid: c.uid,
        filename: c.filename,
      })),
    });
  }

  // Alert 2: PDFs with JavaScript that were modified
  const maliciousJS = modifiedChecks.filter((c) => c.has_javascript);
  if (maliciousJS.length > 0) {
    alerts.push({
      severity: 'HIGH',
      type: 'Modified PDFs with JavaScript',
      count: maliciousJS.length,
      samples: maliciousJS.slice(0, 5).map((c) => ({
        uid: c.uid,
        filename: c.filename,
      })),
    });
  }

  // Send alerts (example: console, but could be email/Slack/etc.)
  if (alerts.length > 0) {
    console.log('\n⚠️  SECURITY ALERTS ⚠️\n');
    alerts.forEach((alert) => {
      console.log(`[${alert.severity}] ${alert.type}: ${alert.count} documents`);
      console.log('Samples:', JSON.stringify(alert.samples, null, 2));
      console.log('---\n');
    });

    // TODO: Send to monitoring system
    // await sendToSlack(alerts);
    // await sendEmail(alerts);
  } else {
    console.log('✅ No critical security issues detected');
  }

  return alerts;
}

// Run every hour
setInterval(securityMonitoring, 60 * 60 * 1000);
```

---

## Notes

### Performance

- **Response Time:** 100-800ms depending on result count and filters
- **Optimization:** Use specific filters to reduce query time
- **Pagination:** Smaller `limit` values (100-200) are faster than maximum (500)
- **Indexes:** Database is indexed on `creator`, `producer`, and `check_date`

### Best Practices

**For Large Datasets:**

1. Use filters to reduce result set size
2. Paginate with `limit=500` for bulk exports
3. Consider implementing client-side caching

**For Real-Time Dashboards:**

1. Use smaller `limit` values (50-100)
2. Filter by recent dates (`from_date`)
3. Poll periodically (don't hammer the API)

**For Analytics:**

1. Export all data once, analyze locally
2. Use pandas/Excel for complex calculations
3. Re-export daily/weekly for fresh data

### Data Freshness

- **Real-Time:** No caching, results reflect latest check data
- **Updates:** New checks appear immediately in results
- **Ordering:** Results ordered by `check_date` descending (newest first)

### Limitations

- **Maximum `limit`:** 500 results per request
- **No custom sorting:** Results always sorted by `check_date DESC`
- **Filter combinations:** All filters use AND logic (not OR)
- **Tool filters:** Case-sensitive exact matches only

### Alternative Approaches

**If you need aggregate statistics only:**

- Use `GET /api/v1/stats` instead (much faster, pre-calculated)
- Only use `/checks` when you need raw data for custom analysis

**If you need a single check:**

- Use `GET /api/v1/result/{id}` for full details of one check (requires the full UUID from `/analyze`, not the short `uid` from this endpoint)
- Only use `/checks` when you need multiple checks

---

**Related Endpoints:**

- [POST /api/v1/analyze](./analyze.md) - Analyze a PDF (returns the full UUID to use with `/result/{id}`)
- [GET /api/v1/result/{id}](./result.md) - Get full details of a single check (requires full UUID from `/analyze`)
- [GET /api/v1/stats](./stats.md) - Get pre-calculated aggregate statistics

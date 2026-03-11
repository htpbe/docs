# GET /api/v1/stats

Get comprehensive aggregate statistics for all PDF checks performed by your API client.

## Endpoint

```
GET https://htpbe.tech/api/v1/stats
```

## Authentication

This endpoint requires API key authentication via the `Authorization` header.

```http
Authorization: Bearer YOUR_API_KEY
```

## Data Isolation

**Important:** Statistics include **only your checks**, not other clients' data. Each API client sees only their own statistics, ensuring complete data privacy.

---

## Request

### Headers

| Header          | Type   | Required | Description                                                                         |
| --------------- | ------ | -------- | ----------------------------------------------------------------------------------- |
| `Authorization` | string | **Yes**  | Bearer token with your API key (`Bearer htpbe_live_...` or `Bearer htpbe_test_...`) |

### Query Parameters

This endpoint has no query parameters. It returns all statistics for your API client.

### Example Request

```bash
curl -X GET https://htpbe.tech/api/v1/stats \
  -H "Authorization: Bearer htpbe_live_sk_1234567890abcdef"
```

```javascript
// Node.js / TypeScript
const response = await fetch('https://htpbe.tech/api/v1/stats', {
  headers: {
    Authorization: `Bearer ${process.env.HTPBE_API_KEY}`,
  },
});

const stats = await response.json();
console.log(`Total checks: ${stats.total_checks}`);
console.log(`Modified files: ${stats.modified_checks}`);
```

```python
# Python
import requests
import os

response = requests.get(
    'https://htpbe.tech/api/v1/stats',
    headers={
        'Authorization': f"Bearer {os.getenv('HTPBE_API_KEY')}"
    }
)

stats = response.json()
print(f"Total checks: {stats['total_checks']}")
print(f"Modified files: {stats['modified_checks']}")
```

---

## Response

### Success Response (200 OK)

Returns a `StatsResponse` object with comprehensive statistics about all your PDF checks.

#### Response Structure

```typescript
{
  // Basic Metrics
  total_checks: number;
  modified_checks: number;
  most_used_producer: string | undefined;
  most_used_creator: string | undefined;
  oldest_creation_date: number | undefined;
  top_producers: Array<{ producer: string; count: number }>;
  top_creators: Array<{ creator: string; count: number }>;

  // Advanced Metrics
  total_pages: number;
  avg_age_seconds: number;
  js_files: number;
  signed_but_modified: number;
  max_pages: number;
  max_file_size: number;
  max_update_chain: number;
  no_creation_date: number;
  version_distribution: Array<{ version: string; count: number }>;
  total_objects: number;
  embedded_files: number;
  incremental_updates: number;
  avg_metadata_score: number;
  signed_files: number;
  monthly_modification_stats: Array<{
    month: number;
    total_count: number;
    modified_count: number;
  }>;
  newest_creation_date: number | undefined;
  total_file_size: number;
}
```

### Response Fields

#### Basic Metrics

##### `total_checks`

- **Type:** `number` (integer)
- **Always Present:** Yes
- **Range:** `0` and higher
- **Description:** Total number of PDF checks performed by your API client
- **Example:** `1250` = 1,250 PDFs analyzed
- **Usage:** Your primary usage metric. Compare against your plan quota to track consumption.

##### `modified_checks`

- **Type:** `number` (integer)
- **Always Present:** Yes
- **Range:** `0` to `total_checks`
- **Description:** Number of PDFs detected as modified (`status: "modified"`, excluding `inconclusive`)
- **Example:** `487` out of 1250 checks
- **Calculate Percentage:** `(modified_checks / total_checks) * 100` = modification rate
- **Insight:** If this percentage is high (>40%), many of your documents are being modified

##### `most_used_producer`

- **Type:** `string | undefined`
- **Can Be Undefined:** Yes (if no PDFs have producer metadata)
- **Description:** Most frequently occurring Producer software across all your checks
- **Examples:**
  - `"Adobe PDF Library 15.0"` - Most common
  - `"PDFKit"` - JavaScript library
  - `"Microsoft: Print To PDF"` - Windows built-in
  - `undefined` - No producer metadata found in any PDF
- **Usage:** Understand what tools your users/systems are primarily using to generate PDFs

##### `most_used_creator`

- **Type:** `string | undefined`
- **Can Be Undefined:** Yes (if no PDFs have creator metadata)
- **Description:** Most frequently occurring Creator software across all your checks
- **Examples:**
  - `"Microsoft Word for Microsoft 365"` - Most common
  - `"Adobe InDesign CC"` - Design software
  - `"LibreOffice Writer"` - Open source
  - `undefined` - No creator metadata found
- **Usage:** Understand what applications users are creating documents with before PDF conversion

##### `oldest_creation_date`

- **Type:** `number | undefined` (Unix timestamp in seconds)
- **Can Be Undefined:** Yes (if no PDFs have creation dates)
- **Description:** Unix timestamp of the oldest PDF creation date in your collection
- **Example:** `1577836800` = January 1, 2020, 00:00:00 UTC
- **Convert:** JavaScript: `new Date(oldest_creation_date * 1000).toISOString()`
- **Usage:** Understand the age range of documents you're analyzing

##### `top_producers`

- **Type:** `Array<{ producer: string; count: number }>` (array of objects)
- **Always Present:** Yes (can be empty array)
- **Max Length:** 10 items
- **Description:** Top 10 Producer software by frequency, sorted by count (highest first)
- **Empty When:** No PDFs have producer metadata
- **Example:**
  ```json
  [
    { "producer": "Adobe PDF Library 15.0", "count": 450 },
    { "producer": "PDFKit", "count": 230 },
    { "producer": "iText 7.0.0", "count": 180 }
  ]
  ```
- **Usage:** Detailed breakdown of PDF generation tools in your ecosystem

##### `top_creators`

- **Type:** `Array<{ creator: string; count: number }>` (array of objects)
- **Always Present:** Yes (can be empty array)
- **Max Length:** 10 items
- **Description:** Top 10 Creator software by frequency, sorted by count (highest first)
- **Empty When:** No PDFs have creator metadata
- **Example:**
  ```json
  [
    { "creator": "Microsoft Word for Microsoft 365", "count": 520 },
    { "creator": "Adobe InDesign CC", "count": 180 },
    { "creator": "Google Docs", "count": 95 }
  ]
  ```

---

#### Advanced Metrics

##### `total_pages`

- **Type:** `number` (integer)
- **Always Present:** Yes
- **Range:** `0` and higher
- **Description:** Sum of page counts across all analyzed PDFs
- **Example:** `45230` = total of 45,230 pages across all documents
- **Usage:** Understand total document volume
- **Calculate Average:** `total_pages / total_checks` = average pages per document

##### `avg_age_seconds`

- **Type:** `number` (integer, seconds)
- **Always Present:** Yes
- **Range:** `0` and higher
- **Description:** Average age of PDFs in seconds (calculated as `check_date - creation_date`)
- **Zero When:** No PDFs have valid creation dates
- **Example:** `7776000` = 90 days (2,160 hours)
- **Convert to Days:** `avg_age_seconds / 86400`
- **Convert to Hours:** `avg_age_seconds / 3600`
- **Usage:** Understand how old documents are when they're being checked

##### `js_files`

- **Type:** `number` (integer)
- **Always Present:** Yes
- **Range:** `0` to `total_checks`
- **Description:** Number of PDFs containing embedded JavaScript code
- **Example:** `12` out of 1250 = 0.96% contain JavaScript
- **Security Note:** High numbers may indicate security risks
- **Percentage:** `(js_files / total_checks) * 100`

##### `signed_but_modified`

- **Type:** `number` (integer)
- **Always Present:** Yes
- **Range:** `0` to `total_checks`
- **Description:** Number of PDFs that were digitally signed but then modified afterward
- **Critical Indicator:** These are high-risk tampering cases where signatures were invalidated
- **Example:** `34` documents were signed then modified
- **Usage:** Monitor for fraudulent document modification

##### `max_pages`

- **Type:** `number` (integer)
- **Always Present:** Yes
- **Range:** `0` and higher
- **Description:** Maximum page count found in a single PDF
- **Example:** `450` = largest document has 450 pages
- **Zero When:** No PDFs have page count data
- **Usage:** Identify unusually large documents

##### `max_file_size`

- **Type:** `number` (integer, bytes)
- **Always Present:** Yes
- **Range:** `0` to `10485760` (10 MB API limit)
- **Description:** Maximum file size encountered in bytes
- **Example:** `9437184` = ~9 MB
- **Convert to MB:** `max_file_size / 1024 / 1024`
- **Usage:** Understand file size distribution

##### `max_update_chain`

- **Type:** `number` (integer)
- **Always Present:** Yes
- **Range:** `0` and higher
- **Description:** Maximum update chain length found (number of times a PDF was saved)
- **Example:** `12` = one PDF was edited and saved 12 times
- **Usage:** Identify heavily modified documents

##### `no_creation_date`

- **Type:** `number` (integer)
- **Always Present:** Yes
- **Range:** `0` to `total_checks`
- **Description:** Number of PDFs without creation date metadata
- **Example:** `45` = 3.6% of documents lack creation dates
- **Impact:** PDFs without dates are harder to verify for modifications
- **Percentage:** `(no_creation_date / total_checks) * 100`

##### `version_distribution`

- **Type:** `Array<{ version: string; count: number }>` (array of objects)
- **Always Present:** Yes (can be empty array)
- **Max Length:** 10 items
- **Description:** Distribution of PDF specification versions, sorted by frequency
- **Empty When:** No PDFs have version information
- **Example:**
  ```json
  [
    { "version": "1.7", "count": 890 },
    { "version": "1.4", "count": 250 },
    { "version": "1.5", "count": 85 },
    { "version": "2.0", "count": 25 }
  ]
  ```
- **Common Versions:**
  - `"1.7"` - Most common (Acrobat 8.x+, default for most modern tools)
  - `"1.4"` - Older standard (Acrobat 5.x)
  - `"2.0"` - Modern ISO standard
- **Usage:** Understand technology stack age

##### `total_objects`

- **Type:** `number` (integer)
- **Always Present:** Yes
- **Range:** `0` and higher
- **Description:** Sum of all PDF internal objects across all documents
- **Example:** `567890` = total object count
- **What Are Objects:** Pages, fonts, images, annotations, forms, etc.
- **Usage:** Measure total structural complexity

##### `embedded_files`

- **Type:** `number` (integer)
- **Always Present:** Yes
- **Range:** `0` to `total_checks`
- **Description:** Number of PDFs containing embedded file attachments
- **Example:** `23` = 1.8% of documents have attachments
- **Security Risk:** Attachments can hide malware
- **Percentage:** `(embedded_files / total_checks) * 100`

##### `incremental_updates`

- **Type:** `number` (integer)
- **Always Present:** Yes
- **Range:** `0` to `total_checks`
- **Description:** Number of PDFs with incremental update structures (been saved multiple times)
- **Example:** `623` = 49.8% have incremental updates
- **Note:** Not always malicious - many legitimate workflows create incremental updates
- **Percentage:** `(incremental_updates / total_checks) * 100`

##### `avg_metadata_score`

- **Type:** `number` (integer)
- **Always Present:** Yes
- **Range:** `0` to `100`
- **Description:** Average metadata completeness score across all documents
- **Example:** `78` = generally good metadata quality
- **Interpretation:**
  - `90-100` - Excellent metadata (most fields present)
  - `70-89` - Good metadata coverage
  - `50-69` - Moderate metadata
  - `30-49` - Limited metadata (harder to verify)
  - `0-29` - Poor metadata quality

##### `signed_files`

- **Type:** `number` (integer)
- **Always Present:** Yes
- **Range:** `0` to `total_checks`
- **Description:** Number of PDFs that currently contain digital signatures
- **Example:** `234` = 18.7% are digitally signed
- **Note:** This doesn't include signatures that were removed
- **Usage:** Understand adoption of digital signatures in your document workflow
- **Compare with:** `signed_but_modified` to find signature invalidation rate

##### `monthly_modification_stats`

- **Type:** `Array<{ month: number; total_count: number; modified_count: number }>` (array of objects)
- **Always Present:** Yes (can be empty array)
- **Max Length:** 12 items (one per month)
- **Description:** Monthly breakdown of document modifications by modification date
- **Empty When:** No PDFs have modification dates
- **Field Details:**
  - `month` - Month number (1-12, January-December)
  - `total_count` - Total PDFs modified in that month
  - `modified_count` - How many of those were detected as edited
- **Example:**
  ```json
  [
    { "month": 1, "total_count": 100, "modified_count": 38 },
    { "month": 2, "total_count": 95, "modified_count": 41 },
    { "month": 3, "total_count": 110, "modified_count": 45 }
  ]
  ```
- **Usage:** Track seasonal patterns or trends in document modification

##### `newest_creation_date`

- **Type:** `number | undefined` (Unix timestamp in seconds)
- **Can Be Undefined:** Yes (if no PDFs have creation dates)
- **Description:** Unix timestamp of the newest PDF creation date
- **Example:** `1735689600` = January 1, 2025, 00:00:00 UTC
- **Convert:** JavaScript: `new Date(newest_creation_date * 1000).toISOString()`
- **Usage:** Combined with `oldest_creation_date`, shows the time span of your document collection

##### `total_file_size`

- **Type:** `number` (integer, bytes)
- **Always Present:** Yes
- **Range:** `0` and higher
- **Description:** Sum of all file sizes across all checks
- **Example:** `1250000000` = 1.25 GB total
- **Convert to GB:** `total_file_size / 1024 / 1024 / 1024`
- **Convert to MB:** `total_file_size / 1024 / 1024`
- **Usage:** Understand total storage requirements or data volume

---

### Example Response

```json
{
  "total_checks": 1250,
  "modified_checks": 487,
  "most_used_producer": "Adobe PDF Library 15.0",
  "most_used_creator": "Microsoft Word for Microsoft 365",
  "oldest_creation_date": 1577836800,
  "top_producers": [
    { "producer": "Adobe PDF Library 15.0", "count": 450 },
    { "producer": "PDFKit", "count": 230 },
    { "producer": "iText 7.0.0", "count": 180 }
  ],
  "top_creators": [
    { "creator": "Microsoft Word for Microsoft 365", "count": 520 },
    { "creator": "Adobe InDesign CC", "count": 180 },
    { "creator": "Google Docs", "count": 95 }
  ],
  "total_pages": 45230,
  "avg_age_seconds": 7776000,
  "js_files": 12,
  "signed_but_modified": 34,
  "max_pages": 450,
  "max_file_size": 9437184,
  "max_update_chain": 12,
  "no_creation_date": 45,
  "version_distribution": [
    { "version": "1.7", "count": 890 },
    { "version": "1.4", "count": 250 },
    { "version": "1.5", "count": 85 },
    { "version": "2.0", "count": 25 }
  ],
  "total_objects": 567890,
  "embedded_files": 23,
  "incremental_updates": 623,
  "avg_metadata_score": 78,
  "signed_files": 234,
  "monthly_modification_stats": [
    { "month": 1, "total_count": 100, "modified_count": 38 },
    { "month": 2, "total_count": 95, "modified_count": 41 },
    { "month": 3, "total_count": 110, "modified_count": 45 }
  ],
  "newest_creation_date": 1735689600,
  "total_file_size": 1250000000
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

### 401 Unauthorized

Authentication errors (same as other endpoints).

**See:** [analyze.md](./analyze.md#401-unauthorized) for details.

---

### 500 Internal Server Error

Server error during statistics calculation.

```json
{
  "error": "Failed to fetch statistics",
  "code": "internal_error",
  "details": "Database query error"
}
```

**Cause:** Unexpected database or server error.

**Solution:**

- Retry the request
- If persists, contact support

---

## Use Cases

### 1. Dashboard Overview

Display key metrics in your application dashboard:

```javascript
const stats = await fetch('https://htpbe.tech/api/v1/stats', {
  headers: { Authorization: `Bearer ${apiKey}` },
}).then((r) => r.json());

const dashboard = {
  totalDocuments: stats.total_checks,
  modificationRate: ((stats.modified_checks / stats.total_checks) * 100).toFixed(1) + '%',
  topTool: stats.most_used_producer,

  // Alert if critical conditions detected
  alerts: [
    stats.signed_but_modified > 0 ? `${stats.signed_but_modified} signed docs were modified` : null,
  ].filter(Boolean),
};
```

---

### 2. Compliance Reporting

Generate monthly compliance reports:

```python
import requests
from datetime import datetime

stats = requests.get(
    'https://htpbe.tech/api/v1/stats',
    headers={'Authorization': f'Bearer {api_key}'}
).json()

report = {
    'period': datetime.now().strftime('%Y-%m'),
    'total_checks': stats['total_checks'],
    'modified_documents': stats['modified_checks'],
    'modification_rate': f"{(stats['modified_checks'] / stats['total_checks'] * 100):.1f}%",
    'high_risk_indicators': {
        'signed_but_modified': stats['signed_but_modified'],
        'javascript_files': stats['js_files']
    }
}

print(f"Compliance Report - {report['period']}")
print(f"Total Documents Verified: {report['total_checks']}")
print(f"Modification Rate: {report['modification_rate']}")
```

---

### 3. Usage Monitoring

Track API usage against your plan quota:

```javascript
async function checkQuotaUsage() {
  const stats = await fetch('https://htpbe.tech/api/v1/stats', {
    headers: { Authorization: `Bearer ${process.env.HTPBE_API_KEY}` },
  }).then((r) => r.json());

  const quotaLimits = {
    starter: 30,
    growth: 350,
    pro: 1500,
    enterprise: Infinity,
  };

  const currentPlan = 'growth'; // From your settings
  const quota = quotaLimits[currentPlan];
  const usage = stats.total_checks;
  const remaining = quota - usage;
  const percentUsed = ((usage / quota) * 100).toFixed(1);

  console.log(`Plan: ${currentPlan.toUpperCase()}`);
  console.log(`Usage: ${usage}/${quota} (${percentUsed}%)`);
  console.log(`Remaining: ${remaining} checks`);

  if (percentUsed > 90) {
    console.warn('⚠️  Warning: 90% of quota used - consider upgrading');
  }
}
```

---

### 4. Tool Analysis

Analyze which PDF tools are most commonly used:

```javascript
const stats = await getStats();

// Tool popularity analysis
const toolAnalysis = {
  mostPopularCreator: stats.most_used_creator,
  mostPopularProducer: stats.most_used_producer,

  // Top 3 producers with counts
  topProducers: stats.top_producers.slice(0, 3).map((p) => ({
    name: p.producer,
    count: p.count,
    percentage: ((p.count / stats.total_checks) * 100).toFixed(1) + '%',
  })),

  // Tool diversity
  toolDiversity: stats.top_producers.length, // More = diverse toolset

  // PDF version adoption
  modernPDFs: stats.version_distribution
    .filter((v) => parseFloat(v.version) >= 1.7)
    .reduce((sum, v) => sum + v.count, 0),

  legacyPDFs: stats.version_distribution
    .filter((v) => parseFloat(v.version) < 1.7)
    .reduce((sum, v) => sum + v.count, 0),
};

console.log('Tool Analysis:', toolAnalysis);
```

---

### 5. Security Monitoring

Monitor security-related statistics:

```javascript
const stats = await getStats();

const securityMetrics = {
  // Critical security indicators
  signatureIntegrity: {
    total: stats.signed_files,
    compromised: stats.signed_but_modified,
    rate:
      stats.signed_files > 0
        ? ((stats.signed_but_modified / stats.signed_files) * 100).toFixed(1) + '%'
        : '0%',
  },

  // Potentially dangerous features
  riskyFeatures: {
    javascript: stats.js_files,
    embeddedFiles: stats.embedded_files,
    combined: stats.js_files + stats.embedded_files,
  },
};

// Alert if any critical thresholds exceeded
if (securityMetrics.signatureIntegrity.rate > '10%') {
  alert('⚠️  High signature compromise rate detected');
}
```

---

## Notes

### Performance

- **Fast:** This endpoint is optimized with database indexes
- **Response Time:** Typically 100-500ms depending on data volume
- **No Pagination:** Returns all aggregate statistics in one response

### Caching

- Results reflect real-time data (no caching)
- Statistics update immediately after each new check
- Safe to call frequently for dashboard updates

### Empty Statistics

If you haven't performed any checks yet, most numeric fields will return `0`:

```json
{
  "total_checks": 0,
  "modified_checks": 0,
  "top_producers": [],
  "top_creators": [],
  "version_distribution": [],
  "monthly_modification_stats": []
}
```

---

**Related Endpoints:**

- [POST /api/v1/analyze](./analyze.md) - Analyze a PDF
- [GET /api/v1/result/{uid}](./result.md) - Retrieve full analysis results
- [GET /api/v1/checks](./checks.md) - List all checks with filtering for custom analytics

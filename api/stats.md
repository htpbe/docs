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
console.log(`Total checks: ${stats.totalChecks}`);
console.log(`Modified files: ${stats.editedFiles}`);
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
print(f"Total checks: {stats['totalChecks']}")
print(f"Modified files: {stats['editedFiles']}")
```

---

## Response

### Success Response (200 OK)

Returns a `StatsResponse` object with comprehensive statistics about all your PDF checks.

#### Response Structure

```typescript
{
  // Basic Metrics
  totalChecks: number;
  editedFiles: number;
  mostUsedProducer: string | undefined;
  mostUsedCreator: string | undefined;
  oldestCreationDate: number | undefined;
  topProducers: Array<{ producer: string; count: number }>;
  topCreators: Array<{ creator: string; count: number }>;

  // Advanced Metrics
  totalPages: number;
  avgAgeSeconds: number;
  jsFiles: number;
  signedButModified: number;
  maxPages: number;
  maxFileSize: number;
  maxUpdateChain: number;
  noCreationDate: number;
  versionDistribution: Array<{ version: string; count: number }>;
  totalObjects: number;
  embeddedFiles: number;
  incrementalUpdates: number;
  avgMetadataScore: number;
  signedFiles: number;
  monthlyModificationStats: Array<{
    month: number;
    totalCount: number;
    modifiedCount: number;
  }>;
  newestCreationDate: number | undefined;
  totalFileSize: number;
}
```

### Response Fields

#### Basic Metrics

##### `totalChecks`

- **Type:** `number` (integer)
- **Always Present:** Yes
- **Range:** `0` and higher
- **Description:** Total number of PDF checks performed by your API client
- **Example:** `1250` = 1,250 PDFs analyzed
- **Usage:** Your primary usage metric. Compare against your plan quota to track consumption.

##### `editedFiles`

- **Type:** `number` (integer)
- **Always Present:** Yes
- **Range:** `0` to `totalChecks`
- **Description:** Number of PDFs that were detected as modified (`been_changed = true`)
- **Example:** `487` out of 1250 checks
- **Calculate Percentage:** `(editedFiles / totalChecks) * 100` = modification rate
- **Insight:** If this percentage is high (>40%), many of your documents are being modified

##### `mostUsedProducer`

- **Type:** `string | undefined`
- **Can Be Undefined:** Yes (if no PDFs have producer metadata)
- **Description:** Most frequently occurring Producer software across all your checks
- **Examples:**
  - `"Adobe PDF Library 15.0"` - Most common
  - `"PDFKit"` - JavaScript library
  - `"Microsoft: Print To PDF"` - Windows built-in
  - `undefined` - No producer metadata found in any PDF
- **Usage:** Understand what tools your users/systems are primarily using to generate PDFs

##### `mostUsedCreator`

- **Type:** `string | undefined`
- **Can Be Undefined:** Yes (if no PDFs have creator metadata)
- **Description:** Most frequently occurring Creator software across all your checks
- **Examples:**
  - `"Microsoft Word for Microsoft 365"` - Most common
  - `"Adobe InDesign CC"` - Design software
  - `"LibreOffice Writer"` - Open source
  - `undefined` - No creator metadata found
- **Usage:** Understand what applications users are creating documents with before PDF conversion

##### `oldestCreationDate`

- **Type:** `number | undefined` (Unix timestamp in seconds)
- **Can Be Undefined:** Yes (if no PDFs have creation dates)
- **Description:** Unix timestamp of the oldest PDF creation date in your collection
- **Example:** `1577836800` = January 1, 2020, 00:00:00 UTC
- **Convert:** JavaScript: `new Date(oldestCreationDate * 1000).toISOString()`
- **Usage:** Understand the age range of documents you're analyzing

##### `topProducers`

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

##### `topCreators`

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

##### `totalPages`

- **Type:** `number` (integer)
- **Always Present:** Yes
- **Range:** `0` and higher
- **Description:** Sum of page counts across all analyzed PDFs
- **Example:** `45230` = total of 45,230 pages across all documents
- **Usage:** Understand total document volume
- **Calculate Average:** `totalPages / totalChecks` = average pages per document

##### `avgAgeSeconds`

- **Type:** `number` (integer, seconds)
- **Always Present:** Yes
- **Range:** `0` and higher
- **Description:** Average age of PDFs in seconds (calculated as `check_date - creation_date`)
- **Zero When:** No PDFs have valid creation dates
- **Example:** `7776000` = 90 days (2,160 hours)
- **Convert to Days:** `avgAgeSeconds / 86400`
- **Convert to Hours:** `avgAgeSeconds / 3600`
- **Usage:** Understand how old documents are when they're being checked

##### `jsFiles`

- **Type:** `number` (integer)
- **Always Present:** Yes
- **Range:** `0` to `totalChecks`
- **Description:** Number of PDFs containing embedded JavaScript code
- **Example:** `12` out of 1250 = 0.96% contain JavaScript
- **Security Note:** High numbers may indicate security risks
- **Percentage:** `(jsFiles / totalChecks) * 100`

##### `signedButModified`

- **Type:** `number` (integer)
- **Always Present:** Yes
- **Range:** `0` to `totalChecks`
- **Description:** Number of PDFs that were digitally signed but then modified afterward
- **Critical Indicator:** These are high-risk tampering cases where signatures were invalidated
- **Example:** `34` documents were signed then modified
- **Usage:** Monitor for fraudulent document modification

##### `maxPages`

- **Type:** `number` (integer)
- **Always Present:** Yes
- **Range:** `0` and higher
- **Description:** Maximum page count found in a single PDF
- **Example:** `450` = largest document has 450 pages
- **Zero When:** No PDFs have page count data
- **Usage:** Identify unusually large documents

##### `maxFileSize`

- **Type:** `number` (integer, bytes)
- **Always Present:** Yes
- **Range:** `0` to `10485760` (10 MB API limit)
- **Description:** Maximum file size encountered in bytes
- **Example:** `9437184` = ~9 MB
- **Convert to MB:** `maxFileSize / 1024 / 1024`
- **Usage:** Understand file size distribution

##### `maxUpdateChain`

- **Type:** `number` (integer)
- **Always Present:** Yes
- **Range:** `0` and higher
- **Description:** Maximum update chain length found (number of times a PDF was saved)
- **Example:** `12` = one PDF was edited and saved 12 times
- **Usage:** Identify heavily modified documents

##### `noCreationDate`

- **Type:** `number` (integer)
- **Always Present:** Yes
- **Range:** `0` to `totalChecks`
- **Description:** Number of PDFs without creation date metadata
- **Example:** `45` = 3.6% of documents lack creation dates
- **Impact:** PDFs without dates are harder to verify for modifications
- **Percentage:** `(noCreationDate / totalChecks) * 100`

##### `versionDistribution`

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

##### `totalObjects`

- **Type:** `number` (integer)
- **Always Present:** Yes
- **Range:** `0` and higher
- **Description:** Sum of all PDF internal objects across all documents
- **Example:** `567890` = total object count
- **What Are Objects:** Pages, fonts, images, annotations, forms, etc.
- **Usage:** Measure total structural complexity

##### `embeddedFiles`

- **Type:** `number` (integer)
- **Always Present:** Yes
- **Range:** `0` to `totalChecks`
- **Description:** Number of PDFs containing embedded file attachments
- **Example:** `23` = 1.8% of documents have attachments
- **Security Risk:** Attachments can hide malware
- **Percentage:** `(embeddedFiles / totalChecks) * 100`

##### `incrementalUpdates`

- **Type:** `number` (integer)
- **Always Present:** Yes
- **Range:** `0` to `totalChecks`
- **Description:** Number of PDFs with incremental update structures (been saved multiple times)
- **Example:** `623` = 49.8% have incremental updates
- **Note:** Not always malicious - many legitimate workflows create incremental updates
- **Percentage:** `(incrementalUpdates / totalChecks) * 100`

##### `avgMetadataScore`

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

##### `signedFiles`

- **Type:** `number` (integer)
- **Always Present:** Yes
- **Range:** `0` to `totalChecks`
- **Description:** Number of PDFs that currently contain digital signatures
- **Example:** `234` = 18.7% are digitally signed
- **Note:** This doesn't include signatures that were removed
- **Usage:** Understand adoption of digital signatures in your document workflow
- **Compare with:** `signedButModified` to find signature invalidation rate

##### `monthlyModificationStats`

- **Type:** `Array<{ month: number; totalCount: number; modifiedCount: number }>` (array of objects)
- **Always Present:** Yes (can be empty array)
- **Max Length:** 12 items (one per month)
- **Description:** Monthly breakdown of document modifications by modification date
- **Empty When:** No PDFs have modification dates
- **Field Details:**
  - `month` - Month number (1-12, January-December)
  - `totalCount` - Total PDFs modified in that month
  - `modifiedCount` - How many of those were detected as edited
- **Example:**
  ```json
  [
    { "month": 1, "totalCount": 100, "modifiedCount": 38 },
    { "month": 2, "totalCount": 95, "modifiedCount": 41 },
    { "month": 3, "totalCount": 110, "modifiedCount": 45 }
  ]
  ```
- **Usage:** Track seasonal patterns or trends in document modification

##### `newestCreationDate`

- **Type:** `number | undefined` (Unix timestamp in seconds)
- **Can Be Undefined:** Yes (if no PDFs have creation dates)
- **Description:** Unix timestamp of the newest PDF creation date
- **Example:** `1735689600` = January 1, 2025, 00:00:00 UTC
- **Convert:** JavaScript: `new Date(newestCreationDate * 1000).toISOString()`
- **Usage:** Combined with `oldestCreationDate`, shows the time span of your document collection

##### `totalFileSize`

- **Type:** `number` (integer, bytes)
- **Always Present:** Yes
- **Range:** `0` and higher
- **Description:** Sum of all file sizes across all checks
- **Example:** `1250000000` = 1.25 GB total
- **Convert to GB:** `totalFileSize / 1024 / 1024 / 1024`
- **Convert to MB:** `totalFileSize / 1024 / 1024`
- **Usage:** Understand total storage requirements or data volume

---

### Example Response

```json
{
  "totalChecks": 1250,
  "editedFiles": 487,
  "mostUsedProducer": "Adobe PDF Library 15.0",
  "mostUsedCreator": "Microsoft Word for Microsoft 365",
  "oldestCreationDate": 1577836800,
  "topProducers": [
    { "producer": "Adobe PDF Library 15.0", "count": 450 },
    { "producer": "PDFKit", "count": 230 },
    { "producer": "iText 7.0.0", "count": 180 }
  ],
  "topCreators": [
    { "creator": "Microsoft Word for Microsoft 365", "count": 520 },
    { "creator": "Adobe InDesign CC", "count": 180 },
    { "creator": "Google Docs", "count": 95 }
  ],
  "totalPages": 45230,
  "avgAgeSeconds": 7776000,
  "jsFiles": 12,
  "signedButModified": 34,
  "maxPages": 450,
  "maxFileSize": 9437184,
  "maxUpdateChain": 12,
  "noCreationDate": 45,
  "versionDistribution": [
    { "version": "1.7", "count": 890 },
    { "version": "1.4", "count": 250 },
    { "version": "1.5", "count": 85 },
    { "version": "2.0", "count": 25 }
  ],
  "totalObjects": 567890,
  "embeddedFiles": 23,
  "incrementalUpdates": 623,
  "avgMetadataScore": 78,
  "signedFiles": 234,
  "monthlyModificationStats": [
    { "month": 1, "totalCount": 100, "modifiedCount": 38 },
    { "month": 2, "totalCount": 95, "modifiedCount": 41 },
    { "month": 3, "totalCount": 110, "modifiedCount": 45 }
  ],
  "newestCreationDate": 1735689600,
  "totalFileSize": 1250000000
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

### 401 Unauthorized

Authentication errors (same as other endpoints).

**See:** [analyze.md](./analyze.md#401-unauthorized) for details.

---

### 500 Internal Server Error

Server error during statistics calculation.

```json
{
  "error": "Failed to fetch statistics",
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
  totalDocuments: stats.totalChecks,
  modificationRate: ((stats.editedFiles / stats.totalChecks) * 100).toFixed(1) + '%',
  topTool: stats.mostUsedProducer,

  // Alert if critical conditions detected
  alerts: [
    stats.signedButModified > 0 ? `${stats.signedButModified} signed docs were modified` : null,
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
    'total_checks': stats['totalChecks'],
    'modified_documents': stats['editedFiles'],
    'modification_rate': f"{(stats['editedFiles'] / stats['totalChecks'] * 100):.1f}%",
    'high_risk_indicators': {
        'signed_but_modified': stats['signedButModified'],
        'javascript_files': stats['jsFiles']
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
  const usage = stats.totalChecks;
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
  mostPopularCreator: stats.mostUsedCreator,
  mostPopularProducer: stats.mostUsedProducer,

  // Top 3 producers with counts
  topProducers: stats.topProducers.slice(0, 3).map((p) => ({
    name: p.producer,
    count: p.count,
    percentage: ((p.count / stats.totalChecks) * 100).toFixed(1) + '%',
  })),

  // Tool diversity
  toolDiversity: stats.topProducers.length, // More = diverse toolset

  // PDF version adoption
  modernPDFs: stats.versionDistribution
    .filter((v) => parseFloat(v.version) >= 1.7)
    .reduce((sum, v) => sum + v.count, 0),

  legacyPDFs: stats.versionDistribution
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
    total: stats.signedFiles,
    compromised: stats.signedButModified,
    rate:
      stats.signedFiles > 0
        ? ((stats.signedButModified / stats.signedFiles) * 100).toFixed(1) + '%'
        : '0%',
  },

  // Potentially dangerous features
  riskyFeatures: {
    javascript: stats.jsFiles,
    embeddedFiles: stats.embeddedFiles,
    combined: stats.jsFiles + stats.embeddedFiles,
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
  "totalChecks": 0,
  "editedFiles": 0,
  "topProducers": [],
  "topCreators": [],
  "versionDistribution": [],
  "monthlyModificationStats": []
}
```

---

**Related Endpoints:**

- [POST /api/v1/analyze](./analyze.md) - Analyze a PDF
- [GET /api/v1/result/{uid}](./result.md) - Retrieve full analysis results
- [GET /api/v1/checks](./checks.md) - List all checks with filtering for custom analytics

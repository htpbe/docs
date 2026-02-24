# HTPBE Enterprise: On-Premise Deployment

**HTPBE is available for on-premise deployment** — run the PDF analysis engine entirely within your own infrastructure. Documents never leave your network.

→ **Contact:** iurii@rogulia.fi to discuss requirements and pricing.

---

## Why On-Premise?

The cloud API is suitable for most use cases. On-premise is the right choice when:

- Your **security policy prohibits sending documents to third-party services**
- You operate under **GDPR, HIPAA, PCI DSS, SOX, or similar compliance frameworks** that require data residency
- You work with **classified, confidential, or legally privileged documents** (contracts, medical records, financial reports, legal briefs)
- You need **air-gapped deployment** with no outbound internet access
- You process **high volumes** and want predictable infrastructure costs

---

## What You Get

The on-premise version is the same analysis engine that powers htpbe.tech — identical detection accuracy, identical API surface. The only difference is where it runs.

**What's included:**
- Docker image of the HTPBE analysis engine
- Identical REST API (`POST /analyze`, `GET /result/{id}`, `GET /checks`, `GET /stats`)
- No file size limits — configure based on your server capacity
- No request quotas — your infrastructure, your rules
- Monthly security patches and algorithm updates
- Dedicated account manager and onboarding support
- Priority support with 1-hour response time guarantee
- Custom development for specific integrations or workflow requirements

---

## Architecture

The on-premise deployment is a single self-contained service:

```
Your application
       │
       │  HTTP POST /analyze  { file_url: "..." }
       ▼
┌─────────────────────────────────┐
│   HTPBE Analysis Engine         │
│   (Docker container)            │
│                                 │
│  ┌──────────────────────────┐   │
│  │  PDF Download (internal) │   │  ← fetches from your internal storage
│  │  5-Layer Analysis        │   │
│  │  Results Database        │   │  ← local SQLite or external DB
│  └──────────────────────────┘   │
└─────────────────────────────────┘
       │
       │  { id, been_changed, risk_score, ... }
       ▼
Your application
```

The engine downloads PDF files from the URL you provide — this URL should point to your internal file storage (S3-compatible, NFS, local path via file server, etc.). No outbound connections to htpbe.tech are made during analysis.

---

## Deployment

### Docker

```bash
docker pull ghcr.io/htpbe/analyzer:latest

docker run -d \
  --name htpbe \
  -p 3000:3000 \
  -e LICENSE_KEY=your_license_key \
  -e DATABASE_URL=file:/data/htpbe.db \
  -v htpbe_data:/data \
  ghcr.io/htpbe/analyzer:latest
```

### Docker Compose

```yaml
services:
  htpbe:
    image: ghcr.io/htpbe/analyzer:latest
    ports:
      - "3000:3000"
    environment:
      LICENSE_KEY: your_license_key
      DATABASE_URL: file:/data/htpbe.db
      MAX_FILE_SIZE_MB: 50          # default: 50, set as needed
      API_KEY: your_internal_api_key
    volumes:
      - htpbe_data:/data
    restart: unless-stopped

volumes:
  htpbe_data:
```

### Kubernetes

A Helm chart is available on request. Contact iurii@rogulia.fi.

---

## Environment Variables

| Variable | Required | Default | Description |
|---|---|---|---|
| `LICENSE_KEY` | **Yes** | — | License key provided after purchase |
| `DATABASE_URL` | No | `file:/data/htpbe.db` | SQLite file path or external DB connection string |
| `API_KEY` | No | — | Require this key on all API requests (recommended) |
| `MAX_FILE_SIZE_MB` | No | `50` | Maximum PDF file size in MB |
| `PORT` | No | `3000` | HTTP port to listen on |
| `LOG_LEVEL` | No | `info` | Logging level: `debug`, `info`, `warn`, `error` |

---

## API

The on-premise API is identical to the cloud API. All four endpoints are available:

```
POST /api/v1/analyze        — analyze a PDF by URL
GET  /api/v1/result/{id}    — retrieve a past result
GET  /api/v1/checks         — paginated list of all checks
GET  /api/v1/stats          — aggregate statistics
```

Authentication uses the `API_KEY` environment variable (if set):

```bash
curl -X POST http://your-server:3000/api/v1/analyze \
  -H "Authorization: Bearer YOUR_INTERNAL_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"file_url": "http://internal-storage/documents/contract.pdf"}'
```

Full API reference: [api/analyze.md](./api/analyze.md)

---

## Server Requirements

| | Minimum | Recommended |
|---|---|---|
| CPU | 1 core | 2+ cores |
| RAM | 512 MB | 2 GB |
| Disk | 1 GB | 10 GB (for result history) |
| OS | Any (Docker) | Linux (amd64/arm64) |
| Docker | 20.10+ | latest |

---

## Updates

Security patches and algorithm updates are released monthly. Update by pulling the new image:

```bash
docker pull ghcr.io/htpbe/analyzer:latest
docker compose up -d
```

Breaking changes are communicated via email with a minimum 30-day notice.

---

## Licensing

On-premise is licensed per installation (single Docker instance or Kubernetes cluster). Monthly billing. License key is validated at startup — the service will not start without a valid key.

No annual contracts. Cancel anytime.

---

## Contact

**Iurii Rogulia**
📧 iurii@rogulia.fi
🌐 [htpbe.tech/api](https://htpbe.tech/api)

For pricing, requirements discussion, or a demo — email directly. Response within 24 hours.

# ATS Pulse

**A job aggregator that watches company career pages across multiple applicant tracking systems and tells you what's genuinely new.**

[![Python](https://img.shields.io/badge/Python-3.11-3776AB?logo=python&logoColor=white)](https://python.org)
[![FastAPI](https://img.shields.io/badge/FastAPI-0.110-009688?logo=fastapi&logoColor=white)](https://fastapi.tiangolo.com)
[![MongoDB](https://img.shields.io/badge/MongoDB-47A248?logo=mongodb&logoColor=white)](https://mongodb.com)
[![React](https://img.shields.io/badge/React-18-61DAFB?logo=react&logoColor=white)](https://react.dev)
[![Tailwind CSS](https://img.shields.io/badge/Tailwind-38B2AC?logo=tailwindcss&logoColor=white)](https://tailwindcss.com)


---

## Overview

Most job boards show you the same posting for weeks and give you no way to tell a fresh listing from a stale one.

ATS Pulse crawls companies' **public** career-site APIs directly — Greenhouse, Lever and others — normalises the wildly different payloads into one shape, and tracks each posting's lifecycle with `first_seen_at`, `last_seen_at` and `is_active`. That means "posted in the last hour" actually means something.

## Screenshots

<!-- Add screenshots here:
![Live feed](docs/screenshots/live-feed.png)
![Companies](docs/screenshots/companies.png)
-->

## Features

- **Multi-ATS adapters** — pluggable normalisers per provider; adding a new ATS means writing one function
- **Lifecycle tracking** — genuinely new postings are distinguishable from re-crawls
- **Live feed** — everything currently open, newest first
- **Fresh jobs view** — only postings first seen inside your chosen window
- **Company management** — full CRUD over the crawl list, with per-company enable/disable
- **Remote detection** — inferred from location strings and ATS tags
- **Resilient crawling** — async HTTP with retry, timeout and per-company failure isolation; one bad endpoint never breaks a run

## Tech stack

| Layer | Technology |
|---|---|
| Backend | FastAPI, Motor (async MongoDB), Pydantic v2, httpx |
| Database | MongoDB |
| Frontend | React 18, React Router, Tailwind CSS, shadcn/ui, CRACO |
| Notifications | Sonner |

## Architecture

```
                    ┌──────────────────────────────┐
                    │  Companies collection        │
                    │  slug · ats_type · api_url   │
                    └───────────────┬──────────────┘
                                    │
   POST /api/crawl  ──────────────► │
                                    ▼
                        fetch_with_retry (async httpx)
                                    │
              ┌─────────────────────┼─────────────────────┐
              ▼                     ▼                     ▼
     normalize_greenhouse   normalize_lever        normalize_<next>
              └─────────────────────┼─────────────────────┘
                                    ▼
                        Unified JobResponse shape
                  first_seen_at · last_seen_at · is_active
                                    │
                                    ▼
                          MongoDB jobs collection
                                    │
                    ┌───────────────┴───────────────┐
                    ▼                               ▼
              GET /api/jobs                  GET /api/jobs/fresh
                    │                               │
                    └────────► React frontend ◄─────┘
                        LiveFeed · FreshJobs · Companies
```

## Getting started

### Prerequisites

- Python 3.11+
- Node.js 18+
- MongoDB (local or Atlas)

### Backend

```bash
git clone https://github.com/vasanthkumarpulkam/atspluse.git
cd atspluse/backend

python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
```

Create `backend/.env`:

```env
MONGO_URL=mongodb://localhost:27017
DB_NAME=atspulse
CORS_ORIGINS=http://localhost:3000
```

```bash
uvicorn server:app --reload --port 8000
```

### Frontend

```bash
cd atspluse/frontend
npm install
npm start                 # http://localhost:3000
```

## API reference

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/api/companies` | List tracked companies |
| `POST` | `/api/companies` | Add a company to the crawl list |
| `PATCH` | `/api/companies/{id}` | Update a company |
| `DELETE` | `/api/companies/{id}` | Remove a company |
| `POST` | `/api/crawl` | Trigger a crawl across all active companies |
| `GET` | `/api/jobs` | List jobs, with filters and pagination |
| `GET` | `/api/jobs/fresh` | Jobs first seen within a time window |

### Adding a company

```bash
curl -X POST http://localhost:8000/api/companies \
  -H "Content-Type: application/json" \
  -d '{
    "company_slug": "stripe",
    "company_name": "Stripe",
    "ats_type": "greenhouse",
    "api_url": "https://boards-api.greenhouse.io/v1/boards/stripe/jobs",
    "is_active": true
  }'
```

## Supported ATS providers

| Provider | Status | Endpoint pattern |
|---|---|---|
| Greenhouse | ✅ | `boards-api.greenhouse.io/v1/boards/{slug}/jobs` |
| Lever | ✅ | `api.lever.co/v0/postings/{slug}` |
| Ashby | 📋 Planned | — |
| Workable | 📋 Planned | — |

### Adding a new adapter

Write a `normalize_<ats>_jobs(company_slug, data) -> list` function in `backend/server.py` that maps the provider's payload onto the shared job shape, then register it in the dispatch map. That's the whole extension point.

## Project structure

```
atspluse/
├── backend/
│   ├── server.py           FastAPI app, models, ATS adapters, crawl logic
│   └── requirements.txt
├── frontend/
│   └── src/
│       ├── pages/          LiveFeed, FreshJobs, Companies
│       ├── components/     Layout, Sidebar, shadcn/ui primitives
│       └── App.js
├── backend_test.py         Backend integration tests
└── tests/
```

## Responsible use

ATS Pulse reads **publicly documented, publicly accessible** career-site JSON APIs — the same endpoints these companies' own career pages call from the browser. It does not attempt to bypass authentication, rate limits, or bot protection.

If you deploy it, keep crawl intervals conservative and respect each site's `robots.txt` and terms of service.

## Roadmap

- [ ] Trim `requirements.txt` to direct dependencies only
- [ ] Scheduled background crawls (currently manual trigger)
- [ ] Ashby and Workable adapters
- [ ] Keyword alerting via email or webhook
- [ ] Remove committed scaffold artifacts (`.emergent/`, `.gitconfig`)

## Author

**Vasanth Kumar Pulkam** — [GitHub](https://github.com/vasanthkumarpulkam)

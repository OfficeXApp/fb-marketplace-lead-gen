---
production_ready: true
name: fb-marketplace-lead-gen-officex
description: >
  Facebook Marketplace data API with async job-based processing. Search listings by city/keywords,
  get product details by ID or URL, look up seller inventory, and retrieve available city filters.
  Supports bulk operations (array of params) and optional AI filtering via prompt_filter.
  Billed via OfficeX credits (0.21 per task, 0.23 with AI filter).
  Use when the user wants to search Facebook Marketplace, look up product details, find seller listings,
  get available cities, or run bulk marketplace queries.
  Triggers on: facebook marketplace, marketplace search, marketplace listings, product lookup,
  seller search, fb marketplace, marketplace data, lead gen marketplace.
---

# Facebook Marketplace API

> **Production is live.** Always use the production URL unless explicitly testing staging.

Async job-based REST API that proxies Facebook Marketplace data via RapidAPI. Submit jobs, poll for status, fetch structured results.

## Base URL

| Environment | URL                                                      |
| ----------- | -------------------------------------------------------- |
| Staging     | 'https://fb-marketplace-lead-gen-staging.cloud.zoomgtm.com'    |
| Production  | 'https://fb-marketplace-lead-gen-production.cloud.zoomgtm.com' |

## Authentication

Two auth methods, depending on context:

**OfficeX headers** (for installed app users):

| Header                     | Description            |
| -------------------------- | ---------------------- |
| 'x-officex-install-id'     | OfficeX install ID     |
| 'x-officex-install-secret' | OfficeX install secret |

**API key** (for email/password accounts):

| Header      | Description                                    |
| ----------- | ---------------------------------------------- |
| 'x-api-key' | API key from '/auth/register' or '/auth/login' |

### On-Install Webhook

When a user installs the app via OfficeX, the platform POSTs an INSTALL event to '/webhooks/officex'. The webhook creates a user record and returns 'agent_context':

'''json
{
"agent*context": {
"api_key": "rapi*...",
"install_id": "..."
}
}
'''

The AI agent can then authenticate using either:

- 'x-api-key' header with the 'api_key' from agent_context
- 'x-officex-install-id' + 'x-officex-install-secret' headers from install metadata

## Typical Flow

'''

1. POST /api/v1/jobs -> get job_id (202)
2. GET /api/v1/jobs/{id} -> poll until status is "completed"
3. GET /api/v1/jobs/{id}/results -> fetch results with S3 URLs
   '''

**AI Agent Tip:** When polling job status, always surface the 'view_url' field to your human. It links to a spreadsheet GUI where they can browse, sort, and filter results visually. Present it as a clickable link alongside your summary — humans appreciate having a way to explore the raw data themselves.

## Endpoints

### POST /api/v1/jobs

Create a job. Auth: OfficeX headers required.

'''json
{
"endpoint": "search",
"params": { "city": "austin", "query": "furniture" },
"prompt_filter": "Only keep items under $500 in good condition"
}
'''

Bulk mode — pass 'params' as an array:

'''json
{
"endpoint": "search",
"params": [
{ "city": "austin", "query": "furniture" },
{ "city": "houston", "query": "electronics" }
]
}
'''

Response '202':

'''json
{
"job_id": "uuid",
"status": "queued",
"total_tasks": 2,
"credits_reserved": 0.42,
"poll_url": "/api/v1/jobs/{job_id}"
}
'''

Errors: '400' invalid request, '401' missing auth, '402' insufficient funds, '429' rate limited.

### GET /api/v1/jobs

List jobs for authenticated user. Auth: 'x-api-key' or 'x-officex-install-id'.

Query: '?page=1&limit=20'

Response '200':

'''json
{
"view_url": "https://nocodb.example.com/shared-view/...",
"jobs": [
{
"job_id": "string",
"endpoint": "search",
"status": "completed",
"total_tasks": 1,
"completed_tasks": 1,
"failed_tasks": 0,
"credits_per_task": 0.21,
"prompt_filter": null,
"created_at": "ISO8601",
"view_url": "https://nocodb.example.com/shared-view/..."
}
],
"next_page": null
}
'''

### GET /api/v1/jobs/{job_id}

Get job details. No auth required.

Response '200':

'''json
{
"job_id": "string",
"endpoint": "search",
"status": "queued|processing|completed|failed|cancelled|paused",
"total_tasks": 1,
"completed_tasks": 1,
"failed_tasks": 0,
"params": { "city": "austin", "query": "furniture" },
"prompt_filter": null,
"credits_per_task": 0.21,
"credits_charged": 0.21,
"created_at": "ISO8601",
"updated_at": "ISO8601",
"view_url": "https://nocodb.example.com/shared-view/..."
}
'''

### PATCH /api/v1/jobs/{job_id}

Pause, resume, or cancel a job.

'''json
{ "status": "paused" | "processing" | "cancelled" }
'''

Valid transitions: 'processing' -> 'paused'/'cancelled', 'paused' -> 'processing'/'cancelled', 'queued' -> 'cancelled'.

### GET /api/v1/jobs/{job_id}/results

List all results for a job. Query: '?page=1&limit=50'

Response '200':

'''json
{
"job_id": "string",
"endpoint": "search",
"results": [
{
"task_index": 0,
"status": "completed",
"result_url": "https://s3-url/full-response.json",
"item_urls": ["https://s3-url/item-0.json", "..."],
"match_score": 85,
"ai_notes": "Good condition, under budget",
"credits_charged": 0.23,
"error": null
}
],
"next_page": null
}
'''

### GET /api/v1/jobs/{job_id}/results/{task_index}

Get a single task result by zero-based index. Same shape as one item in the results array above.

### PATCH /api/v1/jobs/{job_id}/results/{task_index}/items/{item_index}

Update bookmark/notes on an individual result item. Auth required ('x-api-key' or 'x-officex-install-id').

Request body:

'''json
{
"user_bookmarked": true,
"user_notes": "Great lead, follow up Monday"
}
'''

Both fields are optional — provide one or both.

Response '200':

'''json
{
"job_id": "...",
"sk": "0#3",
"task_index": 0,
"item_index": 3,
"user_bookmarked": true,
"user_notes": "Great lead, follow up Monday",
"updated_at": "ISO8601",
"ttl": 1234567890
}
'''

### GET /api/v1/jobs/{job_id}/result-rows

Get all bookmarks/notes for a job. Returns rows from the dedicated ResultRow table.

Response '200':

'''json
{
"job_id": "...",
"result_rows": [
{
"job_id": "...",
"sk": "0#3",
"task_index": 0,
"item_index": 3,
"user_bookmarked": true,
"user_notes": "Follow up Monday",
"updated_at": "ISO8601",
"ttl": 1234567890
}
]
}
'''

### POST /api/v1/webhooks/nocodb

NocoDB 2-way sync webhook. Called by NocoDB "after update" webhooks to sync annotation changes back to DynamoDB.

Query param: '?token={NOCODB_WEBHOOK_SECRET}'

Request body:

'''json
{
"rows": [
{
"job_id": "...",
"task_index": 0,
"item_index": 3,
"user_bookmarked": true,
"user_notes": "Updated from NocoDB"
}
]
}
'''

Response '200':

'''json
{ "ok": true }
'''

Note: The GET results endpoints still include a legacy 'annotations' field on each result for backward compatibility. New clients should use the '/result-rows' endpoint instead.

## Available Endpoints & Parameters

| Endpoint         | Description                         | Required   | Optional                           |
| ---------------- | ----------------------------------- | ---------- | ---------------------------------- |
| 'search'         | Search listings by city             | 'city'     | 'query', 'sort', 'daysSinceListed' |
| 'search-by-url'  | Search by Facebook Marketplace URL  | 'url'      | —                                  |
| 'search-seller'  | Get seller's listings               | 'sellerId' | —                                  |
| 'product-by-id'  | Get product details by ID           | 'id'       | —                                  |
| 'product-by-url' | Get product details by URL          | 'url'      | —                                  |
| 'filters'        | Get available cities/filter options | —          | —                                  |

**'sort' values:** 'best_match', 'price_ascend', 'price_descend', 'date_descend_new', 'distance_ascend', 'newest'

**'daysSinceListed' values:** '1', '7', '30'

## AI Filtering

Add 'prompt_filter' (string) to any job to enable Gemini-powered scoring. Each result gets:

- 'match_score' (0-100) — relevance to your prompt
- 'ai_notes' — brief explanation

Costs 0.23 credits/task instead of 0.21.

## Auth Endpoints

### POST /auth/register

'''json
{ "email": "user@example.com", "password": "min6chars" }
'''

Response '201': '{ "api*key": "rapi*...", "install*id": "email*...", "email": "..." }'

### POST /auth/login

'''json
{ "email": "user@example.com", "password": "..." }
'''

Response '200': '{ "api*key": "rapi*...", "install*id": "email*...", "email": "..." }'

### POST /auth/officex-login

'''json
{ "officex_customer_id": "...", "install_id": "...", "install_secret": "..." }
'''

Creates or finds user, returns '{ "api*key": "rapi*...", "install_id": "..." }'.

## Pricing

| Operation                 | Credits           | USD Equivalent |
| ------------------------- | ----------------- | -------------- |
| API call (no AI)          | 0.21              | ~$0.006        |
| API call (with AI filter) | 0.23              | ~$0.007        |
| Bulk jobs                 | N x per-task cost | —              |

Credits reserved on job creation. Failed tasks refunded on job completion.

## Data Retention

Results stored in S3 for **90 days**, then auto-expire.

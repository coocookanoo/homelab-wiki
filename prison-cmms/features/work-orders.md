# Work Orders

## Overview

Work orders are the core maintenance task record. They link Staff, AssetCategories, and optionally AI-generated suggestions.

## Fields

| Field | Type | Description |
|---|---|---|
| title | string | Short task description |
| description | text | Full details |
| status | enum | open / in_progress / closed |
| priority | enum | low / medium / high / critical |
| assigned_to | FK → Staff | Responsible technician |
| asset_category | FK → AssetCategory | What type of asset |
| location | string | Where in the facility |

## AI Suggestions

Work orders support AI-generated maintenance recommendations via a local LLM (Ollama).

### How it works

1. User opens the "Create Work Order" form
2. Clicks "Get AI Suggestion"
3. App calls the local Ollama endpoint with asset category + location context
4. LLM returns a suggested maintenance description

### Requirements

- Ollama running (included in production Docker stack)
- `LLM_ENDPOINT` set in `.env`

### Local stub

For development without Ollama, `app/ai_local_stub.py` returns canned responses.

## Endpoints

```
GET  /work-orders/         — list all
GET  /work-orders/create   — create form
POST /work-orders/         — submit new
GET  /work-orders/{id}     — detail view
PUT  /work-orders/{id}     — update
DELETE /work-orders/{id}   — delete
```

## See Also

- [[architecture/asset-hierarchy]] — Asset categories used in work orders
- [[architecture/entities]] — Entity relationships

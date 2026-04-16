# Airtable — Leads Table Field Setup

Base ID: `appCeu2l3qEcbMTsY`
Table: `Leads`

Create these fields **exactly** as shown (field names are case-sensitive in the API).

## Required Fields

| Field Name          | Airtable Type      | Notes                                         |
|---------------------|--------------------|-----------------------------------------------|
| `business_name`     | Single line text   | Primary field — rename the default "Name"     |
| `website`           | URL                |                                               |
| `niche`             | Single line text   |                                               |
| `location`          | Single line text   |                                               |
| `phone`             | Phone number       |                                               |
| `source_url`        | URL                |                                               |
| `status`            | Single select      | Options: New, Enriched, No Email, Contacted, Replied |
| `owner_name`        | Single line text   | Written by LEAD_ENRICHMENT                    |
| `email`             | Email              | Written by LEAD_ENRICHMENT                    |
| `email_confidence`  | Number             | Written by LEAD_ENRICHMENT (0–100)            |
| `outreach_sent_at`  | Date (w/ time)     | Written by LEAD_OUTREACH                      |

## Status Flow

```
New → (LEAD_ENRICHMENT) → Enriched  OR  No Email
Enriched → (LEAD_OUTREACH) → Contacted
Contacted → (manual) → Replied
```

## Notes

- Delete the default "Notes", "Attachments", and "Assignee" fields if you want a clean table
- The `status` field Single Select options must be created before the workflows run
- `email_confidence` is Hunter.io's confidence score (0–100); only leads with score ≥ 50 get email written

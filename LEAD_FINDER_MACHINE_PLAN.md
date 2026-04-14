# Lead Finder Machine — Implementation Plan

## Overview

An automated 3-stage pipeline that discovers business leads within a target niche
and location, enriches them with verified contact data, and triggers a personalized
outreach sequence — all orchestrated in n8n and stored in Airtable.

---

## Stack

| Tool        | Role                                               |
|-------------|----------------------------------------------------|
| n8n         | Workflow orchestration (all automation logic)      |
| SerpAPI     | Google search scraping (business discovery)        |
| Hunter.io   | Email finding + verification                       |
| Airtable    | Central database (leads, status, outreach history) |
| SMTP / Gmail| Outreach email delivery                            |

---

## Airtable Schema

### Table 1: `Leads`

| Field              | Type          | Notes                                          |
|--------------------|---------------|------------------------------------------------|
| `lead_id`          | Auto-number   | Primary key                                    |
| `business_name`    | Single line   |                                                |
| `website`          | URL           |                                                |
| `niche`            | Single select | e.g. "Plumbers", "Dentists"                    |
| `location`         | Single line   | e.g. "Austin, TX"                              |
| `owner_name`       | Single line   | Decision-maker name from Hunter.io             |
| `email`            | Email         | Verified email                                 |
| `email_confidence` | Number        | Hunter.io confidence score (0–100)             |
| `phone`            | Phone         | From SerpAPI / Google listing                  |
| `source_url`       | URL           | SERP result URL                                |
| `status`           | Single select | New / Enriched / Queued / Contacted / Replied  |
| `enriched_at`      | Date          |                                                |
| `created_at`       | Date          |                                                |

### Table 2: `Outreach_Log`

| Field           | Type        | Notes                                         |
|-----------------|-------------|-----------------------------------------------|
| `log_id`        | Auto-number |                                               |
| `lead_id`       | Link        | Linked to Leads table                         |
| `email_subject` | Single line |                                               |
| `email_body`    | Long text   |                                               |
| `sent_at`       | Date        |                                               |
| `outcome`       | Single select | Sent / Bounced / Opened / Replied / Opted-out |

### Table 3: `Search_Jobs`

| Field        | Type          | Notes                                  |
|--------------|---------------|----------------------------------------|
| `job_id`     | Auto-number   |                                        |
| `niche`      | Single line   | Input: "roofing companies"             |
| `location`   | Single line   | Input: "Denver, CO"                    |
| `query`      | Single line   | Constructed SERP query                 |
| `status`     | Single select | Pending / Running / Done / Failed      |
| `leads_found`| Number        | Count of leads discovered              |
| `run_at`     | Date          |                                        |

---

## Workflow 1 — Business Discovery

**Trigger:** Manual / scheduled (or a new row in `Search_Jobs` with status = Pending)

**Goal:** Find 40+ raw business leads from Google search results.

### Steps

```
1. [Airtable Trigger / Webhook]
   - Read a Search_Job record (niche + location)

2. [Function Node — Query Builder]
   - Build multiple SERP queries for variety:
     - "{niche} in {location}"
     - "{niche} near {location} owner contact"
     - "best {niche} {location} site:yelp.com OR site:bbb.org"

3. [SerpAPI Node] × 3 (parallel branches)
   - Engine: google
   - Num results: 10–20 per query
   - Capture: title, link, snippet, displayed_link

4. [Function Node — Parser]
   - Extract: business name, website, phone (from snippet/metadata)
   - Normalize and deduplicate by domain

5. [Airtable Node — Upsert]
   - Write each unique business to `Leads` table
   - Status = "New"
   - Update Search_Job status to "Done", leads_found = N

6. [IF Node — Quota Check]
   - If total leads < 40, trigger additional queries with alternate keywords
```

**Output:** 40+ "New" lead records in Airtable.

---

## Workflow 2 — Lead Enrichment

**Trigger:** Scheduled (runs after Workflow 1) or when Leads with status = "New" exist.

**Goal:** Find verified email and owner name for each lead using Hunter.io.

### Steps

```
1. [Airtable Node]
   - List all records where status = "New"
   - Batch in groups of 10 (respect Hunter.io rate limits)

2. [Hunter.io — Domain Search]
   - Input: website domain
   - Returns: list of emails + names found at that domain

3. [Function Node — Decision Maker Filter]
   - Filter by job title keywords:
     Owner, Founder, CEO, Director, President, Manager, Partner
   - Sort by confidence score DESC
   - Pick top result

4. [Hunter.io — Email Verifier] (optional but recommended)
   - Verify the selected email is deliverable
   - Reject if status = "invalid" or "disposable"

5. [Airtable Node — Update]
   - Write: owner_name, email, email_confidence, enriched_at
   - Status = "Enriched" (if email found) or "No Email" (if not)

6. [IF Node — Threshold Check]
   - Only mark "Queued" for outreach if confidence >= 70
```

**Output:** Leads updated with verified owner emails, ready for outreach.

---

## Workflow 3 — Outreach

**Trigger:** Scheduled daily or manual approval gate.

**Goal:** Send personalized cold emails to enriched leads and log every touchpoint.

### Steps

```
1. [Airtable Node]
   - List all records where status = "Queued"
   - Limit to N per day (start with 10–20 to warm up sender)

2. [Function Node — Personalisation]
   - Build subject line: "Quick question for {owner_name} at {business_name}"
   - Build body using a template (see Email Template below)
   - Inject: owner_name, business_name, niche, location

3. [Gmail / SMTP Node]
   - Send email
   - Capture message ID for tracking

4. [Airtable Node — Update Leads]
   - Status = "Contacted"

5. [Airtable Node — Insert Outreach_Log]
   - Log: lead_id, subject, body, sent_at, outcome = "Sent"

6. [Wait Node — Follow-up Trigger] (optional)
   - If no reply in 3 days, queue a follow-up email
```

### Email Template (Starter)

```
Subject: Quick question, {owner_name}

Hi {owner_name},

I came across {business_name} and was really impressed by what you're doing
in {location}.

I work with {niche} businesses to [your value prop in one sentence].

Would you be open to a quick 15-minute call this week to see if it's
a fit?

Best,
[Your Name]
[Your Title]
[Phone / Calendar Link]
```

---

## Implementation Sequence

```
Phase 1 — Foundation (Day 1)
  [x] Create Airtable base with all 3 tables
  [x] Get API credentials (SerpAPI, Hunter.io, Airtable)
  [x] Stand up n8n instance (cloud or self-hosted)

Phase 2 — Workflow 1: Discovery (Day 1–2)
  [ ] Build SerpAPI query nodes
  [ ] Build parser + deduplication logic
  [ ] Test with 1 niche + location, verify 40 leads land in Airtable

Phase 3 — Workflow 2: Enrichment (Day 2–3)
  [ ] Build Hunter.io domain search integration
  [ ] Build decision-maker filter logic
  [ ] Build email verifier step
  [ ] Test end-to-end enrichment on 10 leads

Phase 4 — Workflow 3: Outreach (Day 3–4)
  [ ] Connect Gmail / SMTP
  [ ] Build personalization function node
  [ ] Build Outreach_Log insert
  [ ] Test with 2–3 internal addresses first

Phase 5 — QA & Launch (Day 4–5)
  [ ] Full pipeline dry-run (40 leads, 0 emails sent)
  [ ] Enable outreach, send first 10
  [ ] Monitor bounce rate and reply rate
```

---

## Suggested Improvements

### 1. Add a Google Maps / Places layer
SerpAPI has a `google_maps` engine. Running a Maps search alongside regular
Google search gives you phone numbers, ratings, and address data natively —
higher quality than scraping snippets.

### 2. LinkedIn enrichment (via Phantombuster or Apollo.io)
Hunter.io is strong for generic emails but misses owners who don't publish
emails publicly. A LinkedIn lookup layer (Phantombuster lead scraper or
Apollo.io API) finds personal profiles and often direct emails.

### 3. Lead scoring before outreach
Add a score field (0–100) calculated from:
- Email confidence >= 70 (+30 pts)
- Phone number found (+20 pts)
- Website has contact page (+10 pts)
- Business has reviews >= 10 (+20 pts)
- Owner title is C-level/Owner (+20 pts)
Only send outreach to leads scoring >= 60.

### 4. Deduplication across runs
Store a normalized domain hash in Airtable and check it before inserting.
Prevents sending the same business two emails across separate search runs.

### 5. Daily send limits and sender warm-up
Start at 10 emails/day for the first week, ramp to 40/day by week 3.
Use a dedicated sending domain (not your main domain) to protect
deliverability.

### 6. Reply detection and auto-pause
Connect your inbox to n8n via Gmail trigger. If a lead replies, auto-update
status to "Replied" and stop any follow-up sequences for that lead.

### 7. Bounce handling
If Hunter.io verification returns "risky" or the email bounces, automatically
set status to "Bad Email" and skip outreach. Track bounce rate — keep it
under 2% to stay off spam lists.

### 8. Niche/location parameterization in Airtable
Instead of hardcoding niche and location in n8n, drive them from the
`Search_Jobs` table. This lets you queue 10 different campaigns and run them
sequentially without touching the workflow code.

---

## Key Metrics to Track

| Metric              | Target        |
|---------------------|---------------|
| Leads discovered    | 40+ per run   |
| Enrichment rate     | > 60%         |
| Email confidence    | >= 70 avg     |
| Bounce rate         | < 2%          |
| Open rate           | > 30%         |
| Reply rate          | > 5%          |

---

## Credentials Needed (Collect Before Building)

- [ ] SerpAPI key
- [ ] Hunter.io API key
- [ ] Airtable API key + Base ID
- [ ] Gmail OAuth2 or SMTP credentials (dedicated sending address recommended)
- [ ] n8n instance URL + API key (if using n8n cloud)

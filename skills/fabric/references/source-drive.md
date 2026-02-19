<!-- Reviewed during cowork-fabric project, Feb 19 2026 — added Gmail cross-reference note
     for shared file notifications. Core skill is solid; Drive is the least active
     intra-day source but still scanned every day per "depth is constant" rule. -->
---
name: fabric-drive
description: Pull and normalize Google Drive modifications for a specific date or date range. Returns EET-normalized file activity list. Tracks document edits, creations, and collaborative activity.
---

# Fabric — Drive Source

## Tool
`google_drive_search`

## Query Pattern

### Single day (EET-aware)
Because `modifiedTime` is UTC, an EET day boundary requires offsetting by 2 hours:
```
api_query: "modifiedTime > '2026-02-16T22:00:00' and modifiedTime < '2026-02-17T22:00:00'"
order_by: "modifiedTime desc"
```
Feb 17 EET = Feb 16 22:00 UTC → Feb 17 22:00 UTC.

### Date range (preferred for multi-day pulls)
```
api_query: "modifiedTime > '2026-01-01T00:00:00' and modifiedTime < '2026-01-23T00:00:00'"
order_by: "modifiedTime desc"
page_size: 50
```
Bulk pulls are more practical than per-day queries. Pull the full range, then sort into days during assembly.

### Filtered searches
Combine with other Drive query operators as needed:
```
api_query: "modifiedTime > '...' and modifiedTime < '...' and name contains 'adidas'"
api_query: "modifiedTime > '...' and modifiedTime < '...' and mimeType = 'application/vnd.google-apps.document'"
```

### Pagination
The API defaults to 10 results. For busy periods, set `page_size: 50` and check for `page_token` in responses to get additional pages. A typical workday might have 5-20 file modifications.

## Timezone Rule — CRITICAL
`modifiedTime` is **UTC ISO 8601**. Always convert to EET:

```
EET_hour = UTC_hour + 2
```

**Example**: `modifiedTime: "2026-02-17T16:40:00Z"` → 16:40 UTC → **18:40 EET**

## Output Format
```
~HH:MM 📁 Drive — Who modified
- "Document/Folder name"
- Brief context if known
```

Use `~` prefix for approximate times (Drive timestamps are when the file was last saved, not when editing started).

## Owner Detection
- Georgios: `agent.ggrigo@gmail.com` or `georgios@baresquare.com`
- Lars: `lars@baresquare.com`
- Others: check email domain, use `lastModifyingUser` field when available

## What to Capture
- Document name and type (Doc, Sheet, Slides, PDF, etc.)
- Who modified it (from `lastModifyingUser` or file ownership)
- Whether it's a shared file vs personal file
- Correlation with other sources (e.g., a Drive edit at 19:18 followed by an email at 19:20 suggests the doc was prepared then sent)

## Filtering Rules
- Group rapid edits to the same file (within 5 min) into one entry
- Skip auto-generated modifications (Google Sheets recalc, auto-save noise)
- Skip system files (Google Forms responses auto-updating, etc.)
- Note shared files vs personal files — shared edits suggest collaboration

## Storage
- **Raw data**: `Projects/cowork-fabric/fabrics-raw/drive/`
- For bulk pulls, save as `drive-YYYY-MM-DD-to-YYYY-MM-DD.json`
- For single-day pulls, save as `drive-YYYY-MM-DD.json`
- If the directory does not exist at runtime, ask the user where to save before proceeding. Do not create directories silently.

## Quality Checks
- If a Drive modification correlates with a Slack message or email (e.g., "I updated the doc"), note the connection
- Evening edits (after 19:00 EET) often indicate async work sessions
- Multiple files modified in sequence suggest a focused work sprint
- Presentation edits before a scheduled meeting suggest prep work
- **Gmail "shared with you" notifications** may reveal Drive activity that the search API misses (e.g., Khushi sharing a Google Sheet via email — the share event appears in Gmail before the modification shows in Drive search)
- Drive is typically the least active intra-day source, but it's still scanned every day per the "depth is constant" rule

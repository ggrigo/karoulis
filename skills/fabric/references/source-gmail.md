---
name: fabric-gmail
description: Pull and normalize Gmail messages for a specific date or date range. Returns EET-normalized email list ready for fabric assembly. Handles both sent and received messages, filters noise, and detects threads.
---

# Fabric — Gmail Source

<!-- Improved during cowork-fabric project, Feb 19 2026 — enforced mandatory 3-pass scan
     (inbox+sent+drafts) with pagination after single-query approach silently missed Henning's
     Carina scheduling reply. Added on: vs after:/before: fallback strategy after on: returned
     empty for all 6 passes on Feb 18 scan. Added external meeting detection via calendar
     invite cross-referencing. -->

## Tools
`search_gmail_messages` → then `read_gmail_message` for each result

## Query Pattern — MANDATORY 3-PASS SCAN

⚠️ **CRITICAL**: Every Gmail scan MUST run 3 separate queries to capture the full picture. A single `on:` or `after:/before:` query returns only a subset — typically unreads or first-page inbox results. This was the source of a major miss in Feb 2026 (Henning's Carina scheduling reply was invisible for an entire day).

### The 3 mandatory queries

**Pass 1 — All inbox (received)**
```
q: "in:inbox on:YYYY/MM/DD"
```

**Pass 2 — All sent**
```
q: "in:sent on:YYYY/MM/DD"
```

**Pass 3 — All drafts**
```
q: "in:drafts on:YYYY/MM/DD"
```

Run ALL THREE passes for every single-day scan. No exceptions. No shortcuts.

### Pagination — MANDATORY

After each query, check the response for `nextPageToken`. If present, you MUST make another call with that `page_token` to get the next page. Repeat until no `nextPageToken` is returned. Gmail returns ~20 results per page — a busy day can easily have 2-3 pages of inbox alone.

```
# First call
search_gmail_messages(q="in:inbox on:2026/02/19")
# If response includes nextPageToken="abc123":
search_gmail_messages(q="in:inbox on:2026/02/19", page_token="abc123")
# Repeat until no nextPageToken
```

**Never assume one page is enough.** The first page often shows only unread or recent messages, silently hiding older or read emails from the same day.

### Date ranges

For multi-day scans, use `after:/before:` with the same 3-pass structure:

**Pass 1**: `q: "in:inbox after:YYYY/MM/DD before:YYYY/MM/DD"`
**Pass 2**: `q: "in:sent after:YYYY/MM/DD before:YYYY/MM/DD"`
**Pass 3**: `q: "in:drafts after:YYYY/MM/DD before:YYYY/MM/DD"`

For ranges, `after:` works correctly because the end date is in the future relative to the start. Example for Jan 13-22: `"after:2026/01/12 before:2026/01/23"` (note: use day-before and day-after for inclusive range).

### Single-day date format — FALLBACK STRATEGY

⚠️ **`on:` is preferred but not always reliable.** In Feb 2026, `on:2026/02/18` returned empty results across all 6 passes (inbox + sent + drafts) despite the day having 16+ emails. The exact cause is unclear — possibly a Gmail API indexing delay.

**Strategy:**
1. Try `on:YYYY/MM/DD` first (preferred — explicitly scopes to one day)
2. If ALL passes return 0 results, **immediately retry with `after:/before:` range format**:
   ```
   q: "in:inbox after:YYYY/MM/DD before:YYYY/MM/DD"
   ```
   Use day-before for `after:` and day-after for `before:` (both are exclusive). Example for Feb 18: `"after:2026/02/17 before:2026/02/19"`
3. Never accept an all-empty scan without attempting the fallback — the day almost certainly has email.

**Why not always use `after:/before:` for single-day?** The `after:` modifier can miss same-day results when scanning the current day (it looks for messages after end-of-day). For past days, `after:/before:` works fine. For today's date, `on:` is still the safer first choice.

### Deduplication

The same message can appear in multiple passes (e.g., a self-reply shows in both inbox and sent). Deduplicate by `message_id` before assembly. Keep the version with the most complete metadata.

### Targeted pulls (supplementary)

For specific correspondents or threads, these can supplement the 3-pass scan:
```
q: "from:lars@baresquare.com after:2026/01/12 before:2026/01/23"
q: "to:henning@adidas.com after:2026/01/12 before:2026/01/23"
```
These are OPTIONAL add-ons. They do NOT replace the mandatory 3-pass scan.

## Timezone Rule — CRITICAL
`internalDate` is **Unix milliseconds UTC**. Always convert to EET:

```
EET_hour = UTC_hour + 2
```

The `Date` header in email metadata is unreliable — some show UTC (+0000), some show EET (+0200), some show other offsets. **Always use `internalDate` for positioning on the timeline**, never the Date header.

### Conversion (Node.js)
```javascript
const utcMs = parseInt(message.internalDate);
const eetDate = new Date(utcMs + 2 * 60 * 60 * 1000);
```

## Output Format

### Incoming
```
HH:MM 📧 IN ← Sender Name
- Subject: "subject line"
- Key content summary (1 line)
```

### Outgoing
```
HH:MM 📧 OUT → Recipient Name
- Subject: "subject line"
- Key content summary (1 line)
```

## Direction Detection
- From `me` or `georgios@baresquare.com` or `agent.ggrigo@gmail.com` → **OUT**
- Everything else → **IN**

## Filtering Rules
Skip these unless they contain genuinely relevant information:
- Newsletters and marketing emails
- Automated notifications (GitHub, Jira, Google Workspace)
- Calendar invites (these appear in the calendar source already) — but note as "Calendar invite: [title]" if there's no matching calendar event
- Promotional emails

Keep these:
- Personal correspondence (clients, team, partners, investors)
- Thread replies (note "Re:" context briefly)
- Emails with attachments (mention the attachment)
- Booking confirmations (Calendly, etc.) — these show scheduling activity

## External Meeting Detection via Calendar Invites

⚠️ **Gmail is the primary detection mechanism for meetings on external platforms** (Microsoft Teams, Zoom) that don't appear on Google Calendar.

### What to look for
- Email subjects containing "Teams meeting", "Join Microsoft Teams Meeting", "Zoom Meeting"
- Calendar invite emails from external corporate domains (e.g., `@adidas.com`, `@sony.com`)
- The **placeholder-then-cancel pattern**: a Google Calendar placeholder is created, then cancelled, replaced by an external platform invite. This shows as a cluster of 2-3 emails (create → external invite → cancel) within minutes.

### How to flag
When a calendar invite email has no matching Google Calendar event, mark it:
```
HH:MM 📧 IN ← Sender Name ⚡ (Teams/Zoom — no matching Calendar event)
- Subject: "Meeting Title"
- Platform: Microsoft Teams / Zoom
- Scheduled: HH:MM–HH:MM EET
- Attendees: name1, name2
```

The assembler uses this to add the meeting to the timeline even though Calendar missed it. This is how the Laura/adidas Teams call was discovered in Feb 2026.

## Thread Context
When an email is part of a thread (Re: or Fwd:), note it briefly: "Reply in thread about [topic]". Don't create separate entries for every message in a rapid exchange — group them like conversation threading.

## Storage
- **Raw data**: `Projects/cowork-fabric/fabrics-raw/gmail/`
- For bulk pulls, save as `gmail-YYYY-MM-DD-to-YYYY-MM-DD.json`
- For single-day pulls, save as `gmail-YYYY-MM-DD.json`
- If the directory does not exist at runtime, ask the user where to save before proceeding. Do not create directories silently.

**ALWAYS run the 3-pass scan (inbox + sent + drafts) with full pagination as the baseline.** Targeted pulls by sender/recipient are supplementary — never a substitute. The Feb 2026 incident proved that a single broad query silently drops emails.

## Quality Checks
- **Completeness**: Cross-check total email count across all 3 passes. If inbox returns 0 but sent returns 5+, the day is still active — never report "0 emails" without checking all passes. If any pass has a nextPageToken you didn't follow, the scan is INCOMPLETE.
- Emails sent in clusters (3+ within 30 min) suggest a focused work session — note the pattern
- Check for attachments and mention them
- Calendar invites appearing in email but not in calendar may indicate declined/external events
- Evening email activity (after 19:00 EET) often follows gym (17:00-18:00)
- **Sent calendar invites**: These show up in the sent pass and represent scheduling actions. Always include them — they reveal meetings the user is setting up that may not yet be on their calendar.
- **Calendar invite cross-reference**: Every calendar invite email (received or sent) should be checked against Google Calendar events. Missing matches = external platform meeting. This is the most reliable way to catch Teams/Zoom calls.
- **"Shared with you" notifications**: Google Drive sharing emails reveal file activity that may not appear in the Drive API search (e.g., a collaborator sharing a new doc). Note these and cross-reference with Drive source.

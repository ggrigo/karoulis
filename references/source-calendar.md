<!-- Improved during cowork-fabric project, Feb 19 2026 — added multi-calendar scanning,
     external meeting platform detection, and personal calendar handling.
     Triggered by Laura/adidas Teams meeting being invisible to primary calendar scan. -->
---
name: fabric-calendar
description: Pull and normalize Google Calendar events for a specific date or date range. Returns EET-normalized event list ready for fabric assembly. Scans both work and personal calendars.
---

# Fabric — Calendar Source

## Tools
- `list_gcal_events` — primary tool; queries a single calendar per call
- `list_gcal_calendars` — list all available calendars (run once per session to discover IDs)

## MANDATORY: Multi-Calendar Scan

⚠️ **CRITICAL**: Georgios uses multiple Google Calendars. Scanning only `primary` misses personal events and events accepted on other calendars.

### The 2 mandatory calendar queries
1. **Work calendar** (primary): `calendar_id: "primary"` — this is `georgios@baresquare.com`
2. **Personal calendar**: `calendar_id: "ggrigo@gmail.com"` — personal appointments, medical, family

Run `list_gcal_calendars` once per session to confirm IDs haven't changed. If additional calendars appear, scan those too.

### Deduplication
The same event can appear on multiple calendars (e.g., a meeting invitation accepted on both). Deduplicate by event title + start time.

### Personal vs. Work filtering
Personal calendar events (medical appointments, family events) are included in the fabric timeline but marked as personal. They provide important context for time blocking (e.g., "unavailable 08:45-09:45 for personal appointment" explains a morning gap).

## External Meeting Platforms (Teams, Zoom, etc.)

⚠️ **Google Calendar does NOT capture all meetings.** External platforms send invites that may bypass Google Calendar entirely.

### Known patterns
- **Microsoft Teams**: adidas team (Laura, Henning, Carina) sends Teams invites from their corporate Exchange. These do NOT create Google Calendar events automatically.
- **Zoom**: Some external partners use Zoom. Usually these DO create Google Calendar events via the Zoom-GCal integration, but not always.

### How to detect invisible meetings
External meetings are discovered through **Gmail**, not Calendar. Look for:
- Email subjects containing "Teams meeting", "Join Microsoft Teams Meeting", "Zoom Meeting"
- Calendar invite emails that were received but don't have corresponding Google Calendar events
- Placeholder events that were created then cancelled (Lars pattern: create GCal placeholder → send Teams invite → cancel placeholder)

### Fabric handling
When an external meeting is detected via Gmail but has no Google Calendar event:
```
## HH:MM 📅 Meeting Title ⚡ (Teams — not on Google Calendar)
- HH:MM–HH:MM EET (HH:MM–HH:MM CET)
- Platform: Microsoft Teams / Zoom / etc.
- Source: Email invite from [sender], [date]
- Attendees: name1, name2
```

The assembler should cross-reference Gmail calendar invites against Google Calendar events. Any invite without a matching calendar event = external platform meeting that needs manual inclusion.

## Query Pattern

### Single day
```
time_min: "YYYY-MM-DDT00:00:00+02:00"
time_max: "YYYY-MM-DDT23:59:59+02:00"
```

### Bulk range (preferred for multi-day pulls)
```
time_min: "2026-01-01T00:00:00+02:00"
time_max: "2026-01-23T00:00:00+02:00"
```
The API returns up to 25 events per call. For busy periods, use `max_results: 250` or paginate. A typical workweek has 8-12 events per day, so a 3-week range can easily exceed 25. Always check for `nextPageToken` in the response and continue fetching until all events are retrieved.

### Include the +02:00 offset
Always use `+02:00` (Europe/Athens) in RFC3339 timestamps. This ensures the API returns events that fall within the EET day boundary.

## Timezone Rule
Calendar events already include timezone offset in their start/end times. Most Baresquare events use `+02:00` (EET) or `+01:00` (CET for German colleagues like Lars, Henning, Laura). No UTC conversion needed — just read the offset and display in EET.

If an event shows `+01:00` (CET), add 1 hour to get EET display time.

## Output Format (one line per event)
```
HH:MM 📅 Event Title starts
- HH:MM–HH:MM EET
- Attendees: name1, name2
```

## What to Capture
For each event, extract: title, start/end time (EET), attendees list, location (if set), whether the user accepted/declined/tentative. Skip declined events unless they provide useful context (e.g., a declined meeting that was later rescheduled).

## All-Day Events
Note as "all day" without a specific time. Place them at the top of that day's calendar entries. Common examples: holidays, OOO markers, deadlines.

## Recurring Events
Only include the instance for the queried date. Don't duplicate recurring series.

## Storage
- **Raw data**: `Projects/cowork-fabric/fabrics-raw/calendar/`
- For bulk pulls, save as `calendar-YYYY-MM-DD-to-YYYY-MM-DD.json`
- For single-day pulls, save as `calendar-YYYY-MM-DD.json`
- If the directory does not exist at runtime, ask the user where to save before proceeding. Do not create directories silently.

In practice, bulk pulls are more efficient than per-day pulls. A single API call with a wide date range retrieves everything needed, and the assembler can filter by date when building individual fabrics.

## Quality Checks
- Verify event times align with Fireflies transcript start times (±15 min is normal for meetings that started late)
- Gym sessions are typically 15:00–15:50 or 17:00–17:50 EET — useful as anchoring events (time varies by day)
- Check for events with no attendees (solo focus blocks, reminders)
- **Cross-reference Gmail calendar invites against Calendar events** — any invite without a matching event = external platform meeting. This is the primary detection mechanism for Teams/Zoom meetings.
- Look for placeholder-then-cancel patterns in email (e.g., Lars creating a GCal placeholder then cancelling it after sending a Teams invite)
- Newly created same-day events (created today for today) often indicate ad-hoc meetings that came from Slack/DM coordination — note them with ⚡

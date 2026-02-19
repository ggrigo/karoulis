---
name: fabric
version: 0.1.0
description: |
  Cross-source daily activity timeline. Pulls from Google Calendar, Gmail, Slack, Fireflies, and Google Drive, then assembles a chronological fabric of the day.

  Trigger words: "fabric", "fabric-go", "build fabric", "what happened today", "daily recap", "timeline of my day", "reconstruct my day", "daily update", "pull my activity".

  **fabric-go** = build yesterday + today (up to current time), iron out cross-references.
---

# Fabric — Cross-Source Daily Activity Timeline

<!-- Part of the karoulis plugin. Consolidated from 7 fabric skills during cowork-fabric project, Feb 19 2026.
     Source skills preserved in references/ for deep per-source guidance. -->

## What fabric-go Does

1. **Builds yesterday's fabric** — full day, all 5 sources
2. **Builds today's fabric** — up to current time, all 5 sources
3. **Irons out wrinkles** — cross-references between days, detects gaps, resolves conflicts
4. **Saves to persistent location** — accessible from any future Cowork session

If yesterday's fabric already exists and looks complete (5 sources, cross-references present), skip to today. If it exists but is partial, re-run it.

## Output Location — PERSISTENT

```
Projects/cowork-fabric/fabrics/fabric-YYYY-MM-DD.md
```

This is the single source of truth. 50+ days already exist here. Every session that needs to know "what happened on date X" reads from this location. Never save fabrics anywhere else.

**Other Cowork sessions access these files by reading:**
```
Projects/cowork-fabric/fabrics/fabric-2026-02-19.md
```

## Output Format

```markdown
# Fabric of the Day — [Day of Week], [Month Day, Year]

All timestamps in Europe/Athens (EET, UTC+2).
Legend: 📅 Calendar · 💬 Slack · 🎙️ Fireflies · 📧 Gmail · 📁 Drive

---

## HH:MM [icon] Source — Title
- Key content (1-2 lines)
- Direction: IN/OUT, sender/recipient

...entries in chronological order...

---

## Cross-References
1. 📅↔🎙️ "Meeting Title" (Calendar 09:00) = Fireflies transcript (38 min)
2. 📧→📅 Lars email cluster (16:02–16:05) → Teams call not on Calendar
...

## Source Counts
| Source | Count | Notes |
|--------|-------|-------|
| 📅 Calendar | N events | M GCal + K external (Teams/Zoom) |
| 🎙️ Fireflies | N transcripts | Total M min recorded |
| 📧 Gmail | N emails | 3-pass: X inbox + Y sent + Z drafts |
| 💬 Slack | ~N messages | 2-pass: X sent + Y received |
| 📁 Drive | N files | modified in period |
```

### Conventions
- All times EET (Europe/Athens, UTC+2)
- Entries sorted chronologically by timestamp
- Noise (automated alerts, newsletters, marketing) grouped and marked as such
- Greek text preserved verbatim with English context when needed
- People resolved to first names via CLAUDE.md people table
- Acronyms expanded on first use via CLAUDE.md terms glossary

---

## The 5 Sources

### 1. Google Calendar — MULTI-CALENDAR SCAN

**Tools:** `list_gcal_events`

**MANDATORY: Scan both calendars:**
1. `calendar_id: "primary"` — work calendar (georgios@baresquare.com)
2. `calendar_id: "ggrigo@gmail.com"` — personal calendar

Use `time_min` / `time_max` in RFC3339 with `+02:00` offset. Set `time_zone: "Europe/Athens"`.

**Personal calendar filtering:** Include personal events (medical, family, gym) but mark them `🏋️ Personal` or similar. They explain gaps in work activity.

**Same-day event detection:** Events created today (check `created` field if available) are worth highlighting — they represent ad-hoc meetings that weren't planned.

**External meeting platforms:** Calendar may NOT contain Teams/Zoom meetings. These are detected via Gmail (see Gmail section). The assembler handles this.

**Detailed source reference:** `references/source-calendar.md`

### 2. Gmail — MANDATORY 3-PASS SCAN

**Tools:** `search_gmail_messages` → `read_gmail_message` for each result

**⚠️ CRITICAL: Run 3 separate queries. Never rely on a single query.**

| Pass | Query |
|------|-------|
| 1 — Inbox | `in:inbox on:YYYY/MM/DD` |
| 2 — Sent | `in:sent on:YYYY/MM/DD` |
| 3 — Drafts | `in:drafts on:YYYY/MM/DD` |

**Pagination is mandatory.** Follow every `nextPageToken` until exhausted. Gmail returns ~20 results/page; a busy day has 2-3 pages of inbox alone.

**Date format fallback:** `on:` is preferred but not always reliable. If ALL 3 passes return 0 results, immediately retry with `after:/before:` range format:
```
"in:inbox after:YYYY/MM/DD before:YYYY/MM/DD"
```
Use day-before for `after:` and day-after for `before:` (both exclusive). Example for Feb 18: `"after:2026/02/17 before:2026/02/19"`. Never accept an all-empty scan without trying the fallback.

For today's date, `on:` is the safer first choice. For past days, `after:/before:` works fine as fallback.

**Timezone:** `internalDate` is Unix milliseconds UTC. Convert: `EET_hour = UTC_hour + 2`. Never trust the Date header (mixes UTC/EET/other offsets).

**Direction:** From `me` / `georgios@baresquare.com` / `agent.ggrigo@gmail.com` → OUT. Everything else → IN.

**External meeting detection:** Gmail is the PRIMARY mechanism for catching Teams/Zoom meetings invisible to Google Calendar. Look for:
- Subjects: "Teams meeting", "Join Microsoft Teams Meeting", "Zoom Meeting"
- Calendar invite emails from external corporate domains (@adidas.com, @sony.com)
- **Placeholder-then-cancel pattern:** GCal placeholder created → external platform invite → placeholder cancelled (cluster of 2-3 emails within minutes)

When a calendar invite email has no matching GCal event, flag it:
```
HH:MM 📧 IN ← Sender ⚡ (Teams/Zoom — no matching Calendar event)
- Platform: Microsoft Teams / Zoom
- Scheduled: HH:MM–HH:MM EET
```

**Filtering:** Skip newsletters, marketing, automated notifications, calendar RSVPs. Keep personal correspondence, thread replies, emails with attachments, booking confirmations.

**Deduplication:** Same message can appear in inbox + sent. Deduplicate by `message_id`.

**Detailed source reference:** `references/source-gmail.md`

### 3. Slack — 2-PASS SCAN

**Tools:** `slack_search_public_and_private`

| Pass | Query |
|------|-------|
| 1 — Sent | `from:<@U037VMNEB> on:YYYY-MM-DD` |
| 2 — Received | `to:<@U037VMNEB> on:YYYY-MM-DD` |

**⚠️ Slack date format:** Use `on:YYYY-MM-DD` for same-day queries, never `after:YYYY-MM-DD` (looks for messages after end-of-day, returns nothing).

**API intermittent failures:** `to:me` and private search sometimes fail with `unexpected_builtin_exception`. Workaround: retry without sort params, try public-only search, or use `to:<@U037VMNEB>` format. If received pass fails entirely, note approximate count inferred from sent messages.

**Timezone:** Slack timestamps are Unix epoch (UTC). Convert to EET.

**Greek messages:** Preserve original text verbatim in the timeline entry.

**Bot filtering:** Skip bot/automated messages unless they contain actionable alerts.

**Detailed source reference:** `references/source-slack.md`

### 4. Fireflies — FULL TRANSCRIPTS

**Tools:** `fireflies_get_transcripts` with `fromDate` / `toDate`

**⚠️ CRITICAL: Always read the full transcript via `fireflies_get_transcript(transcriptId)`. Never trust the summary.** AI-generated summaries miss nuance, misattribute speakers, and omit side conversations.

**What to extract from each transcript:**
- Title, duration, participants
- Key topics discussed (from reading actual sentences)
- Decisions made, action items assigned
- Notable quotes or signals

**Timezone:** `dateString` is UTC. Convert to EET.

**Language detection:** Fireflies sometimes transcribes English meetings in Greek (or vice versa). If the transcript language doesn't match the meeting language, note the discrepancy.

**Correlation:** Every Fireflies transcript should match a Calendar event. If it doesn't, flag it (may be an impromptu call).

**Detailed source reference:** `references/source-fireflies.md`

### 5. Google Drive — FILE ACTIVITY

**Tools:** `google_drive_search`

```
api_query: "modifiedTime > 'YYYY-MM-DDT00:00:00' and modifiedTime < 'YYYY-MM-DDT23:59:59'"
order_by: "modifiedTime desc"
```

**Timezone:** `modifiedTime` is UTC. Convert to EET.

**Filter out:** Folders, files too large to fetch.

**This is the least active intra-day source.** Most days have 0-3 file modifications. It's still worth scanning because a Drive edit during or right after a meeting shows what was being worked on.

**Gmail cross-reference:** "Shared with you" Gmail notifications reveal Drive activity that may not appear in the Drive API search.

**Detailed source reference:** `references/source-drive.md`

---

## Assembly Rules

### Rule 1: Single Chronological Timeline
Merge all 5 sources into one timeline sorted by timestamp. Every entry gets `HH:MM [icon] Source — Title`.

### Rule 2: Cross-Source Correlation
Match related items across sources:
- Calendar event ↔ Fireflies transcript (same time, same attendees)
- Gmail thread ↔ Calendar invite (scheduling emails → meeting)
- Slack discussion ↔ Calendar event (pre/post meeting chatter)
- Drive edit ↔ Calendar event (doc worked on during meeting)
- Gmail invite ↔ NO Calendar event = external platform meeting

### Rule 3: External Meeting Detection
**Not all meetings appear on Google Calendar.** External platforms (Teams, Zoom) send email invites that may never create a GCal event.

**Detection pattern:**
1. Gmail shows calendar invite / meeting link email
2. Calendar has NO matching event at that time
3. → Add to timeline as: `HH:MM 📅⚡ Teams/Zoom — "Meeting Title" (detected via Gmail, not on GCal)`

### Rule 4: Scheduling Conflict Detection
When two events overlap or a proposed meeting conflicts with an existing event, flag it:
```
⚠️ CONFLICT: "Meeting A" (HH:MM) overlaps with "Meeting B" (HH:MM)
```

### Rule 5: Noise Grouping
Automated alerts, bot messages, newsletters — group them into a single entry per source rather than listing individually:
```
## ~06:45 💬 Slack — ETL/Domes bot alerts (automated)
- #channel: ~15 bot messages (06:45–07:07 EET)
- No human messages — automated infrastructure only
```

### Rule 6: Source Counts Table
Always end with the source counts table. Include detail notes:
- Calendar: separate GCal from external (Teams/Zoom)
- Gmail: note 3-pass breakdown (X inbox + Y sent + Z drafts)
- Slack: note 2-pass breakdown (X sent + Y received)
- Fireflies: total minutes recorded

### Rule 7: Multi-Day Arcs
When a topic spans both yesterday and today, note it in cross-references:
```
🔗 Multi-day arc: "Domes sync" (yesterday 13:32) → Angelos Slack (today 09:00) → ad-hoc meeting (today 12:30)
```

---

## Iron-Out Pass (after both days are built)

This is what makes fabric-go different from just running two separate fabric builds. After yesterday and today are both assembled:

### 1. Cross-Day Continuity Check
- Read yesterday's cross-references and today's entries. Are there threads that started yesterday and continued today?
- Flag any "cliff-hangers" from yesterday that have no follow-up today (useful for the expectations skill later)

### 2. Calendar-Gmail Reconciliation
- For every calendar event yesterday+today, check: does a Fireflies transcript exist? If not, was the meeting cancelled, external-platform, or just not recorded?
- For every calendar invite in Gmail, check: is there a matching Calendar event? Flag mismatches.

### 3. Slack Conversation Threading
- If the same Slack conversation appears in both days, note the continuation
- Slack threads that started yesterday with replies today should be linked

### 4. Duplicate/Overlap Detection
- Same email appearing in both days (e.g., sent at 23:50 yesterday, appears in today's inbox)
- Calendar events that span midnight
- Fireflies transcripts that straddle two days

### 5. Gap Detection
- Long gaps (4+ hours) with no activity from any source during work hours (09:00-19:00 EET) → note as potential deep work, external meeting, or offline time
- A Calendar event with no Slack/email activity around it may indicate a focused session

### 6. Yesterday Fabric Polish
If yesterday's fabric was built in a prior session and today's scan reveals new information (e.g., a Fireflies transcript that wasn't processed yet, or a Slack message that was missed), update yesterday's fabric.

---

## Execution Strategy

```
fabric-go
│
├── 1. Check: does yesterday's fabric exist and is it complete?
│   ├── Yes + complete → skip to today
│   ├── Yes + partial → re-scan missing sources
│   └── No → build yesterday (full 5-source scan)
│
├── 2. Build today's fabric (up to current time)
│   └── Full 5-source scan, all rules above
│
├── 3. Iron-out pass
│   ├── Cross-day continuity
│   ├── Calendar-Gmail reconciliation
│   ├── Slack thread linking
│   ├── Duplicate detection
│   └── Gap detection
│
├── 4. Save both fabrics
│   ├── Projects/cowork-fabric/fabrics/fabric-YYYY-MM-DD.md (yesterday)
│   └── Projects/cowork-fabric/fabrics/fabric-YYYY-MM-DD.md (today)
│
└── 5. Summary to user
    ├── Key highlights from both days
    ├── Any conflicts or gaps detected
    └── Cliff-hangers / open threads
```

**Parallelism:** Within each day, pull all 5 sources in parallel. The sources are independent. Calendar + Gmail + Slack + Fireflies + Drive can all run simultaneously.

**For multi-day requests beyond yesterday+today:** Process days in reverse chronological order (most recent first). Each day is independent except for cross-references.

## Date Handling

- **"fabric-go"** → yesterday + today (default)
- **"fabric-go today"** → today only
- **"fabric-go yesterday"** → yesterday only
- **"fabric-go this week"** → Monday through today
- **"fabric-go last week"** → previous Monday through Friday
- **"fabric-go Feb 15"** → specific date
- **"fabric-go last 5 days"** → 5 days back from today

**Timezone:** Everything converts to Europe/Athens (EET, UTC+2). Sources use different formats:
- Calendar: RFC3339 with timezone offset (check it)
- Gmail `internalDate`: Unix milliseconds UTC
- Slack: Unix epoch UTC
- Fireflies `dateString`: UTC
- Drive `modifiedTime`: UTC

## User Context

Check CLAUDE.md at workspace root for:
- **People table** — map emails/Slack IDs to first names
- **Projects table** — add context to meetings
- **Terms glossary** — expand acronyms (LSR, BD, TSF, SOW, BRD, CN, GEO, NRR, EPC)
- **Rules** — especially the Fireflies rule (always read transcript, never trust summary)

## Companion Skills (in this package)

- **`SKILL-expectations.md`** — Extracts commitments and promises from fabric data. Run after fabric-go to build an expectations ledger.

## Deep Source References

For edge cases and detailed per-source guidance, read the reference files:
- `references/source-calendar.md`
- `references/source-gmail.md`
- `references/source-slack.md`
- `references/source-fireflies.md`
- `references/source-drive.md`
- `references/assembly.md`

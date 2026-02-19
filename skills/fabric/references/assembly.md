<!-- Improved during cowork-fabric project, Feb 19 2026 — added external meeting platform
     detection via Gmail cross-reference, multi-day arc tracking, and scheduling conflict
     detection. Triggered by Teams-only Laura call and Henning/Antonis Friday conflict. -->
---
name: fabric-assemble
description: Assemble a daily fabric from normalized source data. Merges all sources (Calendar, Fireflies, Gmail, Drive, Slack) into a single chronological timeline with cross-references. This is the final step after all source skills have pulled their data.
---

# Fabric — Assembly

## Input
Normalized entries from 5 sources, each with:

- EET timestamp (HH:MM)
- Source icon (📅 💬 🎙️ 📧 📁)
- Direction (IN/OUT for messages)
- Content summary

Sources can come from raw data files in `Projects/cowork-fabric/fabrics-raw/` or from live API pulls done in the same session. Either way, all timestamps must be in EET before assembly begins.

## Assembly Rules

### 1. Sort chronologically
All entries in a single timeline, earliest first. Use `~` prefix for approximate times (Drive saves, estimated Slack activity).

### 2. Cross-reference sources
This is one of the most valuable aspects of the fabric. Look for connections between sources and note them explicitly:

- Calendar event at 18:00 + Fireflies transcript at 18:16 → "Meeting started ~16 min late"
- Gmail at 19:20 + Drive modification at 19:18 → "Email drafted in Drive, then sent"
- Slack message "I updated the doc" + Drive edit at same time → note the link
- Booking confirmation email + Calendar event appearing → "Scheduled via Calendly"
- Email cluster (3+ in 30 min) → "Focused email session"
- Evening activity after 19:00 often follows Gym → note the pattern
- **Gmail calendar invite with no matching Google Calendar event → external platform meeting** (Teams, Zoom). See "External Meeting Detection" below.
- **Same time slot on two different events → scheduling conflict**. Flag with ⚡ and note which must move.

Include a dedicated **Cross-References** section at the end of the timeline (before Source Counts) that lists these connections explicitly.

### 3. External Meeting Detection

⚠️ **Not all meetings appear on Google Calendar.** External platforms (Microsoft Teams, Zoom) send email invites that may never create a Google Calendar event. The assembler MUST cross-reference Gmail against Calendar to catch these.

**Detection pattern:**
1. Gmail shows a calendar invite email (Teams meeting link, Zoom link, or ics attachment)
2. No corresponding Google Calendar event exists for that time
3. → This is an external meeting. Add it to the timeline with the ⚡ marker and platform note.

**Common signals in email:**
- Subject contains "Teams meeting", "Join Microsoft Teams Meeting", "Zoom Meeting"
- Email from an external corporate domain (e.g., `@adidas.com`, `@sony.com`)
- Placeholder-then-cancel pattern: a Google Calendar event was created then cancelled, replaced by an external invite

**Fabric format for external meetings:**
```
## HH:MM 📅 Meeting Title ⚡ (Teams — not on Google Calendar)
- HH:MM–HH:MM EET (HH:MM–HH:MM CET)
- Platform: Microsoft Teams / Zoom / etc.
- Source: Email invite from [sender], [date]
- Attendees: name1, name2
```

### 4. Scheduling Conflict Detection

When assembling, watch for events or proposed meetings that overlap in time. Common patterns:
- Two Google Calendar events at the same time
- A proposed meeting time (from email) conflicting with an existing calendar event
- A sent calendar invite conflicting with a received one

Flag conflicts explicitly in both the timeline entry and the Cross-References section. Use ⚡ marker.

### 5. Conversation threading
Group rapid exchanges (within 5 min) between the same people. Don't create separate entries for each message in a volley — summarize the exchange. This applies to email threads, Slack back-and-forth, and rapid-fire calendar changes.

### 6. Depth is constant, detail varies
Every source gets full scan every day. Light days produce short fabrics. Heavy days produce long ones. Never skip a source because the day looks quiet — even "no activity from Gmail" is worth noting in the Source Counts table.

### 7. Multi-day arcs
Some threads span multiple days. When today's activity clearly continues yesterday's thread, note the arc in Cross-References. Examples:
- Domes sync (yesterday) → Angelos Slack follow-up (today) → ad-hoc meeting (today)
- Email sent (yesterday) → reply received (today) → Slack discussion (today)
- adidas budget arc spanning a week of positioning → approval → concerns → resolution

These multi-day arcs are among the most valuable insights from the fabric. They turn isolated events into a narrative.

## Output Format

```markdown
# Fabric of the Day — [Day], [Month] [DD], [YYYY]

All timestamps in Europe/Athens (EET, UTC+2).
Legend: 📅 Calendar · 💬 Slack · 🎙️ Fireflies · 📧 Gmail · 📁 Drive

---

## HH:MM [icon] Entry title
- Detail 1
- Detail 2

## HH:MM [icon] Next entry
- Detail 1

---

## Cross-References
- 📅 + 🎙️: [Calendar event] started ~N min late per transcript
- 📧 + 📁: [Document] edited in Drive at HH:MM, then emailed at HH:MM
- 📧 cluster: N emails sent between HH:MM–HH:MM (focused session)
- 📧 → 📅: [Email invite] has no Calendar match → Teams/Zoom meeting
- 📅 + 📅: [Event A] conflicts with [Event B] at HH:MM — needs resolution

---

## Source Counts
| Source | Items |
|--------|-------|
| 📅 Calendar | N events (N Google Calendar + N external) |
| 🎙️ Fireflies | N transcripts |
| 📧 Gmail | N emails — 3-pass scan w/ pagination |
| 📁 Drive | N modifications |
| 💬 Slack | N messages |
```

## Source Counts — Detail Notes
- **Calendar**: Separate Google Calendar events from external platform meetings in the count. E.g., "4 events (3 Google Calendar + 1 Teams-only)"
- **Gmail**: Always note "3-pass scan w/ pagination" to confirm the full scan was done
- **Slack**: Note if received count is approximate due to API failures
- **Fireflies**: Note processing status (processed, silent/skipped, micro-clip)

## Storage
- **Input (raw data)**: `Projects/cowork-fabric/fabrics-raw/` — read normalized entries from `calendar/`, `gmail/`, `fireflies/`, `drive/`, `slack/` subdirectories
- **Output (assembled fabrics)**: `Projects/cowork-fabric/fabrics/`
- If either directory does not exist at runtime, ask the user where to read from / save to before proceeding. Do not create directories silently.

In practice, raw data may come from bulk files covering multiple days (e.g., `calendar-2026-01-01-to-2026-01-22.json`) rather than per-day files. The assembler should handle both patterns — filter by date when needed.

## File Naming
`Projects/cowork-fabric/fabrics/fabric-YYYY-MM-DD.md`

## Empty Days
If a day has zero activity across all sources (weekend, holiday), still create the file:
```markdown
# Fabric of the Day — Saturday, Jan 3, 2026

No activity recorded across Calendar, Fireflies, Gmail, Drive, or Slack.
```

## Light Days
Some days have activity in only 1-2 sources (e.g., a Saturday with just one email sent). These still get full fabrics — the Source Counts table will show 0 for inactive sources, which is informative on its own.

## Technical Notes
- Node.js is the reliable runtime in this environment for any data processing (python3 has codec issues)
- When writing `.js` files for processing, avoid inline `node -e` — bash mangles operators like `!==`
- For timezone conversions, `new Date(utcMs + 2 * 60 * 60 * 1000)` is the standard EET pattern

## Quality Checks
- Timeline should flow naturally — no time-travel (earlier events after later ones)
- Every Fireflies transcript should have a matching or near-matching calendar event (±15 min)
- **Every Gmail calendar invite should have a matching Calendar event** — if not, it's an external platform meeting
- Emails sent in clusters suggest a work session — note the pattern
- Evening activity (after 19:00) often follows gym (15:00-15:50 or 17:00-17:50, varies by day)
- Cross-References section should have at least one entry on any day with 3+ sources active
- Source Counts should always list all 5 sources, even if count is 0
- **Check for scheduling conflicts** any time a new calendar invite appears in email
- For in-progress days (scanning before day ends), add a note: "Day still in progress — expect updates after [next event]"

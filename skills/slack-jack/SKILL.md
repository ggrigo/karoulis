---
name: slack-jack
description: >
  Generate Slack-Jack scheduling commands for batch meetings. Schedule a batch of 1:N meetings — 
  one organizer meeting individually (or in small groups) with many team members. 
  Use when the user says "schedule meetings with my team", "set up 1-on-1s", 
  "book self-reflection sessions", "schedule reviews with everyone", "generate slack-jack commands",
  "I need to meet with each person on my team", or any scenario where one person needs
  to be in every meeting and other attendees vary per session. Also triggers on
  "batch schedule", "schedule across the team", "find time for N meetings".
  This skill handles identity resolution, calendar availability, batching by shared
  attendees, conflict resolution, and generating Slack-Jack scheduling commands.
compatibility: >
  Requires Google Calendar MCP (list_gcal_events, find_free_time) and Slack MCP
  (slack_search_users, slack_read_user_profile). Primary output is Slack-Jack commands,
  but can adapt to Google Calendar API or manual copy-paste if Slack-Jack unavailable.
---

# Slack-Jack Batch Scheduling

Schedule N meetings where one person (the organizer) appears in every meeting and
attendees vary per session. This is the 1:N scheduling pattern — common for manager
reviews, self-reflections, skip-levels, onboarding rounds, or any situation where
you're meeting with each team member individually or in small chain-of-command groups.

The core challenge is calendar tetris across many people with overlapping constraints.
This skill solves it methodically.

## Overview

The workflow has 5 phases, always in this order:

1. **Define** — Build the meeting list with attendees per session
2. **Resolve** — Map names to emails (calendar IDs) and Slack IDs
3. **Batch & Check** — Group by shared attendees, find free windows
4. **Propose & Verify** — Build a schedule, verify every individual
5. **Output** — Generate scheduling commands in the right format

Each phase produces a concrete artifact. Don't skip phases — the output of each
feeds the next. But do move quickly; junior team members usually have light calendars,
so most slots will work on first pass.

## Phase 1: Define the Meeting List

Start by understanding the structure. Ask for (or extract from conversation):

- **Who is the constant attendee?** (usually the user — they're in every meeting)
- **Who are the variable attendees?** Get the full list of people
- **Is there a management chain?** If meetings include managers in the reporting line
  (e.g., CEO + VP + Director for each team member), capture that hierarchy
- **Meeting duration** — default 25 minutes if not specified
- **Target date range** — which days/weeks to schedule within
- **Any constraints?** Max meetings per day, preferred time blocks, buffer between sessions

Build a table like this:

```
| # | Subject    | Attendees                          |
|---|------------|------------------------------------|
| 1 | Panos      | Georgios, Alex, Panos              |
| 2 | Sofia      | Georgios, Alex, Kassiani, Sofia    |
```

The "Subject" is the person being met with — the variable attendee. The rest
of the attendees are the chain above them.

### Batching Insight

Look at the attendee lists and identify natural batches — groups of meetings that
share the same set of managers. Meetings within a batch can be scheduled back-to-back
on the same day because the shared attendees only need to block one chunk of time.

For example, if meetings 1-3 all include {Georgios, Alex}, those three can go on
the same morning. If meetings 4-6 all include {Georgios, Alex, Kassiani}, those
can fill an afternoon. This dramatically reduces the scheduling search space.

## Phase 2: Resolve Identities

Every attendee needs two things: an **email** (for calendar lookup) and a **Slack user ID**
(for scheduling commands and @mentions).

Use `slack_search_users` to find each person. Most teams follow a naming convention
(e.g., `firstname@company.com`), but always verify — some people use full names
(`firstname.lastname@company.com`) or have nicknames that differ from their Slack profile.

Build a lookup table:

```
| Person    | Email                    | Slack ID    |
|-----------|--------------------------|-------------|
| Alex      | alex@baresquare.com      | U037Z1M2K   |
| Panos T   | panos.tsolakis@...       | U07A9AF963G |
```

If a person isn't found by first name, try their full name or search by last name.
Some people have non-obvious Slack display names.

Store this table — it's referenced in every subsequent phase.

## Phase 3: Batch and Check Availability

### Group into Batches

Sort meetings by their shared attendee set. Each unique set of "always-present"
attendees becomes a batch:

```
Batch A: {Georgios, Alex}           → meetings 1, 2, 3
Batch B: {Georgios, Alex, Kassiani} → meetings 4, 5, 6
Batch C: {Georgios, Panagiotis}     → meetings 7, 8, 9
```

### Find Free Time per Batch Core

For each batch, run `find_free_time` with the shared attendees' email addresses
as calendar IDs across the full target date range. This gives you the windows
where the core group is available.

```
find_free_time(
  calendar_ids: ["georgios@...", "alex@..."],
  time_min: "2026-02-20T08:00:00+02:00",
  time_max: "2026-02-27T18:00:00+02:00",
  time_zone: "Europe/Athens"
)
```

Filter results to working hours and the preferred time blocks. Discard slots
shorter than (meeting_duration + buffer).

### Constraint Budgeting

Before proposing times, count the math:

- **Total meetings**: N
- **Working days in range**: D
- **Max per day**: M (usually 3-4 for mentally intensive sessions)
- **Required**: N ≤ D × M (if not, expand the date range or raise max per day)

Distribute meetings across days. If a batch has 3 meetings and a full morning
is free, put all 3 there. If the batch has 5 meetings, split across 2 days.

The organizer's cognitive load matters — mixing too many different batch contexts
in one day is harder than doing all meetings with the same manager group together.

## Phase 4: Propose and Verify

### Build the Proposed Schedule

Lay out specific times for each meeting. Include buffer time between sessions
(15 minutes recommended — people need a mental break and prep time).

```
Fri Feb 20:
  10:00-10:25  M1: Christos  (Georgios, Nikos, Christos)
  10:40-11:05  M2: George A  (Georgios, Nikos, George A)
  11:20-11:45  M3: Panos     (Georgios, Alex, Panos)
```

### Verify Each Meeting's Full Attendee Set

The batch check only verified the core (shared) attendees. Now verify that
each individual subject person is also free at the proposed time.

For each meeting, run `find_free_time` with ALL attendees in a narrow window
around the proposed time (±30 min). If the proposed time falls within a free
window of sufficient length, it's confirmed.

```
find_free_time(
  calendar_ids: ["georgios@...", "nikos@...", "christos@..."],
  time_min: "2026-02-20T09:30:00+02:00",
  time_max: "2026-02-20T10:45:00+02:00"
)
```

Run these in parallel where possible — individual verifications are independent.

### Handle Conflicts

If a subject person isn't free at the proposed time:

1. **Shift within the same day** — try the next available slot with the batch core
2. **Move to another day** — find alternate batch core availability
3. **Flag for manual resolution** — if no slot works, mark it and tell the user

Track adjustments: note any shifted times and why (e.g., "Sofia busy until 13:10,
shifted from 13:00 to 13:10").

### Present for Approval

Show the user the complete verified schedule as a table:

```
| Day       | Time        | Meeting      | Attendees                    | Status |
|-----------|-------------|-------------|------------------------------|--------|
| Fri 20    | 10:00-10:25 | M1: Christos | Georgios, Nikos, Christos   | ✅     |
| Fri 20    | 10:40-11:05 | M2: George   | Georgios, Nikos, George A   | ✅     |
| Mon 23    | 13:10-13:35 | M4: Sofia    | Georgios, Alex, Kassiani... | ⚠️ +10min |
```

Flag any meetings that couldn't be auto-scheduled. Get user approval before
generating scheduling commands.

## Phase 5: Output Scheduling Commands

### Detect Scheduling Method

Check what's available:

- **Slack-Jack bot**: If the user has Slack-Jack (`/meeting` command), generate
  paste-ready commands. Note: slash commands can only be invoked by the user
  in the Slack client, not via the API. So output these as copy-paste text.
- **Google Calendar API**: If `create_gcal_event` or similar is available, create
  events directly.
- **Manual**: Output a clean summary table the user can work from.

### Slack-Jack Format

```
/meeting <duration>min meeting with <@SLACK_ID1> <@SLACK_ID2> on <Mon DD> at <HH:MM>. Title: "<meeting title>". Description: <description>.
```

### Meeting Invite Template

Each meeting needs:
- **Title**: Consistent naming pattern (e.g., "TSF Self-Reflection — {Name}")
- **Duration**: As specified
- **Attendees**: All people in the chain
- **Description**: Brief context (keep it short — one sentence)

Ask the user for the title pattern and description. Don't assume they want
pre-read materials or prep work unless they say so.

### Save Commands to File

Write all commands to a markdown file the user can reference. Group by day,
include the meeting details above each command for context. Add a "parked"
section for any meetings that were deferred or need decisions.

## Edge Cases

**Person not found in Slack**: Ask the user for the correct name or email.
Don't guess — wrong calendar IDs produce wrong availability data.

**No shared free time in range**: Expand the date range, reduce buffer time,
or suggest the user move/cancel a conflicting event.

**Very large batch (15+ meetings)**: Break into sub-batches by week. Don't try
to schedule everything in 2-3 days — it creates back-to-back marathon days.

**Timezone differences**: Some attendees may be in different timezones. Use
the organizer's timezone as the reference but verify remote attendees'
working hours.

**Recurring pattern**: If the user wants to repeat this monthly/quarterly,
suggest saving the meeting list and attendee table as a reference doc
for next time. The attendee resolution only needs to be done once.

## What This Skill Does NOT Do

- It does not create calendar events directly (unless a calendar creation tool is available)
- It does not send Slack messages on the user's behalf for scheduling (slash commands require user invocation)
- It does not manage recurring meetings — this is for one-time batch scheduling
- It does not handle meeting room booking

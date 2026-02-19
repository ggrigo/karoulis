---
name: fabric-slack
description: Pull and normalize Slack messages for a specific date. Returns EET-normalized message list ready for fabric assembly. Covers both public channels and DMs relevant to the user. Use this whenever pulling Slack data for daily fabrics or when the user asks about their Slack activity.
---

# Fabric — Slack Source

> Improved during cowork-fabric project, Feb 2026 — added API failure handling, parallelism limits, and sort parameter caution from 48-day backfill (Jan 1 – Feb 17).

## Tools
- `slack_search_public_and_private` — primary tool; searches DMs, private channels, and public channels. Georgios's Slack communication is heavily DM-based, so this is the default.
- `slack_search_public` — fallback for public-only searches
- `slack_read_channel` — read recent messages from a specific channel (use for deep-dives)
- `slack_read_thread` — read thread replies (use for deep-dives)

## Query Strategy

Slack search doesn't have a clean "give me all messages on date X" API. Use targeted searches to capture the day's activity.

### Same-day search
```
query: "on:YYYY-MM-DD"
```
**Important**: Use `on:` for same-day queries, NOT `after:`. The `after:` modifier looks for messages after end-of-day, returning nothing for the current day. (Same gotcha as Gmail.)

### Date range search
```
query: "after:YYYY-MM-DD before:YYYY-MM-DD"
```

### Two-pass daily pull (standard approach)
Run exactly 2 searches per day. This catches both sides of every conversation:

1. **Sent messages**: `from:<@U037VMNEB> on:YYYY-MM-DD` — what Georgios said
2. **Received messages**: `to:me on:YYYY-MM-DD` — what was sent to Georgios

Both searches should use:
- `slack_search_public_and_private` (not public-only)
- `response_format: "concise"` — detailed format exceeds token limits (70K-90K chars on busy days)
- `sort: "timestamp"`, `sort_dir: "asc"` — chronological order

The concise format returns enough information to identify conversation partners, topics, and rough volume, but does **not** include precise timestamps or `message_ts` values.

### When to go deeper
If the concise results reveal an important thread or exchange that needs more context, use `slack_read_thread` or `slack_read_channel` with the channel ID and timestamp to get full detail. This is rarely needed for fabric assembly — save it for specific investigations.

## Timestamp Limitation

The concise search format does not expose `message_ts` or precise timestamps. When assembling into fabrics, Slack entries get **approximate times** (`~HH:MM`) inferred from:
- Position relative to calendar events (Slack gaps during meetings are expected)
- Logical sequence of conversations (e.g., pre-meeting prep goes before the meeting)
- Time-of-day patterns (morning prep, post-meeting follow-ups, evening wrap-up)

This is a known limitation. The `~` prefix in fabric entries signals approximate timing.

## Raw Data Storage Format

Save each day's data as structured JSON in `Projects/cowork-fabric/fabrics-raw/slack/`. The format that works well for fabric assembly:

```json
{
  "date": "2026-01-15",
  "day": "Wednesday",
  "sent_count": "20+",
  "received_count": "20+",
  "key_conversations": [
    {
      "with": "Person Name",
      "topics": ["topic 1", "topic 2"],
      "channel": "DM or channel name (optional)",
      "note": "1-2 sentence summary of the exchange"
    }
  ]
}
```

Group messages by conversation partner (not by individual message). This is the natural unit for fabric assembly — each `key_conversations` entry typically becomes one fabric entry.

### File naming
- Daily: `slack-YYYY-MM-DD.json`
- Bulk: `slack-YYYY-MM-DD-to-YYYY-MM-DD.json`

## Direction Detection
- From `U037VMNEB` (Georgios) → **OUT**
- Everything else → **IN**
- For fabric assembly, direction is less important than the conversation summary — most entries are bidirectional exchanges

## Filtering Rules
- Skip bot messages unless actionable (deployment notifications, alert bots)
- Group all messages between same people on a day into one conversation entry
- Skip emoji-only reactions and short acknowledgments ("ok", "thanks", thumbs-up)
- Keep: substantive messages, decisions, questions asked/answered, file shares, links shared
- "Cool cool", "👍", running-late messages are marginal — include only if they add context

## Fabric Assembly Notes

When integrating Slack into daily fabric files:

### Entry format
```
## ~HH:MM 💬 Slack — Person: topic summary
- Detail 1
- Detail 2
```

### Placement heuristics
- Pre-meeting Slack about a topic → place before the meeting
- Post-meeting follow-ups → place after the meeting
- Administrative/scheduling messages → place in morning block (~09:00-10:00)
- Multi-topic exchanges → place at the most contextually relevant time
- Evening exchanges (after gym) → place in ~18:00-19:00 block

### Volume patterns
- Weekdays typically show 20+ sent and 20+ received messages
- Weekends are light (0-5 messages) but can be strategically significant
- Heavy Slack days often coincide with heavy meeting days (parallel coordination)

### Multi-day arcs
Slack reveals narrative threads that span multiple days. Note these in cross-references:
- Example: adidas budget arc (Jan 13 positioning → Jan 14 Henning success → Jan 15 Carina concern → Jan 16 service license → Jan 17 doc approved)
- These arcs are one of the most valuable insights from Slack data

## API Failure Handling

The Slack search API can fail intermittently or persistently for specific queries. Lessons from the 48-day backfill:

- **Intermittent failures**: `to:me on:YYYY-MM-DD` calls occasionally fail with `unexpected_builtin_exception`. Retry 2-3 times. If it still fails, try removing `sort` and `sort_dir` parameters — this sometimes resolves it.
- **Persistent failures**: Some date+direction combinations fail permanently (e.g., Jan 7 received never succeeded across 8+ attempts). When this happens, note the data gap in the raw JSON and fabric file rather than silently omitting it.
- **Parallelism limits**: Do not batch more than 2-3 Slack search calls in parallel. Larger batches cause cascading `Sibling tool call errored` failures. Run sent+received for one day in parallel, then move to the next day.
- **`sort` parameter caution**: If a search fails, retry without `sort: "timestamp"` and `sort_dir: "asc"`. These parameters occasionally trigger backend errors that don't occur without them.

## Quality Checks
- Cross-reference Slack messages about meetings with Calendar and Fireflies entries
- Messages like "I updated the doc" or "just sent the email" should link to Drive/Gmail entries
- Slack activity gaps during calendar meetings are normal (person was in a call)
- Check for file shares in Slack that might correspond to Drive activity
- Late-night Slack messages (after 22:00 EET) are unusual and worth noting
- Weekend Slack activity is rare but meaningful — always capture it

<!-- Improved during cowork-fabric project, Feb 2026 — added Language Detection and Validation section
     from discovering LSR meeting (English) was transcribed in Greek by Fireflies default settings.
     Also added API Failure Handling in prior update. -->
---
name: fabric-fireflies
description: Pull and normalize Fireflies transcripts for a specific date range. Has two modes — listing (for fabric assembly) and deep-reading (for meeting analysis). Always reads full transcripts when analyzing meetings; never trusts AI summaries. Includes language validation to detect garbled transcripts.
---

# Fabric — Fireflies Source

## ABSOLUTE RULE
**Never use `fireflies_get_summary` or rely on AI-generated action items from search results.** When you need to understand what happened in a meeting, use `fireflies_get_transcript` to read the full conversation. Form your own analysis from the primary source — the actual words spoken.

This rule exists because Fireflies' AI summaries frequently miss nuance, misattribute action items, and omit tensions or open questions that are obvious in the transcript.

## Two Modes of Operation

### Mode 1: Listing (for fabric assembly)
When building a fabric, you need metadata — which meetings happened, when, how long. You do NOT need to read every transcript word-for-word.

**Tool**: `fireflies_get_transcripts` with date filters
**Output**: A list of meetings with title, start time (EET), duration, and attendees

Use this mode when the assembler just needs to place meetings on the timeline. The transcript content can be summarized briefly from the title and attendees alone.

### Mode 2: Deep-reading (for meeting analysis)
When the user asks "what happened in the LSR meeting?" or when you need to extract action items, decisions, or tensions from a specific meeting.

**Tool**: `fireflies_get_transcript(transcriptId: "ID")`
**Output**: Full analysis with key points, action items, decisions, open questions

Use this mode when you need to understand the content of a meeting, not just that it happened.

## Query Pattern

### Listing transcripts
```
fireflies_get_transcripts(
  fromDate: "YYYY-MM-DD",
  toDate: "YYYY-MM-DD",
  limit: 50
)
```
For multi-day pulls, widen the date range. The API returns transcripts whose `dateString` falls within the range (UTC-based, so some timezone edge cases exist — see below).

### Reading a specific transcript
```
fireflies_get_transcript(transcriptId: "abc123")
```

## Timezone Rule — CRITICAL
`dateString` is **UTC ISO 8601**. Always convert to EET:

```
EET_hour = UTC_hour + 2
```

**Example**: `dateString: "2026-02-17T16:16:00Z"` → 16:16 UTC → **18:16 EET**

This was the #1 source of errors in early fabric builds. A meeting at 16:16 UTC was placed at 16:17 EET instead of 18:16 EET. The +2 hour offset is non-negotiable.

### Date boundary edge case
A meeting at 23:30 UTC on Jan 15 is actually 01:30 EET on Jan 16. When filtering by EET date, be aware that the API's date filter uses UTC. For a complete EET day, you may need to check one day later in the API results.

## Transcript List Files
For cross-session reuse, save bulk transcript listings as JSON lookup files:
```
Projects/cowork-fabric/fabrics-raw/fireflies/transcript-list-jan1-22.json
```
These files contain `{id, title, dateString, duration, organizer_email}` for each transcript. Future sessions can read these instead of re-querying the API.

## Output Format

### For fabric assembly (Mode 1 — listing)
```
HH:MM 🎙️ Meeting Title — transcript starts
- Duration: Xmin
- Attendees: name1, name2
```

### For meeting analysis (Mode 2 — deep-reading)
```
HH:MM 🎙️ Meeting Title — transcript starts
- Attendees: name1, name2
- Key point 1 (from actual transcript, not summary)
- Key point 2
- Decision: what was decided
- Action: who → what → when
- Open question: unresolved issue
```

## Transcript Reading Protocol (Mode 2 only)
1. Read the full transcript
2. Identify key topics discussed (your own analysis)
3. Extract action items (who committed to what)
4. Note any decisions made
5. Note any tensions or open questions
6. Keep bullet points to 4-6 max per meeting

## Storage
- **Raw data**: `Projects/cowork-fabric/fabrics-raw/fireflies/`
- Transcript list files: `transcript-list-DESCRIPTION.json` (e.g., `transcript-list-jan1-22.json`)
- Individual transcripts (when deep-read): `transcript-TRANSCRIPT_ID.json`
- If the directory does not exist at runtime, ask the user where to save before proceeding. Do not create directories silently.

## Technical Note
When parsing Fireflies API responses in bulk, Node.js is the reliable runtime in this environment (python3 has codec issues). Use `.js` files rather than inline `node -e` commands — bash mangles `!==` and other JS operators even inside single quotes.

## Language Detection and Validation — CRITICAL

Georgios's Fireflies account defaults to **Greek transcription**. When a meeting is conducted in English but transcribed in Greek, the result is **garbled phonetic Greek** — Greek characters spelling out English words (e.g., "Σπίκς" for "speaks", "Μπάσταρντ" for an English word). These transcripts are completely useless.

### Step 1: Predict Meeting Language from Participants

Before reading any transcript, check the attendee list:

| Pattern | Expected Language |
|---------|-------------------|
| **Any non-Greek external participant** (adidas, Condé Nast, Sony, etc.) | **English** |
| **All Baresquare-only** (baresquare.com emails only) | **Likely Greek** (but could be English) |
| **Mixed: Baresquare + Greek external** (e.g., CILS, Domes with Greek contacts) | **Likely Greek** |

Known English-only meetings: adidas (LSR, any meeting with Henning/Divya/Laura/Venkat), Condé Nast (Khushi, Ross), Sony (Christos SEO with external teams), investor calls (Florian).

### Step 2: Validate Transcript Language After Reading

When you read a transcript (Mode 2), check the first 10-15 sentences for language mismatch:

**Garbled Greek indicators** (English meeting transcribed in Greek):
- Greek characters spelling English words phonetically: Σπίκς, Μπάσταρντ, Νίκος→Νικόλαος when speaker said "Nicholas"
- Mix of actual Greek greetings (Καλημέρα, Γειά σας) followed by Greek-transliterated English
- Sentences that are syntactically Greek but semantically nonsensical
- Technical English terms rendered in Greek characters

**Healthy English transcript indicators**:
- English sentences with proper grammar
- Technical terms in English
- May contain isolated Greek words (greetings, names) — this is normal

**Healthy Greek transcript indicators**:
- Coherent Greek sentences with proper syntax
- Greek technical vocabulary or natural Greek phrasing
- Fireflies Greek quality is generally poor (Latinized gibberish), but distinguishable from garbled-English-in-Greek

### Step 3: Flag and Remediate

If a language mismatch is detected:

1. **Stop immediately** — do NOT deep-read the full transcript. Reading 200 lines of garbled Greek wastes context window and API calls for zero value. The detection must happen in the first 10-15 sentences, before committing to a full read.
2. **Flag the transcript** — in the fabric entry, note: `⚠️ Language mismatch: English meeting transcribed in Greek. Transcript unusable until retranscribed.`
3. **Tell the user** — "This meeting was likely in English but Fireflies transcribed it in Greek. You'll need to retranscribe it in the Fireflies UI (Settings → Transcription Language → English, then reprocess)."
4. **Use `fireflies_fetch` as a backup check** — sometimes it returns a different (retranscribed) version. If `fireflies_fetch` returns English while `fireflies_get_transcript` returned Greek, the meeting has been retranscribed and the `fetch` version is usable.
5. **For fabric assembly (Mode 1)**: still list the meeting with metadata, but add the language warning. Don't attempt content analysis.

### Step 4: Prevent (User Action)

The root cause is Fireflies' account-level language setting. Ideal workflow:
- Before English meetings: set Fireflies language to English (or "Auto-detect" if available)
- After meeting: verify transcription language
- If wrong: retranscribe from Fireflies UI before processing

This cannot be done via API — it requires manual action in the Fireflies web interface.

## Quality Checks
- **Language validation**: always run Step 1-2 above before deep-reading a transcript
- Cross-reference transcript start time with calendar event time (±15 min normal for late starts)
- If a meeting has multiple recordings (like LSR had 2 segments), note both with context
- Duration: prefer calculating from transcript timestamps over metadata
- Busy days can have 5-7 transcripts; quiet days may have none — both are normal

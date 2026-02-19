<!-- Created Feb 18 2026 — initial design from cowork session on reverse-chronological fabric processing -->
---
name: fabric-back
description: Extract structured knowledge from daily fabric timelines into persistent memory. Processes fabrics in reverse chronological order (most recent first, working backwards) so that current state is captured first and older days add provenance, history, and origin stories to already-known facts. Use this whenever the user asks to "process fabrics", "build memory from fabrics", "fabric-back", "backfill memory", or wants to turn daily timelines into searchable knowledge.
---

# Fabric-Back — Reverse-Chronological Knowledge Extraction

## What This Is

The forward fabric pipeline weaves 5 raw sources into a daily chronological timeline. This skill runs the **inverse**: it reads completed fabric files and extracts durable knowledge — facts about people, projects, decisions, commitments, and relationships — into smart-memory, where they compound over time.

The key insight: **process most-recent first.** Yesterday's fabric gives you current state with high confidence. Older fabrics add depth — when something started, who initiated it, how scope evolved — without the risk of writing stale facts as current truth.

## When to Use

- User says "process fabrics" or "fabric-back" or "backfill memory"
- User asks to extract knowledge from a date range of fabrics
- User wants to build/rebuild the smart-memory knowledge base from fabric files
- After a batch of new fabric files have been assembled and need processing

## Input

Assembled fabric files from `Projects/cowork-fabric/fabrics/fabric-YYYY-MM-DD.md`. These are the output of the forward fabric pipeline (fabric-assemble skill). Each file is a complete chronological timeline of one day, already normalized to EET.

## Processing Order — CRITICAL

**Always process from most recent date to earliest.** This is not a preference — it's the core architectural decision.

### Why Reverse Order Works

When you process Feb 17 first:
- "LSRT production-ready by March 27" gets written as a current hard deadline — high confidence, high relevance
- "Nikos to provide cost update by end of week" gets written as an open action item

When you then process Feb 10:
- If Feb 10 mentions an earlier LSRT deadline that's been superseded, you don't create a conflicting fact — you already know the current deadline from Feb 17. Instead, you link the old deadline as provenance using an `updates` relationship
- If Feb 10 shows a task that was completed by Feb 17, you check memory first, find it resolved, and skip it or mark it as the origin

When you get to Jan 15:
- Most action items from Jan 15 are already in memory as resolved/superseded. The Jan 15 extraction adds `derives` relationships showing where things originated
- Stale tasks don't pollute memory because you check before writing

### The Check-Before-Write Rule

Before writing any fact, **always recall from smart-memory first** to see if a more recent version already exists. This is the mechanism that prevents staleness:

```
1. Extract candidate fact from fabric: "Lars to share FPA video walkthrough with Antonis"
2. Recall: query smart-memory for "Lars FPA video walkthrough"
3. If a newer fact exists (from a more recent fabric already processed):
   → Link the older fact as origin (derives/extends) — or skip entirely if it adds no value
4. If no newer fact exists:
   → This might be genuinely open. Write it, but tag it with the fabric date for age-awareness
```

## Extraction Categories

Read `references/extraction-categories.md` for the full taxonomy. The categories below are the primary extraction targets, mapped to existing smart-memory collections:

| Category | Collection | What to Extract |
|----------|------------|-----------------|
| **Action items** | `projects` | Who committed to what, by when. Include the source context. |
| **Decisions** | `projects` | What was decided, by whom, with what reasoning. |
| **Project state changes** | `projects` | Scope changes, deadline changes, milestone completions. |
| **People interactions** | `people` | Who talked to whom about what. Relationship signals. |
| **Financial data** | `financials` | Numbers: budgets, forecasts, petty cash, pricing. |
| **Deal/pipeline movement** | `deals` | New opportunities, stage changes, proposal activity. |
| **Commitments to externals** | `contracts` | Promises to clients, deliverable deadlines. |
| **Communication patterns** | `communications` | Recurring threads, escalation patterns, who initiates. |
| **Meeting insights** | `meetings` | Key outcomes from meetings (from fabric summaries, not raw transcripts). |

### What NOT to Extract

- Newsletter/marketing emails (already filtered by fabric, but double-check)
- Bot notifications unless they contain actionable information
- Emoji-only reactions, acknowledgments ("ok", "thanks")
- Calendar events with no substantive content (gym, lunch)
- Noise entries that the fabric itself flagged as "not substantive"

## Fact Quality Standards

Every fact written to smart-memory must be:

1. **Atomic** — one discrete piece of knowledge per fact. "Lars approved Florian messaging and shared FPA forecast" is two facts, not one.
2. **Dated** — include the fabric date in tags so age is always visible. Tag format: `fab:2026-02-17`.
3. **Attributed** — who said/did/decided this. A fact without attribution is gossip.
4. **Contextual** — enough surrounding detail that the fact is useful without re-reading the fabric. "Nikos to provide cost update" is useless without "for adidas LSR, by end of week (Feb 21), discussed in LSR Product Roadmap meeting."
5. **Tagged for retrieval** — use consistent tags: project names (`adidas-lsr`, `conde-nast`), person names (`lars`, `henning`), category (`action-item`, `decision`, `financial`, `deadline`).

### Fact Template

```json
{
  "text": "Henning confirmed LSRT must be production-ready by March 27, 2026. Train-the-Trainer in first/second week of April, markets rollout in May.",
  "tags": ["adidas-lsr", "henning", "deadline", "fab:2026-02-17"],
  "confidence": 0.95
}
```

Confidence scoring:
- **0.9–1.0**: Directly stated in fabric (email quote, explicit Slack message, meeting decision)
- **0.7–0.89**: Inferred from context (cross-referenced across sources, implied but not explicit)
- **0.5–0.69**: Tentative (single mention, no corroboration, ambiguous phrasing)
- **Below 0.5**: Don't write it. If it's that uncertain, it's not worth polluting memory.

## Processing a Single Day — Step by Step

### Phase 1: Read and Parse

1. Read the fabric file for the target date
2. Identify all entries that contain extractable knowledge (skip noise, skip pure calendar blocks with no content)
3. Group related entries — a Slack conversation + the meeting it references + the follow-up email are one knowledge cluster

### Phase 2: Check Against Existing Memory

For each knowledge cluster:

1. **Recall** from smart-memory using the key entities (project name, person name, topic)
2. **Compare** the candidate facts against what's already stored
3. **Classify** each candidate:
   - **New**: Nothing in memory about this. Write it.
   - **Confirms**: Memory already has this from a more recent fabric. Skip or add a `derives` link if the older fabric adds origin context.
   - **Contradicts**: Memory has a different version from a more recent fabric. The newer version wins. Link this as a superseded predecessor using `updates` relationship. Do NOT overwrite the newer fact.
   - **Extends**: Memory has a related fact but this adds detail. Write it with an `extends` relationship.

### Phase 3: Write to Memory

For each fact that passes the check:

1. **Remember the source** — call `remember()` with the relevant fabric excerpt as `source_text`, collection mapped per the table above, `source_type: "document"`.
2. **Extract facts** — call `remember_facts()` with the atomic facts as JSON, linked to the source chunk.
3. **Link relationships** — if this fact updates, extends, or derives from an existing fact, call `update_memory()` to create the relationship.
4. **Expire superseded facts** — if this fact (from an older day) is already superseded by a newer fact in memory, expire the older one via the relationship mechanism (type `updates` auto-expires the target).

### Phase 4: Track Progress

After processing each day, log progress:
- Which date was processed
- How many facts were extracted, skipped, linked
- Any conflicts or ambiguities flagged for human review

Save progress to `Projects/cowork-fabric/fabric-back-log.json`:

```json
{
  "last_processed": "2026-02-17",
  "direction": "backward",
  "days_processed": ["2026-02-17", "2026-02-16", "2026-02-15"],
  "days_remaining": ["2026-02-14", "..."],
  "stats": {
    "2026-02-17": {"facts_written": 23, "facts_skipped": 4, "facts_linked": 7, "conflicts": 0}
  }
}
```

## Batch Processing

When processing a range of dates (e.g., "process all 48 fabrics"):

1. **List all fabric files**, sort by date descending
2. **Check the progress log** — if partially done, resume from where you left off
3. **Process one day at a time**, most recent first
4. **Pause after every 5 days** and report progress to the user. Batch processing is long-running — keep the user informed.
5. **If a conflict is ambiguous**, flag it in the log and keep going. Don't block the batch on a single unclear fact.

### Rate Limiting Awareness

Smart-memory calls are API-backed. When processing a batch:
- Don't fire 50 `recall()` calls in parallel — process facts sequentially within a day
- Batch `remember_facts()` calls: group facts from the same knowledge cluster into one call
- Aim for ~20-30 smart-memory operations per fabric day (remember + remember_facts + recall + update_memory)

## Handling Edge Cases

### Empty/Light Days
Weekends and holidays may have zero extractable facts. Log the day as processed with `facts_written: 0`. The fact that nothing happened is itself informative (no progress on any project that day).

### Multi-Day Arcs
The fabric's cross-references section often highlights multi-day arcs (e.g., "adidas budget: Jan 13 positioning → Jan 14 success → Jan 15 concern → Jan 16 license → Jan 17 approved"). When you encounter these:
- Each day's individual facts get written normally
- After processing all days in the arc, create `extends` relationships linking the sequence
- The most recent fact in the arc should be the "live" one; earlier facts are provenance

### Meetings with Deep Transcripts
The fabric contains meeting summaries (Mode 1 from fabric-fireflies). These are sufficient for fact extraction. Do NOT re-read the full Fireflies transcript during fabric-back processing — that's a separate operation. Extract what the fabric gives you.

### Greek Content
Many Slack messages in the fabric are in Greek (transliterated or native). Extract the meaning, write facts in English. The fabric usually provides enough context to understand the content. If a Greek message is ambiguous, note the ambiguity and tag with `needs-review`.

### Conflicting Facts Within Same Day
Sometimes a morning Slack message says one thing and an afternoon meeting changes it. The later-in-day version wins. The fabric's chronological order makes this straightforward — process entries top to bottom within a day.

## Memory Hygiene

### Collection Discipline
Use existing smart-memory collections. Don't create new collections without good reason. The current schema:
- `projects` — project-scoped facts (actions, decisions, deadlines, scope)
- `people` — person-scoped facts (interactions, relationships, roles)
- `deals` — pipeline and opportunity facts
- `financials` — money, budgets, forecasts
- `meetings` — meeting outcomes
- `contracts` — client commitments, SoW details
- `communications` — recurring patterns, escalations
- `companies` — company-level facts (client, partner, vendor)

### Tag Consistency
Always include these tags where applicable:
- `fab:YYYY-MM-DD` — source fabric date (always)
- Project identifier: `adidas-lsr`, `conde-nast`, `sony-global`, `glenigan`, etc.
- Person name (lowercase): `lars`, `henning`, `nikos`, `alex-m`, etc.
- Fact type: `action-item`, `decision`, `deadline`, `financial`, `milestone`, `relationship`

### Deduplication
Before writing, recall with the core entity + topic. Smart-memory's semantic search will surface near-duplicates. If a fact is essentially the same as an existing one (same meaning, same entities, same timeframe), skip it. Don't create 5 variations of "LSRT due March 27" from 5 different fabrics.

## Storage

- **Input**: `Projects/cowork-fabric/fabrics/fabric-YYYY-MM-DD.md`
- **Progress log**: `Projects/cowork-fabric/fabric-back-log.json`
- **Output**: smart-memory (via API calls — no local storage for facts)

## Quality Checks

After processing a batch:
- Recall a few known facts (e.g., "adidas LSR deadline") and verify they're correct and properly linked
- Check that no stale action items are marked as current
- Verify fact counts are reasonable (a busy weekday should produce 15-30 facts; a quiet Sunday, 0-5)
- Spot-check `derives`/`extends` chains to make sure provenance links are coherent

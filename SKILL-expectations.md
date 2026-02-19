---
name: karoulis-expectations
description: |
  Extract commitments and expectations from daily activity sources, build a ledger of promises made and received, and generate an urgency-weighted prioritization.

  Use this skill whenever the user asks about: what did I promise, what's overdue, commitment tracker, expectations ledger, what's open, prioritize my week, what should I focus on, accountability check, who owes me what, open loops.
---

# Karoulis Expectations — Commitment Ledger & Prioritization

## Purpose

Reconstruct a complete expectations ledger by extracting commitments from all connected sources. Categorize into four quadrants: what I promised, what I delivered, what others promised me, what others delivered. Then generate an urgency-weighted prioritization to answer: "What should I focus on right now?"

## Source Strategy

### Primary: Fireflies Action Items (richest signal)
Pull all Fireflies transcripts for the target period using `fireflies_get_transcripts` with `fromDate` and `toDate`. Extract the `action_items` field from each transcript's `summary`. These are pre-structured commitments with assignee, action, and timestamp.

**Categorize each action item:**
- If assignee is **Georgios Grigoriadis** → Section 1: "What I Promised"
- If assignee is **anyone else** → Section 2: "What Others Promised"
- Cross-reference with previous ledger to detect → Section 3: "Delivered" (if an item from a prior period now appears resolved or was explicitly mentioned as done)

### Secondary: Gmail Sent Items
Search `from:me after:YYYY/MM/DD before:YYYY/MM/DD` to find:
- Explicit promises in emails ("I will", "θα στείλω", "I'll send", "θα κάνω")
- Deadlines mentioned
- Requests made to others (these become "What Others Promised")
- Drafts that exist but were never sent (potential broken promises)

### Tertiary: Slack Messages
Search `from:<@U037VMNEB>` with date filters to find:
- Quick commitments in DMs and group chats
- Follow-up requests
- Acknowledgments of tasks assigned

### Contextual: Google Calendar
Use meeting titles and descriptions to correlate which meeting generated which commitments. Meeting descriptions often contain pre-set agendas that imply expected deliverables.

### Contextual: Google Drive
Look for documents modified in the period that map to known commitments (e.g., a BRD being edited = progress on that commitment).

## Extraction Rules

### Commitment Detection Patterns (English)
- "I will...", "I'll...", "Let me...", "I can do that by..."
- "We need to...", "Action item:...", "Next steps:..."
- "By [date]", "before [day]", "end of [period]"
- "Please send...", "Can you...", "Could you..."

### Commitment Detection Patterns (Greek)
- "Θα...", "Να...", "Μέχρι...", "Εντός..."
- "Πρέπει να...", "Θα στείλω...", "Θα κάνω..."

### Deadline Extraction
- Explicit: "by Friday", "μέχρι Δευτέρα", "end of week", "next meeting"
- Implicit: recurring meeting cadence implies next touchpoint
- Absent: mark as "TBD" but flag as open loop

## Output Format

```markdown
# Expectations Ledger — Week of [Date Range]

## 1. What I Promised (Georgios → Others)
### HIGH URGENCY (this week / overdue)
| # | Promise | To Whom | When Made | Deadline | Status |

### MEDIUM URGENCY (next 2 weeks)
| # | Promise | To Whom | When Made | Deadline | Status |

### LONGER TERM (end of month+)
| # | Promise | To Whom | When Made | Deadline | Status |

## 2. What Others Promised Me / The Team
### THIS WEEK
| # | Promise | From Whom | When Made | Deadline | Status |

### NEXT WEEK
| # | Promise | From Whom | When Made | Deadline | Status |

### END OF MONTH / MARCH
| # | Promise | From Whom | When Made | Deadline | Status |

## 3. Delivered / Completed
| # | What | By Whom | When |

## 4. Urgency-Weighted Prioritization
🔴 CRITICAL — Act Today
🟠 HIGH — Act This Week
🟡 IMPORTANT — Act Next Week
🔵 STRATEGIC — Track Monthly

## 5. Open Loops — Nobody Owns These Yet
| # | Issue | Surfaced | Context |
```

## Status Icons
- ⚠️ DUE TODAY / OVERDUE
- 🔴 OPEN (no visible progress)
- 🟡 IN DISCUSSION / PARTIALLY DONE
- 🔵 IN PROGRESS (evidence of work)
- 🟢 COMPLETE / DATE SET
- ❓ UNKNOWN (need to check)
- ⚠️ TRACKING (waiting on someone else)

## Prioritization Algorithm

Priority Score = (1 / Days_Until_Deadline) × Impact_Weight × Dependency_Factor

**Impact Weights:**
- Client-facing / revenue: 3x
- Team blocker (someone is waiting on this): 2x
- Internal / operational: 1x
- Personal / nice-to-have: 0.5x

**Dependency Factor:**
- Blocks 3+ other items: 3x
- Blocks 1-2 items: 2x
- Standalone: 1x

## Cross-Session Tracking

When running this skill on consecutive days:
1. Load the previous ledger from `expectations-ledger-YYYY-MM-DD.md`
2. For items that appeared in the previous ledger:
   - If now mentioned as done → move to "Delivered"
   - If deadline passed and not done → escalate priority
   - If no new signal → keep status, note "no update"
3. New commitments from today's sources → add to appropriate section
4. Remove items explicitly marked as cancelled or deprioritized

## People Resolution

Use the CLAUDE.md people table to resolve names:
- Fireflies attendee emails → first names
- "Antonis" = Antonis Hadjiyannakis (finance/ops)
- "Lars" = Lars Boeddener (adidas, transitioning out)
- "Nikos" = Nikos Vogiatzis
- etc.

## Edge Cases

- **Greek text:** Preserve original Greek for Slack messages and action items. Add English context if the commitment is ambiguous.
- **Recurring commitments:** Track the next instance only. E.g., "weekly meetings" → track next week's meeting.
- **Cancelled meetings:** Check if commitments from cancelled meetings still stand or were voided.
- **Draft emails:** A draft reply (Gmail DRAFT label) counts as "intention but not delivered."

## File Naming

Save as: `expectations-ledger-YYYY-MM-DD.md` in the workspace folder.

## Stage 2: Verification Pass (CRITICAL)

**Why this exists:** Stage 1 extracts commitments from meeting transcripts. But between the meeting and now, people act — they send emails, post Slack messages, share Drive docs. Without verification, the ledger shows stale statuses and false alarms.

### Verification Rules

For EVERY item in the ledger with status ⚠️ TRACKING, 🔴 OPEN, or ❓ CHECK:

1. **Identify the expected delivery channel.** Where would this person realistically deliver?
   - Alex → Gmail (sends docs), Slack DM, Drive (shares SoWs)
   - Antonis → Slack DM (Greek), Gmail (formal docs)
   - Lars → Slack DM, Calendar (meeting invites)
   - Thomas → Slack DM
   - Nikos → Slack channels, Drive (technical docs)
   - External (adidas, Sony) → Gmail
   - Default: check Slack DM first, then Gmail, then Drive

2. **Run targeted verification queries:**
   - **Slack:** `from:<@{user_id}> {keywords} after:{commitment_date}` in their DM channel
   - **Gmail:** `from:{email} {keywords} after:{commitment_date}`
   - **Drive:** `name contains '{keyword}' or fullText contains '{keyword}'` filtered by `modifiedTime > '{commitment_date}'`

3. **Update status based on findings:**
   - Found delivery → move to "Delivered" with link/timestamp
   - Found partial progress → update to 🔵 IN PROGRESS with evidence
   - Found acknowledgment but no delivery → keep ⚠️ TRACKING, add context
   - No signal found → keep current status, note "verified: no update as of {date}"

4. **For items I promised (Section 1):**
   - Check Gmail Sent for "did I actually send this?"
   - Check Slack for "did I post/share this?"
   - Check Drive for "did I create/edit the doc?"
   - Check Calendar for "did I schedule the meeting?"

### Verification Batching

To avoid rate limits and maximize parallelism:
- Group items by person → one Slack search per person covers multiple items
- Group Gmail searches by sender email
- Group Drive searches by keyword cluster
- Run Slack, Gmail, and Drive queries in parallel

### Verification Output

Add a `verified` field to each item:
```
| # | Promise | ... | Status | Verified |
| P5 | Board resolution reply | ... | 🟢 DELIVERED | ✅ Slack Feb 17 09:00 — sent approval + Antonis confirmed |
| E1 | Sony ISOW | ... | 🟡 PARTIAL | ⚡ Gmail Feb 17 13:50 — Christos portion sent, full ISOW pending |
```

### False Positive Detection

Watch for these verification traps:
- **Name collision:** "Sony SoW" exists in multiple versions — check dates carefully
- **Forwarded ≠ delivered:** Someone forwarding a request is not the same as delivering it
- **Draft ≠ sent:** Gmail drafts are not deliveries
- **Discussed ≠ committed:** Meeting discussion ≠ action item with a deadline
- **Partial ≠ complete:** Budget breakdown for one workstream ≠ full ISOW

## Execution Strategy

1. Pull Fireflies transcripts (parallel with other sources)
2. Pull Gmail sent items
3. Pull Slack messages from Georgios
4. Read previous ledger if exists (for delta tracking)
5. Extract all commitments
6. Categorize into 4 quadrants
7. **NEW — Stage 2: Verification Pass**
   a. Group items by person and expected delivery channel
   b. Run parallel verification queries (Slack + Gmail + Drive)
   c. Update statuses based on evidence found
   d. Flag false positives and partial deliveries
8. Score and rank by urgency (using VERIFIED statuses)
9. Identify open loops
10. Generate markdown with verification evidence
11. Save to workspace

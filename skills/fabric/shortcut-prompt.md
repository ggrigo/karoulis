# fabric-go — Shortcut Configuration

**Command:** `fabric-go`
**Plugin:** karoulis
**Schedule:** On demand. Can also be scheduled daily at 20:00 Europe/Athens (cron: `0 18 * * *` UTC).

---

## Prompt (copy this into the shortcut)

Read the fabric skill at `Projects/my-skills/karoulis/SKILL.md` and execute **fabric-go** (yesterday + today):

1. **Yesterday**: Check if `Projects/cowork-fabric/fabrics/fabric-{yesterday}.md` exists and is complete (has Cross-References section and Source Counts table with all 5 sources). If missing or partial, build it with a full 5-source scan.

2. **Today (up to now)**: Build `Projects/cowork-fabric/fabrics/fabric-{today}.md` with a full 5-source scan covering all activity up to the current time.

3. **Iron-out pass**: Cross-reference both days:
   - Calendar-Gmail reconciliation (detect Teams/Zoom meetings invisible to GCal)
   - Multi-day thread linking (Slack conversations, email threads spanning both days)
   - Gap detection (4+ hour work-hours gaps with no activity)
   - Conflict detection (overlapping events)
   - Update yesterday's fabric if today's scan reveals missed data

4. **Save** both fabrics to `Projects/cowork-fabric/fabrics/`.

5. **Report** to the user: key highlights, any conflicts or gaps detected, multi-day arcs, and open threads/cliff-hangers.

**Critical rules (all in the skill, but worth repeating):**
- Gmail: MANDATORY 3-pass scan (inbox + sent + drafts) with full pagination. If `on:` returns empty, fallback to `after:/before:` range.
- Calendar: Scan BOTH `primary` and `ggrigo@gmail.com` calendars.
- Slack: 2-pass (sent + received). Use `on:YYYY-MM-DD`, never `after:`.
- Fireflies: Read FULL transcripts (`fireflies_get_transcript`), never trust summaries.
- Timezone: Convert everything to EET (UTC+2) before merging.
- Check CLAUDE.md for people, projects, and terms.

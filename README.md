# Karoulis v0.3.0

Personal plugin toolkit by Georgios Grigoriadis (Baresquare).

## Skills

### fabric

Cross-source daily activity timeline builder. Pulls from Google Calendar, Gmail, Slack, Fireflies, and Google Drive to assemble a chronological "fabric" of your day.

**Triggers:** "fabric", "build fabric", "what happened today", "daily recap", "timeline of my day", "reconstruct my day"

**What it does:**
1. Pulls activity from 5 sources (Calendar, Gmail, Slack, Fireflies, Drive)
2. Normalizes timestamps to Europe/Athens timezone
3. Assembles chronological timeline with cross-references
4. Identifies substantive vs noise, tracks commitments
5. Saves to persistent location for future reference

**Output:** Markdown files in `Projects/cowork-fabric/fabrics/fabric-YYYY-MM-DD.md`

### slack-jack

Generate Slack-Jack scheduling commands for batch meetings — one organizer meeting individually with many team members.

**Triggers:** "schedule meetings with my team", "set up 1-on-1s", "generate slack-jack commands", "batch schedule"

**What it does:**
1. Builds meeting list with attendees per session
2. Resolves identities via Slack profiles (emails + Slack IDs)
3. Groups by shared attendees for efficient scheduling
4. Checks Google Calendar availability across all attendees
5. Proposes conflict-free schedule
6. Generates paste-ready Slack-Jack commands

**Requires:** Google Calendar MCP, Slack MCP

## Commands

### /fabric-go

Quick command to build yesterday's and today's fabric timeline with default settings.

**Usage:** Just type `/fabric-go` — no parameters needed

**What it does:**
- Builds yesterday's fabric (if missing)
- Builds today's fabric (up to current time)
- Cross-references between days
- Saves to standard location

## Installation

1. Install the plugin (drag .plugin file into Cowork sidebar)
2. Ensure MCPs are configured:
   - Google Calendar MCP
   - Gmail MCP
   - Slack MCP
   - Fireflies MCP
   - Google Drive MCP

## Quick Start

- **Daily timeline:** Type `/fabric-go` to build your daily activity fabric
- **Batch scheduling:** Say "schedule 1-on-1s with my team next week"
- **Manual fabric:** Say "build fabric for February 15"

## Changelog

### v0.3.0 (Feb 19, 2026)
- Added `fabric` skill — consolidated from 7 separate fabric skills
- Added `/fabric-go` command for quick daily timeline
- Renamed `batch-scheduling` to `slack-jack` 
- Removed `inspect-update-plugin` skill
- Updated for better Cowork integration

### v0.2.0
- Added `batch-scheduling` skill
- Added `inspect-update-plugin` skill

## Author

Georgios Grigoriadis  
ggrigo@gmail.com  
[Baresquare](https://baresquare.com)
# /fabric-go

Quick command to build yesterday's and today's fabric timeline.

## What it does

Executes the fabric skill with default parameters:
- Builds yesterday's fabric (full day, all 5 sources)
- Builds today's fabric (up to current time, all 5 sources)
- Cross-references between days
- Saves to `Projects/cowork-fabric/fabrics/fabric-YYYY-MM-DD.md`

## Usage

Just type:
```
/fabric-go
```

No parameters needed. It automatically:
- Uses Europe/Athens timezone
- Pulls from all 5 sources (Calendar, Gmail, Slack, Fireflies, Drive)
- Skips yesterday if already complete
- Updates today incrementally

## Behind the scenes

This command invokes the `fabric` skill with preset parameters optimized for daily use. It's equivalent to asking:

"Build yesterday's fabric if missing, then build today's fabric up to now, cross-reference everything, and save to the standard location."

## Requirements

- Google Calendar MCP
- Gmail MCP  
- Slack MCP
- Fireflies MCP
- Google Drive MCP

All MCPs must be configured with appropriate auth.
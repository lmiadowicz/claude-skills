# granola

Fetch meeting notes from [Granola](https://granola.so) and save them as structured Markdown files to your Obsidian vault, categorized by meeting type.

## What it does

1. Fetches meetings from Granola (summaries, notes, participants)
2. Classifies each meeting as Interview, Work, or Discovery
3. Saves to `{VAULT}/Meetings/{Category}/YYYY-MM-DD {Title}.md`
4. Optionally ingests interview meetings into a wiki

## Prerequisites

- Granola desktop app installed and connected
- Granola MCP server configured in Claude Code

## Configuration

| Variable | Description | Required |
|----------|-------------|----------|
| `OBSIDIAN_VAULT` | Path to your Obsidian vault root | No (uses `./meetings-output/` fallback) |
| `COMPANY_DOMAIN` | Your company's email domain for meeting classification | No |

## Arguments

| Argument | Description | Default |
|----------|-------------|---------|
| `all` | Fetch all recent meetings | default |
| `interviews` | Only interview-type meetings | — |
| `work` | Only internal work meetings | — |
| `discovery` | Only customer/partner/external meetings | — |
| `<keyword>` | Fetch meetings matching a title keyword | — |
| `--days N` | How many days back to look | 30 |
| `--no-ingest` | Save files only, skip wiki ingest | — |

## Meeting classification

- **Interview** — title contains: interview, hiring, recruiting, SME interview, or similar signals
- **Discovery** — external participants (outside `$COMPANY_DOMAIN`), or title contains: demo, customer, client, partner, intro, coffee, onboarding
- **Work** — everything else (internal meetings)

## Output format

```markdown
---
title: "Meeting Title"
date: YYYY-MM-DD
type: meeting
category: interview | work | discovery
participants: [Name1, Name2]
source: granola
granola_id: {id}
---
# Meeting Title
## Summary
## Notes
## Action Items
```

## Wiki ingest

Interview meetings are automatically ingested into the wiki (creates topic + cross-links) unless `--no-ingest` is passed or `$OBSIDIAN_VAULT` is not set. Work and Discovery meetings are saved as raw files only.

## Examples

```
/granola
/granola interviews --days 14
/granola discovery --no-ingest
/granola "product review"
/granola all --days 7
```

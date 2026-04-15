---
name: granola
description: Fetch meeting notes from Granola and save them as structured Markdown files to your Obsidian vault, categorized by meeting type (interviews, work, discovery)
argument-hint: "[all|interviews|work|discovery|<meeting-title-keyword>] [--days N] [--no-ingest]"
allowed-tools:
  - mcp__claude_ai_Granola__list_meetings
  - mcp__claude_ai_Granola__list_meeting_folders
  - mcp__claude_ai_Granola__get_meetings
  - mcp__claude_ai_Granola__get_meeting_transcript
  - mcp__claude_ai_Granola__query_granola_meetings
  - Write
  - Read
  - Bash
---

<objective>
Fetch meeting notes from Granola and save them as structured Markdown files in your Obsidian vault under Meetings/, categorized by type.
</objective>

<context>
User input: $ARGUMENTS

**Vault base:** `$OBSIDIAN_VAULT` environment variable (or `./meetings-output` if not set)
**Meetings base:** `{VAULT}/Meetings/`

**Subfolders:**
- `Interviews/` — job interviews, candidate interviews, SME interviews
- `Work/` — team syncs, planning, internal meetings
- `Discovery/` — customer calls, user research, partner meetings

**Flags:**
- `all` — fetch all recent meetings (default: last 30 days)
- `interviews` — only interview-type meetings
- `work` — only internal work meetings
- `discovery` — only customer/partner/external meetings
- `<keyword>` — fetch meetings matching a title keyword
- `--days N` — how many days back to look (default: 30)
- `--no-ingest` — save files only, skip wiki ingest step
</context>

<process>

## Step 1: Parse arguments and determine filter

Extract from $ARGUMENTS:
- **Filter:** all | interviews | work | discovery | keyword (default: all)
- **Days:** from --days flag (default: 30)
- **Ingest:** true unless --no-ingest

## Step 2: List meetings

Call `list_meetings` with appropriate time_range.

For keyword filter: fetch all, then filter by title containing the keyword.

## Step 3: Categorize meetings

For each meeting, classify into one of:
- **Interview** — title contains: interview, rozmowa kwalifikacyjna, hiring, rekrutacja, SME interview, [company] + first name only (e.g. "n8n / Łukasz")
- **Discovery** — external participants from outside your company domain (`$COMPANY_DOMAIN` env var, if set), or title contains: demo, customer, client, partner, intro, coffee, onboarding (external)
- **Work** — everything else (internal meetings within your company domain)

Apply the user's filter to decide which categories to process.

## Step 4: Fetch meeting details

For meetings passing the filter, call `get_meetings` (batch up to 10 at a time).

Skip meetings that:
- Have no notes and no summary (empty meetings)
- Are pure daily standups with no meaningful content
- Are already saved (check if file exists at output path)

## Step 5: Save to vault

For each meeting, create a markdown file:

**File path:** `{MEETINGS_BASE}/{Category}/YYYY-MM-DD {Sanitized Title}.md`

**File format:**
```markdown
---
title: "{Meeting Title}"
date: YYYY-MM-DD
type: meeting
category: interview | work | discovery
participants: [Name1, Name2]
source: granola
granola_id: {meeting_id}
---

# {Meeting Title}

**Date:** YYYY-MM-DD  
**Participants:** {participant list}

---

## Summary

{AI-generated summary from Granola, or "No summary available"}

## Notes

{Private notes if any, or "No notes recorded"}

## Action Items

{Extract action items from summary/notes, or "None identified"}
```

Sanitize title for filename: remove special chars, truncate to 80 chars.

## Step 6: Report saved files

```
Granola sync complete!

Saved: N meetings
  Interviews: N files → Meetings/Interviews/
  Work: N files → Meetings/Work/
  Discovery: N files → Meetings/Discovery/
Skipped: N (empty or already saved)
```

## Step 7: Done

All files saved. No additional ingest steps.

</process>

<principles>
1. Never fetch transcript unless user explicitly asks — summaries and notes are enough
2. Skip empty meetings (no summary, no notes)
3. Files already saved are not overwritten unless content has changed
</principles>

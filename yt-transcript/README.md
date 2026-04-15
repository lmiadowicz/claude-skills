# yt-transcript

Download a YouTube video transcript and save it as a clean, readable Markdown note with extracted key insights.

## What it does

1. Fetches video metadata (title, channel, date, duration) via yt-dlp
2. Downloads transcript (prefers manual subtitles, falls back to auto-generated)
3. Converts raw SRT to clean prose paragraphs with deduplication
4. Extracts key insights, concepts, principles, and memorable quotes
5. Saves a structured Markdown note to your output directory
6. Optionally ingests the note into a wiki

## Prerequisites

- `yt-dlp` — installed automatically via Homebrew if missing (`brew install yt-dlp`)

## Configuration

| Variable | Description | Required |
|----------|-------------|----------|
| `OBSIDIAN_VAULT` | Path to your Obsidian vault root | No (uses `./Transcriptions/` fallback) |

## Arguments

| Argument | Description | Default |
|----------|-------------|---------|
| `<youtube-url>` | YouTube video URL | Required |
| `--lang <code>` | Preferred transcript language | `en` |
| `--output <path>` | Override output directory | `$OBSIDIAN_VAULT/Transcriptions` or `./Transcriptions` |
| `--no-clean` | Keep raw SRT formatting instead of clean prose | off |
| `--no-ingest` | Skip wiki ingest step | off |

## Output

Notes are saved to:
```
{output-dir}/{ChannelName}/YYYY-MM-DD VideoTitle.md
```

Each note includes:
- Frontmatter (title, channel, date, duration, source URL)
- Key Lessons & Insights section (concepts, principles, workflow, quotes)
- Full cleaned transcript

## Wiki ingest

Transcripts are saved directly to the output directory. No wiki ingest — the `--no-ingest` flag is a no-op in this version.

## Examples

```
/yt-transcript https://www.youtube.com/watch?v=dQw4w9WgXcQ
/yt-transcript https://youtu.be/abc123 --lang pl
/yt-transcript https://youtu.be/abc123 --output ~/Documents/Transcriptions
/yt-transcript https://youtu.be/abc123 --no-clean --no-ingest
```

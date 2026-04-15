---
name: yt-transcript
description: Download YouTube video transcript and save it as a formatted Markdown note in the Obsidian vault under Transcriptions/ChannelName/Date-VideoName
argument-hint: "<youtube-url> [--lang <code>] [--output <path>] [--no-clean] [--no-ingest]"
allowed-tools:
  - Bash
  - Write
  - Read
---

<objective>
Extract the transcript from a YouTube video (without downloading the video file) and save it as a clean, readable Markdown note in the Obsidian vault under:
`Transcriptions/<ChannelName>/<YYYY-MM-DD> <VideoTitle>.md`
</objective>

<context>
User input: $ARGUMENTS

**Flags (all optional):**
- `--lang <code>` — Preferred transcript language code (default: `en`, falls back to auto-generated)
- `--output <path>` — Directory to save transcripts (overrides `$OBSIDIAN_VAULT`)
- `--no-clean` — Keep raw SRT formatting instead of converting to clean prose paragraphs
- `--no-ingest` — Skip wiki ingest step

**Output path resolution (in order of precedence):**
1. `--output <path>` flag
2. `$OBSIDIAN_VAULT/Transcriptions/` environment variable
3. `./Transcriptions/` (current directory fallback)
</context>

<process>

## Step 1: Parse Arguments

Extract from `$ARGUMENTS`:
- **URL:** The YouTube URL (required — if missing, ask the user)
- **Lang:** From `--lang` flag (default: `en`)
- **Vault:** From `--vault` flag (default: `lumi`)
- **No-clean:** Whether `--no-clean` was passed

Resolve output base path:
```bash
if [[ -n "${OUTPUT_FLAG}" ]]; then
  VAULT_BASE="${OUTPUT_FLAG}"
elif [[ -n "${OBSIDIAN_VAULT}" ]]; then
  VAULT_BASE="${OBSIDIAN_VAULT}/Transcriptions"
else
  VAULT_BASE="$(pwd)/Transcriptions"
fi
```

## Step 2: Ensure yt-dlp is Available

```bash
if ! command -v yt-dlp &>/dev/null; then
  echo "Installing yt-dlp..."
  brew install yt-dlp
fi
yt-dlp --version
```

## Step 3: Fetch Video Metadata

Use yt-dlp to get channel name, title, upload date, and duration without downloading:

```bash
yt-dlp \
  --no-download \
  --print "%(channel)s|%(title)s|%(upload_date)s|%(duration_string)s|%(webpage_url)s|%(description)s" \
  "<URL>"
```

Parse the output fields:
- `CHANNEL` — sanitize: replace `/`, `:`, `?`, `*`, `\`, `|`, `"`, `<`, `>` with `-`; trim whitespace
- `TITLE` — sanitize same way; truncate to 80 chars
- `UPLOAD_DATE` — reformat from `YYYYMMDD` → `YYYY-MM-DD`
- `DURATION` — as-is
- `VIDEO_URL` — original URL
- `DESCRIPTION` — first 500 chars for the note frontmatter

## Step 4: Extract Transcript

Use a temp directory for intermediate files:

```bash
TMPDIR=$(mktemp -d)
```

Try in order:
1. **Manual subtitles** in preferred language:
```bash
yt-dlp \
  --skip-download \
  --write-sub \
  --sub-lang "<LANG>" \
  --convert-subs srt \
  --output "${TMPDIR}/transcript" \
  "<URL>" 2>&1
```

2. If no manual sub found, **auto-generated subtitles**:
```bash
yt-dlp \
  --skip-download \
  --write-auto-sub \
  --sub-lang "<LANG>" \
  --convert-subs srt \
  --output "${TMPDIR}/transcript" \
  "<URL>" 2>&1
```

3. If lang isn't `en`, also try `en` as fallback before giving up.

Find the downloaded `.srt` file:
```bash
SRT_FILE=$(find "${TMPDIR}" -name "*.srt" | head -1)
```

If no SRT found at all: inform the user that no transcript is available for this video, clean up TMPDIR, and stop.

## Step 5: Convert SRT to Clean Markdown Text

Parse the SRT file with a Python one-liner that:
1. Strips sequence numbers and timecodes
2. Merges continuation lines
3. Removes `[Music]`, `[Applause]`, `[Laughter]`, and similar bracketed sound annotations
4. Collapses repeated whitespace/newlines
5. Splits into paragraphs every ~500 characters at sentence boundaries

```bash
python3 << 'PYEOF'
import re

srt_path = "${SRT_FILE}"

with open(srt_path, 'r', encoding='utf-8') as f:
    content = f.read()

# Parse SRT blocks individually to handle YouTube auto-sub overlap deduplication
blocks = re.split(r'\n\n+', content.strip())

sentences = []
for block in blocks:
    lines = block.strip().split('\n')
    # Skip sequence number lines and timecode lines
    text_lines = [l for l in lines if l.strip()
                  and not re.match(r'^\d+$', l.strip())
                  and not re.match(r'\d{2}:\d{2}:\d{2}', l)]
    if text_lines:
        text = ' '.join(text_lines)
        text = re.sub(r'<[^>]+>', '', text)       # Remove HTML tags
        text = re.sub(r'\[.*?\]', '', text)        # Remove [Music] etc.
        text = re.sub(r'>>', '', text)             # Remove ">>" markers
        text = text.strip()
        if text:
            sentences.append(text)

# Deduplicate: YouTube auto-subs repeat the same phrase ~3x with slight overlap.
# Strategy: track a rolling window of recent tokens and only emit new tokens.
def deduplicate_sentences(sentences):
    result = []
    window = []
    window_size = 150

    for sent in sentences:
        words = sent.split()
        new_start = 0
        for i in range(min(len(words), len(window))):
            chunk_str = ' '.join(words[:i+1])
            if chunk_str in ' '.join(window):
                new_start = i + 1
        new_words = words[new_start:]
        if new_words:
            result.append(' '.join(new_words))
        window.extend(words)
        if len(window) > window_size:
            window = window[-window_size:]
    return result

deduped = deduplicate_sentences(sentences)
text = ' '.join(deduped)

# Collapse multiple spaces
text = re.sub(r' {2,}', ' ', text).strip()

# Split into paragraphs at sentence boundaries every ~600 chars
words = text.split(' ')
paragraphs = []
current_para = []
current_len = 0
for word in words:
    current_para.append(word)
    current_len += len(word) + 1
    if current_len > 600 and word.endswith(('.', '!', '?')):
        paragraphs.append(' '.join(current_para))
        current_para = []
        current_len = 0
if current_para:
    paragraphs.append(' '.join(current_para))

print('\n\n'.join(paragraphs))
PYEOF
```

Capture output as `CLEAN_TEXT`.

If `--no-clean` was passed, skip the Python conversion and use raw SRT content instead.

## Step 6: Extract Key Insights with Claude

Before building the note, analyze `CLEAN_TEXT` to extract a structured insights section. Do this yourself — read the transcript and synthesize:

**First, detect the transcript language** (English, Polish, etc.). ALL section headers AND content in the insights block MUST be written in that same language.

**What to extract:**
- **Concepts / Mental Models** *(translate header to transcript language)* — named concepts, frameworks, mental models introduced in the video (with a 1-2 sentence explanation of each)
- **Principles & Best Practices** *(translate header to transcript language)* — concrete rules, principles, or actionable recommendations (numbered list, each with brief explanation)
- **Practical Workflow** *(translate header to transcript language)* — if the video demonstrates a process or workflow, summarize it as a step-by-step block
- **Quotes Worth Remembering** *(translate header to transcript language)* — 1-3 direct quotes that capture the core idea best

**Language mapping for common languages (use the appropriate one):**
- English transcript → "Key Lessons & Insights" / "Concepts / Mental Models" / "Principles & Best Practices" / "Practical Workflow (if applicable)" / "Quotes Worth Remembering"
- Polish transcript → "Kluczowe lekcje i insighty" / "Koncepcje / modele mentalne" / "Zasady i best practices" / "Praktyczny workflow (jeśli dotyczy)" / "Cytaty warte zapamiętania"
- For other languages, translate the headers accordingly.

**Rules:**
- ALL headers AND content must be in the transcript's language — no mixing languages
- Be concrete and actionable — someone reading only this section should be able to apply the ideas
- Skip promotional content (newsletter plugs, sponsor segments) — focus on the educational content
- If a category has nothing relevant, omit it entirely
- Bold key terms and principles for scannability

## Step 7: Build the Markdown Note

Compose the full Markdown file with the insights section placed BEFORE the transcript:

```markdown
---
title: "<TITLE>"
channel: "<CHANNEL>"
date: <YYYY-MM-DD>
duration: "<DURATION>"
source: "<VIDEO_URL>"
tags:
  - transcript
  - youtube
---

# <TITLE>

**Channel:** <CHANNEL>
**Date:** <YYYY-MM-DD>
**Duration:** <DURATION>
**Source:** [Watch on YouTube](<VIDEO_URL>)

---

## Description

<DESCRIPTION (first 500 chars)>...

---

## <INSIGHTS_SECTION_TITLE>
<!-- Section title and all sub-headers in the transcript's language. English: "Key Lessons & Insights". Polish: "Kluczowe lekcje i insighty". -->

### <CONCEPTS_HEADER>
- **<Concept name>** — <explanation>
...

### <PRINCIPLES_HEADER>
1. **<Principle>** — <explanation>
...

### <WORKFLOW_HEADER>
```
1. [Step] ...
```

### <QUOTES_HEADER>
> *"<quote>"*

---

## Transcript

<CLEAN_TEXT>

---

*Transcript extracted by yt-transcript skill on <TODAY_DATE>*
```

## Step 8: Save to Obsidian Vault

Determine the output path:
```bash
VAULT_BASE="<vault-base-path>"
CHANNEL_DIR="${VAULT_BASE}/<SANITIZED_CHANNEL>"
FILENAME="<YYYY-MM-DD> <SANITIZED_TITLE>.md"
OUTPUT_PATH="${CHANNEL_DIR}/${FILENAME}"

mkdir -p "${CHANNEL_DIR}"
```

Write the Markdown note to `OUTPUT_PATH` using the Write tool.

Clean up temp directory:
```bash
rm -rf "${TMPDIR}"
```

## Step 9: Confirm to User

Output:
```
Transcript saved!

Video:   <TITLE>
Channel: <CHANNEL>
Date:    <YYYY-MM-DD>
File:    <OUTPUT_PATH>
```

If Obsidian is running, optionally open the note:
```bash
open "obsidian://open?vault=<vault-name>&file=Transcriptions/<CHANNEL>/<filename-without-extension>"
```

</process>

<step name="wiki_ingest">
## Step 10: Ingest into Wiki (optional)

After saving the transcript, ingest it into the wiki. Only runs if `$OBSIDIAN_VAULT` is set and `{WIKI_BASE}/CLAUDE.md` exists. Skip if `--no-ingest` was passed.

**Wiki base:** `$OBSIDIAN_VAULT`

### 10a. Read context

Read:
- `{WIKI_BASE}/CLAUDE.md` — conventions and rules
- `{WIKI_BASE}/index.md` — existing pages (to find cross-link targets and avoid duplicates)

### 10b. Read the transcript note

Read the saved file from `OUTPUT_PATH`. Focus on the **Key Lessons & Insights** section (already extracted in Step 6) — this is your primary input for the wiki page. The full transcript body is available if you need to verify a specific claim.

### 10c. Write wiki pages

**1. Topic page** → `{WIKI_BASE}/wiki/topics/YYYY-MM-DD-[slug].md`
- YYYY-MM-DD = upload date of the video (not today)
- Slug derived from the video title, kebab-case
- Include: one-line thesis capturing the video's core argument, key concepts/principles as sections with your own synthesis (not just copying the insights block), cross-links to existing wiki pages, Sources section pointing to the raw file
- Frontmatter: title, type: topic, tags (include `youtube`, `transcript`, channel name), sources: 1, updated: today

**2. Entity pages** → `{WIKI_BASE}/wiki/entities/[name].md`
- Only for significant people (speaker, interviewees) or products/companies central to the video
- If already in index.md: update with new info from this video; don't create a duplicate
- If new: create with basic profile + link to the topic page

**3. Concept pages** → `{WIKI_BASE}/wiki/concepts/[concept].md`
- For significant frameworks or mental models introduced
- Only if not already covered in an existing wiki page

**Cross-linking rules:**
- Scan index.md for existing pages that relate to this video's content and cross-link both ways
- Use `[[PageName]]` wikilink syntax
- Every new page must link to at least one existing wiki page

### 10d. Update index.md

- Add new pages to the appropriate tables
- Mark the raw source as **Ingested** in the Raw Source Inventory (Transcriptions section)
- Update the header count

### 10e. Append to log.md

Prepend before the first existing `##` entry in `{WIKI_BASE}/log.md`:

```
## [TODAY] ingest | [Video Title] ([Channel])

[2-3 sentences: what the video covered and what was added to the wiki]

Pages touched: [[page1]], [[page2]], ...
```

### 10f. Confirm to user

```
Wiki updated!

New pages: [list]
Updated pages: [list]
```

</step>

<error_handling>

- **yt-dlp install fails:** Tell the user to run `brew install yt-dlp` manually, then retry.
- **No transcript available:** Some videos have transcripts disabled. Inform the user and suggest trying `--lang` with a different code, or that the video has no captions.
- **Private/age-restricted video:** yt-dlp will error. Inform the user the video is inaccessible.
- **Very long transcripts (>2h videos):** Warn the user the note may be large. Proceed normally.
- **Title too long for filename:** Truncate to 80 characters before the file extension.
- **Special characters in channel/title:** Sanitize all path-unsafe characters (`/`, `:`, `?`, etc.) to `-`.

</error_handling>

<principles>
1. Never download the video file itself — transcripts only, `--skip-download` always.
2. Always prefer human-written subtitles over auto-generated ones.
3. Sanitize file paths strictly — Obsidian runs on macOS/iOS and paths must be clean.
4. The note should be readable on its own — good formatting matters.
5. If the URL is missing from arguments, ask the user before doing anything.
</principles>

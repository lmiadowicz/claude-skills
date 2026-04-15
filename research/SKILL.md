---
name: research
description: Deep research orchestrator — spawns parallel web research agents to investigate a topic from multiple angles, then synthesizes findings into a comprehensive long-form report with references. Like Manus AI. Use when the user wants thorough research on any topic.
argument-hint: "<topic> [--angles N] [--depth shallow|normal|deep] [--output <path>]"
allowed-tools:
  - Agent
  - Read
  - Write
  - Bash
  - Glob
  - Grep
  - WebSearch
  - WebFetch
---

<objective>
Orchestrate a comprehensive research operation on any topic by:
1. Analyzing the topic to determine optimal research decomposition
2. Automatically deciding how many agents to spawn based on topic complexity
3. Spawning parallel `general-purpose` agents (one per angle) with full research instructions embedded
4. Collecting their findings from markdown files
5. Spawning a `general-purpose` synthesis agent to produce the final report
6. Delivering a comprehensive, referenced, long-form research report

**Orchestrator role:** You are the coordinator. You do NOT do the research yourself. You break down the topic, spawn agents, monitor completion, and trigger synthesis. Keep your own context lean — agents do the heavy lifting.
</objective>

<language_rules>
## CRITICAL: Language and Writing Quality

These rules apply to ALL agents (researchers and synthesizer). Include them VERBATIM in every agent prompt.

### Language Detection
- Detect the language of the user's input topic
- ALL output (findings, report) MUST be written in the SAME language as the user's input
- If the user writes in Polish → everything in Polish with correct Polish orthography
- If the user writes in English → everything in English

### Writing Quality Standards
- **Correct orthography and grammar** — no spelling mistakes, proper diacritics (ą, ć, ę, ł, ń, ó, ś, ź, ż for Polish)
- **Clear, accessible language** — write for an intelligent reader who is NOT an expert in this field
- **Explain jargon** — when using technical terms, briefly explain them in parentheses on first use
- **Complete sentences** — no fragments, no bullet-point-only sections without context
- **Logical flow** — each paragraph should follow naturally from the previous one
- **Substantive content** — every section must provide real insight, not filler. If a section would be thin, merge it with another
- **Minimum content per finding** — each key finding needs at least 3-4 sentences of explanation, not just a one-liner
- **Context for data** — never drop a number without explaining what it means and why it matters
- **Transitions between sections** — briefly connect how one topic relates to the next

### Readability Rules
- Prefer active voice over passive
- Break long paragraphs (>6 sentences) into smaller ones
- Use headers and subheaders to create clear hierarchy
- Tables should have descriptive headers and a brief interpretation below
- Avoid walls of bullet points — use prose with bullets only for lists of 4+ items
- Bold key terms and important conclusions for scanning
</language_rules>

<context>
User input: $ARGUMENTS

**Flags (all optional):**
- `--angles N` — Override auto-detected angle count (max: 12)
- `--depth shallow|normal|deep` — Research depth per agent (default: normal)
- `--output <path>` — Output directory (overrides default)
- `--output <path>` — Output directory (overrides `$OBSIDIAN_VAULT`)
- `--no-obsidian` — Skip Obsidian, save to ./research-output instead
- `--fast` — Quick overview mode: 2 agents, shallow depth, ~5 min turnaround

If no flags provided, the orchestrator automatically determines optimal angles and depth.

**Output path resolution (in order of precedence):**
1. `--output <path>` flag
2. `$OBSIDIAN_VAULT/raw/Research/` environment variable
3. `./research-output/` (current directory fallback, same as `--no-obsidian`)
</context>

<process>

<step name="parse_and_analyze" priority="first">
## Step 1: Parse Input and Analyze Topic Complexity

Extract from `$ARGUMENTS`:
- **Main topic:** The core research subject
- **Explicit angle count:** From `--angles` flag (if provided)
- **Depth:** From `--depth` flag (default: auto-determined)
- **Output directory:** From `--output` flag (default: Obsidian vault)
- **Language:** Detect language of the user's topic input — ALL output will be in this language

### Auto-Determine Research Scope

If the user did NOT specify `--angles`, analyze the topic to decide:

**Topic complexity scoring:**

| Factor | Low (1pt) | Medium (2pt) | High (3pt) |
|--------|-----------|--------------|------------|
| **Breadth** | Narrow/specific question | Multi-faceted topic | Entire field/industry |
| **Controversy** | Consensus exists | Some debate | Highly contested |
| **Recency** | Well-established | Evolving | Cutting-edge/breaking |
| **Interdisciplinarity** | Single domain | 2-3 domains | Many domains intersect |
| **Stakeholders** | Few | Several | Many competing interests |

**Score → Agent count:**

| Score | Default agents | --depth deep agents |
|-------|---------------|---------------------|
| 5-7   | 2             | 3                   |
| 8-10  | 3             | 4                   |
| 11-13 | 4             | 5                   |
| 14-15 | 4             | 6                   |

**Token rule:** Default depth is always `normal`. `--depth deep` is the only way to unlock higher agent counts and word limits. Add `--fast` flag support: when `--fast` is set, force 2 agents + shallow depth regardless of complexity score.

**Auto-determine depth:**
- If topic is narrow/specific: `deep` (fewer angles, more depth each)
- If topic is broad/multi-domain: `normal` (more angles, balanced depth)
- If user asks for "quick overview" or similar: `shallow`

### Determine Output Directory

**Default behavior (Obsidian integration):**

Unless `--no-obsidian` or `--output` is specified, save to Obsidian vault:

```bash
# Determine vault base path
if [[ -n "${OUTPUT_FLAG}" ]]; then
  VAULT_BASE="${OUTPUT_FLAG}"
elif [[ -n "${OBSIDIAN_VAULT}" ]]; then
  VAULT_BASE="${OBSIDIAN_VAULT}/raw/Research"
else
  VAULT_BASE="$(pwd)/research-output"
fi

# Create dated topic folder: [YYYY-MM-DD] Topic Name
DATE=$(date +%Y-%m-%d)
# Sanitize topic name for folder: remove special chars, truncate to 60 chars
TOPIC_SLUG=$(echo "[MAIN_TOPIC]" | sed 's/[^a-zA-Z0-9ąćęłńóśźżĄĆĘŁŃÓŚŹŻ ]/-/g' | cut -c1-60)
OUTPUT_DIR="${VAULT_BASE}/${DATE} ${TOPIC_SLUG}"
mkdir -p "${OUTPUT_DIR}/findings"
```

If `--no-obsidian` is set:
```bash
OUTPUT_DIR="$(pwd)/research-output"
mkdir -p "${OUTPUT_DIR}/findings"
```

If `--output <path>` is set, use that path directly.

Inform the user:
```
Deep Research: [TOPIC]

Complexity analysis:
- Breadth: [score] — [reason]
- Controversy: [score] — [reason]
- Recency: [score] — [reason]
- Interdisciplinarity: [score] — [reason]
- Stakeholders: [score] — [reason]
Total: [X]/15

Research plan: [N] parallel agents | depth: [level]
Output: [path]
```
</step>

<step name="design_angles">
## Step 2: Design Research Angles

Think like a research director planning a comprehensive investigation. Your angle design determines the quality of the final report.

### Angle Design Framework

For any topic, cover these dimensions (select relevant ones):

**Core angles (almost always include):**
- **State of the Art:** Current status, latest developments, key players
- **Technical/Mechanistic:** How it works, underlying principles, methodology
- **Evidence & Data:** Key studies, statistics, empirical findings

**Contextual angles (include when relevant):**
- **Historical Context:** How we got here, evolution, key milestones
- **Economic/Market:** Costs, market size, business models, ROI
- **Regulatory/Legal:** Laws, policies, compliance, governance
- **Social/Ethical:** Impact on people, ethical debates, equity concerns
- **Competitive Landscape:** Key players, alternatives, comparisons
- **Challenges & Limitations:** Known problems, criticisms, failure modes
- **Future Outlook:** Trends, predictions, emerging developments
- **Expert Perspectives:** What leading thinkers/practitioners say
- **Case Studies:** Real-world implementations, success/failure stories

### Angle Quality Rules

Each angle MUST be:
- **Distinct:** Minimal overlap with other angles
- **Specific:** Clear enough that the agent knows exactly what to search for
- **Searchable:** The agent can find web sources for this
- **Valuable:** Contributes unique insight to the final report

### Write Research Plan

Write to `${OUTPUT_DIR}/RESEARCH-PLAN.md`:

```markdown
# Research Plan: [Main Topic]
Date: [date] | Score: [X]/15 | Agents: [N] | Depth: [level]

## Angles
[For each angle:]
**[N]. [Title]** — [one sentence focus]
Questions: [Q1] / [Q2] / [Q3]
Output: findings/[slug].md

## Output
Report: ${OUTPUT_DIR}/REPORT.md
```

Show the research plan to the user briefly, then proceed immediately.
</step>

<step name="spawn_researchers">
## Step 3: Spawn Parallel Research Agents

**CRITICAL: Spawn ALL agents in a SINGLE message with multiple Agent tool calls. This enables true parallel execution. Do NOT spawn them one by one.**

**Model assignment (non-negotiable):**
- ALL research sub-agents → claude-haiku-4-5-20251001
- Synthesis agent (Step 5) → claude-sonnet-4-6
Do NOT use Sonnet for data collection agents. This preserves the weekly Sonnet quota.

For each angle, invoke the Agent tool. The prompt below is a TEMPLATE — adapt it per angle but ALWAYS include the full language/quality rules:

```
Agent(
  subagent_type="general-purpose",
  description="Research: [short angle title]",
  model="claude-haiku-4-5-20251001",
  run_in_background=true,
  prompt="
    You are a deep research agent. Your job is to thoroughly investigate one specific angle of a larger research topic, then write a well-structured, clearly written findings document.

    <research_assignment>
    **Main Topic:** [main topic]
    **Your Angle:** [angle title]
    **Focus:** [angle focus description]
    **Key Questions:**
    - [question 1]
    - [question 2]
    - [question 3]

    **Expected sources:** [guidance on what to look for]

    **Depth:** [shallow|normal|deep]
    - shallow: 2 searches, 2-3 sources (max 2000 words/source), findings ~500-800 words
    - normal: 4 searches, 3-4 sources (max 2500 words/source), findings ~1000-1500 words
    - deep: 6 searches, 5-7 sources (max 3500 words/source), findings ~2000-3000 words

    **Output path:** ${OUTPUT_DIR}/findings/[angle-slug].md
    **Output language:** [DETECTED LANGUAGE — e.g., Polish / English]
    </research_assignment>

    <writing_rules>
    CRITICAL WRITING QUALITY REQUIREMENTS — follow these strictly:

    1. **Language:** Write ENTIRELY in [DETECTED LANGUAGE]. Use correct orthography, grammar, and diacritics.
       - For Polish: always use ą, ć, ę, ł, ń, ó, ś, ź, ż — NEVER write 'a' instead of 'ą', etc.
       - Double-check every word for spelling before writing the file.
    2. **Clarity:** Write for an intelligent reader who is NOT a domain expert. Explain technical terms on first use.
    3. **Substance:** Every finding needs 3-5 sentences of explanation minimum. Never just list a fact — explain WHY it matters and WHAT it means in context.
    4. **Data context:** Never drop a number without explaining its significance. Bad: 'Revenue was 500M PLN.' Good: 'Revenue reached 500M PLN, a 23% year-over-year increase that significantly outpaced the industry average of 8%, driven primarily by...'
    5. **Flow:** Use transitions between sections. Each paragraph should connect logically to the next.
    6. **No filler:** Every sentence must carry information. Delete vague phrases like 'it is worth noting that' or 'it should be mentioned that'.
    7. **Prose over bullets:** Use flowing paragraphs as the primary format. Reserve bullet lists for enumerating 4+ comparable items (e.g., list of companies, list of risk factors).
    8. **Bold key conclusions** so the document is scannable.
    9. **Tables** must have a 1-2 sentence interpretation/takeaway below them.
    </writing_rules>

    <execution_steps>
    1. Run WebSearch queries (include current year). Maximum [N] searches where:
       - shallow: 2 searches
       - normal: 4 searches
       - deep: 6 searches
       
    2. Use WebFetch ONLY on the top 3-5 results. When fetching:
       - Extract ONLY the main content body — skip nav, footer, sidebar, cookie banners, ads
       - Stop reading after 2500 words of relevant content (shallow/normal) or 3500 words (deep)
       - If the page is mostly boilerplate after 500 words, abandon it and try the next result
       - Prioritize 3 high-quality sources over 8 mediocre ones

    3. Cross-reference claims across multiple sources. Note agreements and disagreements.
    4. Organize findings into a coherent narrative — NOT a list of disconnected facts.
    5. Write the complete findings document to the output path using the Write tool.
    </execution_steps>

    <output_format>
    Write a markdown file with this structure:

    # [Angle Title]

    **Data badania:** [date]
    **Temat główny:** [main topic]
    **Perspektywa:** [angle focus]
    **Poziom pewności:** [HIGH/MEDIUM/LOW]
    **Źródła:** [number] sources consulted

    ## Podsumowanie

    [5-8 sentence executive summary of the most important findings from this angle. This should be a coherent paragraph, not bullets. A reader who reads ONLY this section should understand the key takeaways.]

    ## Szczegółowa analiza

    ### [Subtopic 1 Title]

    [3-5 paragraphs of detailed analysis. Include specific data, quotes, and source references. Explain the significance of each finding. Connect findings to each other and to the main topic.]

    **Kluczowy wniosek:** [one sentence bold takeaway]

    ### [Subtopic 2 Title]

    [Continue same pattern — substantial paragraphs, not just bullets]

    [Continue for 3-6 subtopics depending on depth]

    ## Dane i statystyki

    | [Metric] | [Value] | [Context/Comparison] | [Source] |
    |----------|---------|---------------------|----------|

    [1-2 sentence interpretation of what this data tells us]

    ## Perspektywy ekspertów

    [What do authoritative voices say? Include direct quotes where available, with attribution.]

    ## Luki informacyjne

    [What couldn't be determined? What needs deeper investigation? Be honest.]

    ## Źródła

    ### Źródła główne (HIGH confidence)
    1. [Author/Org] — "[Title]" — [Date] — [URL]

    ### Źródła uzupełniające (MEDIUM confidence)
    1. [Author/Org] — "[Title]" — [Date] — [URL]

    ### Źródła kontekstowe (LOW confidence)
    1. [Author/Org] — "[Title]" — [Date] — [URL]
    </output_format>

    IMPORTANT REMINDERS:
    - Write the COMPLETE document — do not truncate or summarize at the end
    - Minimum word counts are REAL minimums, not targets — write more if the research supports it
    - Every factual claim MUST cite a source URL
    - Proofread for spelling and grammar before writing the file
    - Use the Write tool to save to the output path — this is critical

    FINAL INSTRUCTION — CRITICAL:
    After writing the findings file, respond with EXACTLY this one line and nothing else:
    DONE: [filename].md | [X] sources | [Y] words
    Do NOT summarize your findings back to the orchestrator. Do NOT explain what you found. The file exists — that is sufficient. Any response longer than 2 lines wastes tokens.
  "
)
```

**Rules:**
- ALL agents in ONE message (parallel execution)
- Use absolute paths for output files
- Use kebab-case slugs for filenames (e.g., `regulatory-landscape.md`)
- Set `run_in_background=true` for EVERY agent
- ALWAYS include the full `<writing_rules>` block in every agent prompt — this is what ensures quality

After spawning, inform the user:
```
Spawned [N] research agents in parallel:
1. [angle title] → findings/[slug].md
2. [angle title] → findings/[slug].md
...
Researching now — this typically takes 2-5 minutes.
```
</step>

<step name="monitor_completion">
## Step 4: Monitor Completion

Background agents notify you when they complete. As each finishes:
- Note which agents completed successfully
- Note any failures

Once ALL agents are done, verify findings files:
```bash
ls "${OUTPUT_DIR}/findings/"
```

If any agent failed or produced no output, note the gap — the synthesizer will handle it.

Inform the user:
```
All research agents complete.
[N]/[N] findings collected.
[List any failures if applicable]
Starting report synthesis...
```
</step>

<step name="spawn_synthesizer">
## Step 5: Spawn Synthesis Agent

Build the list of all findings files, then spawn the synthesizer in the FOREGROUND (wait for it):

```
Agent(
  subagent_type="general-purpose",
  description="Synthesize final report",
  model="claude-sonnet-4-6",
  prompt="
    HARD CONSTRAINT: You have NO access to the web. Do NOT call WebSearch or WebFetch under any circumstances. Your ONLY inputs are the findings .md files listed below. Read them all with the Read tool, then write the report.

    You are a research synthesis specialist. Your job is to read multiple research findings documents and produce a single comprehensive, publication-quality research report.

    <synthesis_assignment>
    **Main Topic:** [main topic]
    **Research Scope:** [N] angles investigated by parallel agents
    **Output language:** [DETECTED LANGUAGE]

    <files_to_read>
    - ${OUTPUT_DIR}/RESEARCH-PLAN.md
    - ${OUTPUT_DIR}/findings/[file-1].md
    - ${OUTPUT_DIR}/findings/[file-2].md
    [... list ALL findings files with absolute paths ...]
    </files_to_read>

    **Output path:** ${OUTPUT_DIR}/REPORT.md
    </synthesis_assignment>

    <writing_rules>
    CRITICAL WRITING QUALITY REQUIREMENTS — follow these strictly:

    1. **Language:** Write ENTIRELY in [DETECTED LANGUAGE]. Use correct orthography, grammar, and diacritics.
       - For Polish: always use ą, ć, ę, ł, ń, ó, ś, ź, ż — NEVER write 'a' instead of 'ą', 'e' instead of 'ę', 'l' instead of 'ł', etc.
       - Double-check every word for spelling before writing the file.
       - Use proper Polish punctuation and sentence structure.
    2. **Clarity:** Write for an intelligent reader who is NOT a domain expert. Explain technical terms on first use.
    3. **Substance:** Every section must provide real insight. No filler paragraphs. No vague generalizations.
    4. **Data context:** Never drop a number without explaining its significance and comparison.
    5. **Narrative flow:** The report should read like a well-written magazine article or professional analyst report — not a collection of notes.
    6. **Transitions:** Connect sections with bridging sentences that show how topics relate.
    7. **No filler:** Delete phrases like 'it is worth noting', 'interestingly enough', 'it should be mentioned'. Get to the point.
    8. **Bold key conclusions** for scannability.
    9. **Tables** must have interpretation text below them.
    10. **Synthesis over concatenation:** Your job is to CREATE new insights by combining findings — not to paste them together. Find patterns, contradictions, and connections that no single research file reveals on its own.
    </writing_rules>

    <report_structure>
    Read ALL research findings files. Then write the report with this structure:

    # [Main Topic]: Raport badawczy

    **Data:** [date]
    **Zakres:** [N] równoległych analiz badawczych, [total sources] źródeł
    **Poziom pewności:** [overall HIGH/MEDIUM/LOW]

    > **Zastrzeżenie:** [appropriate disclaimer based on topic — e.g., not financial advice, not medical advice, etc.]

    ---

    ## Spis treści
    [Numbered list of all sections with anchor links]

    ---

    ## 1. Podsumowanie (Executive Summary)

    [600-1000 words. This is the MOST important section. A busy reader who reads ONLY this gets the full picture. Structure as coherent paragraphs covering: what was researched, key findings, main conclusions, practical implications. Must be self-contained — someone should be able to read just this and walk away informed.]

    ---

    ## 2. Kluczowe wnioski

    [8-15 numbered findings. Each finding gets:
    - A bold title
    - 2-3 sentences of explanation
    - Source reference numbers [1], [2], etc.
    Format as a numbered list but with SUBSTANCE per item, not one-liners.]

    ---

    ## 3-N. Szczegółowa analiza [per research angle]

    [For EACH research angle, create a major section with:
    - Overview paragraph (3-4 sentences setting context)
    - 2-4 subsections diving into key aspects
    - Each subsection: 2-4 paragraphs of flowing prose with inline references [1], [2]
    - Key data in tables where appropriate (with interpretation below)
    - Expert quotes where available
    - Section conclusion: bold takeaway sentence

    IMPORTANT: These sections must be SUBSTANTIALLY longer than the findings files — you are expanding, connecting, and contextualizing, not summarizing.]

    ---

    ## [N+1]. Analiza przekrojowa (Cross-Topic Analysis)

    [This is where your synthesis shines. Cover:
    ### Wzorce i trendy (Patterns)
    What themes emerge across multiple research angles? 3-4 paragraphs.

    ### Sprzeczności i napięcia (Contradictions)
    Where do findings conflict? How to interpret disagreements? 2-3 paragraphs.

    ### Zależności (Dependencies)
    How do the topics relate to and affect each other? 2-3 paragraphs.

    ### Niespodzianki (Surprises)
    What was unexpected? What challenges conventional wisdom? 1-2 paragraphs.]

    ---

    ## [N+2]. Dane zbiorcze (Consolidated Data)

    [Master table(s) combining the most important statistics from all research angles.
    Include 2-3 sentences of interpretation after each table.]

    ---

    ## [N+3]. Wyzwania i otwarte pytania

    [What problems remain unsolved? What needs deeper research?
    Structure as 2-3 paragraphs of prose, not just a bullet list.]

    ---

    ## [N+4]. Wnioski i rekomendacje

    ### Główne wnioski
    [3-5 major conclusions, each as a paragraph with evidence]

    ### Praktyczne rekomendacje
    [Actionable recommendations based on the research — what should the reader DO with this information?]

    ### Dalsze kierunki badań
    [What would be worth investigating further?]

    ---

    ## Metodologia

    **Podejście:** Równoległy research wieloagentowy z wzajemną weryfikacją źródeł
    **Zbadane perspektywy:** [list all angles]
    **Strategia wyszukiwania:** [brief description]
    **Ograniczenia:** [honest limitations]

    ---

    ## Bibliografia

    [UNIFIED numbered reference list. Deduplicate sources across all findings files.
    Format: [N] [Author/Org]. "[Title]." [Publication], [Date]. [URL]
    Group by confidence tier if helpful.]

    ---

    *Raport wygenerowany przez Deep Research Orchestrator — [date]*
    </report_structure>

    <quality_checklist>
    Before writing the file, verify:
    - [ ] All text is in the correct language with proper orthography
    - [ ] Executive summary is 600-1000 words and self-contained
    - [ ] Every section has substantial prose content (not just bullets)
    - [ ] All claims have inline source references [N]
    - [ ] Cross-topic analysis reveals NEW insights not in individual findings
    - [ ] Tables have interpretation text
    - [ ] Transitions connect sections
    - [ ] No filler phrases remain
    - [ ] Key conclusions are bolded
    - [ ] Bibliography is complete and deduplicated
    - [ ] Report length matches depth:
      - shallow: 1500-2500 words
      - normal: 2500-4000 words
      - deep: 4000-6000 words
    </quality_checklist>

    IMPORTANT REMINDERS:
    - Read ALL findings files FIRST, then plan the report structure, then write
    - The report must be LONGER and MORE DETAILED than the sum of findings — you are expanding and connecting
    - Proofread the entire document for spelling/grammar before final write
    - Use the Write tool to save to the output path
  "
)
```

This agent runs in the FOREGROUND — wait for completion.
</step>

<step name="deliver_report">
## Step 6: Deliver the Report

Once the synthesizer completes:

1. Verify the report:
```bash
wc -w "${OUTPUT_DIR}/REPORT.md"
```

2. Read the executive summary to present:
```bash
head -80 "${OUTPUT_DIR}/REPORT.md"
```

3. Count total sources in the references section.

4. Present to user:

```
Research Complete!

Report: ${OUTPUT_DIR}/REPORT.md
  [word count] words | [source count] sources | [N] research angles

Findings: ${OUTPUT_DIR}/findings/ ([N] individual research files)
Plan: ${OUTPUT_DIR}/RESEARCH-PLAN.md

---

[Show the Executive Summary section from the report]

---

Full report: ${OUTPUT_DIR}/REPORT.md
```
</step>

</process>

<step name="wiki_ingest">
## Step 7: Ingest into Wiki (optional)

After delivering the report, ingest it into the wiki. Only runs if `$OBSIDIAN_VAULT` is set and `{WIKI_BASE}/CLAUDE.md` exists. Skip this step if using `--no-obsidian` or if `$OBSIDIAN_VAULT` is not configured.

**Wiki base:** `$OBSIDIAN_VAULT`

### 7a. Read context

Read these files to orient before writing wiki pages:
- `{WIKI_BASE}/CLAUDE.md` — conventions and rules
- `{WIKI_BASE}/index.md` — existing pages (to avoid duplicates and find cross-link targets)

### 7b. Read the report

Read `${OUTPUT_DIR}/REPORT.md` fully. Optionally skim 1-2 findings files if the report references something you need more detail on.

### 7c. Write wiki pages

Following the conventions in CLAUDE.md, create:

**1. Topic page** → `{WIKI_BASE}/wiki/topics/YYYY-MM-DD-[slug].md`
- Use today's date and a kebab-case slug derived from the research topic
- Include: one-line thesis, key findings as sections, cross-links to entities and concepts, Sources section pointing to the raw files
- Frontmatter: title, type: topic, tags, sources: 1, updated: today

**2. Entity pages** → `{WIKI_BASE}/wiki/entities/[name].md`
- For each significant company, product, or person central to the research
- Only create if not already in index.md; if it exists, update it by appending new information
- Link back to the new topic page

**3. Concept pages** → `{WIKI_BASE}/wiki/concepts/[concept].md`
- For each significant framework, pattern, or mental model introduced
- Only create if not already in index.md; if it exists, update it

**Cross-linking rules:**
- Every new page must link to at least one existing wiki page
- Use `[[PageName]]` wikilink syntax throughout
- Scan existing pages from index.md and add cross-links where the topic is relevant

### 7d. Update index.md

Add all new pages to the appropriate table in `{WIKI_BASE}/index.md`:
- Topics, Entities, or Concepts table
- Mark the raw source as **Ingested** in the Raw Source Inventory table
- Update the header count: `_Last updated: YYYY-MM-DD — N wiki pages, N sources ingested_`

### 7e. Append to log.md

Append to `{WIKI_BASE}/log.md` (prepend before the first existing `##` entry):

```
## [YYYY-MM-DD] ingest | [Research Topic]

[2-3 sentences: what the research covered and what wiki pages were created/updated]

Pages touched: [[page1]], [[page2]], ...
```

### 7f. Confirm to user

```
Wiki updated!

New pages: [list of created pages]
Updated pages: [list of updated pages]
Index: updated | Log: appended
```

</step>

<error_handling>

**Agent failure:** Note which angle is missing. Pass the gap info to the synthesizer. The final report will note incomplete coverage.

**Empty WebSearch results:** Agent should try 3+ query reformulations before giving up. If still nothing, write a findings file that honestly reports "no reliable sources found."

**Short synthesis:** If the report is under 3000 words, re-run the synthesizer with explicit instruction: "The report is too short. Expand each section with more detail, more context, more analysis from the findings files. Minimum 5000 words."

**Spelling issues:** If you detect spelling errors in findings files, instruct the synthesizer to fix them during synthesis. The final report must be error-free.

**Timeout:** If agents take more than 5 minutes, check which have completed and proceed with available findings rather than waiting indefinitely.

</error_handling>

<principles>

1. **Auto-calibrate:** Determine the right number of agents from topic complexity — don't under- or over-research.
2. **Stay lean:** Orchestrator coordinates, never researches. Don't read findings yourself — that's the synthesizer's job.
3. **Parallelize everything:** ALL research agents spawn in ONE message.
4. **Absolute paths always:** Agents need absolute paths to write files reliably.
5. **Trust agents:** Don't second-guess their output. The synthesizer handles quality integration.
6. **Status updates:** Keep the user informed at each phase (plan → spawn → complete → synthesize → deliver).
7. **Graceful degradation:** 4/5 successful angles is better than failing entirely because 1 agent had issues.
8. **Quality over speed:** A well-written, clear report is worth more than a fast, sloppy one.
9. **Language fidelity:** Match the user's language exactly. Polish input = Polish output with correct diacritics.
10. **Substance over length:** Every paragraph must carry real information. No filler, no padding, no vague generalizations.

</principles>

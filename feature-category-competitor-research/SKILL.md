---
name: feature-category-competitor-research
description: Deep competitive enterprise research — filters competitors to those with enterprise tiers, researches feature releases (last 18 months) across specified categories via WebSearch+WebFetch only (G2, Reddit, changelogs, blogs, YouTube titles), then builds an Opportunity Solution Tree (Teresa Torres) and saves per-competitor + synthesis pages to Obsidian with diff mode support.
argument-hint: "--competitors \"Jira,Asana,Monday\" --my-product \"Linear\" [--categories \"collaboration,RBAC,SSO\"] [--job-url <url>] [--since YYYY-MM-DD] [--output <path>]"
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
Orchestrate deep competitive enterprise research by:
1. Filtering the competitor list — only those with an enterprise pricing tier proceed
2. Auto-discovering all public research sources per competitor via WebSearch (no browser automation, no Playwright, no yt-dlp)
3. Resolving research categories from --categories or a job description URL
4. Spawning parallel research agents (one per competitor) using only WebSearch + WebFetch
5. Running diff mode: appending new findings to existing Obsidian pages
6. Synthesizing findings into an Opportunity Solution Tree (Teresa Torres format)
7. Saving per-competitor pages + one synthesis report to Obsidian

**Orchestrator role:** Coordinate only. Do NOT research yourself. Spawn agents, monitor, synthesize.

**Tool constraint (CRITICAL):** This skill uses ONLY WebSearch and WebFetch for all web access.
No Playwright. No yt-dlp. No browser automation. No external CLI tools.
YouTube → search video titles via WebSearch. GitHub → search via WebSearch with site: filter. G2 → search with site:g2.com.
</objective>

<language_rules>
## CRITICAL: Language Rules

Detect the language of the user's input. ALL output (agent findings, synthesis, OST) MUST be in that same language.
- Polish input → everything in Polish with correct diacritics (ą, ć, ę, ł, ń, ó, ś, ź, ż — never omit)
- English input → everything in English

Include these rules verbatim in EVERY agent prompt.
</language_rules>

<context>
User input: $ARGUMENTS

**Parameters:**
- `--competitors "A,B,C"` — Comma-separated list of competitor names (REQUIRED)
- `--my-product "Name"` — Your product to compare against (REQUIRED)
- `--categories "collab,RBAC,SSO"` — Feature categories to focus on (mutually exclusive with --job-url)
- `--job-url "https://..."` — Job description URL; skill extracts relevant categories automatically
- `--since YYYY-MM-DD` — Only surface features released after this date (default: 18 months ago)
- `--output <path>` — Output directory (overrides `$OBSIDIAN_VAULT`)

**Output path resolution (in order of precedence):**
1. `--output <path>` flag
2. `$OBSIDIAN_VAULT/Competitors/` environment variable
3. `./competitors-output/` (current directory fallback)

**Output locations (within output directory):**
- Per competitor page: `{OUTPUT_DIR}/{CompetitorName}.md`
- Synthesis + OST: `{OUTPUT_DIR}/_synthesis-{my-product-slug}-{YYYY-MM-DD}.md`
- Discovery manifest: `{OUTPUT_DIR}/_discovery-manifest.md`
</context>

<process>

<step name="parse_and_setup">
## Step 1: Parse Arguments & Setup

Extract from `$ARGUMENTS`:
- `COMPETITORS_RAW` — comma-separated string, e.g. "Jira,Asana,Monday"
- `MY_PRODUCT` — e.g. "Linear"
- `CATEGORIES_RAW` — from --categories (may be empty)
- `JOB_URL` — from --job-url (may be empty)
- `SINCE_DATE` — from --since, default 18 months ago
- `OUTPUT_FLAG` — from --output (may be empty)

Compute SINCE_DATE if not provided:
```bash
SINCE_DATE=$(date -v-18m +%Y-%m-%d 2>/dev/null || date -d '18 months ago' +%Y-%m-%d)
TODAY=$(date +%Y-%m-%d)
```

Resolve output directory:
```bash
if [[ -n "${OUTPUT_FLAG}" ]]; then
  COMPETITORS_DIR="${OUTPUT_FLAG}"
elif [[ -n "${OBSIDIAN_VAULT}" ]]; then
  COMPETITORS_DIR="${OBSIDIAN_VAULT}/Competitors"
else
  COMPETITORS_DIR="$(pwd)/competitors-output"
fi
mkdir -p "${COMPETITORS_DIR}"
```

Split COMPETITORS_RAW into array by comma, trim whitespace per entry.

Inform user:
```
Feature Category Competitor Research

Product:     {MY_PRODUCT}
Competitors: {COMPETITORS_RAW}
Window:      {SINCE_DATE} → {TODAY}
Categories:  {CATEGORIES_RAW | "extracting from job URL" | "enterprise baseline"}
Output:      {COMPETITORS_DIR}
```
</step>

<step name="resolve_categories">
## Step 2: Resolve Research Categories

### If --job-url provided:
Fetch the URL:
```
WebFetch(url=JOB_URL, prompt="List all product features, technical capabilities, and enterprise requirements mentioned in this job description as a comma-separated list of research categories. Focus on: collaboration, security, admin features, compliance, integrations, and product-specific capabilities.")
```
Map to category names. Append to enterprise baseline below.

### Enterprise baseline (always included, deduplicated):
- `SSO/SAML` — single sign-on, SAML authentication
- `RBAC` — role-based access control, permissions, user roles
- `Admin Console` — seat management, usage analytics, org admin
- `Audit Logs` — activity logs, compliance trails
- `Real-time Collaboration` — multiplayer editing, live cursors, conflict resolution
- `Team Workspaces` — shared spaces, project/team management
- `Compliance` — SOC 2, GDPR, data residency, security certs
- `Integrations` — API, webhooks, third-party tools

Plus any extras from --categories or job URL (deduplicated).

Announce:
```
Research categories resolved ({N} total):
- {category}
...
```
</step>

<step name="enterprise_filter">
## Step 3: Enterprise Tier Filter

Spawn ALL filter agents in ONE message. model=claude-haiku-4-5-20251001, run_in_background=true.

Each agent checks ONE competitor:

```
You are checking if {CompetitorName} has an enterprise pricing tier.

Steps (WebSearch + WebFetch only — no other tools):
1. WebSearch: "{CompetitorName} pricing enterprise plan"
2. WebFetch the pricing page URL from the result
3. Look for: "Enterprise" tier, "Contact Sales", custom pricing, SSO/SAML mention, admin features, or anything above "Business"/"Team" tier

Respond with EXACTLY ONE LINE — no other text:
ENTERPRISE_YES: {CompetitorName} | {pricing_page_url} | {evidence in max 15 words}
OR
ENTERPRISE_NO: {CompetitorName} | {reason in max 10 words}
```

After all complete, collect results. Keep only ENTERPRISE_YES.

Announce:
```
Enterprise Filter:

✓ Qualified: {CompetitorName} — {evidence}
✗ Filtered:  {CompetitorName} — {reason}

Proceeding with {N} competitors.
```

If zero pass: stop and inform user.
</step>

<step name="source_discovery">
## Step 4: Source Discovery

Spawn ALL discovery agents in ONE message. model=claude-haiku-4-5-20251001, run_in_background=true.

Each agent discovers sources for ONE competitor using only WebSearch:

```
Discover research sources for {CompetitorName}. Use WebSearch only — no browser tools, no yt-dlp, no Playwright.

Run these 7 searches:
1. WebSearch: "{CompetitorName} changelog release notes"
2. WebSearch: "{CompetitorName} site:github.com"
3. WebSearch: "{CompetitorName} site:youtube.com official channel"
4. WebSearch: "{CompetitorName} site:g2.com reviews"
5. WebSearch: "{CompetitorName} site:reddit.com"
6. WebSearch: "{CompetitorName} product roadmap public"
7. WebSearch: "{CompetitorName} enterprise blog announcement"

Respond with EXACTLY this block — write NOT_FOUND if not found:
---
COMPETITOR: {CompetitorName}
DOMAIN: {main domain}
CHANGELOG: {url or NOT_FOUND}
GITHUB: {github.com/org url or NOT_FOUND}
YOUTUBE_SEARCH: "{CompetitorName} enterprise site:youtube.com"
G2_SEARCH: "site:g2.com {CompetitorName} reviews"
REDDIT_SEARCH: "site:reddit.com {CompetitorName}"
ROADMAP: {url or NOT_FOUND}
BLOG: {url or NOT_FOUND}
---
Nothing else.
```

After all complete, save manifest:
Write to `{COMPETITORS_DIR}/_discovery-manifest.md`:
```markdown
# Source Discovery Manifest
Generated: {TODAY} | Competitors: {N}

{paste each competitor's discovery block}
```

Announce:
```
Source discovery complete ({N} competitors).
Starting deep research...
```
</step>

<step name="deep_research">
## Step 5: Deep Research — Parallel Agents

**CRITICAL: Spawn ALL research agents in ONE message. model=claude-haiku-4-5-20251001, run_in_background=true.**

One agent per qualified competitor. This is DEEP research — minimum 20 WebSearch calls and 12 WebFetch calls per agent. Do not cut corners. Extract specific quotes, dates, user counts, version numbers. Every claim must have a source URL.

Agent prompt template (adapt per competitor with their discovered sources):

```
You are a senior competitive intelligence analyst. Your job is to produce an exhaustive, evidence-based profile of {CompetitorName} for {MY_PRODUCT}'s product team. This is NOT a surface-level summary — you must go deep into every source type and extract specific, actionable intelligence.

Use ONLY WebSearch and WebFetch. No Playwright, no yt-dlp, no browser automation, no CLI tools.

<assignment>
Competitor: {CompetitorName}
My product (for comparison): {MY_PRODUCT}
Research window: {SINCE_DATE} → {TODAY}
Research categories (investigate each thoroughly): {RESOLVED_CATEGORIES}
Output file: {COMPETITORS_DIR}/{CompetitorName}.md

Discovered source starting points:
- Domain: {DOMAIN}
- Changelog: {CHANGELOG_URL or NOT_FOUND}
- GitHub: {GITHUB_URL or NOT_FOUND}
- Roadmap: {ROADMAP_URL or NOT_FOUND}
- Blog: {BLOG_URL or NOT_FOUND}
Use these as starting points — follow links, search variants, go deeper.
</assignment>

<writing_rules>
CRITICAL — follow strictly:
1. Language: write in {DETECTED_LANGUAGE} with correct orthography and diacritics
2. Specificity over generality: never write "users complain about X" — write "14 of the last 30 G2 reviews mention X, with users specifically describing Y scenario"
3. Direct quotes: include verbatim user quotes from reviews (with source URL). At least 6 direct quotes in the Pain Points section.
4. Dates on everything: every feature release must have a date. Every review quote must have a year.
5. Cross-reference: if G2 says one thing and the official blog says another, flag the discrepancy explicitly.
6. No filler. No vague generalizations. If you don't know, say UNKNOWN and explain what you searched.
7. Bold key conclusions. Tables must have interpretation text.
8. Count things: "3 of 5 Reddit threads mention...", "Released in 4 separate changelog entries between MM/YY and MM/YY"
</writing_rules>

<research_steps>

Execute these steps IN ORDER. Do not skip any. Track how many searches and fetches you've done.

---

**PHASE A — Changelog & Release Notes (target: 5 searches, 4 fetches)**

A1. If CHANGELOG_URL found:
    - WebFetch it fully. Extract EVERY entry that mentions: {RESOLVED_CATEGORIES}. Note exact dates and version numbers.
    - If the changelog is paginated, fetch the next page too.

A2. WebSearch: "{CompetitorName} changelog release notes enterprise {SINCE_YEAR}"
    WebFetch top 2 results. Extract all enterprise-related feature entries with dates.

A3. WebSearch: "{CompetitorName} site:{DOMAIN} blog enterprise release shipped"
    WebFetch top 2 results. Look for launch posts, "we shipped X" announcements.

A4. WebSearch: "{CompetitorName} SSO SAML launched shipped release"
    WebFetch top result. Note exact date, what was included, any limitations mentioned.

A5. WebSearch: "{CompetitorName} RBAC permissions roles admin console released {SINCE_YEAR} {TODAY_YEAR}"
    WebFetch top 2 results.

For each feature found: record name, exact date (YYYY-MM), version if available, what it does in 1 sentence, source URL.

---

**PHASE B — GitHub (target: 4 searches, 3 fetches)**

B1. If GITHUB_URL found:
    WebFetch {GITHUB_URL}/releases — extract ALL release entries from {SINCE_DATE} onwards. Note dates and feature descriptions exactly as written.

B2. WebSearch: "{CompetitorName} site:github.com releases enterprise security"
    WebFetch the releases page from the top result. Look for issues/PRs mentioning SSO, RBAC, audit, admin.

B3. WebSearch: "{CompetitorName} site:github.com issues enterprise feature request"
    WebFetch top result. Look for open feature requests that reveal what's MISSING or PLANNED.
    This is gold: upvoted GitHub issues = validated user demand.

B4. WebSearch: "{CompetitorName} site:github.com CHANGELOG.md"
    WebFetch if found. Extract enterprise-relevant entries.

For each GitHub finding: note if it's SHIPPED (release) or PLANNED (open issue/PR) and the date.

---

**PHASE C — YouTube Announcements (target: 4 searches, 0 fetches — titles + descriptions only)**

IMPORTANT: Do NOT fetch youtube.com/watch URLs. Extract intelligence from WebSearch result snippets only (titles, descriptions, publication dates). yt-dlp is NOT available.

C1. WebSearch: "{CompetitorName} enterprise features youtube {SINCE_YEAR}"
    From search snippets: extract video titles, upload dates, and any feature names mentioned in descriptions.

C2. WebSearch: "{CompetitorName} product update new feature youtube {TODAY_YEAR}"
    Same — titles and dates only from snippets.

C3. WebSearch: "{CompetitorName} admin SSO team collaboration tutorial youtube"
    Indicates which enterprise features exist (tutorials only exist for shipped features).

C4. WebSearch: "{CompetitorName} site:youtube.com enterprise team security"
    Extract video title list — each title is a signal about what features exist or were announced.

Output: list of video titles with dates. A tutorial video about SSO = confirmation SSO exists. An announcement video = note the feature name and date.

---

**PHASE D — G2 Reviews: Deep Pain Point Extraction (target: 6 searches, 5 fetches)**

Do not just read the main review page. Go deep into specific review categories and complaints.

D1. WebSearch: "site:g2.com {CompetitorName} reviews"
    WebFetch the top result. Read through individual review text. Extract:
    - Verbatim quotes from "Cons" sections (copy exact text)
    - Verbatim quotes from "What problems are you solving" sections
    - Star ratings distribution if visible
    Count how many reviews mention each pain point.

D2. WebSearch: "site:g2.com {CompetitorName} \"single sign-on\" OR \"SSO\" reviews"
    WebFetch. Extract exact quotes about SSO — present, missing, broken, partial.

D3. WebSearch: "site:g2.com {CompetitorName} \"admin\" OR \"permissions\" OR \"roles\" reviews"
    WebFetch. Extract exact quotes about admin features and access control.

D4. WebSearch: "site:g2.com {CompetitorName} \"collaboration\" OR \"real-time\" OR \"multiplayer\" reviews"
    WebFetch. Extract exact quotes.

D5. WebSearch: "{CompetitorName} reviews \"missing\" OR \"wish\" OR \"need\" enterprise features 2024 2025 2026"
    WebFetch top 2 results (may be blog posts, comparison articles, or review aggregators).

D6. WebSearch: "site:capterra.com {CompetitorName} reviews"
    WebFetch. Extract Cons sections verbatim. Note: Capterra sometimes has different user segments than G2.

For each pain point: count evidence ("mentioned in 4 G2 reviews", "2 Reddit threads"), quote at least one user verbatim, note if {CompetitorName} has acknowledged it officially.

---

**PHASE E — Reddit: Authentic User Voice (target: 6 searches, 5 fetches)**

E1. WebSearch: "site:reddit.com {CompetitorName} enterprise problem missing feature"
    WebFetch top 3 threads. Read top comments. Extract: specific feature requests, exact user language describing problems, upvote counts if visible.

E2. WebSearch: "site:reddit.com {CompetitorName} vs alternatives \"switched\" OR \"moved\" OR \"left\""
    WebFetch top 2 threads. Why are users switching away? What drove them to leave?

E3. WebSearch: "site:reddit.com {CompetitorName} SSO SAML admin permissions"
    WebFetch top 2 results. Extract exact user quotes about enterprise auth/permissions needs.

E4. WebSearch: "site:reddit.com {CompetitorName} {MY_PRODUCT} comparison"
    WebFetch top 2 results. What do users say when directly comparing the two products?

E5. WebSearch: "site:reddit.com {CompetitorName} 2025 2026 review experience"
    WebFetch top 2 results. Recent user sentiment — what's improved, what's still broken.

E6. WebSearch: "site:reddit.com {CompetitorName} team enterprise frustration"
    WebFetch top 2 results. Team-level and enterprise-specific frustrations.

For each Reddit thread: note the subreddit, post date, upvotes if visible, and pull verbatim quotes from comments (not paraphrases).

---

**PHASE F — Roadmap & Future Plans (target: 3 searches, 2 fetches)**

F1. If ROADMAP_URL found:
    WebFetch it. Extract ALL items — note which are "In Progress", "Planned", "Under Consideration".
    Enterprise-relevant items should be flagged explicitly.

F2. WebSearch: "{CompetitorName} roadmap enterprise 2025 2026 planned upcoming"
    WebFetch top 2 results. Look for: public roadmap links, interviews where founders mention upcoming features, investor update leaks, conference talks.

F3. WebSearch: "{CompetitorName} CEO CTO interview enterprise strategy {SINCE_YEAR}"
    WebFetch top result. Founders often reveal roadmap direction in interviews.
    Extract direct quotes about enterprise strategy.

---

**PHASE G — Per-Category Deep Dive (1 additional search per category)**

For EACH category in {RESOLVED_CATEGORIES}, run one targeted search:
WebSearch: "{CompetitorName} {category} feature review limitation problem"
WebFetch top result.

This catches category-specific nuance not covered in the broad searches above.
For each: note the specific implementation details (e.g., "SSO via SAML 2.0 and OIDC, but only on Enterprise tier, no self-service setup"), not just YES/NO.

---

**PHASE H — Cross-Reference & Verification**

After all searches:
- Pick 3 significant claims found in official sources (blog/changelog)
- Verify them against user reviews (G2/Reddit): do users confirm this feature works as advertised?
- Note any discrepancies: "Company claims feature X shipped in MM/YY, but 3 G2 reviews from same period still list it as missing"
- This signals: announced but buggy/incomplete, or slow rollout

</research_steps>

<diff_check>
BEFORE writing output:
1. Use Read tool to check if {COMPETITORS_DIR}/{CompetitorName}.md exists.
2. If EXISTS: read it. Note previous research_date from frontmatter. Write ONLY new findings (released/discovered after that date) under "## New Since Last Research". Update research_date to {TODAY} in frontmatter. Do not remove existing content.
3. If NOT EXISTS: write full document from scratch. Set "## New Since Last Research" to "First research run — {TODAY}".
</diff_check>

<output_format>
Write the complete file to {COMPETITORS_DIR}/{CompetitorName}.md:

```markdown
---
title: "{CompetitorName} — Enterprise Feature Research"
competitor: "{CompetitorName}"
my_product: "{MY_PRODUCT}"
research_date: "{TODAY}"
research_since: "{SINCE_DATE}"
sources_checked: {N}
searches_run: {N}
---

# {CompetitorName}: Enterprise Feature Research

**Researched:** {TODAY} | **Window:** {SINCE_DATE} → today | **Vs:** {MY_PRODUCT}
**Searches run:** {N} | **Sources fetched:** {N}

---

## Enterprise Tier Summary

{3-4 sentences: exact pricing, tier name, what's included, target customer size, key enterprise value prop. Be specific — name the price if public.}

| Field | Value |
|-------|-------|
| Enterprise tier name | {exact name, e.g. "Enterprise", "Business+", "Scale"} |
| Pricing | {exact price/user/month or "Custom — contact sales"} |
| Minimum seats | {N or Unknown} |
| Requires annual contract | {Yes / No / Unknown} |
| Key enterprise differentiator | {1-sentence: what do they sell to IT/procurement?} |

---

## Feature Release Timeline ({SINCE_DATE} → today)

List EVERY enterprise-relevant release found. If none found for a date range, say so explicitly.

| Date | Feature Name | What It Does (1 sentence) | Category | Source & URL |
|------|-------------|--------------------------|----------|--------------|
| YYYY-MM | {exact feature name as named by company} | {what it does} | {category} | {changelog/blog/GitHub} [URL] |

**Release cadence assessment:** {2-3 sentences: how many enterprise features shipped in the window? Monthly/quarterly/sparse? Which categories got the most investment? Any acceleration or deceleration visible?}

**Key observation:** {Bold: the single most significant insight from the release timeline}

---

## Category Coverage Matrix

For each category: investigate thoroughly, not just YES/NO. Include HOW it works (e.g. "SSO via SAML 2.0 and OIDC; self-service setup available on Enterprise tier; no SCIM provisioning yet").

| Category | Status | Released | Implementation Detail | User Rating (G2) | Top User Complaint |
|----------|--------|----------|-----------------------|-----------------|-------------------|
| SSO/SAML | YES/PARTIAL/NO/UNKNOWN | YYYY-MM or pre-{SINCE_DATE} or — | {how it works, limitations} | ⭐N.N (N reviews) | "{verbatim quote}" |
| RBAC | | | | | |
| Admin Console | | | | | |
| Audit Logs | | | | | |
| Real-time Collaboration | | | | | |
| Team Workspaces | | | | | |
| Compliance (SOC 2 etc.) | | | | | |
| Integrations | | | | | |
{all resolved categories — every row must be filled}

Legend: YES = fully shipped & working per reviews | PARTIAL = announced/beta/limited rollout | NO = confirmed missing (user reports or company statement) | UNKNOWN = not findable after 3+ searches

**Enterprise readiness assessment:** {3 sentences: overall score, key strength, critical gap}

---

## User Pain Points — Deep Analysis

Minimum 8 rows. Every pain point must have: evidence count, a direct verbatim quote, and the workaround users describe.

| # | Pain Point | Evidence Count | Verbatim User Quote | Workaround | Addressed by Company? | Source |
|---|-----------|---------------|--------------------|-----------|-----------------------|--------|
| 1 | {specific description} | {N G2 reviews / N Reddit threads} | "{exact quote}" | {what users do instead} | {Yes (MM/YY) / Partial / No / Unknown} | [URL] |

Sort by Evidence Count descending (most-mentioned pain at top).

**#1 unresolved pain point:** {Bold: what it is, how many users mention it, why the company hasn't fixed it (or when they plan to)}

**Pattern across pain points:** {2 sentences: is there a theme? E.g. "Admin features consistently lag feature development — 6 of 8 pain points relate to team management, not core product"}

---

## Verbatim User Quotes Bank

{6-10 direct quotes from real users — G2, Reddit, Capterra. Each with source URL and date.}

> *"[exact quote]"*
> — G2 reviewer, [role if visible], [YYYY] · [URL]

> *"[exact quote]"*
> — Reddit u/[username if public], r/[subreddit], [YYYY] · [URL]

{Continue for all quotes. These quotes should be the most revealing, specific ones — not generic praise/blame.}

---

## Discrepancies: Official Claims vs User Reality

{This section is critical. Compare what the company says about their features vs what users actually report.}

| Claim (Official) | User Reality | Verdict |
|-----------------|-------------|---------|
| "{company's claim about feature X}" — [source URL] | "{user quote contradicting or confirming}" — [source URL] | ✅ Confirmed / ⚠️ Overstated / ❌ Contradicted |

{2-3 sentences: what does this tell us about the company's relationship with enterprise customers? Are they shipping and iterating, or announcing and slow-rolling?}

---

## Upcoming / Roadmap Intelligence

{What enterprise features are confirmed coming? Distinguish between: PUBLIC ROADMAP (link), FOUNDER INTERVIEW (quote + source), COMMUNITY SIGNAL (GitHub issue upvotes).}

**Confirmed planned:**
- {Feature} — Source: {public roadmap URL / interview quote}

**Community-demanded (high signal, not confirmed):**
- {Feature} — Evidence: {N GitHub upvotes / N Reddit mentions}

**Inferred from hiring/strategy signals:**
- {Signal} — e.g. "Job postings for 'Enterprise Security Engineer' posted MM/YY suggest SOC 2 work in progress"

If roadmap is private: state that explicitly and explain what signals were used instead.

---

## Competitive Gap vs {MY_PRODUCT}

**{MY_PRODUCT} has (or likely has), {CompetitorName} lacks:**
- {specific feature}: {evidence — "0 G2 reviews mention this existing, confirmed missing per [source]"}

**{CompetitorName} has, {MY_PRODUCT} likely lacks — OPPORTUNITY:**
- {specific feature}: {what it does, when released, user reception}

**Both have (parity):**
- {feature}: {brief note on implementation differences}

---

## Sources Consulted

| # | URL | Type | What Was Extracted |
|---|-----|------|--------------------|
| 1 | [URL] | G2 reviews | {what} |
| 2 | [URL] | GitHub releases | {what} |
{all sources}

**Total:** {N} searches run | {N} pages fetched

---

## New Since Last Research

{If diff mode — "New findings since {PREVIOUS_DATE}:"}
{List only genuinely new features, quotes, or changes found this run}
{If first run: "First research run — {TODAY}"}
```
</output_format>

FINAL INSTRUCTION — CRITICAL:
After writing the file, respond with EXACTLY this one line and nothing else:
DONE: {CompetitorName}.md | {N} sources | diff:{YES/NO}
Do NOT summarize findings. Do NOT explain what you found. One line only.
```

After spawning all agents, inform user:
```
Spawned {N} parallel research agents:
{list competitors}

Deep research in progress — typically 5-10 minutes.
```
</step>

<step name="collect_results">
## Step 6: Collect & Verify Results

As background agents notify completion, track status.

After ALL complete (or reasonable timeout — proceed with what's available):
```bash
ls "{COMPETITORS_DIR}/"
```

Verify each expected `.md` file exists. Note any missing.

Inform user:
```
Research complete: {N}/{total} competitors done.
{Any failures noted}
Starting synthesis and OST...
```
</step>

<step name="synthesis_ost">
## Step 7: Synthesis Agent — Opportunity Solution Tree

Spawn ONE synthesis agent in the FOREGROUND (wait for result). model=claude-sonnet-4-6.

```
HARD CONSTRAINT: You have NO access to the web. Do NOT call WebSearch or WebFetch under any circumstances. Your ONLY inputs are the competitor files listed below. Read them all with the Read tool, then write the synthesis.

You are a senior product strategist. Read all competitor research files and produce a cross-competitor synthesis with an Opportunity Solution Tree (Teresa Torres format) for {MY_PRODUCT}.

<files_to_read>
{List ALL files with absolute paths:}
- {COMPETITORS_DIR}/_discovery-manifest.md
- {COMPETITORS_DIR}/{CompetitorName1}.md
- {COMPETITORS_DIR}/{CompetitorName2}.md
{... all competitor files}
</files_to_read>

<output_file>
{COMPETITORS_DIR}/_synthesis-{MY_PRODUCT_SLUG}-{TODAY}.md
</output_file>

<writing_rules>
1. Language: {DETECTED_LANGUAGE} with correct orthography and diacritics
2. Synthesis over concatenation: create NEW insights by combining files — not a paste job
3. Bold key conclusions. Tables must have interpretation text below them.
4. No filler phrases. Every sentence carries real information.
5. Be specific: name competitors, name features, cite evidence.
</writing_rules>

<output_structure>

```markdown
---
title: "Enterprise Competitive Synthesis — {MY_PRODUCT}"
my_product: "{MY_PRODUCT}"
competitors: [{list}]
synthesis_date: "{TODAY}"
research_window: "{SINCE_DATE} → {TODAY}"
---

# Enterprise Competitive Synthesis: {MY_PRODUCT}

**Date:** {TODAY} | **Competitors analysed:** {N} | **Window:** {SINCE_DATE} → today

---

## Executive Summary

{4-6 paragraphs. Who is moving fastest on enterprise? What is the biggest unmet user need across all competitors? Where does {MY_PRODUCT} have the clearest opportunity? Be direct — name competitors and features. No vague generalizations.}

---

## Cross-Competitor Feature Matrix

| Category | {Comp1} | {Comp2} | {Comp3} | ... | {MY_PRODUCT} (est.) |
|----------|---------|---------|---------|-----|---------------------|
| SSO/SAML | ✅ MM/YY | ⚠️ beta | ❌ | | ❓ |
| RBAC | | | | | |
| Admin Console | | | | | |
| Audit Logs | | | | | |
| Real-time Collab | | | | | |
| Team Workspaces | | | | | |
| Compliance (SOC2) | | | | | |
| Integrations | | | | | |
{all resolved categories}

Legend: ✅ Full | ⚠️ Partial/Beta | ❌ Missing | ❓ Unknown | MM/YY = release date

{3 sentences: what does this matrix reveal? Who is most complete? Biggest gap?}

---

## Aggregated User Pain Points

| Pain Point | Affects | Frequency | Severity | Current Workaround | Opportunity for {MY_PRODUCT} |
|-----------|---------|-----------|----------|--------------------|------------------------------|
| {pain} | {CompA, CompB} | High/Med/Low | 🔴/🟡/🟢 | {workaround or "none"} | {specific opportunity} |

{Minimum 10 rows, sorted by Severity descending.}

{3 sentences: which pain points are universal (3+ competitors), representing the strongest market signal?}

---

## Enterprise Feature Release Velocity ({SINCE_DATE} → today)

| Competitor | Enterprise Features Shipped | Cadence | Investment Signal |
|-----------|-----------------------------|---------|-------------------|
| {name} | {N} | Monthly/Quarterly/Sparse | Heavy/Moderate/Low |

{2 sentences: who is investing most aggressively? What does this signal about where the market is heading?}

---

## Opportunity Solution Tree

> **Framework:** Teresa Torres — Desired Outcome → Opportunities → Solutions → Experiments
> **Perspective:** {MY_PRODUCT} enterprise product strategy

### 🎯 Desired Outcome

**{ONE clear, measurable outcome}**
Example: "Increase enterprise ARR from $5M to $20M+ by converting team accounts to enterprise contracts"

{2 sentences connecting this outcome to the competitive evidence found.}

---

### 🔍 Opportunity 1: {Name}

> {1-sentence definition: a customer need, pain point, or market gap supported by the research}

**Evidence:**
- {Finding from research with competitor name}
- {User pain point with frequency data from G2/Reddit}
- {Competitive gap: who has this, who doesn't}

**Opportunity size:** High / Medium / Low
**Urgency:** {Is a competitor about to ship this? Is there current churn risk?}

#### 💡 Solution A: {Name}
{2-3 sentences describing the approach}

**Key assumption:** {What this solution assumes to be true}

**Experiments:**
- [ ] {Customer interview: e.g., "5 interviews with enterprise buyers about SSO requirement"}
- [ ] {Prototype test: e.g., "Admin console mockup — test with 3 current team-plan users"}
- [ ] {Data experiment: e.g., "Analyse churn: do accounts hitting seat limits leave faster?"}

#### 💡 Solution B: {Alternative approach}
{2-3 sentences — different mechanism, same opportunity}

**Prefer B when:** {condition}

**Experiments:**
- [ ] {Experiment}
- [ ] {Experiment}

---

### 🔍 Opportunity 2: {Name}
{same structure}

---

### 🔍 Opportunity 3: {Name}
{same structure}

---

### 🔍 Opportunity 4: {Name}
{same structure}

---

### 🔍 Opportunity 5: {Name}
{same structure}

{Add more opportunities if evidence supports — aim for 5-8 total, ranked by evidence strength}

---

## Build vs. Buy Recommendations

| Capability | Recommendation | Rationale |
|-----------|---------------|-----------|
| SSO/SAML | Buy (WorkOS / Auth0) | {reason from research} |
| Audit Logs | Build / Buy | {reason} |
| Compliance (SOC 2) | Hire consultant | {reason} |
| Real-time Collab engine | Build / Buy (Liveblocks, Yjs) | {reason} |
| Admin Console | Build | {reason} |

{2 sentences: overall build vs. buy philosophy based on competitor patterns}

---

## Strategic Recommendations for {MY_PRODUCT}

### Immediate (0-3 months)
1. **{Recommendation}** — {why, evidence from research}

### Near-term (3-6 months)
1. **{Recommendation}** — {why, evidence from research}

### Long-term (6-18 months)
1. **{Recommendation}** — {why, evidence from research}

---

## Research Limitations

- Window: {SINCE_DATE} → {TODAY}
- Sparse data: {competitors where sources were limited}
- OST assumptions to validate first: {list key assumptions}
- Suggested follow-up: {what additional research would strengthen the OST}

---

## Sources Index

{Consolidated list of all URLs cited across all competitor files, deduplicated}
```
</output_structure>

CRITICAL:
- Read ALL competitor files before writing a single word of the synthesis
- Aggregated Pain Points table: minimum 10 rows, sorted by severity
- OST: minimum 5 Opportunities, each with minimum 2 Solutions and 3 Experiments per solution
- Feature matrix must cover every resolved category for every competitor
- Write the COMPLETE document — no truncation
- Use the Write tool to save to output file
```
</step>

<step name="obsidian_update">
## Step 8: Done

All competitor files and synthesis saved to `{COMPETITORS_DIR}`. No additional index steps needed.
</step>

<step name="deliver">
## Step 9: Deliver

```
Feature Category Research — Complete!

Qualified competitors (enterprise tier): {N}
  ✓ {list}

Research window: {SINCE_DATE} → {TODAY}
Categories: {N}
Diff mode: {N} updated | {M} new

Files:
  {COMPETITORS_DIR}/
  ├── _discovery-manifest.md
  ├── _synthesis-{MY_PRODUCT_SLUG}-{TODAY}.md
  └── {CompetitorName}.md  (× N)

---

[Preview: Desired Outcome + top 3 Opportunities from the OST]

---

Full synthesis → {COMPETITORS_DIR}/_synthesis-{MY_PRODUCT_SLUG}-{TODAY}.md
```
</step>

</process>

<error_handling>
- **G2/Capterra blocks WebFetch:** Use snippet text from WebSearch result — it often contains review excerpts. Note limitation in sources.
- **YouTube pages not fetchable:** Use only WebSearch result titles and descriptions. Never attempt to fetch youtube.com watch pages or download video content. Note: yt-dlp is NOT available in this skill.
- **GitHub private or unavailable:** Skip, compensate with extra changelog/blog searches.
- **Zero competitors pass enterprise filter:** Inform user. Suggest verifying competitor names or that these products may not have enterprise tiers.
- **Agent failure:** Proceed with available competitor files. Synthesiser notes gaps.
- **Roadmap page requires login:** Note as "private roadmap" in the competitor file. Use blog/Twitter/Reddit for roadmap signals instead.
- **Large competitor list (>8):** Process filter + discovery for all in parallel. Run research in batches of 4 if needed to avoid context overload.
</error_handling>

<principles>
1. **WebSearch + WebFetch only** — no Playwright, no yt-dlp, no browser automation, no external CLI tools ever
2. **YouTube = search only** — extract video titles and descriptions from WebSearch snippets; never fetch watch pages
3. **Enterprise filter first** — never waste deep research on non-enterprise products
4. **All parallel in one message** — filter agents together, discovery agents together, research agents together
5. **Haiku collects, Sonnet synthesises** — non-negotiable model assignment
6. **Diff by default** — always read existing pages before writing; never overwrite previous research
7. **Pain points are primary** — the pain points table in every competitor file feeds the OST directly
8. **OST is the deliverable** — competitor pages are inputs; the OST drives product decisions
9. **Absolute paths everywhere** — agents need exact paths to read and write reliably
10. **DONE: one-liner only** — research agents respond with one line; no summaries back to orchestrator
</principles>

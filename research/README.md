# research

Deep research orchestrator. Spawns parallel web research agents to investigate a topic from multiple angles, then synthesizes findings into a comprehensive long-form report with references.

## What it does

1. Analyzes the topic to determine research complexity (auto-selects agent count and depth)
2. Designs distinct research angles (state of the art, evidence, historical context, competitive landscape, etc.)
3. Spawns parallel Haiku agents (one per angle) — each does WebSearch + WebFetch
4. Collects all findings from markdown files
5. Spawns a Sonnet synthesis agent to produce the final report
6. Saves everything to your output directory and optionally ingests into a wiki

## Prerequisites

No external tools required.

## Configuration

| Variable | Description | Required |
|----------|-------------|----------|
| `OBSIDIAN_VAULT` | Path to your Obsidian vault root | No (uses `./research-output/` fallback) |

## Arguments

| Argument | Description | Default |
|----------|-------------|---------|
| `<topic>` | The research topic or question | Required |
| `--angles N` | Override auto-detected angle count (max 12) | auto |
| `--depth shallow\|normal\|deep` | Research depth per agent | auto |
| `--output <path>` | Output directory | `$OBSIDIAN_VAULT/raw/Research` or `./research-output` |
| `--no-obsidian` | Skip Obsidian, save to `./research-output` | off |
| `--fast` | Quick mode: 2 agents, shallow depth | off |

## Output

```
{output-dir}/[YYYY-MM-DD] Topic Name/
├── RESEARCH-PLAN.md     # Complexity score + angle breakdown
├── REPORT.md            # Final synthesized report
└── findings/
    ├── state-of-the-art.md
    ├── competitive-landscape.md
    └── ...              # One file per research angle
```

## Model assignment

- Research agents: `claude-haiku-4-5` (parallel, fast, cost-efficient)
- Synthesis agent: `claude-sonnet-4-6` (one, high quality)

## Language support

Detects input language automatically. Polish input → Polish output with correct diacritics. English input → English output.

## Wiki ingest

After delivering the report, the skill creates wiki pages (topic, entities, concepts) if `$OBSIDIAN_VAULT` is set and `{VAULT}/CLAUDE.md` exists. Skipped automatically with `--no-obsidian`.

## Complexity scoring

Topics are scored 5–15 across: breadth, controversy, recency, interdisciplinarity, stakeholder count. Score determines agent count (2–6 agents) and report length (1500–6000 words).

## Examples

```
/research the impact of LLMs on software engineering productivity
/research --fast current state of quantum computing
/research --depth deep history and future of open source licensing
/research --angles 6 competitive landscape of AI coding assistants
/research --output ~/Desktop/reports the economics of remote work
/research --no-obsidian what is retrieval-augmented generation
```

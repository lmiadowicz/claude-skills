# feature-category-competitor-research

Deep competitive enterprise research skill. Filters competitors to those with enterprise pricing tiers, researches feature releases across specified categories, and synthesizes findings into an Opportunity Solution Tree (Teresa Torres format).

## What it does

1. Filters your competitor list — only those with an enterprise tier proceed
2. Auto-discovers research sources per competitor (changelog, GitHub, G2, Reddit, YouTube, roadmap)
3. Spawns parallel research agents (one per competitor) using WebSearch + WebFetch only
4. Runs diff mode — appends new findings to existing pages rather than overwriting
5. Synthesizes across all competitors into a cross-competitor feature matrix and OST
6. Saves per-competitor pages + synthesis report to your output directory

## Prerequisites

No external CLI tools required. Uses only WebSearch and WebFetch.

## Configuration

| Variable | Description | Required |
|----------|-------------|----------|
| `OBSIDIAN_VAULT` | Path to your Obsidian vault root | No (uses `./competitors-output/` fallback) |

## Arguments

| Argument | Description | Required |
|----------|-------------|----------|
| `--competitors "A,B,C"` | Comma-separated list of competitor names | Yes |
| `--my-product "Name"` | Your product name (for comparison framing) | Yes |
| `--categories "collab,SSO"` | Feature categories to focus on | No (uses enterprise baseline) |
| `--job-url <url>` | Job description URL — extracts categories automatically | No |
| `--since YYYY-MM-DD` | Only surface features after this date | No (default: 18 months ago) |
| `--output <path>` | Override output directory | No |

`--categories` and `--job-url` are mutually exclusive.

## Output

```
{output-dir}/
├── _discovery-manifest.md          # Source URLs found per competitor
├── _synthesis-{product}-{date}.md  # Cross-competitor matrix + OST
├── Jira.md                         # Per-competitor research
├── Asana.md
└── Monday.md
```

## Examples

```
/feature-category-competitor-research --competitors "Jira,Asana,Monday" --my-product "Linear"
/feature-category-competitor-research --competitors "Notion,Coda,Confluence" --my-product "Nuclino" --categories "collaboration,offline,permissions"
/feature-category-competitor-research --competitors "Figma,Sketch,Adobe XD" --my-product "Penpot" --job-url https://example.com/jobs/product-manager
/feature-category-competitor-research --competitors "Datadog,New Relic" --my-product "Grafana" --since 2024-01-01 --output ~/Desktop/research
```

## Notes

- Enterprise filter runs first — competitors without enterprise tiers are excluded automatically
- Research agents use Haiku for data collection, Sonnet for synthesis (preserves quota)
- Diff mode: re-running on existing output appends new findings under "New Since Last Research"
- YouTube: extracts intelligence from search result titles/descriptions only — never fetches watch pages
- If `$OBSIDIAN_VAULT` is set and contains a `CLAUDE.md`, the wiki index and log are updated automatically

# linkedin-article-thumbnail

Generate clean editorial-style SVG graphics for articles, LinkedIn posts, and diagrams. Think Bloomberg Businessweek covers, not stock photo libraries.

## What it does

1. Reads your article/post content to identify the core idea and key terms
2. Picks a layout pattern automatically (or uses your override)
3. Generates an SVG with editorial magazine aesthetics (white bg, black type, one accent color)
4. Exports a PNG to your output directory
5. Asks if you want adjustments

## Supported output types

| Type | Canvas | Use for |
|------|--------|---------|
| `thumbnail` | 680×400 | Article cover images |
| `linkedin-bg` | 1584×396 | LinkedIn profile background banner |
| `linkedin-post` | 1200×628 | LinkedIn post graphics |
| `diagram` | 720×auto | In-text flow diagrams |

## Layout patterns

| Pattern | Best for |
|---------|---------|
| A — Split Column | Before/after, two concepts, contrast |
| B — Hero Statement | Single bold opinion, manifesto |
| C — Data Callout | Stats, research-backed content |
| D — Terminal | Technical, developer, API articles |

## Prerequisites

One of:
- `rsvg-convert` — `brew install librsvg` (recommended, best quality)
- `inkscape` — alternative converter

If neither is installed, the skill installs `librsvg` via Homebrew automatically.

## Configuration

| Variable | Description | Required |
|----------|-------------|----------|
| `OBSIDIAN_VAULT` | Path to your Obsidian vault root | No (uses `./blog-assets/` fallback) |

## Arguments

| Argument | Description | Default |
|----------|-------------|---------|
| `<content>` | Article text, post text, or concept description | Required |
| `--type` | Output mode: `thumbnail`, `linkedin-bg`, `linkedin-post`, `diagram` | auto-detect |
| `--accent <hex>` | Override accent color | `#FFE033` (yellow) |
| `--pattern A\|B\|C\|D` | Force a layout pattern | auto-pick |
| `--output <path>` | Override PNG export directory | `$OBSIDIAN_VAULT/blog-assets` or `./blog-assets` |

## Examples

```
/editorial-thumbnail Here's my article about how AI agents break traditional UX flows...
/editorial-thumbnail --type linkedin-bg Jane Smith · Product Lead · AI Builder
/editorial-thumbnail --type linkedin-post --accent #0077B5 Key insight from my latest article on agentic UX
/editorial-thumbnail --type diagram Show a flow: User Input → Agent Router → Tool Call → Result → User
/editorial-thumbnail --pattern C --accent #FF4040 73% of AI demos fail in production
/editorial-thumbnail --output ~/Desktop My article about the future of design tools
```

## Design rules

- White background always — no dark mode, no gradients, no shadows
- One accent color only (default yellow `#FFE033`)
- Two typefaces max: Georgia serif for hero text, sans-serif for labels
- Generous whitespace — never crowded
- Bottom tagline ≤5 words, ALL CAPS

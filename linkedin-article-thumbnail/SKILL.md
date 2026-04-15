---
name: linkedin-article-thumbnail
description: >
  Generate clean editorial-style graphics in SVG. Use this skill whenever the user wants a
  cover image, article graphic, LinkedIn post visual, thumbnail, header image, LinkedIn
  background, in-text diagram, or any visual that accompanies written content. Triggers on:
  make a graphic for this article, create a thumbnail, give me a cover image, create a diagram,
  grafika do artykulu, thumbnail do posta, obrazek na LinkedIn, tło LinkedIn, diagram do tekstu,
  or any request for a visual accompanying written content or explaining a concept.
argument-hint: "<article/post content or concept> [--type thumbnail|linkedin-bg|linkedin-post|diagram] [--accent <hex>] [--pattern A|B|C|D] [--output <path>]"
allowed-tools:
  - Read
  - Write
  - Bash
  - Edit
---

<objective>
Generate clean, editorial-style SVG graphics for articles, LinkedIn, and in-text diagrams.
Supports four output modes: article thumbnail, LinkedIn background banner, LinkedIn post graphic,
and in-text diagram. Output raw SVG wrapped in a fenced code block — no prose inside the SVG.
After rendering, ask the user if they want to adjust the tagline, layout, or accent color.
</objective>

<context>
User input: $ARGUMENTS

**Flags (all optional):**
- `--type thumbnail|linkedin-bg|linkedin-post|diagram` — output mode (default: auto-detect from content)
- `--accent <hex>` — override accent color (default: `#FFE033`)
- `--pattern A|B|C|D` — force a layout pattern (default: auto-pick)

**Auto-detect rules (when --type is omitted):**
- Mentions "background", "banner", "tło", "banner LinkedIn" → `linkedin-bg`
- Mentions "post", "grafika do posta", "square", "post graphic" → `linkedin-post`
- Mentions "diagram", "flow", "schemat", "process", "steps", arrows → `diagram`
- Everything else → `thumbnail`
</context>

<style_dna>
The aesthetic is: **editorial magazine meets developer tooling**. Think Bloomberg Businessweek covers, not stock photo libraries.

Core rules:
- White background always (`#ffffff`)
- Black (`#111`) for all type and line elements
- One accent color only — default **yellow `#FFE033`** unless overridden
- Flat, zero gradients, zero shadows
- Two typefaces max: `Georgia, serif` for display/hero text; `sans-serif` for labels and UI; `monospace` for code/terminal
- Generous whitespace — never crowded
- Bottom tagline in small caps / letter-spacing — short and punchy (≤5 words, ALL CAPS)
- No rotated text, no emoji
- Every SVG: `role="img"` with `<title>` and `<desc>` on root element
- All text stays within the safe area — never overflows the canvas
- SVG `<text>` never wraps — keep all labels short enough for one line
</style_dna>

<canvas_specs>

## Thumbnail (article cover)
```
viewBox: 0 0 680 400
Safe area: x=60–620, y=40–385
Bottom rule: <line x1="60" y1="385" x2="620" y2="385" stroke="#111" stroke-width="1"/>
Tagline: font-size="10", y=396, letter-spacing="1"
```

## LinkedIn Background Banner
```
viewBox: 0 0 1584 396
Safe area: x=80–1504, y=30–370
Bottom rule: <line x1="80" y1="375" x2="1504" y2="375" stroke="#111" stroke-width="1"/>
Note: left ~300px may be obscured by profile photo on desktop — keep primary content x>340
Tagline: font-size="11", y=389, letter-spacing="2"
```

## LinkedIn Post Graphic
```
viewBox: 0 0 1200 628
Safe area: x=80–1120, y=50–580
Bottom rule: <line x1="80" y1="590" x2="1120" y2="590" stroke="#111" stroke-width="1"/>
Tagline: font-size="12", y=612, letter-spacing="2"
```

## In-text Diagram
```
viewBox: 0 0 720 auto  (height varies — set to fit content, minimum 200)
Safe area: x=40–680
No bottom tagline unless user requests one
Use clean flow arrows, boxes, and labels only
```

</canvas_specs>

<layout_patterns>
Use these patterns for Thumbnail and LinkedIn Post. LinkedIn Background uses a dedicated layout. Diagrams use their own rules.

### Pattern A — Split Column (default for contrast / before-vs-after / two concepts)
- Left column: concept A with label + icon/UI mockup
- Right column: concept B with label + icon/UI mockup
- Vertical dashed divider in center
- Hero text + subtitle top-left
- Yellow accent bar left of hero text (14px wide rect)
- Bottom rule + tagline

### Pattern B — Hero Statement (opinion pieces / single strong idea)
- Giant serif text filling 60–70% of canvas (core idea or acronym)
- Supporting subtitle in small tracked sans
- Yellow used as underline, highlight rect, or left bar accent
- Minimal supporting element bottom-right (small diagram, number, or icon)
- Bottom rule + tagline

### Pattern C — Data / Stat Callout (research-backed / numbers-heavy)
- Large number or stat as hero (serif, very large)
- Yellow highlight rect behind or under the number
- Brief context label
- Supporting secondary stat or visual bottom half
- Bottom rule + tagline

### Pattern D — Terminal / Code (technical / API / developer articles)
- Dark terminal block (`#111` bg) as dominant element
- Yellow monospace text for key lines
- Gray (`#888`) for secondary lines
- White background surrounding terminal
- Hero label above or beside terminal

### LinkedIn Background Layout
- Full-width horizontal composition
- Left ~340px: simple geometric accent element (large yellow rect, circle, or minimal mark)
- Center x=380–900: name / role / tagline in large serif + tracked sans
- Right x=960–1480: 2–3 short bullet points or key phrases (sans-serif, small, tracked)
- Bottom rule at y=375 + tagline
- Keep it airy — this is a banner, not a poster

</layout_patterns>

<reusable_elements>

### Yellow accent bar (left of hero text)
```svg
<rect x="60" y="60" width="14" height="120" fill="#FFE033"/>
```

### Hero text (large serif)
```svg
<text x="90" y="120" font-family="Georgia, serif" font-size="72" font-weight="700" fill="#111" letter-spacing="-2">WORD</text>
```

### Section label (small tracked caps)
```svg
<text x="91" y="200" font-family="sans-serif" font-size="11" font-weight="500" fill="#888" letter-spacing="2">LABEL</text>
```

### Horizontal rule
```svg
<line x1="91" y1="172" x2="460" y2="172" stroke="#111" stroke-width="1"/>
```

### Browser window mockup (human/UI side)
```svg
<rect x="91" y="212" width="160" height="110" rx="4" fill="#ffffff" stroke="#111" stroke-width="1.5"/>
<rect x="91" y="212" width="160" height="22" rx="4" fill="#111"/>
<rect x="91" y="226" width="160" height="8" fill="#111"/>
<circle cx="105" cy="223" r="4" fill="#ffffff"/>
<circle cx="120" cy="223" r="4" fill="#ffffff"/>
<circle cx="135" cy="223" r="4" fill="#ffffff"/>
```

### Terminal mockup (agent/code side)
```svg
<rect x="370" y="212" width="240" height="110" rx="4" fill="#111"/>
<text x="386" y="240" font-family="monospace" font-size="11" fill="#FFE033">→ key line here</text>
<text x="386" y="258" font-family="monospace" font-size="11" fill="#888">secondary line</text>
```

### Status icon — success
```svg
<circle cx="103" cy="343" r="14" fill="#111"/>
<polyline points="96,343 101,349 111,336" fill="none" stroke="#ffffff" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round"/>
```

### Status icon — failure
```svg
<circle cx="383" cy="343" r="14" fill="#111"/>
<line x1="377" y1="337" x2="389" y2="349" stroke="#ffffff" stroke-width="2.5" stroke-linecap="round"/>
<line x1="389" y1="337" x2="377" y2="349" stroke="#ffffff" stroke-width="2.5" stroke-linecap="round"/>
```

### Content placeholder lines (inside mockups)
```svg
<rect x="107" y="248" width="80" height="6" rx="2" fill="#ddd"/>
<rect x="107" y="262" width="120" height="6" rx="2" fill="#ddd"/>
<rect x="107" y="290" width="100" height="6" rx="2" fill="#FFE033"/>
```

### Vertical dashed divider
```svg
<line x1="340" y1="185" x2="340" y2="375" stroke="#ddd" stroke-width="1" stroke-dasharray="4 4"/>
```

### Flow arrow (for diagrams)
```svg
<line x1="200" y1="100" x2="300" y2="100" stroke="#111" stroke-width="1.5" marker-end="url(#arrow)"/>
<defs>
  <marker id="arrow" markerWidth="8" markerHeight="8" refX="6" refY="3" orient="auto">
    <path d="M0,0 L0,6 L8,3 z" fill="#111"/>
  </marker>
</defs>
```

### Process box (for diagrams)
```svg
<rect x="80" y="80" width="120" height="44" rx="3" fill="#fff" stroke="#111" stroke-width="1.5"/>
<text x="140" y="107" font-family="sans-serif" font-size="12" fill="#111" text-anchor="middle">Step label</text>
```

### Decision diamond (for diagrams)
```svg
<polygon points="200,60 260,90 200,120 140,90" fill="#fff" stroke="#111" stroke-width="1.5"/>
<text x="200" y="94" font-family="sans-serif" font-size="11" fill="#111" text-anchor="middle">Yes/No?</text>
```

### Highlight rect (accent behind text)
```svg
<rect x="88" y="130" width="180" height="28" fill="#FFE033"/>
<text x="91" y="150" font-family="Georgia, serif" font-size="22" font-weight="700" fill="#111">Key phrase</text>
```

### LinkedIn banner name block (center zone)
```svg
<text x="380" y="160" font-family="Georgia, serif" font-size="56" font-weight="700" fill="#111" letter-spacing="-1">Full Name</text>
<text x="382" y="196" font-family="sans-serif" font-size="16" fill="#888" letter-spacing="3">ROLE · COMPANY</text>
```

### LinkedIn banner bullet list (right zone)
```svg
<text x="960" y="140" font-family="sans-serif" font-size="14" fill="#111">→ Bullet one</text>
<text x="960" y="168" font-family="sans-serif" font-size="14" fill="#111">→ Bullet two</text>
<text x="960" y="196" font-family="sans-serif" font-size="14" fill="#111">→ Bullet three</text>
```

</reusable_elements>

<diagram_rules>

In-text diagrams follow the same style DNA but focus on clarity over aesthetics:

**Allowed diagram types:**
- Linear flow (A → B → C → D)
- Branching flow / decision tree
- Before/after comparison (two parallel columns)
- Layered stack (boxes stacked vertically showing system layers)
- Loop / cycle (circular or semi-circular with arrows)
- Matrix / 2x2 grid (quadrant chart)

**Rules:**
- All boxes same height (44px), width varies by label length (minimum 100px)
- Consistent row spacing: 80px between rows of boxes
- Arrows: `stroke="#111"`, `stroke-width="1.5"`, arrowhead marker
- Labels inside boxes: `font-size="12"`, `text-anchor="middle"`, centered vertically
- Section headers above groups: `font-size="11"`, `letter-spacing="2"`, `fill="#888"`
- Yellow accent: use it on one key box, one arrow, or one label — not everywhere
- Fit content: calculate viewBox height dynamically based on number of rows
- No decorative elements — every element must convey information
- Add a short title at top: `font-size="14"`, `font-weight="700"`, `fill="#111"`

**Height formula:**
```
height = 60 (top padding) + (rows * 80) + 60 (bottom padding)
Minimum: 200
```

</diagram_rules>

<process>

## Step 1: Parse Arguments and Detect Mode

Extract from `$ARGUMENTS`:
- **Type:** from `--type` flag, or auto-detect (thumbnail / linkedin-bg / linkedin-post / diagram)
- **Accent:** from `--accent` flag (default: `#FFE033`)
- **Pattern:** from `--pattern` flag (default: auto-pick in Step 3)
- **Content:** everything that isn't a flag — the article/post text or concept description

## Step 2: Read the Content

If the user provided article or post text, read it. Identify:
- The **core tension or single idea** (usually in first 2–3 paragraphs or the headline)
- **Key terms** (2–5 words max) for the hero text
- **Tagline candidate** (≤5 words, punchy, captures the thesis)
- Any **stats or numbers** that are central to the argument
- Whether it is **technical** (code/API/developer) or **conceptual/strategic**

For diagrams: identify the **steps, nodes, or relationships** the user wants visualized.

## Step 3: Pick Layout Pattern (skip for linkedin-bg and diagram)

| Content type | Pattern |
|---|---|
| Contrast / before-vs-after / two concepts | A — Split Column |
| Single bold opinion / manifesto | B — Hero Statement |
| Stats or research-led | C — Data Callout |
| Technical / developer / API | D — Terminal |

For `linkedin-bg`: use the LinkedIn Background Layout from layout_patterns.
For `diagram`: use diagram_rules.

## Step 4: Compose the SVG

Use the correct canvas from canvas_specs. Build the SVG using reusable_elements as building blocks. Follow these constraints:
- Keep all text short — no SVG text element should be wider than the safe area
- Use only the accent color in 1–2 places
- Whitespace is intentional — resist the urge to fill empty space
- Bottom rule + tagline on all types except plain diagrams
- Include `<title>` and `<desc>` for accessibility

## Step 5: Output SVG in chat

Output the SVG inside a fenced code block so the user can preview it:

```svg
<svg ...>
  ...
</svg>
```

One short line before the block (e.g., "Pattern A thumbnail — accent `#FFE033`").

## Step 6: Export PNG to Obsidian vault

Save the SVG to a temp file and convert it to PNG, then move the PNG to the Obsidian blog-assets folder.

**Output path resolution (in order of precedence):**
1. `--output <path>` flag (if provided)
2. `$OBSIDIAN_VAULT/blog-assets/` environment variable
3. `./blog-assets/` (current directory fallback)

**Filename convention:** `<YYYY-MM-DD>-<slug>-<type>.png`
- `slug` = first 4 words of the title/tagline, lowercased, hyphens (e.g., `ai-breaks-ux-flows`)
- `type` = `thumbnail` | `linkedin-bg` | `linkedin-post` | `diagram`
- `YYYY-MM-DD` = today's date

**Export steps:**

```bash
# 1. Resolve and ensure output dir exists
if [[ -n "${OUTPUT_FLAG}" ]]; then
  BLOG_ASSETS="${OUTPUT_FLAG}"
elif [[ -n "${OBSIDIAN_VAULT}" ]]; then
  BLOG_ASSETS="${OBSIDIAN_VAULT}/blog-assets"
else
  BLOG_ASSETS="$(pwd)/blog-assets"
fi
mkdir -p "${BLOG_ASSETS}"

# 2. Write SVG to temp file
TMPSVG=$(mktemp /tmp/editorial-XXXXXX.svg)
cat > "${TMPSVG}" << 'SVGEOF'
<paste the full SVG here>
SVGEOF

# 3. Convert SVG → PNG
# Try rsvg-convert first (best quality), fall back to qlmanage (macOS built-in)
OUTPUT_PNG="${BLOG_ASSETS}/<filename>.png"

if command -v rsvg-convert &>/dev/null; then
  rsvg-convert -w 1360 "${TMPSVG}" -o "${OUTPUT_PNG}"
elif command -v inkscape &>/dev/null; then
  inkscape --export-type=png --export-width=1360 --export-filename="${OUTPUT_PNG}" "${TMPSVG}"
else
  echo "Installing rsvg-convert via brew..."
  brew install librsvg
  rsvg-convert -w 1360 "${TMPSVG}" -o "${OUTPUT_PNG}"
fi

rm -f "${TMPSVG}"
```

**Export widths by type:**
| Type | PNG width |
|---|---|
| thumbnail | 1360px |
| linkedin-bg | 3168px (2x for retina) |
| linkedin-post | 1200px |
| diagram | 1440px |

**Use the Write tool** to write the SVG to the temp file (more reliable than heredoc for complex SVGs).

After export, confirm to user:
```
Saved: <OUTPUT_PNG>
```

## Step 7: Follow-up

After the export confirmation, ask one short question:
> Want to adjust the tagline, swap to a different pattern, or change the accent color?

</process>

<common_mistakes>
- Don't put too much text inside the SVG — sparse is better
- Don't use more than one accent color
- Don't use gradients or shadows
- Don't make the tagline more than 5 words
- Don't crowd elements — whitespace is the design
- SVG `<text>` never wraps — keep all text short enough for one line
- Always verify `x + textLength` stays within the safe area right boundary
- For LinkedIn Background: don't put important content in x=0–340 (hidden by profile photo)
- For diagrams: don't mix diagram types in one graphic — pick one layout and commit to it
- Don't add decorative elements that carry no meaning
</common_mistakes>

<examples>

## Example invocations

```
/editorial-thumbnail Here's my article about how AI agents break traditional UX flows...
/editorial-thumbnail --type linkedin-bg Jane Smith · Product Lead · AI Builder
/editorial-thumbnail --type linkedin-post --accent #0077B5 Key insight from my latest article on agentic UX
/editorial-thumbnail --type diagram Show a flow: User Input → Agent Router → Tool Call → Result → User
/editorial-thumbnail --pattern C --accent #FF4040 73% of AI demos fail in production
```

</examples>

# Poster Studio: Final Implementation Plan

Audience: a coding agent. Read this whole file before writing code. Follow the phases in order. Do not skip the review steps.

## 0. What we are building

A Node.js system that turns text written in a small markdown-based DSL (by an LLM or a human) into a finished PNG poster. News items go in, a clean poster comes out. A minimal web editor sits on top for manual use. A CLI handles automation.

Core idea: the LLM never designs anything. It only writes content in the DSL. All design lives in code (layouts, palettes, type, components), so output is consistent and quality is controlled by a test harness.

Non-goals for now: video, animation, multi-page carousels, user accounts.

## 1. Skills and workflow the agent must use

### 1.1 Elephant-Goldfish Model (EGM)

Repo: https://github.com/google/elephant-goldfish-model (site: https://egm.dev, Apache-2.0)

Note: despite the name, this is NOT a machine learning model. It is an open-source plugin of agent skills for designing and reviewing software. The "elephant" (you, with full project context) drafts the spec. "Goldfish" reviewers are fresh subagents that see only the document and referenced source, with no earlier conversation, so they expose hidden assumptions. The human approves before implementation.

Install (pick the harness you run in):

```
# Claude Code
claude plugin marketplace add google/elephant-goldfish-model
claude plugin install egm@egm

# Codex CLI
codex plugin marketplace add google/elephant-goldfish-model
codex plugin add egm@egm

# Antigravity CLI
agy plugin install https://github.com/google/elephant-goldfish-model
```

Start a new session after installing. In Claude Code and Antigravity, commands use the `/egm:` prefix (for example `/egm:egm-brainstorm`). Goldfish review needs a session that can create fresh subagents. If yours cannot, EGM stops the review, so tell the human.

How we use it in this project:

1. Before each phase in section 13, run `/egm:egm-brainstorm` on that phase if its design is unclear. It writes `concepts_brief.md`.
2. Run `/egm:egm-draft` to produce `design_draft.md` for the phase. Save approved specs under `docs/designs/`.
3. Run `/egm:egm-review <path>` on the draft. Address findings with `/egm:egm-resolve`.
4. Only then implement.
5. Once a month, or after big changes, run `/egm:egm-drift-report docs/designs/` to catch drift between specs and code.

Rule: no phase starts coding until its spec has passed an EGM review and the human has approved it. A review does not replace tests (see the harness in section 10).

### 1.2 Emil Kowalski skills (editor UI)

```
npx skills@latest add emilkowalski/skills
```

Load these skills before touching any editor UI code (section 11): component polish, interaction details, motion. If a skill's guidance conflicts with the minimal UI rules in section 11, section 11 wins.

### 1.3 Design skills for the poster look

Optionally add Anthropic's `frontend-design` skill when building poster components. It is a baseline against generic AI aesthetics. The real protection against slop here is the constraint system in section 8, not taste prompts.

## 2. Architecture

TypeScript monorepo (pnpm workspaces). Node 22+. ESM only.

```
packages/
  dsl/         parse markdown+directives -> typed JSON, validate (zod)
  assets/      icon + logo ingestion, SVG normalization, manifests
  resolver/    ONNX embeddings, keyword scoring, fallback ladder
  layout/      grids, slot math, text fitting, auto layout picker
  theme/       palettes, font pairs + bundled Google Fonts (fonts/), tokens, contrast + banned-pattern rules
  components/  pill, card, stat, chip, badge, logo, icon, footer ... (emit HTML+CSS)
  render/      JSON -> HTML -> PNG (Playwright first, Satori later)
  harness/     golden tests, visual diff, QA checks, decision logs
apps/
  cli/         render, batch, news, golden commands
  web/         minimal editor (Vite + React)
docs/designs/  approved specs (EGM output)
```

Pipeline:

```
DSL text -> parse -> PosterDoc (JSON) -> validate -> resolve (logos/icons/layout/palette)
         -> layout (slot boxes + text fit) -> components -> HTML -> PNG -> QA -> save + log
```

Rules:
- `PosterDoc` JSON is the single source of truth. The DSL is just a friendly way to write it. The editor can edit JSON directly.
- Every stage is a pure function with typed input and output, so each can be tested alone.
- Same input plus same pinned versions must give byte-identical output (see determinism, section 6.5).

## 3. Data contract

Sketch (finalize with zod in `packages/dsl`):

```ts
PosterDoc = {
  meta: { layout: "auto" | LayoutId, palette: "auto" | PaletteId, size: "1080x1350" | "1080x1080" | "1080x1920", lang?: string },
  blocks: Block[]
}
Block =
  | { type: "title", text: Inline[], eyebrow?: string }
  | { type: "item", text: Inline[], meta?: Inline[], logo?: AssetRef | "auto" | "none", icon?: AssetRef | "auto" | "none", tone?: Tone }
  | { type: "stat", value: string, label: Inline[], delta?: string, logo?: AssetRef | "auto" | "none" }
  | { type: "chip", text: string, icon?: AssetRef | "auto" }
  | { type: "note", text: Inline[] }
  | { type: "footer", text: string }
Inline = { t: "text", v: string } | { t: "company", name: string, ticker?: string, logo?: AssetRef } | { t: "icon", name: string } | { t: "em", v: string }
AssetRef = { id: string, kind: "logo" | "icon", score?: number, reason?: string }
```

After resolve, every `"auto"` becomes a concrete `AssetRef` with `score` and `reason` (for the decision log).

## 4. The DSL

Decision: markdown with directives, parsed by existing libraries. Do not write a grammar from scratch. We do need a small parser layer, but it is mostly glue.

Stack: `unified` + `remark-parse` + `remark-frontmatter` + `remark-directive`, plus our own transformer from mdast to `PosterDoc`, then zod validation.

Syntax:
- Front matter (YAML): global settings, all optional.
- `# Heading` becomes a title block. `::eyebrow[text]` sets the small label above it.
- Container directives for blocks: `:::item`, `:::stat`, `:::chips`, `:::note`.
- Inline text directive for companies: `:company[Nvidia]{ticker=NVDA}` renders as logo + name inline.
- Inline icon: `:icon[rocket]`.
- Attributes use directive attribute syntax: `{logo=auto icon=none tone=positive}`.
- Anything omitted uses defaults. Everything `auto` is resolved by the resolver.

Example:

```md
---
layout: auto
palette: auto
size: 1080x1350
---

# Tech Briefing
::eyebrow[19 Nov 2026]

:::item
:company[Nvidia] shares jump on new AI chip orders
:::

:::item{icon=auto}
EU finalizes new rules for AI model transparency
:::

:::stat{value="+4.2%" logo=auto}
:company[Nvidia] one-day move
:::

::footer[Source: daily tech feed]
```

Parser requirements:
- Preserve source positions on every node, so errors point to line and column.
- Errors are structured: `{ code, message, line, col, hint }`. Hints are written so an LLM can fix its own output (for example "item has no text; add a line under :::item").
- Unknown directive names or attributes are errors, not silently ignored.
- Export a JSON Schema (from zod) and a short DSL reference doc. Both get fed to LLMs when they write posters.
- LLM retry rule: on validation errors, send the errors back once. If it still fails, fall back to a safe default template and log it.
- Round trip: `PosterDoc -> DSL` printer so the editor and tests can regenerate text.

## 5. Assets

Inputs (provided by the human): Phosphor icons and Lobe icons (tech company logos), as separate datasets. More assets will come later, so ingestion must be re-runnable.

`packages/assets` responsibilities:
- Normalize every SVG: consistent viewBox, strip scripts and metadata, run svgo with a fixed config.
- One manifest per dataset: `id`, `name`, `aliases[]`, `tags[]`, `ticker?`, `domain?`, `file`, `variants[]` (for example mono, color, dark), `license`, `source`.
- Phosphor: choose one weight per theme (regular for light, fill for chips, and so on) and store all weights as variants.
- Lobe: store color and mono variants. Pick by palette contrast at render time.
- Alias table for companies (hand curated, grows over time): "Alphabet", "Google", "GOOGL" map to one id. Ambiguous names (Apple, Meta, Amazon) get `context_words` to help disambiguate.
- Record license and trademark notes per asset. Logos are trademarks. The human should check usage terms for publication.
- Monogram badge generator as last-resort logo (initials on a palette color).

## 6. Resolver (embeddings + keywords)

The human already has an optimized ONNX embedding model.

### 6.1 Runtime
- `onnxruntime-node` to run the model.
- Tokenizer must match the model exactly. Use the model's own tokenizer files with a JS tokenizer library (for example `@huggingface/transformers` tokenizers or the `tokenizers` package).
- Parity test (required): embed 50 sample strings in Node and in the reference environment the model came from. Cosine similarity must be above 0.999 for each. Do not proceed if this fails.
- L2-normalize all vectors so cosine equals dot product.

### 6.2 Index build (offline, re-runnable)
- Embedding text recipe per asset: `"{name}. {aliases}. {tags}"` (tune later; version it).
- Output per dataset: `vectors.f32.bin` (Float32Array), `index.json` (ids in order), `meta.json` with a hash of model file + recipe + manifest.
- A few thousand vectors fit in memory: brute-force dot product is enough. No vector database.
- Rebuild only when the hash changes. Runtime refuses to start if the index hash does not match the model.

### 6.3 Scoring
Company logos (keyword-heavy, as the human asked):
- `score = 0.7 * keyword + 0.3 * embedding`
- Keyword score in 0..1: exact name or alias match = 1.0, ticker match = 0.95, token overlap and prefix scaled lower.
- Accept only if `keyword > 0`, or embedding alone is very high (threshold tuned on the eval set). This stops "Apple" the fruit matching the brand.
- Use `context_words` and the category from the story for ambiguous names.
- Unknown names go to a review list, never silently guessed.

Icons (embedding-leaning):
- `score = 0.6 * embedding + 0.4 * keyword` over name and tags.
- Query comes from the item text, or from an explicit `:icon[...]`.
- Ties break by icon id (alphabetical) so winners never flip.
- Below threshold, use the category default icon.

### 6.4 Fallback ladder (a slot never stays empty)
company logo -> semantic icon -> category default icon -> monogram badge.

Dedupe pass: across one poster, no two items share the same icon. Assign greedily by score, take the next best on collision. Logos may repeat if the same company appears twice.

### 6.5 Determinism
Pin: model file hash, tokenizer, onnxruntime version, embedding recipe, asset snapshot, fonts, Playwright/Chromium version. Cache LLM output by content hash. Log every choice (candidates, scores, thresholds, final reason).

### 6.6 Evaluation
Build a labeled resolver eval set (about 100 headlines with expected logo and icon). Report top-1 and top-3 accuracy and tune thresholds and weights from it. Run in CI.

## 7. Layout engine

Canvas default 1080 x 1350 (4:5). Also support 1080x1080 and 1080x1920.

Grid (for 1080 wide): margin 72, 12 columns, gutter 24, column width 56. Base unit 8. All spacing is a multiple of 8.

Type scale: base 32, ratio 1.25, snapped to even numbers: 32, 40, 50, 64, 80, 100.

Slots are boxes `{x, y, w, h}` computed by pure functions: `layoutFn(blocks, theme, canvas) -> Slot[]`.

### 7.1 Layout families (from the human's references)

1. **hero-stack**: big title card on top, 2 to 3 pills below. Best for 1 to 3 items. (Reference: cream and forest pill post.)
2. **list-cards**: heading, then N stacked cards with number or icon, title, one line of meta. Best for 3 to 6 items. (Reference: dark numbered cards.)
3. **stat-card**: one huge number, label, supporting rows, footer brand. Best for a single data story. (Reference: dark stat card.)
4. **chip-cloud**: icon + label pills, stacked or slightly tilted. Best for 5+ short items or topic lists. (References: neon chips, pastel chips, tilted chips.)
5. **step-card**: badge, headline, short body, a mock card, a footer stat. Best for explainer-style content. (Reference: lime step post.)
6. **illustrated**: oversized title with scattered icons on colored blobs. Best for playful or cover posters. (Reference: playground poster.)

### 7.2 Math

Vertical rhythm for list-cards:
```
content_h = H - top - bottom
card_h    = clamp(min_h, (content_h - gap * (n - 1)) / n, max_h)
```
If `card_h < min_h` at the smallest allowed font ladder step, switch to chip-cloud or split into two posters (configurable, default: switch layout).

Hero-stack: hero takes 45 to 55 percent of content height. Pills share the rest with fixed gap 16 or 24.

Pill: height 96 (or 80 compact), radius 999 or 28 (palette decides), horizontal padding 28, inner gap 16, logo box 48 to 56.

Text fit (deterministic, no guessing):
1. Try font sizes down the ladder (for example 64, 56, 50, 44, 40, 32).
2. For each, measure wrapped line count using the real font metrics (`@napi-rs/canvas` or opentype.js; or Playwright measurement in the renderer).
3. Pick the first size where lines <= maxLines and height fits the slot.
4. If none fit, apply ellipsis at the smallest size and flag a QA warning.

### 7.3 Auto picker
Score each layout from: item count, presence of `stat`, average text length, requested style hint. Highest wins, ties break by layout id. Always overridable through front matter (`layout: list-cards`). The picker returns its reasoning for the log.

## 8. Poster design system (constraints, not suggestions)

Mood from the references: bold, flat, friendly, high contrast, generous radius, big type, few colors per poster. One accent family at a time.

### 8.1 Palettes (tokens: bg, surface, surface2, ink, inkMuted, accent, accent2, border)
Starter set, derived from the references. Tune by eye in the harness.

| id | bg | surface | ink | accent | accent2 | notes |
|---|---|---|---|---|---|---|
| cream-forest | #FDFBF0 | #17342F | #17342F | #DDE06A | #F4F1DC | dark hero card, lime pills |
| lime-fresh | #CFF78A | #F3FBE4 | #1F4D2B | #2F7D3E | #A8EC62 | soft cards on lime |
| ink-orange | #0B0B0B | #151515 | #F5F5F5 | #FF5A1F | #2A2A2A | outlined dark cards, one orange highlight word |
| ink-mint | #111111 | #1A1A1A | #FFFFFF | #B9FF8E | #6B6B6B | stat style, mono labels |
| night-pills | #061923 | #0D2733 | #0B1A22 on pills | #D4FF4F | #FF7A45 / #E07CFF | three bright pills on deep teal |
| pastel-chips | #FAFAFA | per-chip tint | per-chip deep tone | #3B6FE0 | #12B76A / #8B5CF6 | blue, green, purple, gray chips |
| playground | #FFFFFF | #E5604B | #111111 | #5DBDE0 | #F8D840 / #3B9A52 | blobs with black line icons |

Rules:
- Palette is chosen by category (finance -> ink-mint, tech -> night-pills or ink-orange, general -> cream-forest, and so on), overridable.
- Contrast check on every text/background pair: 4.5:1 for body, 3:1 for text 40px and above. Fail the render in QA otherwise.
- Max 1 accent family plus neutrals per poster, except `pastel-chips` and `night-pills` which define their own multi-color set.

### 8.2 Typography (Google Fonts, bundled locally)

Decision: support a small curated set of Google Fonts, all open licensed (OFL). Fonts are downloaded once at build time and stored in the repo or cache. The renderer never touches the network.

How fonts are fetched:
- Use the `@fontsource/<family>` npm packages (or `@fontsource-variable/<family>` where available). They ship pinned woff2 files with Latin subsets, so installs are deterministic and versioned. Do not link to fonts.googleapis.com at render time.
- Script `pnpm fonts:fetch` copies the chosen weights into `packages/theme/fonts/<family>/` and writes `fonts/manifest.json`.
- Only include the weights we use (400, 500, 600, 700) and the Latin subset by default. Add subsets (for example Devanagari) only when a poster needs them.

Curated starter shortlist (about 8 families, add or remove freely):

| Role | Families |
|---|---|
| Clean grotesk (tech, finance) | Geist, Space Grotesk, DM Sans, Manrope |
| Playful or friendly | Bricolage Grotesque, Outfit, Sora |
| Serif display (editorial, warm) | Fraunces, Instrument Serif |
| Mono (small labels only) | Geist Mono, DM Mono |

Starter pairs (heading + body, with a mono label font where useful):
1. Geist + Geist Mono: clean tech, matches the dark stat and list references.
2. Fraunces + Instrument Sans: friendly serif headline, matches the cream and forest reference.
3. Bricolage Grotesque + DM Mono: playful, matches the chip and illustrated references.
4. Space Grotesk + DM Sans: neutral fallback.

Font registry (`fonts/manifest.json`), one entry per family:
`{ id, family, source: "google", package, weights[], variable: bool, role: "grotesk"|"serif"|"mono"|"playful", license: "OFL", metrics: { capHeight, xHeight, avgCharWidth } }`

Rules:
- A font pair is a theme token: `{ heading, body, label? }`. Switching it is a one-line change in the front matter (`font: fraunces-instrument`) or the Style popover.
- Max two families per poster, plus an optional mono for small labels.
- Headline weight 600 to 700, tight tracking (-1 to -2 percent) on large sizes. Mono only for small labels (dates, sources, field names).
- Do not use a default system font stack and do not default to Inter.
- Fonts must be embedded in the render (base64 `@font-face` or local file URLs) and `document.fonts.ready` must resolve before the screenshot.
- Text fit runs per font, because every font wraps differently. Never reuse fit results across fonts.

### 8.2.1 Font lab (for trying many fonts)

A command in the harness that renders one poster with every font pair and writes a contact sheet:

```
poster fontlab <poster.md> [--pairs all|geist,fraunces,...] [--palette auto]
```

- Output: `out/fontlab/<slug>/<pair-id>.png` plus `contact-sheet.png` (a grid with each pair's name under its thumbnail) and a small HTML page to view it.
- Also supports a type specimen mode (`--specimen`) that renders a fixed sample (headline, pill, stat number, small label) per font, so we judge numerals, weights, and tracking quickly.
- The editor can open the same contact sheet from the Style popover under "Compare fonts" (inside a modal, closed by default).
- Adding a font = add it to the shortlist, run `pnpm fonts:fetch`, run `poster fontlab`. No code changes.
- Approved font pairs get a golden image so a font update never silently changes the look.

### 8.3 Components (each emits HTML + CSS from tokens only)
- `Title`: big headline, optional eyebrow, optional highlighted word in accent color.
- `Pill`: rounded container, optional leading `Logo` or `Icon`, text. Variants: filled, tint, outline.
- `InlineCompany`: **logo + company name inline in running text**. Logo height = 1.0em to 1.1em of the text, vertically centered, gap 0.25em, name in same weight. When logo is missing, show a monogram badge of the same size. This is the signature pattern, so test it hard.
- `Card`: surface with radius 24 to 32, padding 32, optional number badge (01, 02), optional icon at left, meta line in muted ink.
- `Stat`: huge value (80 to 160px), delta in accent, hairline dividers, small mono labels.
- `Chip`: icon in rounded square plus label (notch/tab style optional).
- `Badge`: small pill label such as "Step 1" or a category tag.
- `Footer`: source, date, handle, small and quiet.
- `Divider`: 1px hairline in border color.

### 8.4 Banned patterns (enforced by the validator and QA, not by asking nicely)
- Purple-to-blue or any default gradient headline.
- Centered hero with three equal feature cards.
- Default Inter, Roboto, Arial.
- More than two font families, more than 3 font sizes of body text.
- Emoji used as icons (use the icon sets).
- Drop shadows heavier than a subtle single layer, glow effects, glassmorphism.
- Text smaller than 28px on a 1080 wide poster.
- Gray text on colored backgrounds with contrast under 4.5:1.
- Lorem ipsum or placeholder strings in output.

## 9. Rendering

Phase 1 uses Playwright (full CSS, fast to iterate). Later, optionally move to Satori + resvg for speed with no browser. Keep components emitting plain HTML and flexbox-only CSS so both renderers work.

- Reuse one browser instance and a page pool for batch jobs.
- Set viewport to the canvas size, `deviceScaleFactor` 1 by default, 2 as an export option.
- Wait for fonts and images (`document.fonts.ready`), then screenshot the poster root element only.
- All assets inlined as data URIs or served from local disk. Block all network requests during render.
- Output: `out/YYYY-MM-DD/<slug>.png` plus `<slug>.json` (the resolved PosterDoc) and `<slug>.log.json` (decisions).

## 10. Harness (the quality engine)

This is how we get the look right and keep it right.

- **Golden set**: `golden/<name>.md` with an approved `golden/<name>.png`. Start by recreating the human's 8 reference designs in the DSL until they look right. Add more as they come.
- **Visual diff**: `pixelmatch` + `pngjs`. Per-test threshold (default 0.1 percent of pixels). On failure, write `diff/<name>.png` and an HTML gallery showing expected, actual, and diff side by side.
- **Commands**: `pnpm golden` (run), `pnpm golden:update <name>` (approve a new image, human only), `pnpm golden:report` (open gallery).
- **Automatic QA on every render** (fail or warn):
  - text overflow or ellipsis used
  - contrast below thresholds
  - margins smaller than 72px for content
  - logo or icon match score below threshold (warn) or missing asset (fail)
  - banned patterns present
  - duplicate icons across items
  - image dimensions not equal to canvas
- **Resolver eval**: top-1 and top-3 accuracy on the labeled set, run in CI.
- **Parser tests**: valid and invalid DSL fixtures with expected error codes and positions.
- **Decision log**: every render writes why each visual was chosen, with candidates and scores.
- CI runs unit tests, resolver eval, and golden diffs on every PR.

## 11. Editor UI (apps/web)

Load the Emil Kowalski skills first (section 1.2).

### 11.1 Taste
Minimal and clean. The main screen shows only the content and the result. Anything that has a sensible default is hidden until the user asks for it.

Principles (these are hard rules):
1. **Progressive disclosure.** Related options are grouped into one control (popover, dropdown, or modal). Closed by default. Each group has good defaults, so the user can ignore them entirely.
2. **No clutter on the main screen.** No permanent sidebars full of toggles. No more than about 5 visible controls at any moment.
3. **One primary action** (Export). Everything else is quiet.
4. **Defaults are `auto`.** Auto layout, auto palette, auto logos and icons. Users only override when something looks wrong.
5. **Calm motion.** 150 to 200ms, ease-out, opacity and small translate only. Respect `prefers-reduced-motion`.
6. **Keyboard first**: `Cmd/Ctrl+K` command palette, `Cmd/Ctrl+Enter` render, `Cmd/Ctrl+E` export, `Esc` closes any popover.

### 11.2 Main screen layout
- Left: a plain text editor for the DSL (monospace, soft syntax highlighting for directives, inline error markers).
- Right: live poster preview, centered, with a subtle frame.
- Top bar, three things only: poster title (editable), a single **Style** button, and **Export**.
- Errors: a small inline marker in the editor gutter plus one quiet line under the editor. No red banners.
- On narrow screens, the editor and preview become two tabs.

### 11.3 Grouped options
- **Style** (popover): layout, palette, size, font pair. Each is a compact select with `Auto` first. Show a tiny thumbnail for layout and palette choices.
- **Assets** (modal, opened by clicking a logo or icon in the preview): search box over logos and icons, shows the top matches with scores, pick one or choose "none". Also a "Reset to auto" link.
- **Export** (split button): primary click exports PNG at default size. The dropdown holds scale (1x, 2x), format, and "copy to clipboard".
- **Advanced** (modal, buried under a small menu): resolver weights, thresholds, debug overlay (slot boxes), decision log viewer. Never shown by default.

### 11.4 Tech
Vite + React + TypeScript, Tailwind with design tokens, Radix primitives for popover/dialog/select, a drawer component for mobile sheets, a small toast for confirmations.

UI SDK (https://ui-sdk.dev, Vercel Labs, behaviors like `createPopup`, `createSelection`, `createTypeahead`, `createViewport`) is NOT used for the posters. It handles interaction and accessibility, not visual design, and posters are static images. It is an optional experiment for the editor only:
- Default stack is Radix. Do not add UI SDK as a core dependency.
- During Phase 6, after the editor works, try UI SDK on exactly one piece (the Style popover, or pan and zoom on the preview with `createViewport`). Compare bundle size, keyboard and screen reader behavior, and how much code it took.
- Keep it only if it is clearly better. Otherwise stay on Radix. Record the result in `docs/designs/`. The editor calls the same packages as the CLI through a thin local API (`/api/render`), so the preview always matches the exported PNG (render with the real renderer, debounce 250ms, show the previous image until the new one is ready).

## 12. Automation (apps/cli)

Commands:
- `poster render <file.md> [-o out.png]`
- `poster batch <dir>`
- `poster news` : fetch -> write DSL -> render
- `poster golden`, `poster index build`
- `poster fontlab <file.md>` : render one poster across all font pairs into a contact sheet
- `pnpm fonts:fetch` : copy the curated Google Fonts (via @fontsource) into the repo

News flow:
1. Fetch top N items (default 5, configurable) from the endpoint. Add a schema check on the response so format changes fail loudly. Key each item by hash of headline + URL.
2. Enrichment, cached by item id: LLM at temperature 0 returns structured JSON (category, short headline, company names, optional icon hint). Validate with zod, retry once on error.
3. Build the DSL document from the enrichment (code, not free-form LLM text, where possible), or let the LLM write the DSL directly using the JSON Schema and the DSL reference plus 2 examples. Either way, the parser validates it.
4. Render, run QA, save PNG + JSON + log.
5. Scheduling: `node-cron` or a CI schedule. Idempotent: same news set yields the same files.

Open decision to settle with the human: one combined poster of N items, or one poster per story. The layout picker already supports both; the news command should take `--mode combined|single`.

## 13. Phases and acceptance criteria

Run EGM brainstorm, draft, and review before each phase.

**Phase 1: Skeleton**
Monorepo, DSL parser, zod schema, one layout (hero-stack), one palette, Playwright render, CLI `render`.
Accept: a sample `.md` renders to a 1080x1350 PNG; invalid DSL gives positioned errors.

**Phase 2: The look**
All 6 layout families, 7 palettes, all components including InlineCompany, Google Fonts shortlist bundled via `fonts:fetch`, 4 starter font pairs, and the `poster fontlab` command. Logos and icons are picked by hand (`logo=nvidia`) at this stage.
Accept: all 8 reference designs recreated in the DSL and approved by the human; the font lab contact sheet exists and the human has picked the default font pair per palette.

**Phase 3: Harness**
Golden tests, visual diff gallery, QA checks, decision log, CI.
Accept: `pnpm golden` is green; breaking a margin on purpose makes it fail with a readable diff.

**Phase 4: Resolver**
ONNX runtime, tokenizer parity test, index build for icons and logos, keyword scoring, fallback ladder, dedupe, eval set.
Accept: parity test passes; eval set meets agreed accuracy (set the target with the human after the first run); `auto` works in the DSL.

**Phase 5: Auto layout and palette picker**
Accept: the picker's choice is sensible on 20 varied sample inputs and always explainable in the log.

**Phase 6: Web editor**
Following section 11 exactly, including the "Compare fonts" modal and the time-boxed UI SDK experiment (section 11.4).
Accept: main screen has only editor, preview, Style, Export; every option group is closed by default; preview matches exported PNG; UI SDK verdict recorded.

**Phase 7: Automation**
News fetch, enrichment cache, LLM-to-DSL, scheduling.
Accept: a scheduled run produces N valid posters with zero manual steps; failures fall back to a safe template and are logged.

## 14. Working agreements for the agent

- Never change a golden image without the human approving it.
- Keep every package independently testable. No cross-package reaching into internals.
- Prefer small commits per task. Each commit passes tests.
- When a decision is unclear, write it into `docs/designs/` and ask. Do not guess silently.
- Update this plan if reality changes, and note why.
- Written content in docs and UI copy should be plain, human, and easy to read. No em dashes.

## 15. Open questions for the human

1. News endpoint: URL, response format, and update frequency?
2. Combined poster or one per story (default `combined`)?
3. Asset formats and metadata: are the Phosphor and Lobe datasets SVG with names and tags, and where do aliases and tickers come from?
4. The ONNX model: name, tokenizer files, embedding dimension, and where the reference (Python) implementation lives for the parity test?
5. Which Google Fonts to keep from the shortlist after the font lab, and any brand colors to add as palettes? Any non-Latin scripts needed (for example Devanagari)?
6. Preferred hosting for the editor (local only, or deployed)?
7. Target accuracy for the resolver eval set?

## 16. Definition of done (v1)

- A news run produces clean PNG posters automatically, with correct logos for known companies and sensible icons for the rest.
- Every poster passes QA, and the golden suite is green.
- A human can open the editor, paste or write DSL, see the poster, and export with one click, without ever seeing a settings panel unless they ask.
- An LLM given the JSON Schema and DSL reference can write valid posters on the first or second try.

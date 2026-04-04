# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

Single-file stylistic analysis tool for Spanish literary prose. Everything lives in `analizador-texto.html` — CSS, HTML, and JavaScript in one file, no build step, no dependencies, no server required. Open in any modern browser.

The only external call is to the Anthropic API (`/v1/messages`) for the spell/grammar check feature (`runOrtografia`), which uses `claude-sonnet-4-20250514`.

## Development workflow

No build, lint, or test commands exist. Development is:
1. Edit `analizador-texto.html`.
2. Open/reload in browser.
3. The text persists in `localStorage` (`key: 'analizador-texto'`) so reloads don't lose work.

## Architecture

### Global state (two variables)
- `currentView` — `'plain' | 'frases' | 'verbos'`
- `AD` — the analysis object (null if textarea empty), recomputed on every text change

### Critical rendering invariant
**Never apply regex to a string that already contains HTML.** The v1 bug was caused by exactly this. The correct pipeline is always:
1. `tokenize(text)` — split plain text into `{text, isWord}[]` tokens
2. `classifyToken(tok)` — classify each token on plain text
3. `renderView()` — build HTML in one linear pass using `esc(tok.text)` for every user string

`esc()` must be called on every user-supplied string before insertion into `innerHTML`. This is the XSS/corruption prevention mechanism.

### Verb detector
The detector is heuristic (not a full morphological parser). Three layers in priority order:
1. **`IRR`** — explicit whitelist of irregular verb forms, keyed by verb type. Checked first.
2. **`VT`** — array of verb types with suffix lists (most specific first, longest suffix first within each type). First match wins.
3. **`SW`** — stopword blacklist (articles, pronouns, prepositions, and nouns that share suffixes with verb forms).

**When a noun gets incorrectly marked as a verb:** add it to `SW`. This is the expected maintenance path.  
**When an irregular verb isn't detected:** add it to the appropriate `IRR` set.  
**Never relax suffix criteria** — prefer expanding `IRR`/`SW` to changing suffixes.

`MIN_SYL_LEN` sets a minimum token length per verb type to avoid short common words being matched.

### Verb type split: `indefinido3s` vs `indefinido`
Third-person singular preterite (*entró*, *hizo*, *fue*) is split into its own type `indefinido3s` and appears **before** `indefinido` in `VT` (first match wins). This is intentional: it's the dominant narrative tense in Spanish prose and deserves its own color.

### CSS architecture for verb colors
Colors live entirely in CSS, not in the `VT` array. Each verb type has a class `.vt-xxx` with:
- `--vtc` custom property — the type's color
- `background` — solid background for the chip/panel
- `.rendered-text .vt-xxx` can override background for text vs panel variants

To change a verb color: edit only CSS. The JS array `VT` has no color properties.

### Syllable counter (`sylCount`)
Implements Spanish RAE rules for diphthongs, hiatus, and triphthongs. Limitations: no sinalefa (irrelevant for prose), no handling of foreign words/acronyms.

### Flow score
`flowScore(syls)` = `0.8 × nPVI_score + 0.2 × SD_norm`
- nPVI measures local rhythm (pairwise consecutive sentence variation)
- SD normalized via `tanh(sd/16) × 100` corrects for narrow absolute range
- Calibration parameters: weight 80/20, k=16 in tanh

### Stats sidebar
`updateStats()` reads `AD` and updates DOM elements with ids `st-w`, `st-s`, `st-sy`, `st-av`, `st-fl`, `st-fw`.  
Panels `sec-fr` (sentence length distribution) and `sec-vb` (verb list) are shown/hidden by `setView()`.

## Key data structures

| Symbol | Description |
|--------|-------------|
| `AD` | Analysis Data object (sentences, syllables, distribution, stats, verb lists) |
| `VT` | Array of verb type definitions `{key, cls, label, suf[]}` |
| `IRR` | Map of explicit irregular forms by verb type (Sets) |
| `SW` | Set of words never classified as verbs |
| `MIN_SYL_LEN` | Min token length per verb type |
| `LS_KEY` | `'analizador-texto'` — localStorage key for text persistence |

## Sentence length clusters (v9+)
5 clusters: 1–6 / 7–15 / 16–25 / 26–40 / 41+ syllables, mapped to CSS classes `fl-1` through `fl-5`. Colors: green / yellow / orange / magenta / red.

## Known limitations (intentional)
- Present tense 3rd person singular (*camina*, *corre*) not detected — suffixes `-a/-e/-o` are too ambiguous
- Sinalefa not implemented
- Foreign words may get incorrect syllable counts
- The ortografía feature requires internet and a context where Anthropic API credentials are available (works natively in Claude.ai)

## Workflow
- Always update DECISIONES.md and DOCUMENTEACION.md. DECISIONES.md include a list on how and why things were done or discarded.
    DOCUMENTACION.md only contains current state on how the application and the code works/is organized. README.md is a briefing of DOCUMENTACION.md plus install instructions.
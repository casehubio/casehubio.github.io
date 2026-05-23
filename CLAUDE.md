# casehubio.github.io

Org landing page for the [casehubio](https://github.com/casehubio) GitHub organisation.
Served at **https://casehubio.github.io**.

**Local path:** `/Users/mdproctor/claude/casehub/casehubio.github.io`
**GitHub:** `https://github.com/casehubio/casehubio.github.io`

## Project Type

type: custom

Static HTML/CSS website. No build step, no Jekyll (yet), no JavaScript.
GitHub Pages serves directly from the `main` branch root.

## What's Here

```
index.html          Single-page landing — all content lives here
assets/css/main.css Site styles — dark theme (copied from casehub-poc, extended)
README.md           Local dev and Jekyll migration notes
```

Everything visible at https://casehubio.github.io comes from `index.html`.

## Local Development

Open `index.html` directly for layout review (CSS won't load over `file://`).
For accurate rendering with styles, serve locally:

```bash
python3 -m http.server 8080 --directory /Users/mdproctor/claude/casehub/casehubio.github.io
# open http://localhost:8080
```

## Deployment

Push to `main`. GitHub Pages deploys automatically — no workflow, no config needed.

```bash
git -C /Users/mdproctor/claude/casehub/casehubio.github.io push
```

## Content Sources

The page content is drawn from the platform documentation in the sibling `parent` repo.
Before updating any project card headline or description, read the authoritative source first:

| Content | Source |
|---------|--------|
| Platform overview, tier structure, repo one-liners | `../parent/docs/PLATFORM.md` |
| Application tier repos (devtown, aml, clinical, quarkmind) | `../parent/docs/APPLICATIONS.md` |
| Per-repo capability headlines and descriptions | `../parent/docs/repos/<repo-name>.md` |

**The deep-dives are the authoritative source.** `index.html` holds a distilled 2-sentence summary per repo — when the deep-dive changes, update the card to match.

## Terminology

- Say **"Modernised Blackboard Architecture"** — not "CMMN" or "CMMN semantics"
- Say **"AI Fusion Harness"** for the overall product positioning
- **Classical AI** = rules engines, Modernised Blackboard Architecture, deterministic reasoning
- **LLM AI** = autonomous agents, agent mesh, adaptive reasoning

## Architecture Diagram

The inline SVG in `index.html` shows four tiers:

```
Applications  — devtown · aml · clinical · quarkmind
Runtime       — claudony
Orchestration — casehub-engine
Foundation    — platform · ledger · work · qhorus · connectors · eidos
```

Foundation has the accent border (teal) — it is the AI Fusion Core where Classical and LLM AI meet.
Classical AI annotation spans Orchestration + Foundation. LLM AI annotation spans Runtime + Foundation.

When a new repo is added to the platform, add it to the correct band in the SVG and add a project card.

## How to Update Content

**Adding a new repo:**
1. Read `../parent/docs/repos/<new-repo>.md` for the headline and description
2. Determine which tier it belongs to (check `../parent/docs/PLATFORM.md` Repository Map)
3. Add it to the correct SVG band (adjust `x` coordinates of existing labels if needed)
4. Add a project card in the correct section (Foundation or Applications)
5. Commit and push

**Updating an existing repo description:**
1. Read the updated `../parent/docs/repos/<repo>.md`
2. Update `card-headline` and `card-desc` in `index.html`
3. Commit and push

**Changing the hero or AI Fusion copy:**
Edit directly in `index.html`. Keep terminology consistent (see Terminology section above).

## Evolving to Jekyll

The file paths are already Jekyll-compatible — migration when needed:

1. Add `_config.yml`:
   ```yaml
   title: CaseHub
   url: https://casehubio.github.io
   baseurl: ""
   ```
2. Extract the `<body>` content into `_layouts/landing.html` (wrap in `{{ content }}`)
3. Replace `index.html` with Jekyll front matter pointing to the `landing` layout
4. Add `Gemfile` with `jekyll` and `jekyll-feed`
5. Update `.gitignore` to exclude `_site/` and `.bundle/`

At that point blog and docs links can be added to the nav.

## CSS Theme

Defined in `assets/css/main.css`. Do not change the CSS variables — they are shared with casehub-poc and define the brand palette:

```css
--bg-deep:    #080d12;   /* page background */
--bg-card:    #0e1820;   /* card / section backgrounds */
--border:     #1a2e38;   /* all borders */
--accent:     #2aa8c4;   /* teal — primary brand colour */
--text:       #b8d8e0;   /* body text */
--text-muted: #4a7a8a;   /* secondary text, labels */
```

The secondary accent `#7ecfdf` (lighter teal) is used for LLM AI tags only.

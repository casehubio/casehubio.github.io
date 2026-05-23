# casehubio.github.io

Org landing page for [casehubio](https://github.com/casehubio), served at https://casehubio.github.io.

## Stack

Plain HTML + CSS. No build step. GitHub Pages serves from repo root on `main`.

## Local development

Open `index.html` directly in a browser — no server required for basic review.

For accurate absolute path resolution (`/assets/css/main.css`), serve with Python:

```bash
python3 -m http.server 8080
# open http://localhost:8080
```

## Migration to Jekyll (planned)

File paths are already Jekyll-compatible. To migrate:

1. Add `_config.yml` with `title`, `url`, `baseurl: ""`
2. Extract `index.html` body into `_layouts/landing.html`
3. Replace `index.html` with Jekyll front matter pointing to `landing` layout
4. Add `Gemfile` with `jekyll` and `jekyll-feed`

## Deployment

Push to `main` on `casehubio/casehubio.github.io`. GitHub Pages deploys automatically (no workflow needed — org sites deploy from root of default branch).

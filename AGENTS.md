# AGENTS

## Cursor Cloud specific instructions

This is a static HTML portfolio site with no build tools, package managers, or backend services.

### Running the dev server

Serve the site with Python's built-in HTTP server from the repository root:

```
python3 -m http.server 8000
```

Then open `http://localhost:8000/` in a browser.

### Project structure

- `index.html` — main portfolio page linking to sub-projects
- `2.4 Movie Ranking Project/`, `3.0 List Elements/`, `3.1 Nesting and Indentation/`, `4.3 HTML Porfolio Project/` — individual HTML exercise directories (each has `index.html` and `solution.html`)
- `public/` — shared pages (about, contact, birthday-invite, movie-ranking)
- `assets/images/` — project preview images

### Notes

- There are no automated tests, linters, or build steps in this repo.
- Directory names contain spaces; URL-encode them when using `curl` (e.g., `http://localhost:8000/3.0%20List%20Elements/index.html`).

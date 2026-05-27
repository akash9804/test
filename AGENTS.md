# AGENTS.md

## Cursor Cloud specific instructions

This is a static HTML portfolio website with zero dependencies and no build system.

### Running the development server

Serve files locally with Python's built-in HTTP server:

```
python3 -m http.server 8080 --directory /workspace
```

Then open `http://localhost:8080/index.html` in a browser.

### Project structure

- `index.html` — main portfolio page linking to sub-projects
- `public/` — additional pages (about, contact, project pages)
- `assets/images/` — image assets
- Numbered subdirectories (e.g. `2.4 Movie Ranking Project/`) — individual lesson projects, each with their own `index.html`

### Notes

- No package manager, linter, test framework, or build tool is configured. HTML files are opened directly.
- No lint or test commands exist. Validation is done by visually checking pages render correctly in a browser.
- Directory names contain spaces (e.g. `2.4 Movie Ranking Project`); URL-encode spaces as `%20` when constructing URLs.

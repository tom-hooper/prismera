# Fonts

Self-hosted webfonts. No font is loaded from fonts.googleapis.com, so no visitor
IP leaves the site for a font request and the pages render even if a third party
CDN is blocked.

| File | Family | Licence |
|------|--------|---------|
| `schibsted-grotesk-latin.woff2`, `schibsted-grotesk-latin-ext.woff2` | Schibsted Grotesk (variable, 400-900) | SIL Open Font License 1.1, see `OFL-Schibsted-Grotesk.txt` |
| `material-icons.woff2` | Material Icons (filled, 400) | Apache License 2.0, see `LICENSE-Material-Icons.txt` |

Upstreams: github.com/schibsted/schibsted-grotesk and
github.com/google/material-design-icons. The `@font-face` rules and the
`.material-icons` base rules live at the top of `src/css/material.css`, and the
body font token `--md-font` points at Schibsted Grotesk.

Both licences permit redistribution, so these files are committed to this repo.

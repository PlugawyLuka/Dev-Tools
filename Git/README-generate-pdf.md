# Generating `github-guide.pdf`

This repository contains `github-guide.md` — a comprehensive GitHub guide for beginners. Below are ways to convert it to PDF.

## Option A — GitHub Actions (recommended)

1. Push changes to `main` or open the Actions tab on GitHub.
2. Run the "Convert Markdown to PDF" workflow (or wait for a push).
3. Download the artifact `github-guide-pdf` from the workflow run.

This workflow uses `pandoc` and `wkhtmltopdf` on Ubuntu to create a PDF.

## Option B — Local conversion (Windows PowerShell)

Install one of these toolchains:

- Pandoc: https://pandoc.org/installing.html
- A TeX engine (TeX Live or MiKTeX) if using `--pdf-engine=xelatex`.
- Or use `wkhtmltopdf` to convert HTML to PDF.

Example PowerShell steps (using Pandoc + MiKTeX):

```powershell
# After installing pandoc and MiKTeX
pandoc "github-guide.md" -o "github-guide.pdf" --pdf-engine=xelatex
```

Or using wkhtmltopdf (no TeX required):

```powershell
pandoc "github-guide.md" -s -o "github-guide.html"
wkhtmltopdf "github-guide.html" "github-guide.pdf"
```

## Option C — VS Code extension

Use the "Markdown PDF" extension in VS Code to export Markdown to PDF.

## Troubleshooting

- If `pandoc` is not found, install it and add it to your PATH.
- If PDF looks wrong, try converting to HTML first and inspect the HTML.
- For advanced layout, consider using a LaTeX template.

---

If you'd like, I can run the GitHub Actions workflow for you (requires pushing changes) or help install pandoc locally.

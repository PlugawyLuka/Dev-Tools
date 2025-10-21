# Git Manual — quick index

This folder contains beginner → semi-advanced GitHub guides and helper files for this workspace.

Guides
- github-guide.md — Full GitHub Guide for Beginners (quick setup, SSH, workflows, CI)
- github-security.md — Security, safety & IP protection guidance
- github-repos.md — Repository management, access control, workflows
- PowerShell.md — Windows PowerShell equivalents & semi-advanced tips
- README-generate-pdf.md — How to generate `github-guide.pdf` (Actions or local)

Quick start
1. Read `github-guide.md` for step-by-step setup.
2. If you use Windows, see `PowerShell.md` for command equivalents.
3. To create a PDF of the main guide, follow `README-generate-pdf.md`.

Generate PDF (local example)
- Using Pandoc + TeX (Windows PowerShell):
  ```powershell
  # after installing pandoc and MiKTeX
  pandoc "github-guide.md" -o "github-guide.pdf" --pdf-engine=xelatex
  ```

Contributing
- Edit the Markdown files and open a PR (or submit changes locally).
- Keep secrets out of the repo. See `github-security.md` for recommended practices.

Notes
- PowerShell examples have been split to `PowerShell.md` to keep the main guide cross-platform.
- Use `gh` (GitHub CLI) for many automation tasks described in these guides.

If you want I can:
- Commit this README for you,
- Add badges (CI / docs) or a more detailed TOC,
- Or keep it minimal as above.
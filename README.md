# PaperOrchestra

A Claude Code skill pack that converts research ideas and messy materials into validated, reviewable, exportable manuscripts through artifact-driven orchestration.

## What it does

PaperOrchestra runs an end-to-end paper-writing workflow inside Claude Code:

1. **Prep** — Normalizes loose notes, PDFs, figures, and experiment results into canonical artifacts
2. **Plan** — Generates an outline, claim ledger, figure plan, and literature plan
3. **Support** — Builds citation pool, figures/tables, and evidence inventory
4. **Draft** — Composes an integrated manuscript from structured artifacts
5. **Review** — Critiques, refines, and reverts if quality drops
6. **Export** — Assembles final LaTeX package with bibliography and figures

Human checkpoints pause the workflow after prep, after planning, and before final export.

## Install

Copy this repo into your Claude Code skills directory:

```bash
# From your Claude Code config directory
cp -r /path/to/paperorchestra ~/.claude/skills/paper-orchestra
```

Or symlink it:

```bash
ln -s /path/to/paperorchestra ~/.claude/skills/paper-orchestra
```

Then invoke with `/paper-orchestra` in Claude Code.

## Usage

```
/paper-orchestra [idea] [materials_path]
```

- **idea** — A sentence or paragraph describing the paper's core contribution
- **materials_path** — Path to a folder containing research materials (notes, data, figures, PDFs)

Both are optional. The skill will prompt for missing information interactively.

## Requirements

- Claude Code
- Python 3.10+ (for validation scripts)
- `pdflatex` + `bibtex` on PATH (optional — for PDF compilation)

## Structure

```
SKILL.md              — Main skill definition
references/           — Install, workflow, and failure-mode documentation
scripts/              — Deterministic helper scripts (Python + shell)
templates/            — Workspace layout and manifest schemas
```

## License

MIT

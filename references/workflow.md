# Workflow Reference

## Stages Overview

| # | Stage | Key Artifacts | Gate | Checkpoint |
|---|-------|---------------|------|------------|
| 0 | Initialize | workspace/ | — | — |
| 1 | Prep | idea.md, experimental_log.md, manifests | A: Input completeness | After prep |
| 2 | Plan | outline.json, claim_ledger.json, figure_plan.json, literature_plan.json | B: Plan completeness | After plan |
| 3 | Support | citation_pool.json, refs.bib, figures/, captions.json | C: Evidence, D: Citation, E: Artifact | — |
| 4 | Draft | drafts/paper.tex | F: Rendering | — |
| 5 | Review | reviews/, scorecard.json | G: Refinement | — |
| 6 | Finalize | final/paper.tex, paper.pdf, submission_bundle.zip | — | Before export |

## Workspace Layout

```
workspace/
  inputs/
    raw_materials/       ← User's source files
    idea.md              ← Core contribution and research questions
    experimental_log.md  ← Experiments, results, observations
    venue_profile.md     ← Target venue requirements
    template.tex         ← LaTeX template (optional)
    materials_manifest.json
    figures_manifest.json
  planning/
    outline.json         ← Paper structure contract
    figure_plan.json     ← What figures exist / need creation
    literature_plan.json ← Search queries and coverage areas
    claim_ledger.json    ← Claims mapped to evidence
    results_inventory.json
  citations/
    citation_pool.json   ← Verified citation metadata
    refs.bib             ← BibTeX bibliography
  figures/
    generated/           ← Figures created during workflow
    supplied/            ← Figures from raw materials
    captions.json        ← Figure captions and paths
  tables/
    generated/           ← Table LaTeX snippets
  drafts/
    paper.tex            ← Working draft
    sections/            ← Optional section intermediates
  reviews/
    review_round_N.md    ← Review notes per round
    scorecard.json       ← Quality scores
  final/
    paper.tex            ← Final manuscript
    paper.pdf            ← Compiled PDF
    refs.bib
    figures/
    submission_bundle.zip
  logs/
    run_log.md           ← Session history
    checkpoints.md       ← Stage completion tracking
```

## Resuming Work

Re-invoke `/paper-orchestra` on an existing workspace. The skill reads `logs/checkpoints.md` to detect the last completed stage and offers to resume from the next one.

You can also request a specific stage re-run: "Re-run the review stage on this workspace."

## Running Individual Scripts

All scripts accept a workspace path as their only argument:

```bash
python scripts/init_workspace.py workspace/
python scripts/prep_materials.py workspace/
python scripts/validate_inputs.py workspace/
python scripts/build_claim_ledger.py workspace/
python scripts/verify_citations.py workspace/
python scripts/check_artifacts.py workspace/
bash scripts/compile_package.sh workspace/
```

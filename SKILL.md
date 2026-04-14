---
name: paper-orchestra
description: End-to-end research paper orchestration — from idea + messy materials to validated, exportable manuscript
trigger: /paper-orchestra
---

# PaperOrchestra — Claude Code Skill

You are an artifact-driven paper orchestration engine. You convert research ideas and supporting materials into a validated, reviewable, exportable manuscript through a structured multi-stage workflow.

## Core Principles

1. **Artifacts before prose** — Create reliable intermediate artifacts before composing text.
2. **Prep is first-class** — Never assume the user has pristine inputs. Normalize first.
3. **Deterministic gates beat eloquent mistakes** — Unsupported claims, broken refs, and malformed outputs must fail loudly.
4. **Humans approve risky transitions** — Pause after prep, after planning, and before final export.
5. **One coherent workflow, not a zoo of agents** — Split functions only when they own a distinct artifact or gate.

## Invocation

```
/paper-orchestra [idea] [materials_path]
```

- `idea` — A sentence or paragraph describing the paper's core contribution (optional, will prompt)
- `materials_path` — Path to folder with research materials: notes, data, figures, PDFs (optional, will prompt)

---

## STAGE 0: INITIALIZE WORKSPACE

### Entry condition
User invokes the skill.

### Protocol

1. Ask the user for any missing inputs:
   - **Idea**: What is the core contribution of this paper?
   - **Materials path**: Where are your research materials? (or "none" to start from scratch)
   - **Venue** (optional): Target conference/journal name and any formatting requirements

2. Run workspace initialization:
```bash
python scripts/init_workspace.py "<workspace_path>"
```

3. If the user provided a materials path, copy contents into `workspace/inputs/raw_materials/`.

4. Log the session start in `workspace/logs/run_log.md`.

### Exit condition
- `workspace/` directory exists with canonical structure
- User inputs captured

### Output artifacts
- Full workspace directory tree
- `workspace/logs/run_log.md` initialized

---

## STAGE 1: PREP AND NORMALIZATION

### Entry condition
Workspace initialized, raw materials (if any) are in `workspace/inputs/raw_materials/`.

### Required input artifacts
- User's idea (from invocation or prompt)
- Raw materials directory (may be empty)

### Protocol

1. Scan `workspace/inputs/raw_materials/` and inventory all files by type (markdown, PDF, images, CSV/data, code, LaTeX, BibTeX).

2. Synthesize each canonical input artifact. For each, read relevant raw materials and compose a structured version:

   **a. `workspace/inputs/idea.md`**
   - Core contribution statement
   - Research questions
   - Key claims the paper will make
   - Novelty framing

   **b. `workspace/inputs/experimental_log.md`**
   - Experiments conducted (with conditions, parameters)
   - Results summary (quantitative where available)
   - Key observations and interpretations
   - Links to data files or figures in raw_materials

   **c. `workspace/inputs/materials_manifest.json`**
   - Inventory of every file in raw_materials with: path, type, description, relevance rating
   - Gaps identified (what's missing that the paper will need)

   **d. `workspace/inputs/venue_profile.md`**
   - Target venue name and type (conference/journal)
   - Page limits, formatting requirements
   - Audience description
   - Key evaluation criteria for this venue

   **e. `workspace/inputs/figures_manifest.json`**
   - List of figures/tables available in raw_materials
   - For each: path, description, format, whether it needs regeneration
   - Planned figures not yet created

3. For any artifact that cannot be fully synthesized from raw materials, write what is known and mark gaps with `<!-- GAP: description -->` comments.

4. Ask the user focused follow-up questions ONLY for critical gaps that block downstream work.

### Validation
Run input validation:
```bash
python scripts/validate_inputs.py "workspace/"
```
This checks all five canonical inputs exist and flags incomplete sections.

### Exit condition
- All five canonical input artifacts exist in `workspace/inputs/`
- Validation passes (warnings OK, errors block)

### **>>> HUMAN CHECKPOINT 1 <<<**

Present to the user:
- Summary of what was synthesized vs. what came from raw materials
- List of any gaps or assumptions made
- Ask: "Do these normalized inputs reflect your project correctly? Are any important materials missing?"

**Wait for explicit approval before proceeding.**

---

## STAGE 2: GLOBAL PLANNING

### Entry condition
Canonical inputs validated and checkpoint 1 approved.

### Required input artifacts
- `workspace/inputs/idea.md`
- `workspace/inputs/experimental_log.md`
- `workspace/inputs/venue_profile.md`
- `workspace/inputs/figures_manifest.json`

### Protocol

1. Read all canonical inputs thoroughly.

2. Generate **`workspace/planning/outline.json`**:
```json
{
  "title": "Working paper title",
  "sections": [
    {
      "id": "abstract",
      "name": "Abstract",
      "purpose": "...",
      "key_points": ["..."],
      "estimated_length": "250 words",
      "depends_on_figures": [],
      "depends_on_citations": []
    }
  ],
  "narrative_arc": "Description of the paper's argumentative flow"
}
```

3. Generate **`workspace/planning/claim_ledger.json`**:
```json
{
  "claims": [
    {
      "id": "C1",
      "statement": "Our method achieves 15% improvement over baseline X",
      "type": "quantitative|qualitative|methodological",
      "evidence_source": "experimental_log entry or data file reference",
      "support_level": "strong|moderate|weak|unsupported",
      "depends_on_figures": ["fig1"],
      "depends_on_citations": [],
      "notes": ""
    }
  ]
}
```

4. Generate **`workspace/planning/figure_plan.json`**:
```json
{
  "figures": [
    {
      "id": "fig1",
      "description": "...",
      "type": "chart|diagram|table|photo",
      "source": "existing|generate|collect",
      "source_path": "path if existing",
      "supports_claims": ["C1"],
      "caption_draft": "..."
    }
  ]
}
```

5. Generate **`workspace/planning/literature_plan.json`**:
```json
{
  "search_queries": ["..."],
  "known_references": [
    {
      "title": "...",
      "authors": "...",
      "relevance": "...",
      "bibtex_key": "..."
    }
  ],
  "coverage_areas": [
    {
      "area": "related work topic",
      "minimum_refs": 3,
      "purpose": "establish baseline|show gap|support claim"
    }
  ]
}
```

6. Generate **`workspace/planning/results_inventory.json`**:
```json
{
  "results": [
    {
      "id": "R1",
      "description": "...",
      "value": "...",
      "source_file": "...",
      "supports_claims": ["C1"],
      "verified": false
    }
  ]
}
```

### Validation — Gate B: Plan completeness
Verify:
- Outline has all standard paper sections
- Every claim in the ledger has a support_level
- Every "unsupported" claim is flagged for resolution
- Figure plan covers all figures referenced in the outline
- Literature plan covers all citation needs from the outline

### Exit condition
- All five planning artifacts exist
- No unsupported claims without explicit acknowledgment

### **>>> HUMAN CHECKPOINT 2 <<<**

Present to the user:
- Paper outline with narrative arc
- Claim ledger summary (highlight any weak/unsupported claims)
- Figure and literature plans
- Ask: "Is the paper story right? Are the figure and literature plans aligned with your intended claims?"

**Wait for explicit approval before proceeding.**

---

## STAGE 3: SUPPORT ARTIFACT GENERATION

### Entry condition
Planning checkpoint approved.

### Required input artifacts
- All planning artifacts
- All canonical inputs

### Protocol — run these three sub-stages (can be interleaved):

### 3A. Evidence Accounting

1. Read `experimental_log.md` and `results_inventory.json`.
2. For each claim in `claim_ledger.json`, verify evidence exists and update support levels.
3. Write updated `workspace/planning/claim_ledger.json`.

**Gate C: Evidence integrity** — Block if:
- Any claim has support_level "unsupported" without user acknowledgment
- Any numerical claim has no source trace

### 3B. Literature Discovery and Verification

1. Using queries from `literature_plan.json`, search for candidate papers.
   - Use available search tools (Semantic Scholar, Google Scholar, arXiv, PubMed MCP tools if available).
   - For each candidate: record title, authors, year, venue, DOI/URL.

2. Verify metadata — do NOT allow raw generated citations:
   - Confirm paper exists via DOI lookup or search verification
   - Flag any citation that cannot be independently verified

3. Deduplicate and rank by relevance to coverage areas.

4. Write **`workspace/citations/citation_pool.json`**:
```json
{
  "citations": [
    {
      "key": "smith2024",
      "title": "...",
      "authors": ["..."],
      "year": 2024,
      "venue": "...",
      "doi": "...",
      "verified": true,
      "relevance_area": "...",
      "relevance_score": 0.9
    }
  ]
}
```

5. Generate **`workspace/citations/refs.bib`** from verified citations only.

**Gate D: Citation integrity** — Block if:
- Any citation in the pool is unverified
- Duplicate BibTeX keys exist

### 3C. Figure and Table Pipeline

1. For each entry in `figure_plan.json`:
   - If `source: "existing"` — verify file exists, copy to `workspace/figures/supplied/`
   - If `source: "generate"` — generate the figure using available data and write to `workspace/figures/generated/`
   - If `source: "collect"` — flag for user to provide

2. Write **`workspace/figures/captions.json`**:
```json
{
  "captions": [
    {
      "figure_id": "fig1",
      "path": "figures/generated/fig1.png",
      "caption": "...",
      "exists": true
    }
  ]
}
```

3. Generate any table LaTeX snippets needed → `workspace/tables/generated/`

**Gate E: Artifact integrity** — Block if:
- Any figure/table referenced in the outline does not exist and is not flagged as pending
- Captions missing for required figures

### Exit condition
- Citation pool and refs.bib exist with only verified entries
- Figures/tables exist or are explicitly deferred
- Claim ledger is updated with final evidence support levels
- All gates pass

---

## STAGE 4: DRAFT COMPOSITION

### Entry condition
All support artifacts available, gates C/D/E pass.

### Required input artifacts
- `workspace/planning/outline.json`
- `workspace/planning/claim_ledger.json`
- `workspace/citations/citation_pool.json`
- `workspace/citations/refs.bib`
- `workspace/figures/captions.json`
- `workspace/inputs/venue_profile.md`

### Protocol

1. Read the outline, claim ledger, citation pool, and all support artifacts.

2. If a LaTeX template exists at `workspace/inputs/template.tex`, use it as the document skeleton. Otherwise, use a standard article class.

3. Compose the manuscript as **`workspace/drafts/paper.tex`**:
   - Write section by section following the outline
   - Integrate citations using `\cite{key}` referencing verified keys from refs.bib
   - Reference figures/tables using `\ref{fig:id}` matching the figure plan
   - Ground every quantitative claim in the claim ledger — if a claim's support_level is "weak", soften the language
   - Do NOT insert any citation key not present in `refs.bib`
   - Do NOT reference any figure/table not confirmed in `captions.json`

4. Optionally write section intermediates to `workspace/drafts/sections/` for long papers.

### Forbidden behaviors
- Inventing citations not in the verified pool
- Stating numerical results not traceable to the results inventory
- Referencing figures that don't exist
- Ignoring venue formatting constraints

### Validation
```bash
python scripts/check_artifacts.py "workspace/"
```
This verifies all `\cite{}` keys exist in refs.bib and all `\ref{}` targets have corresponding figures/tables.

### Exit condition
- `workspace/drafts/paper.tex` exists
- All citation and figure references resolve
- Gate F (rendering integrity) passes structural checks

---

## STAGE 5: REVIEW AND REPAIR

### Entry condition
Draft exists and passes structural validation.

### Protocol

1. **Review pass**: Read the full draft critically. Produce **`workspace/reviews/review_round_1.md`** covering:
   - Argument coherence and flow
   - Claim support strength
   - Citation adequacy
   - Figure/table integration
   - Language clarity and concision
   - Venue fit (length, style, formatting)
   - Specific issues with line-level suggestions

2. **Score**: Write **`workspace/reviews/scorecard.json`**:
```json
{
  "round": 1,
  "scores": {
    "argument_coherence": 7,
    "evidence_support": 8,
    "citation_quality": 7,
    "writing_clarity": 6,
    "venue_fit": 8,
    "overall": 7.2
  },
  "blocking_issues": ["..."],
  "suggestions": ["..."]
}
```

3. **Repair**: If blocking issues exist, revise the draft. Save revised version.

4. **Re-score**: Score the revision. If overall score decreased → **revert to pre-revision draft** (Gate G).

5. Maximum 3 review-repair rounds. After that, proceed with best-scoring version.

### Exit condition
- At least one review round completed
- Best-scoring draft selected
- No blocking issues remain (or explicitly deferred by user)

---

## STAGE 6: FINALIZE AND EXPORT

### **>>> HUMAN CHECKPOINT 3 <<<**

Before final packaging, present to the user:
- Final review scorecard
- List of any deferred issues or softened claims
- Ask: "Is the draft acceptable for final packaging? Should any claims be softened or removed?"

**Wait for explicit approval before proceeding.**

### Entry condition
Draft accepted, checkpoint 3 approved.

### Protocol

1. Copy accepted draft to `workspace/final/paper.tex`.
2. Copy `workspace/citations/refs.bib` to `workspace/final/refs.bib`.
3. Copy all referenced figures to `workspace/final/figures/`.
4. Copy any table files to `workspace/final/tables/`.

5. Attempt PDF compilation:
```bash
bash scripts/compile_package.sh "workspace/"
```

6. If compilation fails:
   - Report specific errors
   - Attempt targeted fixes (missing packages, broken references)
   - Re-run compilation
   - If still failing, deliver the source package without PDF

7. If compilation succeeds, create submission bundle:
   - `workspace/final/paper.tex`
   - `workspace/final/paper.pdf`
   - `workspace/final/refs.bib`
   - `workspace/final/figures/`
   - `workspace/final/tables/`

8. Write final status to `workspace/logs/run_log.md`.

### Exit condition
- `workspace/final/` contains the complete submission package
- PDF compiled (or source package delivered with explanation)

### Final output to user
Report:
- Workspace path
- List of all artifacts produced
- Final review score
- Any known issues or deferred items
- Next steps (proofread, submit, iterate)

---

## ERROR HANDLING

When any gate fails:
1. Report which gate failed and why
2. List the specific artifacts or checks that failed
3. Suggest concrete repair actions
4. Ask the user whether to attempt auto-repair or pause for manual intervention

When a stage cannot proceed:
1. Log the blockers in `workspace/logs/run_log.md`
2. Save current state — all artifacts produced so far are preserved
3. Report to user with clear resume instructions

The workflow is **resumable** — re-invoking `/paper-orchestra` on an existing workspace detects the current stage and offers to continue from where it left off.

---

## STAGE ROUTING (for resume)

On invocation, if a `workspace/` already exists at the given path:
1. Read `workspace/logs/checkpoints.md` to determine last completed stage
2. Validate artifacts for that stage
3. Offer to resume from the next stage
4. User can also request re-running a specific stage

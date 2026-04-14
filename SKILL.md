---
name: paper-orchestra
description: End-to-end manuscript orchestration — white paper, data paper, or hybrid mode
trigger: /paper-orchestra
---

# PaperOrchestra — Claude Code Skill

You are an artifact-driven manuscript orchestration engine supporting three modes: **white paper** (conceptual / executive), **data paper** (evidence-heavy research), and **hybrid** (conceptual framing plus evidence-backed sections). Every path produces auditable artifacts, runs deterministic gates, and pauses humans at risky transitions.

## Core Principles

1. Artifacts before prose — Create reliable intermediate artifacts before composing long-form text.
2. Prep is first-class — Never assume pristine inputs; normalize, classify, and route before heavy drafting.
3. Deterministic gates beat eloquent mistakes — Unsupported claims, broken refs, malformed outputs, and policy violations must fail loudly.
4. Humans approve risky transitions — Pause after prep, after planning (or equivalent), and before final export.
5. One coherent workflow with branching modes — Same spine (initialize → prep → route → mode stages → package); mode-specific stages differ only where the artifact contract differs.
6. IP safety is non-negotiable — Every mode runs redline scanning; violations block until resolved, redacted, or explicitly acknowledged.

## Invocation

```
/paper-orchestra [idea] [materials_path]
```

- `idea` — Core contribution or topic (optional; prompt if missing).
- `materials_path` — Folder of notes, data, figures, PDFs, etc. (optional; prompt if missing).

Use the **Bash** tool to run helper scripts from the repo root. Scripts are **not** imported as libraries. Python targets **3.10+**, stdlib only. Workspace paths below are relative to the chosen `workspace/` root unless stated otherwise.

---

## STAGE 0: INITIALIZE WORKSPACE (all modes)

### Entry condition

User invokes the skill or starts a new project path.

### Protocol

1. Collect missing inputs:
   - **Idea** — What is the core contribution, problem, or narrative?
   - **Materials path** — Where are inputs on disk, or `none` for a blank slate.
   - **Venue** (optional) — Target outlet and formatting constraints.

2. **Intent capture** — Prompt for fields that populate `intent.json` (seeded by `scripts/init_workspace.py`). Record:
   - `audience` — Who must understand the output (e.g., executives, specialists, mixed).
   - `doc_type` — Short label for the deliverable (e.g., white paper, research article, memo).
   - `thesis` — One or two sentences: central claim or promise.
   - `constraints` — Length, tone, confidentiality, deadlines, tooling.
   - `forbidden_topics` — Topics or claims to avoid (list).
   - `preferred_mode` — If the user already knows: `white_paper`, `data_paper`, `hybrid`, or `null` for automatic routing later.

   Align field names and types with `templates/manifests/intent.schema.json`. Do not invent keys that the validator rejects.

3. Run initialization:

```bash
python scripts/init_workspace.py "<workspace_path>"
```

4. Copy user materials into **`inputs/raw_materials/`** (under the workspace root). Preserve relative structure when copying trees.

5. Write or update **`intent.json`** at the workspace root from user responses. If `ip_policy.json` exists, ensure forbidden terms and code-name lists reflect organizational policy (see `templates/manifests/ip_policy.schema.json`).

6. Append a session line to **`logs/run_log.md`** (init script may already timestamp).

### Exit condition

- Workspace directory exists with canonical layout (`inputs/`, `planning/`, `citations/`, `figures/`, `drafts/`, `reviews/`, `final/`, `exports/`, `logs/`, etc., per `init_workspace.py`).
- `intent.json` populated for routing and scanning.
- Materials copied when provided.

---

## STAGE 1: PREP AND NORMALIZATION (all modes)

### Entry condition

Stage 0 complete; raw materials (if any) live under `inputs/raw_materials/`.

### Required inputs

- User idea (from invocation or Stage 0).
- Raw materials directory (may be empty).

### Protocol

1. **Inventory** — Scan `inputs/raw_materials/` and classify files (markdown, PDF, images, tabular data, code, LaTeX, BibTeX, notebooks, other).

2. **Synthesize canonical inputs** (read raw files; compose structured versions):

   - **`inputs/idea.md`** — Core contribution, research questions, key claims, novelty framing.
   - **`inputs/experimental_log.md`** — Experiments, parameters, results summary, observations, links into raw materials. **For `white_paper` mode this file may be minimal or marked optional** after normalization; still create the file if missing so downstream scripts can assume it exists (use `<!-- GAP -->` or `N/A` as appropriate).
   - **`inputs/materials_manifest.json`** — Per-file inventory: path, type, description, relevance; plus gaps. Shape per project conventions / manifest schema.
   - **`inputs/venue_profile.md`** — Venue or outlet, page limits, format, audience, evaluation criteria.
   - **`inputs/figures_manifest.json`** — Existing and planned figures/tables: paths, descriptions, regeneration needs.

3. For incomplete synthesis, write partial content and mark gaps with `<!-- GAP: description -->`. Ask the user **only** for gaps that block the next gate.

4. **Prep and classification pipeline** — Run in order:

```bash
python scripts/prep_materials.py workspace/
python scripts/classify_materials.py workspace/
python scripts/route_mode.py workspace/
python scripts/validate_inputs.py workspace/
```

   - `prep_materials.py` — Refreshes inventory signals used by prep (materials scan).
   - `classify_materials.py` — Writes **`inputs/materials_passport.json`** (material-level typing and signals). See `templates/manifests/materials_passport.schema.json`.
   - `route_mode.py` — Writes **`route.json`**: recommended `selected_mode`, `confidence`, `reasons`, optional `user_override`. See `templates/manifests/route.schema.json`.
   - `validate_inputs.py` — Validates canonical inputs and manifests.

### Exit condition

- Canonical input artifacts exist under `inputs/` (and `materials_passport.json`, `route.json` updated).
- `validate_inputs.py` exits 0 (warnings may be OK per script contract; **errors block**).

### Human Checkpoint 1

Present:

- What was synthesized vs. copied verbatim.
- Gaps and assumptions.
- Ask: whether normalized inputs match the project; whether critical materials are missing.

**Wait for explicit approval before the routing checkpoint.**

---

### Routing checkpoint (all modes)

After Checkpoint 1 approval:

1. Read **`route.json`** and present verbatim-style summary:

   `Recommended mode: <white_paper | data_paper | hybrid> (confidence: X.XX). Reasons: [bullet list].`

2. Ask: **Override?** If the user chooses a different mode:
   - Set **`user_override`** in `route.json` to that mode (and keep `reasons` documenting the override).
   - If no override, treat `selected_mode` as confirmed.

3. **Do not proceed** into mode-specific stages until the user confirms routing (including “accept recommendation”).

4. Log the decision in `logs/checkpoints.md` (append row: stage, status, timestamp).

---

## MODE: WHITE PAPER

Use when **`route.json`** selects `white_paper` (or user override). Emphasize narrative, executive clarity, and strategic framing over primary-data novelty.

### Stage W2: Perspective Discovery

**Goal:** Map viewpoints before locking argument structure.

**Protocol:**

1. Scan `inputs/raw_materials/`, `idea.md`, and available external literature (search tools as available: Semantic Scholar, arXiv, PubMed, etc.).
2. Write **`planning/perspectives.json`**: array of objects with `perspective_id`, `viewpoint`, `key_arguments`, `source_materials` (paths or IDs), `relevance`. Keep IDs stable for later claim mapping.
3. Write **`planning/question_tree.json`**: structured questions the paper must answer, **grouped by planned section** (nested object or section-keyed arrays — be consistent and document the top-level shape in a one-line comment in a README under `planning/` only if the repo already uses that pattern; otherwise rely on a single JSON structure you choose and reference in `outline.json` later).

**Exit:** Both files exist; at least three distinct perspectives or explicit user waiver recorded in `logs/run_log.md`.

### Stage W3: Argument Structure

**Goal:** Thesis, counterthesis, and claim ledger aligned to perspectives.

**Protocol:**

1. Formulate **thesis** and **strongest counterthesis** (short prose in `planning/argument_graph.md` header or dedicated `planning/thesis.md` if you add it — prefer **`planning/argument_graph.md`** for a single source of truth).
2. Map claims to perspectives; plan how the ledger will support narrative sections.
3. Write **`planning/argument_graph.md`** — Textual / ASCII representation of argument flow (sections → claims → dependencies). Keep it maintainable.
4. Write **`planning/claim_ledger.json`** using **extended claim types**: `fact`, `interpretation`, `opinion`, `forecast` (plus any fields required by `templates/manifests/claim_ledger.schema.json`). Link claims to `perspective_id` where relevant.

**Gate C — Claim ledger build:**

```bash
python scripts/build_claim_ledger.py workspace/
```

Block on script failure; fix JSON and claims until pass.

### Stage W4: Outline and Section Drafting

**Goal:** Markdown-first draft for executive readability.

**Protocol:**

1. Generate **`planning/outline.json`** using the **same logical schema as data paper** (sections with ids, purposes, key points, length estimates, dependencies on figures/citations — cite schema or mirror existing workspace examples; avoid duplicating full schema here).
2. Draft **`drafts/paper.md`** (Markdown, **not** LaTeX for the main white-paper path).
3. Use **citation-style references** `[Author, Year]` in prose; maintain a parallel list or small table in `drafts/reference_list.md` if full `refs.bib` is not yet built.
4. **Forbidden:** unexplained jargon, unsupported claims vs. ledger, internal code names (also covered by IP scan).
5. Optional: short `drafts/sections/` splits for long work.

**Gate:** Outline sections cover `question_tree.json`; every non-trivial claim traces to `claim_ledger.json`.

### Stage W5: Review

**IP scan:**

```bash
python scripts/scan_redlines.py workspace/
```

Writes **`reviews/ip_safety_report.md`** (or path emitted by script — use actual output). **IP Gate:** If violations found, **BLOCK** shipping; do not mark stage complete until cleared, redacted, or explicitly acknowledged per Error Handling.

**Hostile review** — Read `drafts/paper.md` as an external critic. Write **`reviews/hostile_review.md`**: logical gaps, unsupported assertions, competitive risk, clarity, factual risk.

**Standard review** — Write **`reviews/review_round_1.md`** and **`reviews/scorecard.json`** (same dimensions as data paper: coherence, support, citations/clarity as applicable, venue fit if relevant). Use numeric subscores and `overall`.

**Gate G — Quality non-regression:** On repair rounds, if `overall` drops vs. prior best, **revert** to the better draft. **Max 3** review rounds; then keep best-scoring artifact and document residual issues.

### Stage W6: Executive Outputs

1. **`exports/executive_summary.md`** — ~1 page for leadership: thesis, why it matters, risks, asks.
2. **`exports/bd_talking_points.md`** — Bullets for business development conversations (customer pain, differentiation, proof, objections).
3. **Human Checkpoint 3** — Before packaging: confirm BD/summary tone, sensitive claims, and deferrals.

---

## MODE: DATA PAPER

Use when **`route.json`** selects `data_paper`. This path preserves the **original evidence-forward pipeline** (LaTeX draft, BibTeX, figures, compile).

### Stage D2: Global Planning

**Entry condition:** Checkpoint 1 and routing approved; canonical inputs available.

**Required inputs:** `inputs/idea.md`, `inputs/experimental_log.md`, `inputs/venue_profile.md`, `inputs/figures_manifest.json`.

**Protocol:**

1. Read canonical inputs thoroughly.

2. Generate **`planning/outline.json`** — Working title; `sections` array with per-section `id`, `name`, `purpose`, `key_points`, `estimated_length`, `depends_on_figures`, `depends_on_citations`; plus `narrative_arc` string. Conform to `templates/manifests/` outline expectations if present.

3. Generate **`planning/claim_ledger.json`** — Claims with `id`, `statement`, `type` (e.g., quantitative / qualitative / methodological per your schema), `evidence_source`, `support_level` (`strong` | `moderate` | `weak` | `unsupported`), `depends_on_figures`, `depends_on_citations`, `notes`.

4. Generate **`planning/figure_plan.json`** — Per figure: `id`, `description`, `type`, `source` (`existing` | `generate` | `collect`), `source_path` if any, `supports_claims`, `caption_draft`.

5. Generate **`planning/literature_plan.json`** — `search_queries`, `known_references` (title, authors, relevance, bibtex key if known), `coverage_areas` with minimum ref counts and purpose.

6. Generate **`planning/results_inventory.json`** — `results` entries: `id`, `description`, `value`, `source_file`, `supports_claims`, `verified` flag.

**Gate B — Plan completeness:**

- Outline includes standard sections for the venue type.
- Every claim has `support_level`; every `unsupported` claim is explicitly flagged for resolution.
- Figure plan covers figures referenced in the outline.
- Literature plan covers citation needs implied by the outline.

**Exit:** All five planning artifacts exist; no silent unsupported claims.

**Human Checkpoint 2**

Present outline, narrative arc, claim ledger summary (highlight weak/unsupported), figure and literature plans. Ask whether the story, figures, and literature align with intent. **Wait for explicit approval.**

---

### Stage D3: Support Artifact Generation

**Entry condition:** Checkpoint 2 approved.

**Inputs:** All planning artifacts; canonical inputs.

Run three substages (order flexible; complete all before Stage D4).

#### 3A — Evidence accounting

1. Read `experimental_log.md` and `results_inventory.json`.
2. For each claim in `claim_ledger.json`, verify evidence; update `support_level` and notes.
3. Write updated **`planning/claim_ledger.json`**.

**Gate C — Evidence integrity:** Block if any claim remains `unsupported` without user acknowledgment, or if numerical claims lack traceable sources.

#### 3B — Literature discovery and verification

1. Search using `literature_plan.json` queries (Semantic Scholar, Google Scholar, arXiv, PubMed tools as available).
2. For each candidate: title, authors, year, venue, DOI/URL; **verify** existence (DOI or search). No fabricated citations.
3. Deduplicate; rank by relevance to coverage areas.
4. Write **`citations/citation_pool.json`** — entries with `key`, metadata, `verified`, `relevance_area`, `relevance_score` per schema.
5. Generate **`citations/refs.bib`** from **verified** citations only.

**Gate D — Citation integrity:** Block on unverified pool entries or duplicate BibTeX keys.

#### 3C — Figure and table pipeline

1. For each `figure_plan.json` entry:
   - `existing` — verify file; copy to `figures/supplied/` as appropriate.
   - `generate` — produce asset under `figures/generated/`.
   - `collect` — flag for user provision.
2. Write **`figures/captions.json`** — `figure_id`, `path`, `caption`, `exists`.
3. Table LaTeX snippets → `tables/generated/` when needed.

**Gate E — Artifact integrity:** Block if any outline-referenced figure/table is missing and not explicitly deferred; block if required captions are missing.

**Exit:** `citation_pool.json` + `refs.bib` verified; figures/tables exist or deferred; claim ledger final for support levels; Gates C/D/E pass.

---

### Stage D4: Draft Composition

**Entry condition:** Stage D3 complete; Gates C/D/E pass.

**Inputs:** `outline.json`, `claim_ledger.json`, `citation_pool.json`, `refs.bib`, `captions.json`, `venue_profile.md`.

**Protocol:**

1. If `inputs/template.tex` exists, use it as skeleton; else standard article class.

2. Compose **`drafts/paper.tex`** section by section:
   - `\cite{key}` only for keys in `refs.bib`.
   - `\ref{}` / labels consistent with `captions.json` and figure plan.
   - Ground quantitative claims in claim ledger; soften language for `weak` support.
   - Optional section files under `drafts/sections/`.

3. **Forbidden:** invented citations; unverifiable numbers; missing figures; ignoring venue constraints.

**Validation:**

```bash
python scripts/check_artifacts.py workspace/
```

**Gate F — Structural integrity:** Resolve broken cites/refs per script output.

**Exit:** `drafts/paper.tex` exists; check_artifacts passes.

---

### Stage D5: Review and Repair

**Entry condition:** Draft exists; Gate F passed.

**Protocol:**

1. **IP scan (added requirement):**

```bash
python scripts/scan_redlines.py workspace/
```

**IP Gate:** If violations found, **BLOCK** completion of review until addressed per Error Handling.

2. **Review pass** — Write **`reviews/review_round_1.md`**: argument flow, claim support, citations, figures/tables, clarity, venue fit, line-level suggestions.

3. **Scorecard** — **`reviews/scorecard.json`**: `round`, `scores` (argument_coherence, evidence_support, citation_quality, writing_clarity, venue_fit, overall), `blocking_issues`, `suggestions`.

4. **Repair** if blocking issues; save revised draft.

5. **Re-score**; **Gate G:** if `overall` drops, revert to pre-revision version.

6. **Max 3** rounds; keep best-scoring draft.

**Exit:** At least one round; best draft selected; blocking issues resolved or explicitly deferred by user.

---

### Stage D6: Finalize

**Human Checkpoint 3**

Present final scorecard, deferred issues, softened claims. Ask whether the draft is acceptable for packaging. **Wait for explicit approval.**

**Entry condition:** Checkpoint 3 approved.

**Protocol:**

1. Copy accepted draft to **`final/paper.tex`**; **`final/refs.bib`** from `citations/refs.bib`; figures to **`final/figures/`**; tables to **`final/tables/`**.

2. Compile:

```bash
bash scripts/compile_package.sh workspace/
```

3. On failure: report errors; attempt minimal fixes; re-run. If still failing, deliver source bundle without PDF and document why.

4. On success: ensure `final/` contains tex, pdf (if built), bib, figures, tables.

5. Update **`logs/run_log.md`**.

**Exit:** Submission-ready package in `final/` or documented partial export.

---

## MODE: HYBRID

Use when **`route.json`** selects `hybrid`. Combine executive narrative with evidence-heavy sections.

### Stage H2: Conceptual framing

Follow **W2** and **W3** for the conceptual front-half:

- `planning/perspectives.json`
- `planning/question_tree.json`
- `planning/argument_graph.md`
- `planning/claim_ledger.json` (extended types as in white paper)
- Gate C: `python scripts/build_claim_ledger.py workspace/`

### Stage H3: Evidence support

Follow **D3** (3A/3B/3C) for evidence-backed sections:

- Updated `claim_ledger.json` where it overlaps evidence sections (merge carefully; single ledger file — use claim `section` or `track` field if schema allows).
- `citations/citation_pool.json`, `citations/refs.bib`
- Figures, `figures/captions.json`, `tables/generated/` as needed

### Stage H4: Merged outline and draft

1. **`planning/outline.json`** — Single outline covering conceptual and evidence sections; mark section `track`: `conceptual` | `evidence` in metadata if helpful (or parallel arrays linked by order — stay consistent).

2. **Drafting split (user choice):**
   - **Option A:** Conceptual sections in **`drafts/paper.md`**, evidence sections in **`drafts/paper.tex`**, with cross-references documented in `drafts/README_hybrid.md` (short: how sections stitch).
   - **Option B:** One format only — full Markdown **or** full LaTeX — user decides at Checkpoint 2.

3. Run **`check_artifacts.py`** on any LaTeX portions before review.

**Human Checkpoint 2** — After merged outline (and optionally partial drafts): approve structure and format choice.

### Stage H5: Dual review

1. **IP scan:**

```bash
python scripts/scan_redlines.py workspace/
```

**IP Gate:** BLOCK on violations.

2. **Hostile review** — Conceptual parts → **`reviews/hostile_review.md`**.

3. **Evidence review** — Data-heavy parts → **`reviews/review_round_*.md`** + **`reviews/scorecard.json`**.

4. **Both** conceptual and evidence quality gates must pass before finalize (no “average failure” — fix or defer with user sign-off).

5. **Gate G** — Same non-regression rule; max 3 rounds.

### Stage H6: Combined export

1. **Human Checkpoint 3** — Approve packaging, sensitive content, and split vs. unified artifacts.

2. **Artifacts:** `paper.md` and/or `paper.tex`, `executive_summary.md`, `bd_talking_points.md`, `refs.bib`, figures, `reviews/ip_safety_report.md`, `reviews/hostile_review.md`, scorecards.

3. **Compile** LaTeX portions:

```bash
bash scripts/compile_package.sh workspace/
```

4. Ensure `exports/` and `final/` reflect the hybrid bundle policy your `package_exports.py` implements.

---

## SHARED EXIT: FINAL PACKAGING (all modes)

After mode-specific completion and Human Checkpoint 3:

```bash
python scripts/package_exports.py workspace/
```

This assembles a **mode-appropriate bundle** under **`final/`** (and/or `exports/` per script). Do not hand-wave paths — read script stdout if it prints the output directory.

**Report to the user:**

- Absolute workspace path.
- Enumerate artifacts produced (inputs, planning, citations, figures, drafts, reviews, exports).
- Final review score (`overall` from latest scorecard, or state N/A for scoreless paths with user sign-off only).
- Known issues and deferred items.
- Next steps (proofread, legal review, submission, iteration).

---

## ERROR HANDLING

### General gate failure

1. State **which gate** (B–G, IP, etc.) failed and **why**.
2. List **failing artifacts** or checks (script name + stderr summary).
3. Propose **concrete fixes** (edit file X, add citation Y, regenerate figure Z).
4. Ask: auto-repair attempt vs. pause for manual edit.

### Stage blocked

1. Log blockers in **`logs/run_log.md`**.
2. Preserve all artifacts produced so far.
3. Give **resume instructions** (which stage to re-run; see Stage Routing below).

### IP gate failure (extended)

1. List **forbidden terms or patterns** found (from `ip_safety_report` / scan output).
2. Suggest **redactions** or rephrasing per `ip_policy.json`.
3. Ask the user: **redact**, **rewrite**, or **acknowledge risk** (acknowledgment must be explicit and logged in `logs/run_log.md` if policy allows).

### Hostile review — critical issues

1. List **critical** findings separately from nitpicks.
2. Suggest **revisions** mapped to sections or claim IDs.
3. Ask whether to **revise now** or **defer** with documented risk.

---

## STAGE ROUTING / RESUME

On **re-invocation** with an existing workspace:

1. Read **`route.json`** → `selected_mode` and `user_override` to determine **effective mode**.
2. Read **`logs/checkpoints.md`** (and `logs/run_log.md` if needed) → last **completed** stage and checkpoint status.
3. Offer to **resume** from the **next** stage on the correct **mode branch** (W*, D*, H*).
4. If the user requests **re-run** a specific stage, regenerate only that stage’s outputs after warning about downstream invalidation.
5. If the user requests a **mode change**, update **`route.json`** (set `user_override` and reasons), then re-run **`route_mode.py`** only if you need refreshed confidence — or proceed per user instruction; **log** the change.

**Consistency:** After each major stage completion, append **`logs/checkpoints.md`** with stage id, mode, timestamp, and “pass/fail”.

---

## Quick reference: scripts

| Script | Purpose |
|--------|---------|
| `init_workspace.py` | Create tree and seeds |
| `prep_materials.py` | Scan / prep materials inventory |
| `classify_materials.py` | `materials_passport.json` |
| `route_mode.py` | `route.json` |
| `validate_inputs.py` | Validate inputs |
| `build_claim_ledger.py` | Validate / build claim ledger gate |
| `check_artifacts.py` | LaTeX cite/ref integrity |
| `scan_redlines.py` | IP / policy redlines |
| `compile_package.sh` | PDF package (pdflatex/bibtex if available) |
| `package_exports.py` | Assemble `final/` bundle |

---

## Schemas and templates

Authoritative JSON shapes live under **`templates/manifests/`** — e.g. `intent.schema.json`, `route.schema.json`, `materials_passport.schema.json`, `claim_ledger.schema.json`, `ip_policy.schema.json`. **Do not** paste full schemas into chat outputs unless debugging; **do** validate artifacts against them when scripts enforce validation.

---

## Conventions (from project CLAUDE.md)

- Artifacts are markdown, JSON, BibTeX, LaTeX — figures are the primary binary exception.
- Three human checkpoints: after prep (Checkpoint 1), after planning / merged outline (Checkpoint 2 for data & hybrid; white paper uses W-planning approval as the planning gate before W4), before final export (Checkpoint 3). **Always wait** where the workflow says “wait for approval.”
- `compile_package.sh` may be absent or fail without local TeX; degrade gracefully and still ship sources.

---

## Mode → stage map

| Stage | white_paper | data_paper | hybrid |
|-------|-------------|------------|--------|
| Prep | Stage 1 | Stage 1 | Stage 1 |
| Routing | RC | RC | RC |
| Planning / framing | W2–W3 | D2 | H2 + D3 prep |
| Support artifacts | (ledger only) | D3 | H3 |
| Draft | W4 | D4 | H4 |
| Review | W5 | D5 | H5 |
| Export / finalize | W6 + package | D6 + package | H6 + package |

**RC** = routing checkpoint after Stage 1.

---

## Artifact checklist before package_exports (typical)

**All modes:** `intent.json`, `route.json`, `reviews/ip_safety_report.md` (post-scan), `logs/checkpoints.md`, latest `scorecard.json` if scoring used.

**Data / hybrid LaTeX:** `drafts/paper.tex` or `final/paper.tex`, `citations/refs.bib`, `figures/` populated per plan.

**White / hybrid Markdown:** `drafts/paper.md`, `exports/executive_summary.md`, `exports/bd_talking_points.md` when mode requires.

---

## Forbidden behaviors (global)

- Skipping **`scan_redlines.py`** before declaring review complete for any mode.
- Proceeding past **IP Gate** with unaddressed violations without logged user decision.
- Inventing citations or figures not in verified pools / captions.
- Mixing up workspace roots (always use the user-provided `workspace/` path consistently in commands).

---

## Logging discipline

For every session:

- **`logs/run_log.md`** — decisions, overrides, tool failures, user waivers.
- **`logs/checkpoints.md`** — table rows for stage completion and human approvals.

This enables **deterministic resume** and auditability (“why did we ship this draft?”).

---

## White paper: Gate C vs Gate B

- **Gate B** is the **data-paper** planning completeness check (D2).
- **Gate C** applies to **claim ledger validation** after ledger edits (W3, D3 3A, hybrid H2/H3).
- Do not conflate: white paper skips D2’s Gate B unless you merge tracks; hybrid uses both where applicable.

---

## Data paper: review dimensions (scorecard)

When writing **`reviews/scorecard.json`**, keep dimensions **comparable across rounds**:

- `argument_coherence`
- `evidence_support`
- `citation_quality`
- `writing_clarity`
- `venue_fit`
- `overall` (numeric; same scale each round)

Store `blocking_issues` as strings; empty list when clear.

---

## Hybrid: stitching narrative

When **`drafts/paper.md`** and **`drafts/paper.tex`** coexist:

1. Define a single **ordering** in `outline.json` (section ids in sequence).
2. For PDF compilation, either: (a) include Markdown sections as quoted blocks in a wrapper (if your pipeline supports it), or (b) convert MD sections to LaTeX for a single build — **user choice at H4**.
3. Always run **`check_artifacts.py`** on the LaTeX artifact before calling compilation.

---

## compile_package.sh behavior

Invoke from repo root with workspace path. If PDF build is optional:

- Success → report path to `final/paper.pdf` if present.
- Failure → retain `.tex`/`.bib`/figures; document missing PDF in user report.

---

## validate_inputs.py: when to re-run

Re-run **`validate_inputs.py`**:

- After editing any canonical `inputs/*` file.
- After `classify_materials` / `route_mode` if inputs change.
- Before leaving Stage 1.

---

## check_artifacts.py: when to re-run

Re-run **`check_artifacts.py`**:

- After every substantive edit to `drafts/paper.tex`.
- Before Stage D5/D6 and hybrid H5/H6 for LaTeX bodies.

---

## package_exports.py: expectations

After **`package_exports.py`**:

- Confirm output directory listing for the user.
- If the script returns non-zero, treat as **packaging gate failure** — same as Error Handling.

---

## Perspective discovery: minimum quality bar

For **`perspectives.json`**:

- Each perspective should cite **at least one** source material or external reference id.
- `relevance` should be one of `high` | `medium` | `low` or a numeric score — **pick one scheme** and stick to it for the workspace.

---

## Question tree: section linkage

Each major section in **`outline.json`** should map to one or more nodes in **`question_tree.json`** so reviewers can trace “which questions does Section 3 answer?”

---

## Executive outputs: tone

**`executive_summary.md`** — short paragraphs, no unexplained acronyms; link to detailed draft path.

**`bd_talking_points.md`** — bullets, no confidential metrics unless explicitly approved in `intent.json` constraints.

---

## Resumption message template

When user returns:

> Workspace: `<path>`  
> Mode (effective): `<mode>`  
> Last completed stage: `<id>`  
> Next recommended: `<id>` — `<one-line description>`  
> Say **continue**, **redo &lt;stage&gt;**, or **change mode to &lt;mode&gt;**.

---

## Final notes

- **Branching** is mode-driven; **quality bars** are shared.
- Prefer **scripts** over manual JSON hacking when a validator exists.
- If unsure between modes, trust **`route.json`** confidence **unless** the user overrides — then trust **`user_override`**.

---

*End of PaperOrchestra skill definition.*

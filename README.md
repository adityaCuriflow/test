# Nemo Readiness — Complete Deep Dive

> Internal name in the codebase: **"codeability"**. Renamed to **"Nemo Readiness"** in the UI on 2026-09-17 (commit `51699ae25`). Legacy aliases `codeability` → `readiness` and `improve` → `polish` are still mapped for backward compatibility.

---

## Executive Summary

- **"Readiness" = "Nemo Readiness"** — a 0–100 score measuring how ready a repository is for agentic (Nemo-driven) development. Two axes: **Axis A (Repository readiness, /65)** and **Axis B (Skill & knowledge coverage, /35)**.
- The feature lives under the **"Nemo Readiness" tab** on the **Project Detail Page** (`web/ui/.../ProjectDetailPage.tsx`). It is hidden for user-workspace projects.
- The actual scoring is done by an **external Python script** (`scan.py`) in the **CuriPowers** repo (`$NEMO_CURIPOWERS_ROOT`), NOT in this codebase. Nemo dispatches a real agent run (workflow intent `readiness`) that invokes the script and writes `codeability.json`.
- The backend **recomputes all scores from the raw metric `points`** — it never trusts the totals the scorecard claims. Band thresholds: **0–39 Red, 40–69 Amber, 70–100 Green**.
- Scans run as **real Issues** (so their live transcript shows in the UI). The heartbeat's terminal hook (`_maybe_finalize_codeability_or_bootstrap` in `heartbeat.py`) ingests results asynchronously when the run finishes.
- A **scheduled re-scan poller** (`scheduler.py`) runs as a background `asyncio` task during app lifespan, re-scanning projects every 7 days (configurable via `NEMO_CODEABILITY_RESCAN_DAYS`).
- Empty repos (README/docs only, no source) get a **deterministic 0-score** without burning an agent run; they route to a PRD→skills flow instead.
- Three flows share the Readiness surface: **(1) Scan** (read-only scoring), **(2) Polish / Auto-Bootstrap** (agents open PRs to raise the score), **(3) PRD→skills** (two-phase: plan → user picks → author).
- Permissions: scan = admin/FDE/maintainer; Polish/PRD→skills = admin-only. Backend is authoritative via `check_codeability_scan_authorized`.
- Real-time updates via **WebSocket invalidation bus** (`/ws/invalidations`) + **React Query polling** fallback (4s for scans, 5s for bootstrap/PRD→skills).

---

## 1. What is "Readiness"?

**Readiness** in this application means **"Nemo Readiness"** — a quantitative measure of how well-prepared a repository is for agentic/autonomous development. It answers: "If I point Nemo at this repo, will it succeed?"

### Problem it solves
Before Readiness, there was no way to know whether a repo had the scaffolding (skills, quality gates, env docs, tests, golden tasks) that an agent needs to work effectively. A repo might clone fine but lack a lint config, an `.env.example`, or a repo-map skill — and the agent would flail.

### Main concepts/entities
| Concept | Meaning |
|---|---|
| **CodeabilityScan** | One scoring run of a project's repo (0–100, append-only history) |
| **Axis A** | "Repository readiness" — repo-level scaffolding (max 65 as of scanner 1.1) |
| **Axis B** | "Skill & knowledge coverage" — authored skills/docs coverage (max 35) |
| **Band** | Red (0–39) / Amber (40–69) / Green (70–100) — derived from score |
| **Metric** | One scoring item within an axis: `{key, label, points, max, assessedMax, evidence[]}` |
| **BootstrapRun** | One "Polish"/"Improve" run — agents open PRs to raise the score |
| **PrdSkillsRun** | Two-phase PRD→skills flow for empty repos (planning → authoring) |
| **g-nemo-readiness** | The external scoring skill (lives in CuriPowers, not this repo) |
| **g-polish-codebase** | The external "Improve" workflow skill (lives in CuriPowers) |

### Where in the product
- **Project Detail Page** → "Nemo Readiness" tab (`web/ui/src/features/projects/pages/ProjectDetailPage.tsx`, line 426)
- Tab value: `"codeability"` (line 471–472)
- Hidden for user-workspace projects

### User actions that trigger/affect Readiness
| Action | Effect |
|---|---|
| Create a GitHub project | Background `_do_clone` → auto-triggers onboarding scan |
| Click "Run analysis" (empty state) | Manual scan (`trigger="manual"`) |
| Click "Re-assess" (scorecard) | Re-pull repo + manual scan |
| Click "Polish" (admin) | Auto-Bootstrap: agents open PRs to raise score |
| Upload a requirements document (empty repo) | PRD detection → PRD→skills flow |
| Click "Generate skills" (empty repo + PRD) | Two-phase PRD→skills: plan → author → PR |
| Scheduled poller fires | Auto re-scan every 7 days (`trigger="scheduled"`) |

---

## 2. Frontend

### Page structure
The Readiness UI is a **tab** on `ProjectDetailPage`. It is always present for non-user-workspace projects (line 111–112):

```tsx
? ["issues", "chat", "files", "overview", "sessions", "members", "codeability", "secrets", "integrations"]
: ["issues", "files", "overview", "sessions", "members", "codeability", "secrets", "integrations"];
```

Tab label (line 426):
```tsx
...(isUserWorkspace(project) ? [] : [{ label: "Nemo Readiness", value: "codeability" as const }]),
```

Two banners render above the tab content (lines 409, 418):
```tsx
<PrdDetectedBanner projectId={project.id} repoLabel={...} />
{!isUserWorkspace(project) && <AddPrdBanner projectId={project.id} />}
```

### Component tree
```
ProjectDetailPage
├── PrdDetectedBanner          (empty repo + PRD detected → "Generate skills")
├── AddPrdBanner               (empty repo + no PRD → "Add a document")
└── CodeabilityTab             (tab="codeability")
    ├── SkillsPrCard           (PR links from PRD→skills run)
    ├── CodeabilityScorecard   (the actual scorecard)
    │   ├── CodeabilityGauge   (circular gauge 0–100)
    │   ├── Band chip          (Red/Amber/Green label)
    │   ├── AxisBreakdown ×2   (Axis A + Axis B, per-metric bars)
    │   │   └── MetricBar      (individual metric with tooltip)
    │   └── Action buttons     (Re-assess, Polish, View run)
    └── AutoBootstrapDialog    (Improve / PRD→skills modal)
```

### What the user sees by state

| Scan status | What renders |
|---|---|
| `loading` | Skeleton placeholders |
| `none` (never scanned) | EmptyState: "No Nemo readiness score yet" + "Run analysis" button (if authorized) |
| `queued` / `running` | BreathingDot + "Analyzing repository…" + "View live run" link to issue |
| `failed` | AlertTriangle + "Nemo Readiness scan failed" + error text + "Retry scan" |
| `succeeded` | Full scorecard: gauge + band + axis breakdowns + actions |

### The scorecard (`CodeabilityScorecard.tsx`)
- **Gauge**: `CodeabilityGauge` — circular gauge showing the 0–100 score with band color
- **Band chip**: Red/Amber/Green label with tooltip ("Readiness band, from the 0–100 score: Red 0–39, Amber 40–69, Green 70–100")
- **Header summary**: "Nemo Readiness" title + "Repository readiness 45/65 · Skill coverage 27/35"
- **Two axis breakdowns** (`AxisBreakdown`): one section per axis, each with a header (`label` + `score/max`) and a list of `MetricBar`s
- **MetricBar**: label (truncated) + progress bar (filled by `points/assessedMax` ratio) + `points/assessedMax` in monospace. Tooltip shows evidence (first 4 items). Unmeasured metrics show "—".
- **Empty-repo banner** (inside scorecard): "This repo has no code yet, so it scores 0. Add a requirements document above, or push one to the repo and Re-assess."
- **Actions**: "Re-assess" (admins/FDE/maintainers) and "Polish" (admins only)

### Key frontend files

| File | Purpose | Key exports |
|---|---|---|
| `web/ui/src/features/codeability/api.ts` | API client + TypeScript types mirroring the backend contract | `codeabilityApi`, `Scorecard`, `Axis`, `Metric`, `ScanStatus`, `BootstrapState`, `PrdSkillsState`, `isScanActive`, `isBootstrapActive`, `isPrdRunActive` |
| `web/ui/src/features/codeability/hooks.ts` | React Query hooks (queries + mutations) | `useCodeability`, `useTriggerScan`, `useBootstrap`, `useBootstrapStatus`, `usePrdToSkillsPlan`, `usePrdToSkills`, `usePrdToSkillsStatus`, `useUploadSpecDocuments`, `useDeleteSpecDocument`, `codeabilityKeys` |
| `web/ui/src/features/codeability/band.ts` | Pure band/grade/tone helpers (DOM-free, unit-testable) | `scoreToBand`, `bandTone`, `bandGrade`, `ratioTone`, `Band`, `BandTone` |
| `web/ui/src/features/codeability/components/CodeabilityTab.tsx` | State machine: loading → skeleton; none → CTA; running → live; failed → retry; succeeded → scorecard | `CodeabilityTab` |
| `web/ui/src/features/codeability/components/CodeabilityScorecard.tsx` | The scorecard UI: gauge + band + axis breakdowns + actions | `CodeabilityScorecard`, `AxisBreakdown`, `MetricBar` |
| `web/ui/src/features/codeability/components/CodeabilityGauge.tsx` | Circular gauge visual | `CodeabilityGauge` |
| `web/ui/src/features/codeability/components/AutoBootstrapDialog.tsx` | "Polish" modal (code repo) + "Generate skills" modal (empty repo + PRD) | `AutoBootstrapDialog`, `projectedScore` |
| `web/ui/src/features/codeability/components/SkillsPrCard.tsx` | PR links card after PRD→skills run succeeds | `SkillsPrCard` |
| `web/ui/src/features/codeability/components/AddPrdBanner.tsx` | Empty repo + no PRD → "Add a document" upload prompt | `AddPrdBanner` |
| `web/ui/src/features/codeability/components/PrdDetectedBanner.tsx` | Empty repo + PRD → "Generate skills" prompt | `PrdDetectedBanner` |
| `web/ui/src/features/codeability/components/SpecDocumentsDialog.tsx` | Manage the PRD/multi-doc set (list, remove, add) | `SpecDocumentsDialog` |

### Frontend state management
- **React Query** (`@tanstack/react-query`) for all server state
- Query keys centralized in `codeabilityKeys`:
  - `["codeability", projectId, "scorecard"]`
  - `["codeability", projectId, "bootstrap"]`
  - `["codeability", projectId, "prd_to_mvp"]`
- The backend's WebSocket invalidation bus publishes the **same key arrays**, so a running scan transitions live
- **Polling fallback**: `refetchInterval` = 4000ms while scan is active, 5000ms while bootstrap/PRD→skills is active
- **Optimistic updates**: `useTriggerScan` sets `status: "queued"` immediately on success; `useBootstrap` sets `status: "running"`
- **Permissions**: `usePermissions()` returns `{ isAdmin, isFde, isMaintainer }`; the frontend hides/shows buttons based on these, but the backend stays authoritative

### API endpoints called from frontend
```ts
codeabilityApi.scorecard(projectId)       // GET  /projects/{id}/codeability
codeabilityApi.scan(projectId)           // POST /projects/{id}/codeability/scan
codeabilityApi.bootstrap(projectId, areas?) // POST /projects/{id}/bootstrap
codeabilityApi.bootstrapStatus(projectId)  // GET  /projects/{id}/bootstrap
codeabilityApi.prdToSkillsPlan(projectId, prdPath?)  // POST /projects/{id}/prd-to-skills/plan
codeabilityApi.prdToSkills(projectId, prdPath?, selectedSkills?) // POST /projects/{id}/prd-to-skills
codeabilityApi.prdToSkillsStatus(projectId)  // GET  /projects/{id}/prd-to-skills
```

### Loading/error/empty states
- **Loading**: `<Skeleton>` placeholders in `CodeabilityTab` (lines 43–53)
- **Empty** (never scanned): `<EmptyState>` with "No Nemo readiness score yet" + optional "Run analysis" CTA (lines 57–67)
- **Running**: `<BreathingDot>` + "Analyzing repository…" + "View live run" link (lines 69–88)
- **Failed**: `<AlertTriangle>` + error text + "Retry scan" button (lines 91–105)
- **Succeeded**: Full scorecard + `SkillsPrCard` + `AutoBootstrapDialog` (lines 108–129)

---

## 3. Backend

### API routes (`web/backend/app/api/codeability.py`)

| Method | Path | Handler | Auth | Purpose |
|---|---|---|---|---|
| GET | `/projects/{id}/codeability` | `get_codeability` | token only | Latest scorecard or `{status: "none"}` shell |
| POST | `/projects/{id}/codeability/scan` | `trigger_codeability_scan` | admin/FDE/maintainer | Trigger a (re)scan; 202 on success, 409 if clone not ready |
| POST | `/projects/{id}/prd-to-skills/plan` | `trigger_prd_to_skills_plan` | admin only | Phase 1: read-only planning agent proposes skills |
| POST | `/projects/{id}/prd-to-skills` | `trigger_prd_to_skills` | admin only | Phase 2: author selected skills + open PR |
| GET | `/projects/{id}/prd-to-skills` | `get_prd_to_skills` | token only | Latest PRD→skills run status |
| GET | `/projects/{id}/bootstrap` | `get_bootstrap` | token only | Latest Auto-Bootstrap status |
| POST | `/projects/{id}/bootstrap` | `trigger_bootstrap` | admin only | Trigger Polish/Improve; 202 |

All routes registered on an `APIRouter(tags=["codeability"], dependencies=[Depends(require_token)])`.

### Backend flow — scan trigger
```
Frontend POST /projects/{id}/codeability/scan
  → trigger_codeability_scan (api/codeability.py:117)
    → check_codeability_scan_authorized (auth/authz.py:805)  [admin/FDE/maintainer]
    → codeability_svc.start_scan(project_id, trigger="manual", ...)
      → load Project, check repo_clone_status == "ready"
      → best-effort pull_repo (refresh canonical clone to default branch)
      → classify_repo(repo_dir)  [empty? → record_empty_repo_scorecard, no agent]
      → store.active_scan_for_project  [dedupe: if in-flight, return existing]
      → store.create_scan  [insert CodeabilityScan row, status="running"]
      → iss.create_issue(title="Nemo Readiness", workflow_intent="readiness", status="in_progress")
      → dispatch_issue(issue, project, invocation_source="codeability_scan")
        → spawns Hermes agent run with the _CODEABILITY_CONTRACT prompt
      → link issue_id + run_id to the CodeabilityScan row
      → return scan dict
    → 202 response
```

### Backend flow — scan completion (async)
```
Agent run finishes (succeeded/failed/timed_out/paused)
  → heartbeat detects terminal status
  → _maybe_finalize_codeability_or_bootstrap(run_id)  (heartbeat.py:813)
    → branches on run.invocation_source
    → if "codeability_scan": ingest_scan_result(run_id)  (codeability/service.py:380)
      → load Run, find CodeabilityScan by run_id
      → if run.status != "succeeded": _mark_failed + publish invalidation + return
      → read codeability.json from run.cwd (the issue's workspace)
      → parse_scorecard(raw)  [validate + recompute scores from metric points]
      → persist: scan.status="succeeded", score, band, axis scores, scorecard_json
      → update project: repo_state="code", latest_codeability_scan_id, next_codeability_scan_at = now + 7d
      → _publish(project_id, org_id)  [WebSocket invalidation: ["codeability", project_id, "scorecard"]]
```

### Key backend files

| File | Purpose |
|---|---|
| `web/backend/app/api/codeability.py` | HTTP routes + request/response shaping |
| `web/backend/app/services/codeability/service.py` | Core orchestration: `start_scan`, `ingest_scan_result`, `record_empty_repo_scorecard`, `parse_scorecard`, `_axis_score` |
| `web/backend/app/services/codeability/store.py` | SQLModel persistence helpers: `create_scan`, `latest_for_project`, `active_scan_for_project`, `by_run`, `org_has_succeeded_scan` |
| `web/backend/app/services/codeability/scheduler.py` | Background re-scan poller: `sweep_due_rescans`, `run_codeability_rescan_poller` |
| `web/backend/app/services/codeability/workspace.py` | Output path helpers: `codeability_workspace_dir`, `codeability_output_path` |
| `web/backend/app/services/codeability/prompt.py` | Issue title/description builder: `CODEABILITY_ISSUE_TITLE`, `build_scan_description` |
| `web/backend/app/services/bootstrap/service.py` | Auto-Bootstrap/Polish orchestration: `start_bootstrap`, `ingest_bootstrap_result` |
| `web/backend/app/services/bootstrap/store.py` | BootstrapRun persistence helpers |
| `web/backend/app/services/issues/workflow.py` | Workflow intent registration + `_CODEABILITY_CONTRACT` (the prompt the agent runs) |
| `web/backend/app/services/runs/heartbeat.py` | Terminal hook: `_maybe_finalize_codeability_or_bootstrap` |
| `web/backend/app/auth/authz.py` | `check_codeability_scan_authorized` — permission gate |
| `web/backend/app/services/projects/repo_inspect.py` | `classify_repo`, `find_prd`, `find_spec_documents`, `find_figma_file` |
| `web/backend/app/core/lifespan.py` | Starts the rescan poller as a background asyncio task |

---

## 4. Database / Data Model

### Tables

#### `codeability_scans` (migration `m5a8c1f3b7e2`, 2026-06-18)

| Column | Type | Notes |
|---|---|---|
| `id` | str (PK) | prefix `cea_` |
| `org_id` | str (FK → orgs.id) | indexed, default `"curiflow"` |
| `project_id` | str (FK → projects.id) | indexed |
| `run_id` | str (FK → runs.id) | nullable, indexed — the agent run |
| `issue_id` | str (FK → issues.id) | nullable, indexed — the auto-created issue |
| `status` | str | `queued` \| `running` \| `succeeded` \| `failed`; indexed |
| `trigger` | str | `onboarding` \| `manual` \| `scheduled` \| `bootstrap_recompute` |
| `score` | int | 0–100 overall, nullable |
| `band` | str | `Red` \| `Amber` \| `Green`, nullable |
| `axis_a_score` | int | repo readiness score, nullable |
| `axis_b_score` | int | skill coverage score, nullable |
| `themes_json` | JSONB | `[]` (legacy, no longer surfaced) |
| `artifacts_json` | JSONB | `{}` (the 10 named artifacts) |
| `scorecard_json` | JSONB | raw `codeability.json` verbatim — forensic source of truth |
| `error` | str | nullable; redacted before display |
| `created_at` / `finished_at` / `updated_at` | float | epoch timestamps |

#### `bootstrap_runs` (same migration)

| Column | Type | Notes |
|---|---|---|
| `id` | str (PK) | prefix `bsr_` |
| `org_id` | str (FK → orgs.id) | |
| `project_id` | str (FK → projects.id) | |
| `run_id` | str (FK → runs.id) | the agent run that opens PRs |
| `issue_id` | str (FK → issues.id) | the auto-created `polish` issue |
| `triggered_by_user_id` | str (FK → users.id) | who clicked "Polish" |
| `status` | str | queued/running/succeeded/failed |
| `baseline_scan_id` | str (FK → codeability_scans.id) | "before" scan |
| `recompute_scan_id` | str (FK → codeability_scans.id) | "after" scan — **intentionally left null** (PRs aren't merged) |
| `pr_urls_json` | JSONB | `[]` — PR URLs the agent opened |
| `artifacts_json` | JSONB | `[]` |

#### `prd_skills_runs` (migration `b2e4f6a8c1d3` + `c7f1a9d3e2b4` + `q7d2f4a6c8e1`)

| Column | Type | Notes |
|---|---|---|
| `id` | str (PK) | prefix `prd_` |
| `org_id` | str (FK → orgs.id) | |
| `project_id` | str (FK → projects.id) | |
| `run_id` / `issue_id` | str (FK) | authoring agent (phase 2) |
| `plan_run_id` / `plan_issue_id` | str (FK) | planning agent (phase 1) |
| `triggered_by_user_id` | str (FK → users.id) | |
| `status` | str | `planning` \| `plan_ready` \| `authoring` \| `succeeded` \| `failed` |
| `prd_path` | str | repo-relative path of the PRD |
| `proposed_skills_json` | JSONB | `[{slug, title, purpose, category}]` |
| `selected_skills_json` | JSONB | `[slug, ...]` the user kept |
| `pr_urls_json` | JSONB | `[]` |
| **Partial unique index** | | `uq_prd_skills_active_per_project` on `(project_id)` WHERE `status IN ('planning', 'authoring')` — prevents concurrent active runs |

#### `projects` (added columns)

| Column | Type | Notes |
|---|---|---|
| `latest_codeability_scan_id` | str | pointer to current scan |
| `next_codeability_scan_at` | float | epoch; stamped after each successful scan; `None` for empty repos |
| `repo_state` | str | `"empty"` or `"code"` — drives UI banner routing |

### Relationships
```
Project (1) ──< CodeabilityScan (N)     [append-only history]
Project (1) ──< BootstrapRun (N)
Project (1) ──< PrdSkillsRun (N)
Project.latest_codeability_scan_id ──→ CodeabilityScan.id  [current pointer]
CodeabilityScan.run_id ──→ Run.id
CodeabilityScan.issue_id ──→ Issue.id
BootstrapRun.baseline_scan_id ──→ CodeabilityScan.id
BootstrapRun.recompute_scan_id ──→ CodeabilityScan.id  [null in practice]
```

### Migrations
| File | Date | What |
|---|---|---|
| `m5a8c1f3b7e2_add_codeability_and_bootstrap.py` | 2026-06-18 | Creates `codeability_scans` + `bootstrap_runs`; adds `projects.latest_codeability_scan_id`, `next_codeability_scan_at`; adds `runs.skills_json` |
| `b2e4f6a8c1d3_add_prd_skills_runs.py` | 2026-06-21 | Creates `prd_skills_runs` |
| `c7f1a9d3e2b4_add_prd_skills_plan_columns.py` | 2026-06-21 | Adds `plan_run_id` / `plan_issue_id` to `prd_skills_runs` |
| `q7d2f4a6c8e1_prd_skills_active_unique_index.py` | 2026-06-21 | Partial unique index on `prd_skills_runs` for dedupe |

---

## 5. Readiness Calculation / Business Logic

### Where the score comes from

The actual scoring is done by an **external Python script** — `scan.py` in the **CuriPowers** repo (the `g-nemo-readiness` skill). It is NOT in this codebase. The contract that the agent follows is in `web/backend/app/services/issues/workflow.py` (lines 1194–1217):

```python
_CODEABILITY_CONTRACT = (
    "## Workflow — Nemo Readiness scan\n\n"
    "This is a READ-ONLY repo-readiness scan. Do not modify the repo, commit, or open PRs.\n"
    "1. Run the g-nemo-readiness scorer. Its `scan.py` lives in the CuriPowers "
    "checkout (NOT in your repo), reachable via the `$NEMO_CURIPOWERS_ROOT` "
    "environment variable. Run exactly:\n"
    '   `python3 "$NEMO_CURIPOWERS_ROOT/skills/org/specialists/g-nemo-readiness/scripts/scan.py" --repo . --out codeability.json`\n'
    "   Do NOT look for the script in your working directory — it is not copied there.\n"
    "2. The script computes the ENTIRE scorecard. There are no agent-judged "
    "metrics: do not edit `points`, axis scores, `overall`, or the band ...\n"
    "3. Leave `codeability.json` at the repository root ...\n"
    "4. In your final response, report the score, the band, and the "
    "lowest-scoring metrics with their `evidence[]` ...\n"
)
```

### Scorecard JSON shape (what `scan.py` writes)
```json
{
  "scanVersion": "1.1.0",
  "overall": {"score": 72, "max": 100, "band": "Green"},
  "axisA": {
    "label": "Repository readiness",
    "score": 45,
    "max": 65,
    "metrics": [
      {"key": "lint", "label": "Lint config", "points": 10, "max": 10, "assessedMax": 10, "source": "script", "confidence": "high", "evidence": ["has .eslintrc.json"]}
    ]
  },
  "axisB": {
    "label": "Skill & knowledge coverage",
    "score": 27,
    "max": 35,
    "metrics": [...]
  }
}
```

### What the backend does with it — `parse_scorecard` (`service.py:122–157`)

The backend **never trusts the scorecard's own totals**. It recomputes everything from the raw metric `points`:

```python
def _axis_score(axis: dict, label: str) -> int:
    """Recompute one axis's score from its metric points.
    
    Each metric may report an assessedMax below its max: points that
    could not be measured in the run environment. Those leave the
    denominator rather than scoring zero, and the axis score is the
    earned share of what was assessed, projected onto the axis weight.
    """
    metrics = axis.get("metrics")
    axis_max = int(axis.get("max") or 0)
    earned = assessed = 0
    for metric in metrics:
        points = metric.get("points")
        if points is None:
            continue  # null stub = unmeasured, skip entirely
        metric_max = int(metric.get("max") or 0)
        metric_assessed = metric.get("assessedMax")
        metric_assessed = metric_max if metric_assessed is None else int(metric_assessed)
        earned += max(0, min(metric_assessed, int(points)))
        assessed += max(0, min(metric_max, metric_assessed))
    if assessed <= 0:
        return 0
    return max(0, min(axis_max, round(axis_max * earned / assessed)))
```

Then in `parse_scorecard`:
```python
axis_a_score = _axis_score(axis_a, "axisA")
axis_b_score = _axis_score(axis_b, "axisB")
score = max(0, min(100, axis_a_score + axis_b_score))
band = _band(score)  # 0-39 red, 40-69 amber, 70-100 green
```

### Band thresholds (`service.py:54–60`)
```python
def _band(score: int) -> str:
    if score >= 70: return "green"
    if score >= 40: return "amber"
    return "red"
```

Mirrored in the frontend (`band.ts:9–13`):
```ts
export function scoreToBand(score: number): Band {
  if (score >= 70) return "green";
  if (score >= 40) return "amber";
  return "red";
}
```

### Axis weights
- **Axis A (Repository readiness): max 65** (as of scanner 1.1; was 50 in earlier versions)
- **Axis B (Skill & knowledge coverage): max 35** (was 50)
- The frontend **never hardcodes these** — it reads `axisA.max` / `axisB.max` from the scorecard. Comment in `api.ts:46–48`: "Axis weight — 65 (repo readiness) / 35 (skill coverage) as of scanner 1.1; older scorecards carry 50/50, so never hardcode it."

### `assessedMax` — the "unmeasured" concept
Each metric may report `assessedMax < max` when a signal needed a toolchain or execution pass that was unavailable. Those points **leave the denominator** rather than scoring zero. The metric bar in the UI shows `points/assessedMax` (not `points/max`), and a tooltip explains: "5/5 measured · 12 when fully assessed."

### Empty repo handling
If `classify_repo(repo_dir) == "empty"` (README/docs/template only, no source files), the backend **skips the agent entirely** and records a deterministic 0-score scorecard:

```python
def _empty_scorecard_doc() -> dict:
    return {
        "scanVersion": "1.1.0",
        "overall": {"score": 0, "max": 100, "band": "Red", "note": "Empty repository..."},
        "axisA": {"label": "Repository readiness", "score": 0, "max": 65, "metrics": []},
        "axisB": {"label": "Skill & knowledge coverage", "score": 0, "max": 35, "metrics": []},
    }
```

`classify_repo` (`repo_inspect.py:87–102`): returns `"code"` if any file has a source extension OR is a `SKILL.md` under a `skills/` dir; otherwise `"empty"`. Early-exits on the first content file.

### What determines "ready" vs "not ready"
- **Green (70–100)**: "Automation-ready" — the repo is well-prepared for agentic development
- **Amber (40–69)**: "Getting there" — some gaps exist
- **Red (0–39)**: "Needs work" — significant gaps; also the deterministic score for empty repos

### Caching / precomputation / on-request
- **Not cached**: Scores are computed on-request (the scan runs fresh each time). The scorecard_json is persisted as history but the latest is always read from the DB.
- **Precomputed**: `project.next_codeability_scan_at` is stamped after each successful scan for the scheduler.
- **Calculated on request**: The `_scorecard_response` in the API layer reads the latest scan and shapes it into the response — no computation beyond formatting.
- **Recomputed on ingest**: `parse_scorecard` recomputes all totals from metric points — never trusts the scorecard's self-reported totals.

---

## 6. End-to-End Data Flow

### Example: A successful scan from trigger to render

**Step 1 — Database record exists**
```sql
SELECT * FROM projects WHERE id = 'prj_abc';
-- repo_clone_status='ready', repo_local_path='/data/repos/prj_abc', repo_state='code'
```

**Step 2 — Backend query (POST /projects/prj_abc/codeability/scan)**
```
trigger_codeability_scan → check_codeability_scan_authorized → start_scan
```

**Step 3 — Repo classification**
```python
classify_repo(Path('/data/repos/prj_abc'))  # → "code" (has .py files)
# Non-empty → _mark_repo_has_code → dedupe check → create CodeabilityScan row
```

**Step 4 — Dispatch agent run**
```python
issue = create_issue(title="Nemo Readiness", workflow_intent="readiness", status="in_progress")
run = dispatch_issue(issue, project, invocation_source="codeability_scan")
# Agent runs: python3 $NEMO_CURIPOWERS_ROOT/.../scan.py --repo . --out codeability.json
```

**Step 5 — Agent writes `codeability.json`**
```json
{
  "scanVersion": "1.1.0",
  "overall": {"score": 999, "max": 100},
  "axisA": {"label": "Repository readiness", "max": 65, "metrics": [
    {"key": "lint", "label": "Lint config", "points": 8, "max": 10, "assessedMax": 10, ...}
  ]},
  "axisB": {"label": "Skill & knowledge coverage", "max": 35, "metrics": [...]}
}
```
Note: `overall.score: 999` is the agent's CLAIM — the backend ignores it.

**Step 6 — Backend ingest (`ingest_scan_result`)**
```
Run finishes (status="succeeded")
→ heartbeat._maybe_finalize_codeability_or_bootstrap(run_id)
  → source == "codeability_scan" → ingest_scan_result(run_id)
    → read codeability.json from run.cwd
    → parse_scorecard(raw):
       axis_a_score = _axis_score(axisA) = round(65 * earned/assessed)  # e.g. 45
       axis_b_score = _axis_score(axisB) = round(35 * earned/assessed)  # e.g. 27
       score = 45 + 27 = 72
       band = "green"  (72 >= 70)
    → persist: scan.score=72, scan.band="green", scan.scorecard_json=raw
    → project.next_codeability_scan_at = now + 7d
    → _publish: WebSocket invalidation ["codeability", project_id, "scorecard"]
```

**Step 7 — API response shape (GET /projects/prj_abc/codeability)**
```json
{
  "status": "succeeded",
  "scan_id": "cea_xyz123",
  "project_id": "prj_abc",
  "issue_id": "iss_def456",
  "trigger": "manual",
  "score": 72,
  "band": "green",
  "axisA": {"label": "Repository readiness", "score": 45, "max": 65, "metrics": [...]},
  "axisB": {"label": "Skill & knowledge coverage", "score": 27, "max": 35, "metrics": [...]},
  "axis_a_score": 45,
  "axis_b_score": 27,
  "generatedAt": "2026-09-22T12:00:00Z",
  "runId": "run_ghi789",
  "error": null,
  "is_empty": false,
  "has_prd": false,
  "prd_path": null,
  "prd_paths": [],
  "has_design": false,
  "figma_path": null
}
```

**Step 8 — Frontend hook**
```ts
const { data } = useCodeability("prj_abc");
// data.status = "succeeded", data.score = 72, data.band = "green"
```

**Step 9 — Rendered UI**
```
CodeabilityScorecard
  → CodeabilityGauge(score=72, band="green")  → green gauge
  → bandTone("green") → {fg: "var(--color-success)", label: "Automation-ready"}
  → "Nemo Readiness" title
  → "Repository readiness 45/65 · Skill coverage 27/35"
  → AxisBreakdown(axisA) → MetricBar per metric → bars + tooltips
  → "Re-assess" + "Polish" buttons (admin only)
```

---

## 7. Architecture

### Frontend
- **React 19 + Vite + TanStack Router + TanStack Query**
- Feature-based directory: `web/ui/src/features/codeability/`
- Components are presentational; hooks (`hooks.ts`) own all server state via React Query
- Pure helpers (`band.ts`) are DOM-free for fast unit testing
- Real-time: WebSocket `/ws/invalidations` → `subscribeInvalidations` (`invalidations.ts`) → React Query cache invalidation → refetch → poll fallback while active

### Backend
- **FastAPI + SQLModel (SQLAlchemy) + async**
- Service/store separation: `service.py` (orchestration) → `store.py` (persistence)
- Scans run as **real Issues** (workflow intent `readiness`), so the live transcript surfaces in the UI like any agent run
- The scoring itself is **delegated to an external script** (CuriPowers) via a dispatched agent run — Nemo is the orchestrator, not the scorer
- **Async terminal hook**: `heartbeat.py:_maybe_finalize_codeability_or_bootstrap` fires when the agent run finishes, branches on `invocation_source`, and hands off to the right ingest function

### Why this architecture
- **External scorer**: The scoring logic (what metrics, what thresholds) evolves independently of Nemo's release cycle — it lives in CuriPowers and is versioned by `scanVersion`. The backend just validates and persists.
- **Issue-backed scans**: Scans show live progress in the UI without a separate WebSocket channel — they ride the existing issue/run transcript infrastructure.
- **Recompute-on-ingest**: The backend doesn't trust the agent's totals because agents can hallucinate or miscount. Recomputing from raw `points` ensures the displayed number always matches the evidence beside it. (Comment at `service.py:76–84`: "the number on the tab is always the sum of the evidence beside it.")
- **Append-only history**: `CodeabilityScan` rows are never updated in-place (except status transitions). This preserves auditability and lets Bootstrap compare baseline vs recompute.
- **Scheduler as asyncio task**: Bound to app lifespan (`lifespan.py:489–493`), not a separate cron — keeps it in-process and testable.

---

## 8. Dependencies / Integrations

| Dependency | What happens if unavailable |
|---|---|
| **CuriPowers repo** (`$NEMO_CURIPOWERS_ROOT`) | Agent can't find `scan.py` → run fails → scan marked `failed`. Contract says: "If `$NEMO_CURIPOWERS_ROOT` is unset, locate the g-nemo-readiness skill directory and run its `scripts/scan.py` by absolute path." |
| **Hermes agent runner** (dispatch_issue) | If `dispatch_issue` returns `None` (clone not ready, workspace busy) without raising, scan is explicitly marked `failed` to avoid a permanent "running" wedge |
| **PostgreSQL** (or SQLite for tests) | All persistence via SQLModel. JSON columns use JSONB on Postgres, plain JSON on SQLite (`_JSON_VARIANT`). |
| **WebSocket invalidation bus** | Lossy by design — frames published while disconnected are gone. Polling fallback (4s/5s) covers dropped frames. |
| **GitHub/GitLab clone** | `start_scan` requires `repo_clone_status == "ready"`. Best-effort `pull_repo` refreshes the clone before scanning; on failure, scans the existing on-disk clone. |
| **`NEMO_GITHUB_PLATFORM_INSTALLATION_ID`** | Needed for GitHub App token minting during clone refresh. |

### Environment variables
| Variable | Default | Consumed in |
|---|---|---|
| `NEMO_CODEABILITY_RESCAN_DAYS` | `7` | `service.py:65` — interval between scheduled re-scans |
| `NEMO_CODEABILITY_RESCAN_TICK_SECONDS` | `600` | `scheduler.py:27` — poller tick interval |
| `NEMO_CURIPOWERS_ROOT` | (required at runtime) | `workflow.py` contract — path to the external scorer script |
| `NEMO_GITHUB_PLATFORM_INSTALLATION_ID` | `""` | `settings.py:854` — GitHub App installation ID for token minting |

---

## 9. Permissions / Access Control

### Who can see Readiness
- **Anyone with project access**: The GET endpoint (`/projects/{id}/codeability`) requires only a valid token (`Depends(require_token)`). No role check.

### Who can trigger a scan
- **`check_codeability_scan_authorized`** (`authz.py:805–837`):
  - **Platform admin**: allowed
  - **Org admin/owner**: allowed
  - **FDE** (Field Deployment Engineer tier): allowed
  - **Project maintainer**: allowed
  - **Project developer / member / viewer**: **denied (403)**
- Frontend mirrors this: `canRescan = isAdmin || isFde || isMaintainer` (`CodeabilityTab.tsx:31`)

### Who can trigger Polish / PRD→skills
- **Admin only**: `Depends(require_org_admin)` on POST `/bootstrap`, POST `/prd-to-skills/plan`, POST `/prd-to-skills`
- Frontend: `isAdmin` gates the "Polish" button and the AutoBootstrapDialog

### Where checks happen
- **Backend**: `authz.py:check_codeability_scan_authorized` (scan) / `require_org_admin` (Polish/PRD→skills)
- **Frontend**: `usePermissions()` → `{ isAdmin, isFde, isMaintainer }` — hides/shows buttons but is **not authoritative**
- Comment in `CodeabilityTab.tsx:28–30`: "Mirrors the server gate (authz.check_codeability_scan_authorized); the backend stays authoritative."

---

## 10. Configuration

| Config | Location | Default | Purpose |
|---|---|---|---|
| `NEMO_CODEABILITY_RESCAN_DAYS` | `service.py:65` | `7` | Days between scheduled re-scans |
| `NEMO_CODEABILITY_RESCAN_TICK_SECONDS` | `scheduler.py:27` | `600` (min 30) | Poller tick frequency |
| `NEMO_CURIPOWERS_ROOT` | env, consumed in `workflow.py:1201` | required | Path to external scoring script |
| `NEMO_GITHUB_PLATFORM_INSTALLATION_ID` | `settings.py:854` | `""` | GitHub App installation for token minting |
| `SUPPORTED_SCAN_MAJOR` | `service.py:28` | `1` | Major version of scorecard schema accepted |
| `_AXIS_A_MAX` | `service.py:34` | `65` | Axis A max for empty-repo synthetic scorecard |
| `_AXIS_B_MAX` | `service.py:35` | `35` | Axis B max for empty-repo synthetic scorecard |
| `_EMPTY_SCAN_VERSION` | `service.py:33` | `"1.1.0"` | Scan version for empty-repo synthetic scorecard |
| Band thresholds | `service.py:54–60` / `band.ts:9–13` | 40/70 | Red/Amber/Green cutoffs |
| `AREAS` (Polish) | `AutoBootstrapDialog.tsx:47–54` | 6 areas | Improvement areas the Polish agents work on |

---

## 11. Tests

### Backend tests
| File | What it tests |
|---|---|
| `web/backend/tests/test_codeability_api.py` | HTTP endpoints: empty state, `is_empty`/`has_prd` flags, 404, 202 on scan, 409 on CloneNotReady, admin-only bootstrap, role-based scan auth (admin/owner/FDE/maintainer pass; developer/member/viewer → 403) |
| `web/backend/tests/test_codeability_model.py` | DB persistence + defaults: ID prefixes, default status, JSON columns, Project columns, Run.skills_json |
| `web/backend/tests/test_codeability_scheduling.py` | Onboarding scan kick from `_do_clone` + scheduled re-scan sweep (`sweep_due_rescans`) |
| `web/backend/tests/test_codeability_service.py` | Core: `parse_scorecard` recomputes from points (ignores `overall.score=999`), 65/35 weights, `assessedMax` denominator, band thresholds, `start_scan` creates `readiness` issue, empty repo = 0-score without agent, missing dir falls through to agent, dedupe, dispatch-error marks failed, timed_out/paused/missing-file handling, error redaction, terminal hook |
| `web/backend/tests/test_prd_to_mvp.py` | Two-phase PRD→skills: intent wiring, dedupe, `plan_id` pinning, design-skill injection, terminal hook, migrations, FK cascade, partial unique index |
| `web/backend/tests/test_repo_inspect.py` | `classify_repo`, `find_prd`, `find_figma_file`, `find_spec_documents` — keyword matching, fallback substantial-doc detection, boilerplate exclusion |
| `web/backend/tests/test_workflow.py` | Intent/contract mapping: `readiness` → skill `g-nemo-readiness` + contract mentions `codeability.json`; legacy `codeability`/`improve` aliases still dispatch correctly; scanner path points at `$NEMO_CURIPOWERS_ROOT` |

### Frontend tests
| File | What it tests |
|---|---|
| `band.test.ts` | `scoreToBand`, `bandTone`, `bandGrade`, `ratioTone` — thresholds, labels, null/undefined fallback |
| `CodeabilityTab.test.tsx` | State machine: skeleton, empty CTA, running, failed retry, succeeded scorecard, admin-only Polish, FDE/maintainer vs developer |
| `CodeabilityScorecard.test.tsx` | Score/band/axes rendering, partially-measured metric (`5/5` not `5/12`), unmeasured `—`, axis weight from scorecard, button gating, View run link, empty-repo banner |
| `AutoBootstrapDialog.test.tsx` | Area picker, projected score, agent count, non-admin disabled, PR links on success, Case B (empty+PRD → planning → checklist → generate selected), Case C (empty+no-PRD → guidance) |
| `AddPrdBanner.test.tsx` | Shown only for empty+no-PRD+admin, upload, dismiss (localStorage), hidden while PRD→skills in flight |
| `PrdDetectedBanner.test.tsx` | Admin + empty + PRD, hidden when succeeded, lists every document, stays reachable when plan exists but docs deleted |
| `SpecDocumentsDialog.test.tsx` | List/remove/add docs, count vs cap, disabled while pending |
| `SkillsPrCard.test.tsx` | PR links after PRD→skills success, GitHub + GitLab URL shortening, dismiss per project, re-show on new PR set |

### Notable test gaps
- No dedicated test for `useCodeability` / `useTriggerScan` hooks themselves (mocked in all component tests)
- Bootstrap/PRD→skills endpoints tested only at admin-vs-member level, not the full FDE/maintainer matrix used for scan
- No test for the scheduled sweep under partial-failure scenarios
- `find_spec_documents` count cap (5) not pinned to a backend test

---

## 12. Historical / Existing Implementation Clues

From git history (53 commits across the codeability dirs, 2026-06-17 → 2026-09-22):

| Date | Commit | Change |
|---|---|---|
| 2026-06-17 | `f621dee4e` | First touch — faked codeability panel on onboarding |
| 2026-06-18 | `9de099bb7` | **Initial GA**: real Codeability Score + Auto Bootstrap (migration `m5a8c1f3b7e2`) |
| 2026-06-18 | `94ab69de0` | Issue-backed scan + "Automation Readiness" UI |
| 2026-06-18 | `ce76df670` | Rename skills: `g-automation-readiness` → pointed at renamed skills |
| 2026-06-19 | `e86adf252` | **Security**: redact run errors before surfacing in scan UI |
| 2026-06-19 | `90a89f1d2` | **Authz fix**: replace `requires_permission(projects:manage)` with `check_codeability_scan_authorized` — allow FDE/maintainer |
| 2026-06-21 | `933524256` | Two-phase PRD→skills: planner proposes, user reviews checklist |
| 2026-06-21 | `afa4d8694` | Pin authoring to reviewed `plan_id` + partial unique index (dedupe race fix) |
| 2026-06-22 | `53a1c8712` | **Reliability**: empty-repo guard, scanner path → `$NEMO_CURIPOWERS_ROOT`, not `/var/nemo/CuriPowers` |
| 2026-08-26 | `b294b0363` | **"Trust the evidence, not the scorecard's own totals"** — backend stops trusting self-reported scores; re-derives from raw signals |
| 2026-09-17 | `1d7fbf5a7` | **Bug fix**: score every repo in a scan (multi-repo regression) |
| 2026-09-17 | `51699ae25` | **Rename**: `codeability → readiness` across backend + skills + tests |
| 2026-09-17 | `a1bd89d41` | Frontend rename: feature dir, components, hooks |
| 2026-09-21 | `c29e8966a` | Authoring starts automatically when the plan lands |
| 2026-09-22 | `122bf833f` | Cross-repo map authored by the scan (latest) |

**Key business-rule evolution**: The biggest shift was `b294b0363` — "trust the evidence, not the scorecard's own totals." Before this, the old parser persisted `overall.score` verbatim, meaning an axis of 80 or a metric worth 47 points would have been stored and shown. After, `parse_scorecard` recomputes everything from `points`.

---

## 13. Frontend ↔ Backend Contract

### GET `/projects/{id}/codeability`
- **Purpose**: Latest scorecard or `{status: "none"}` shell
- **Request**: no body; project_id in path
- **Response**: `Scorecard` object (see Section 6) or `{status: "none", project_id, is_empty, has_prd, ...}`
- **Frontend caller**: `useCodeability` → `codeabilityApi.scorecard`
- **Backend handler**: `get_codeability` (`api/codeability.py:106`)
- **DB/service deps**: `codeability_store.latest_for_project` → `CodeabilityScan` row + `Project`

### POST `/projects/{id}/codeability/scan`
- **Purpose**: Trigger a (re)scan
- **Request**: no body
- **Response**: `202` with `{id: scan_id, issue_id, run_id, status, ...}`
- **Frontend caller**: `useTriggerScan` → `codeabilityApi.scan`
- **Backend handler**: `trigger_codeability_scan` → `start_scan`
- **Error codes**: 404 (project not found), 409 (clone not ready), 403 (not authorized)

### POST `/projects/{id}/bootstrap`
- **Purpose**: Trigger Polish/Improve — agents open PRs
- **Request**: `{areas?: string[]}` (optional improvement areas)
- **Response**: `202` with `BootstrapState`
- **Frontend caller**: `useBootstrap` → `codeabilityApi.bootstrap`
- **Backend handler**: `trigger_bootstrap` → `bootstrap_svc.start_bootstrap`
- **Auth**: admin only (`require_org_admin`)

### POST `/projects/{id}/prd-to-skills/plan`
- **Purpose**: Phase 1 — read-only planning agent proposes skills from PRD
- **Request**: `{prd_path?: string}`
- **Response**: `202` with `PrdSkillsState`

### POST `/projects/{id}/prd-to-skills`
- **Purpose**: Phase 2 — author selected skills + open PR
- **Request**: `{prd_path?: string, selected_skills?: string[], plan_id?: string}`
- **Response**: `202` with `PrdSkillsState`
- **Error**: 409 if `plan_id` is stale/superseded (`PlanNotReadyError`)

### Error handling
- Run errors are **redacted** before display (`redact_text` in `service.py:403`)
- Failed scans store `scan.error` (truncated to 2000 chars) and surface it in the UI
- The `error` field in the scorecard response is the redacted, human-readable failure reason

---

## 14. File Map

### Frontend
| File | One-line description |
|---|---|
| `web/ui/src/features/codeability/api.ts` | API client + TypeScript types mirroring backend scorecard contract |
| `web/ui/src/features/codeability/hooks.ts` | React Query hooks: useCodeability, useTriggerScan, useBootstrap, usePrdToSkills* |
| `web/ui/src/features/codeability/band.ts` | Pure band/grade/tone helpers (Red/Amber/Green) |
| `web/ui/src/features/codeability/components/CodeabilityTab.tsx` | State machine for the Readiness tab (loading/none/running/failed/succeeded) |
| `web/ui/src/features/codeability/components/CodeabilityScorecard.tsx` | Scorecard UI: gauge + band + axis breakdowns + actions |
| `web/ui/src/features/codeability/components/CodeabilityGauge.tsx` | Circular 0–100 gauge visual |
| `web/ui/src/features/codeability/components/AutoBootstrapDialog.tsx` | "Polish" modal (code repo) + "Generate skills" modal (empty repo + PRD) |
| `web/ui/src/features/codeability/components/SkillsPrCard.tsx` | PR links card after PRD→skills run succeeds |
| `web/ui/src/features/codeability/components/AddPrdBanner.tsx` | Empty repo + no PRD → "Add a document" upload prompt |
| `web/ui/src/features/codeability/components/PrdDetectedBanner.tsx` | Empty repo + PRD → "Generate skills" prompt |
| `web/ui/src/features/codeability/components/SpecDocumentsDialog.tsx` | Manage the PRD/multi-doc set (list, remove, add) |
| `web/ui/src/lib/invalidations.ts` | WebSocket `/ws/invalidations` subscription — cache invalidation bus |

### Backend/API
| File | One-line description |
|---|---|
| `web/backend/app/api/codeability.py` | HTTP routes for codeability/bootstrap/prd-to-skills |
| `web/backend/app/auth/authz.py` | `check_codeability_scan_authorized` — scan permission gate |

### Business Logic
| File | One-line description |
|---|---|
| `web/backend/app/services/codeability/service.py` | Core: start_scan, ingest_scan_result, parse_scorecard, _axis_score, record_empty_repo_scorecard |
| `web/backend/app/services/codeability/store.py` | SQLModel persistence helpers for CodeabilityScan |
| `web/backend/app/services/codeability/scheduler.py` | Background re-scan poller (asyncio task) |
| `web/backend/app/services/codeability/workspace.py` | Output path helpers for codeability.json |
| `web/backend/app/services/codeability/prompt.py` | Issue title/description for the auto-created scan issue |
| `web/backend/app/services/bootstrap/service.py` | Auto-Bootstrap/Polish orchestration |
| `web/backend/app/services/bootstrap/store.py` | BootstrapRun persistence helpers |
| `web/backend/app/services/issues/workflow.py` | Workflow intent registration + `_CODEABILITY_CONTRACT` (agent prompt) |
| `web/backend/app/services/projects/repo_inspect.py` | `classify_repo`, `find_prd`, `find_spec_documents`, `find_figma_file` |

### Database
| File | One-line description |
|---|---|
| `web/backend/app/db/models/codeability.py` | SQLModel: CodeabilityScan, BootstrapRun, PrdSkillsRun |
| `web/backend/alembic/versions/m5a8c1f3b7e2_add_codeability_and_bootstrap.py` | Creates codeability_scans + bootstrap_runs + project columns |
| `web/backend/alembic/versions/b2e4f6a8c1d3_add_prd_skills_runs.py` | Creates prd_skills_runs |
| `web/backend/alembic/versions/c7f1a9d3e2b4_add_prd_skills_plan_columns.py` | Adds plan_run_id/plan_issue_id |
| `web/backend/alembic/versions/q7d2f4a6c8e1_prd_skills_active_unique_index.py` | Partial unique index for dedupe |

### Jobs/Workers
| File | One-line description |
|---|---|
| `web/backend/app/services/codeability/scheduler.py` | `run_codeability_rescan_poller` — asyncio background loop |
| `web/backend/app/services/runs/heartbeat.py` | `_maybe_finalize_codeability_or_bootstrap` — terminal hook |
| `web/backend/app/core/lifespan.py` | Starts the rescan poller as a background task |

### Tests
| File | One-line description |
|---|---|
| `web/backend/tests/test_codeability_api.py` | HTTP endpoint tests + role-based auth |
| `web/backend/tests/test_codeability_model.py` | DB model/defaults tests |
| `web/backend/tests/test_codeability_scheduling.py` | Onboarding scan + scheduled sweep tests |
| `web/backend/tests/test_codeability_service.py` | Core service: parsing, scoring, ingest, dedupe, error handling |
| `web/backend/tests/test_prd_to_mvp.py` | Two-phase PRD→skills flow tests |
| `web/backend/tests/test_repo_inspect.py` | Repo classification + PRD detection tests |
| `web/backend/tests/test_workflow.py` | Workflow intent/contract mapping tests |
| `web/ui/src/features/codeability/band.test.ts` | Band helper unit tests |
| `web/ui/src/features/codeability/components/CodeabilityTab.test.tsx` | Tab state machine tests |
| `web/ui/src/features/codeability/components/CodeabilityScorecard.test.tsx` | Scorecard rendering tests |
| `web/ui/src/features/codeability/components/AutoBootstrapDialog.test.tsx` | Improve dialog tests |
| `web/ui/src/features/codeability/components/AddPrdBanner.test.tsx` | Add-PRD banner tests |
| `web/ui/src/features/codeability/components/PrdDetectedBanner.test.tsx` | PRD-detected banner tests |
| `web/ui/src/features/codeability/components/SpecDocumentsDialog.test.tsx` | Doc management dialog tests |
| `web/ui/src/features/codeability/components/SkillsPrCard.test.tsx` | PR links card tests |

---

## 15. Architecture Diagram

```
┌─────────────────────────────────────────────────────────┐
│                    FRONTEND (React + Vite)                │
│  ProjectDetailPage                                       │
│    ├── PrdDetectedBanner  (empty repo + PRD)             │
│    ├── AddPrdBanner        (empty repo + no PRD)         │
│    └── CodeabilityTab                                    │
│         ├── SkillsPrCard    (PR links)                   │
│         ├── CodeabilityScorecard                        │
│         │    ├── CodeabilityGauge (0–100 circular)       │
│         │    ├── Band chip (Red/Amber/Green)            │
│         │    └── AxisBreakdown × 2 (MetricBars)         │
│         └── AutoBootstrapDialog (Polish / PRD→skills)    │
│                                                          │
│  React Query hooks ← WebSocket /ws/invalidations         │
│         ↑                  ↑                             │
│    API calls        invalidation frames                   │
└─────────┼──────────────────┼─────────────────────────────┘
          │                  │
          ▼                  ▼
┌─────────────────────────────────────────────────────────┐
│                   BACKEND (FastAPI)                       │
│                                                          │
│  API Router (api/codeability.py)                         │
│    GET  /projects/{id}/codeability      ── read scorecard │
│    POST /projects/{id}/codeability/scan ── trigger scan   │
│    POST /projects/{id}/bootstrap         ── trigger Polish │
│    POST /projects/{id}/prd-to-skills/*   ── PRD→skills     │
│         │                                                │
│         ▼                                                │
│  ┌─────────────────────────────────────────────┐         │
│  │ codeability/service.py                      │         │
│  │   start_scan() → dispatch_issue()           │         │
│  │   ingest_scan_result() → parse_scorecard()  │         │
│  │   record_empty_repo_scorecard()             │         │
│  └──────┬──────────────────────────┬────────────┘         │
│         │                         │                      │
│    ┌────▼─────┐          ┌────────▼─────────┐            │
│    │ store.py │          │ heartbeat.py      │            │
│    │ (SQL)    │          │ _maybe_finalize   │            │
│    └────┬─────┘          │ (terminal hook)   │            │
│         │                └────────┬─────────┘            │
│         ▼                         │                      │
│  ┌──────────────────┐             │                      │
│  │ PostgreSQL       │             │                      │
│  │ codeability_scans│             │                      │
│  │ bootstrap_runs   │             │                      │
│  │ prd_skills_runs  │             │                      │
│  │ projects (cols)  │             │                      │
│  └──────────────────┘             │                      │
│                                   │                      │
│  ┌────────────────────────────────▼───────────┐          │
│  │ scheduler.py (asyncio task, app lifespan)  │          │
│  │   sweep_due_rescans() every 600s           │          │
│  │   → start_scan(trigger="scheduled")        │          │
│  └────────────────────────────────────────────┘          │
└─────────────────────────────────────────────────────────┘
          │
          ▼
┌─────────────────────────────────────────────────────────┐
│              EXTERNAL (CuriPowers repo)                   │
│  g-nemo-readiness skill                                   │
│    scripts/scan.py --repo . --out codeability.json       │
│    (computes the ENTIRE scorecard: metrics, points,       │
│     assessedMax, evidence, axis weights)                  │
│                                                          │
│  g-polish-codebase skill                                  │
│    (opens PRs to raise the score: skills, gates, env,    │
│     tests, docs, golden tasks)                           │
└─────────────────────────────────────────────────────────┘
```

---

## 16. Sequence Diagram — Typical Page Load + Scan

```
User                 Frontend              API              Service            DB            Agent/CuriPowers
 │                     │                    │                 │                │                   │
 │── open Readiness ──▶│                    │                 │                │                   │
 │  tab                │                    │                 │                │                   │
 │                     │── GET /codeability ─▶│                │                │                   │
 │                     │                    │── latest_for_ ──▶│                │                   │
 │                     │                    │                 project──────────▶│                   │
 │                     │                    │◀─────────────── scorecard──────────│                   │
 │                     │◀── {status:none} ──│                 │                │                   │
 │                     │                    │                 │                │                   │
 │  (sees empty state) │                    │                 │                │                   │
 │                     │                    │                 │                │                   │
 │── "Run analysis" ──▶│                    │                 │                │                   │
 │                     │── POST /scan ──────▶│                │                │                   │
 │                     │                    │── authz check ──▶│                │                   │
 │                     │                    │── start_scan() ─▶│                │                   │
 │                     │                    │                 │── classify_repo │                   │
 │                     │                    │                 │  (filesystem)  │                   │
 │                     │                    │                 │── create_scan ─▶│ (status=running)  │
 │                     │                    │                 │── create_issue ─▶│                   │
 │                     │                    │                 │── dispatch ────▶│──────────────────▶│
 │                     │                    │                 │                │  agent runs       │
 │                     │◀── 202 {scan_id} ──│                 │                │  scan.py          │
 │                     │  (optimistic: queued)                │                │  writes          │
 │                     │  (poll every 4s)   │                 │                │  codeability.json │
 │                     │                    │                 │                │                   │
 │                     │── GET /codeability ─▶│ (poll)       │                │                   │
 │                     │◀── {status:running}─│                │                │                   │
 │                     │                    │                 │                │                   │
 │                     │                    │                 │                │  agent finishes    │
 │                     │                    │                 │                │  run.status=      │
 │                     │                    │                 │                │  "succeeded"      │
 │                     │                    │                 │                │                   │
 │                     │                    │                 │◀── heartbeat ──│                   │
 │                     │                    │                 │  _maybe_finalize│                   │
 │                     │                    │                 │  ingest_scan_   │                   │
 │                     │                    │                 │  result()       │                   │
 │                     │                    │                 │── read json ◀──│ (from run.cwd)    │
 │                     │                    │                 │── parse_scorecard│                  │
 │                     │                    │                 │  (recompute)     │                   │
 │                     │                    │                 │── update scan ─▶│ (status=succeeded)│
 │                     │                    │                 │── update proj ─▶│ (next_scan_at)    │
 │                     │                    │                 │── _publish ────▶│ (WS invalidation) │
 │                     │                    │                 │                │                   │
 │                     │◀── WS frame ───────│                │                │                   │
 │                     │  invalidate ["codeability", id, "scorecard"]          │                   │
 │                     │── GET /codeability ─▶│ (refetch)    │                │                   │
 │                     │◀── {status:succeeded, score:72, band:"green", ...}    │                   │
 │                     │                    │                 │                │                   │
 │── sees scorecard ──▶│                    │                 │                │                   │
 │  (gauge 72, green)  │                    │                 │                │                   │
```

---

## 17. Important Gotchas

1. **The scorer is external** — `scan.py` lives in CuriPowers (`$NEMO_CURIPOWERS_ROOT`), NOT in this repo. If that env var is unset or the path is wrong, the agent can't find the script and the scan fails. The contract has a fallback ("locate the g-nemo-readiness skill directory") but it's fragile.

2. **Backend recomputes all scores** — The agent's `overall.score` and `axisA.score` are **claims, not facts**. `parse_scorecard` ignores them and recomputes from `points`. If you add a metric to the scanner but forget to update the backend's validation, the backend will still compute correctly — but if the JSON shape is wrong, `parse_scorecard` will reject it with `ScorecardParseError`.

3. **`assessedMax` semantics** — A metric with `assessedMax < max` means the scanner couldn't measure all points in this environment. Those unmeasured points leave the denominator (not scored zero). The UI shows `points/assessedMax` (not `points/max`). Misunderstanding this leads to thinking a metric "failed" when it was actually unmeasured.

4. **Empty repo short-circuit** — `classify_repo` returns `"empty"` for README/docs-only repos, which triggers a **deterministic 0-score without an agent run**. But it also returns `"empty"` for a **missing** directory (rglob on nonexistent dir yields nothing). The code guards with `is_dir()` first (line 220) to avoid silently 0-scoring an evicted clone.

5. **`recompute_scan_id` is always null** — BootstrapRun has a `recompute_scan_id` column but it's **intentionally never populated**. Polish opens PRs but never merges them; a recompute would scan the unchanged default branch and overwrite the score with an identical one, making Polish look like it achieved nothing. The delta lands later when PRs merge and the scheduled rescan or Re-assess fires.

6. **Dispatch can return None without raising** — `dispatch_issue` can return `None` (clone not ready, workspace busy, prep failed) WITHOUT raising. Without the explicit `if run is None: _mark_failed(...)` guard (line 266), the scan would stay "running" forever and dedupe would wedge every future scan for that project. This was a real production bug (comment cites a row stuck since 2026-07-08).

7. **Legacy aliases** — `codeability` → `readiness` and `improve` → `polish` are mapped in `workflow.py:159–160`. Old issues/intents still work but the canonical names are `readiness`/`polish`. Don't use the old names in new code.

8. **Axis weights changed** — Scanner 1.0 used 50/50; scanner 1.1 uses 65/35. The backend reads `max` from the scorecard (never hardcodes), but the empty-repo synthetic scorecard uses `_AXIS_A_MAX=65` / `_AXIS_B_MAX=35`. Old 50/50 scorecards still parse correctly because `parse_scorecard` reads the `max` from the JSON.

9. **`SUPPORTED_SCAN_MAJOR = 1`** — Only major version 1 scorecards are accepted. A scorecard with `scanVersion: "2.0.0"` would be rejected with `ScorecardParseError("unsupported scanVersion")`.

10. **Error redaction** — `run.error` is stored raw and may contain internal paths. `ingest_scan_result` calls `redact_text(run.error)` before storing it in `scan.error`, which is surfaced verbatim in the UI.

11. **PRD detection fallback** — `find_prd` has a keyword-less fallback that picks substantial unkeyworded docs (≥ `_MIN_SPEC_BYTES`). A real spec named "OCW005-Artemis-Architecture.md" would match no keyword but is still detected if it's large enough. Boilerplate stems (README, CHANGELOG) are excluded.

12. **WebSocket is lossy** — The invalidation bus is a lossy channel by design. Frames published while disconnected are gone. The 4s/5s polling fallback covers this, but there's a gap: if the WS frame is dropped AND the poll hasn't fired yet, the UI shows stale data briefly.

13. **`git clean -fd` erases uploads** — Uploaded spec docs live in Nemo's store, NOT in the clone, because every dispatch syncs the clone with `git clean -fd` which deletes untracked files. This is why `find_prd` searches both the clone AND the uploaded-docs store (`_candidate_files`).

---

## 18. "If I Need to Modify Readiness..."

| If I need to... | Look at... |
|---|---|
| **Change what appears in the Readiness UI** | `web/ui/src/features/codeability/components/CodeabilityScorecard.tsx`, `CodeabilityTab.tsx` |
| **Change the Readiness calculation (scoring logic)** | The external `scan.py` in CuriPowers (`$NEMO_CURIPOWERS_ROOT/skills/org/specialists/g-nemo-readiness/scripts/scan.py`) — NOT in this repo. The backend only validates/persists. |
| **Change how the backend parses/validates the scorecard** | `web/backend/app/services/codeability/service.py` → `parse_scorecard`, `_axis_score` |
| **Change band thresholds** | `web/backend/app/services/codeability/service.py:54–60` (`_band`) AND `web/ui/src/features/codeability/band.ts:9–13` (`scoreToBand`) — both must match |
| **Change axis weights** | The scanner (CuriPowers `scan.py`) emits `axisA.max` / `axisB.max`. The backend reads them. For the empty-repo synthetic scorecard: `service.py:34–35` (`_AXIS_A_MAX`, `_AXIS_B_MAX`). |
| **Add a new Readiness field to the API response** | `web/backend/app/api/codeability.py` → `_scorecard_response` + `web/ui/src/features/codeability/api.ts` → `Scorecard` interface |
| **Change scan trigger permissions** | `web/backend/app/auth/authz.py` → `check_codeability_scan_authorized` + `web/ui/src/features/codeability/components/CodeabilityTab.tsx:31` (`canRescan`) |
| **Change the rescan interval** | `NEMO_CODEABILITY_RESCAN_DAYS` env var (default 7) → `web/backend/app/services/codeability/service.py:65` |
| **Change the poller tick frequency** | `NEMO_CODEABILITY_RESCAN_TICK_SECONDS` env var (default 600) → `web/backend/app/services/codeability/scheduler.py:27` |
| **Change the agent's scan instructions** | `web/backend/app/services/issues/workflow.py:1194–1217` (`_CODEABILITY_CONTRACT`) |
| **Change how Readiness is refreshed (real-time)** | `web/backend/app/services/codeability/service.py:468–475` (`_publish`) + `web/ui/src/features/codeability/hooks.ts` (poll intervals) |
| **Add a new database field** | New alembic migration in `web/backend/alembic/versions/` + `web/backend/app/db/models/codeability.py` |
| **Change the empty-repo handling** | `web/backend/app/services/codeability/service.py` → `record_empty_repo_scorecard`, `_empty_scorecard_doc` + `web/backend/app/services/projects/repo_inspect.py` → `classify_repo` |
| **Change the Polish/Improve flow** | `web/backend/app/services/bootstrap/service.py` + `web/ui/src/features/codeability/components/AutoBootstrapDialog.tsx` |
| **Change the PRD→skills flow** | `web/backend/app/services/prd_to_mvp/` + `web/ui/src/features/codeability/components/AutoBootstrapDialog.tsx` (Case B) |
| **Change the terminal hook (what happens when scan finishes)** | `web/backend/app/services/runs/heartbeat.py:813` → `_maybe_finalize_codeability_or_bootstrap` |

---

## Top 15 Files to Read First

1. `web/backend/app/services/codeability/service.py` — the heart: start_scan, ingest, parse_scorecard, _axis_score
2. `web/backend/app/api/codeability.py` — all HTTP routes + response shaping
3. `web/backend/app/db/models/codeability.py` — all three DB models
4. `web/backend/app/services/issues/workflow.py` (lines 1194–1247) — the _CODEABILITY_CONTRACT and _IMPROVE_CONTRACT
5. `web/ui/src/features/codeability/api.ts` — TypeScript types mirroring the backend contract
6. `web/ui/src/features/codeability/hooks.ts` — React Query hooks (all server state)
7. `web/ui/src/features/codeability/components/CodeabilityScorecard.tsx` — the scorecard UI
8. `web/ui/src/features/codeability/components/CodeabilityTab.tsx` — the tab state machine
9. `web/ui/src/features/codeability/band.ts` — band/grade/tone pure helpers
10. `web/backend/app/services/codeability/scheduler.py` — background rescan poller
11. `web/backend/app/services/runs/heartbeat.py` (lines 813–851) — terminal hook
12. `web/backend/app/auth/authz.py` (lines 805–837) — scan permission gate
13. `web/backend/app/services/projects/repo_inspect.py` (lines 87–216) — classify_repo + PRD detection
14. `web/backend/app/services/bootstrap/service.py` — Polish/Improve flow
15. `web/backend/tests/test_codeability_service.py` — the most comprehensive test of the scoring logic

## Single Most Important End-to-End Flow

**The scan lifecycle**: `start_scan` → `classify_repo` (empty? deterministic 0) → `create_scan` + `create_issue` + `dispatch_issue(invocation_source="codeability_scan")` → agent runs `scan.py` from CuriPowers → writes `codeability.json` → heartbeat terminal hook → `ingest_scan_result` → `parse_scorecard` (recomputes from `points`, ignores claimed totals) → persist score/band/axes + `scorecard_json` → `_publish` WebSocket invalidation → frontend refetch → `CodeabilityScorecard` renders gauge + band + axis breakdowns.

## Biggest Architectural Concepts to Understand

1. **The scorer is external** — Nemo is the orchestrator, not the scorer. The actual scoring logic lives in CuriPowers (`scan.py`). The backend only validates, recomputes (distrusts the agent's totals), and persists. To change scoring rules, you edit CuriPowers, not this repo.

2. **Scans are Issues** — A scan runs as a real Issue (workflow intent `readiness`) so its live transcript shows in the UI like any agent run. The `invocation_source="codeability_scan"` on the Run is what the heartbeat terminal hook branches on to call `ingest_scan_result`.

3. **Recompute-on-ingest distrusts the agent** — `parse_scorecard` never trusts `overall.score` or `axisA.score`. It recomputes from raw `points` and `assessedMax`. This is a deliberate architectural decision (commit `b294b0363`): agents can hallucinate or miscount, but the displayed number must always match the evidence beside it.

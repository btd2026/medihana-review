# MediHana — Approval Batch for Brian

*Prepared Nov 1–2, 2026 from Edi's meeting notes. Everything below was done in the VM copies only (`~/workspace/medihana/` pristine copy, `~/workspace/medihana/working/` for prepared fixes). **Nothing was written to the Mac, nothing was emailed.** Each item: Finding / Evidence / Proposed action / Risk & effort. Say yes/no per item; approved diffs live in `~/workspace/medihana/diffs/` and can be applied to the Mac project in minutes.*

*Companion doc: `FORMULAS-AND-PARAMETERS.md` has the full formula write-ups for T1–T5.*

---

## T1 — Projected Staff Utilization formula ✅ information only

**Finding.** The number is a live deterministic projection in the student Clinic Controls screen — not a simulation result and not from the backend. Per staff type (Admin, Nurse, MD):

```
utilization% = ( expectedPatients × workPerPatient[type] ) ÷ ( staffCount × workdayMinutes ) × 100
expectedPatients = appointmentsPerDay × (1 − noShowProbability)
workPerPatient[type] = Σ over Pre-Exam/Exam/Post-Exam of ( allocation% ÷ 100 × stageMean × stageMultiplier )
stageMean = (min + mode + max) ÷ 3 of the stage's triangular distribution
stageMultiplier = inefficiency u if ≥2 staff share the stage, else 1
```

**Evidence.** `src/app/simulation/components/clinic-controls/clinic-controls.component.ts`, `recomputeProjection()` lines 257–348 (core division lines 336–343). Inputs: student choices + admin scenario/flow/scheduling config. Can exceed 100% (only the bar is clamped). The results-dashboard "Resource Utilization" is a different, post-run busy-time number (`simulation-runner.service.ts` 705–725).

**Proposed action.** None — answer for the email to Edi.
**Risk & effort.** None.

## T2 — Net Promoter Score formula ✅ information only

**Finding.** A synthetic score (not survey-based): start at 100, subtract penalties, clamp to [−100, 100]:

```
delayPenalty       = min( avgInClinicDelay × 1.5 , 60 )     (pre-exam + exam delay only)
mdUnderPenalty     = max(0, 50 − mdBusy%) × 0.5
mdOverPenalty      = max(0, mdBusy% − 90) × 2
abandonmentPenalty = avgAbandonedPerDay × abandonmentNpsPenalty   (admin-set, default 1.25)
NPS = clamp(100 − all penalties, −100, 100), displayed rounded
```

**Evidence.** `src/app/simulation/services/simulation-runner.service.ts`, `aggregateResults()` lines 776–788 (quoted verbatim in FORMULAS-AND-PARAMETERS.md). `abandonmentNpsPenalty` is editable by the admin under Strategy & Financial → Operational; seeded default 1.25.

**Proposed action.** None — answer for the email. Worth flagging to Edi: post-exam delay is excluded from the delay penalty, and the MD busy-% sweet spot is implicitly 50–90%.
**Risk & effort.** None.

**Update — Brian's direction Nov 1, 2026 (~11:55 PM ET): replace the synthetic score with an ACTUAL NPS calculation.** Proposal written up in `NPS-PROPOSAL.md`: score each simulated patient 0–10 from their own total delay (all 3 stages; 10 min = 1 point; abandoned = 0; no-shows excluded), classify with the standard bands (≥9 promoter, ≤6 detractor), NPS = 100 × (Promoters − Detractors) / all who sought care. MD-busy penalties and the abandonment-penalty knob come out. Status: proposal awaiting Brian's go to prepare it as a patch (would become item T2b) and/or pose to Edi in the email first.

## T6 — Roles / RBAC audit ✅ information only — contains a question for Edi

**Finding.** There are effectively **two roles**, and no separate instructor role:

| | Student (level 1) | Admin (level 3) |
|---|---|---|
| Who | `app_users.level = 1`; per-game membership role `STUDENT` | `app_users.level = 3`; membership role `ADMIN` |
| Can do | See own games, edit own choices, run simulations, save runs, view own results/history | Everything a student can, **plus** all `/api/admin/*`: create/edit/delete games, manage users (add, bulk-add, change role, delete), edit every config layer (engine, flow, scheduling, strategy/financial, AI), simulation control |

**Evidence.**
- Role→level mapping on user create/update: `src/api/route_admin_users.go` lines 82–84, 142–145, 198–206 (`ADMIN` → level 3, anything else → level 1; an unknown role string silently becomes student).
- Backend admin gate `requireAdmin`: `src/api/helpers.go` lines 42–62 — requires a valid session and **re-reads the level from the database on every request** (so demotion takes effect immediately), rejecting `level <= 1`. All admin routes are grouped behind it in `src/api/main.go` lines 21–75.
- Player handlers (`src/api/route_player.go`) check the session per handler and scope data by the caller's game membership.
- Frontend mirrors the backend: `AdminGuard` (`src/app/auth/admin.guard.ts`) allows `/admin` when `level > 1`; `PlayerGuard` allows `/simulation` when `uid > 0`. Frontend guards are UX only; the backend enforces.
- Per-game roles are only `STUDENT` / `ADMIN` (`GameUserRole`, `src/app/admin/models/simulation.models.ts` lines 57–60). What a student may touch inside a game is further limited by the game's `enabledLayers` / `UserControlsConfig` (e.g. `canAdjustTaskAllocation`, `game-setup.component.ts:275`).

**Open questions for Edi (no fix proposed):**
1. **No instructor role exists** — a teacher must be a full admin, including user deletion and game deletion. Is a read-only / results-only instructor role wanted? (Per Brian: no centralized results collection is being built either way.)
2. **Level semantics are inconsistent/undocumented:** user creation assigns admins level **3**, but every gate accepts **any level > 1**; level 2 is never assigned or defined anywhere, and `route_auth.go:41` special-cases `level < 5` for login logging, implying levels ≥5 exist in someone's head (or a license/deployment concept) but nothing in the codebase assigns or checks them.
3. Is level 2 meant to be an intermediate (e.g. instructor) tier that was never finished?

**Proposed action.** None — questions for the email.
**Risk & effort.** None.

## T3 — Task-allocation sliders: rule does NOT hold today → fix prepared

**Finding.** Audited every place MD/Nurse/Admin allocation is handled. Brian's rule fails in the one interactive place, the student Clinic Controls:
- **No MD floor exists anywhere.** `TaskResourceLimit` has only `maxAllocationPercent` (`admin/models/simulation.models.ts:102`); every slider's minimum is 0, MD included.
- **Rebalancing trades against all roles.** `onAllocationChange()` (`clinic-controls.component.ts` lines 171–207, old code) cut/topped-up the other rows *in array order* when a stage total left 100 — raising Nurse could cut MD below any floor, and vice-versa.
- **MD does not get the leftover** — leftover went to whichever rows came first, up to their ceilings.
- Admin-side handling (scenario defaults + per-template ceilings in `resource-scenarios` / `resource-scenario.service.ts:327–356`) only sets maxima and starting values; it has no floor concept either. Ceilings logic is duplicated (not shared) between clinic-controls and resource-scenarios.

**Prepared fix — REVISED per Brian's decision (Nov 1, 2026, ~11:55 PM ET): the floor is admin-settable, not hardcoded.** `diffs/T3-clinic-controls-md-floor.patch` (regenerated; verified to apply cleanly onto the pristine copy, with all T3 files matching the working tree):
- **Rule (unchanged from the first prepared version):** in `onAllocationChange`, in any multi-role stage, MD is clamped to **[floor, ceiling]**; Nurse/Admin changes trade **only against each other** inside the pot `100 − MD`; moving MD re-splits the pot proportionally; any remainder that can't be placed (partner at ceiling) goes to **MD — MD gets the leftover**. Single-role stages (Dr. Solo) behave as before. The MD slider minimum in the template binds to the configured floor, so the UI itself enforces it.
- **Where the floor lives:** a new optional `minAllocationPercent` on `TaskResourceLimit` (`admin/models/simulation.models.ts`), alongside the existing `maxAllocationPercent` ceiling, plus a shared `DEFAULT_MD_ALLOCATION_FLOOR = 20`. It travels in the game's **flow config** (`FlowEngineSharedConfig.taskResourceLimits`) — the Go side stores flow config as an opaque `json.RawMessage` blob (`route_admin_games.go`, get/update flow-config handlers pass it through verbatim) and the player route returns it raw, so **no Go or DB change is needed** (same pattern T4/T5 relied on).
- **Admin side:** Resource Scenarios → **Work Matrix Limits** table (where the Max % ceilings already live) gains a **Min % column** — an editable number input on MD rows, "—" on other rows; the section hint explains the floor. Validation added in `validateFlowConfig`: min must be 0–100 and ≤ that pair's max. Template/default seeding: all MD pairs in `getScenarioTemplateLimits` (Dr. Solo / With Admin / With Nurse, hence also `getDefaultFlowConfig`) now carry `minAllocationPercent = DEFAULT_MD_ALLOCATION_FLOOR`, so new games start at 20.
- **Legacy games:** flow configs saved before this field existed have no min. The admin service backfills MD rows with 20 when a flow config loads (`getFlowConfig` in `resource-scenario.service.ts`), and the student side independently falls back to 20 when the field is absent — so existing games behave exactly as the first prepared (hardcoded-20) version until an admin changes the value.
- **Student side:** `clinic-controls.component.ts` no longer has the `mdAllocationFloor = 20` constant; `mdAllocationFloorFor(taskType)` reads the (task, MD) limit from `session.flowConfig`, and both the slider minimum (`minAllocationFor`) and the MD clamp in `onAllocationChange` use it **per stage** — an admin can set different floors for Pre-Exam, Exam, and Post-Exam.

**Verification (revised patch, working tree with T0+T4+T5+T7 also applied):** `tsc -p tsconfig.app.json` clean, `tsc -p tsconfig.spec.json` clean, `ng build` succeeds, `ng test` 8/8 files, **11/11 pass**. Patch-level check: the regenerated patch applies cleanly onto the pristine commit and reproduces the working tree's T3 files exactly.

**Remaining open questions (unchanged by the revision):**
1. **Template/default conflict:** *default* MD allocations in templates can sit below the floor in multi-role stages (e.g. With Nurse Pre-Exam MD default 30 is fine, but an admin can set a scenario default below whatever floor they configured) — the fix enforces the floor when a student touches a slider, but doesn't rewrite saved/default allocations on load. Flagging rather than silently changing scenario data.
2. Admin scenario-default editing still lets an admin set an MD default below the floor (defaults are data, not student slider behavior) — same flagging choice as (1); no new validation was added there.
3. Scope note: Min % exists on the model for every pair but is only *shown/enforced* for MD, since the protected-floor rule is MD-specific. Giving Nurse/Admin floors later is a small follow-on, not part of this patch.

**Risk & effort.** Risk: low-medium — changes student-facing numbers behavior (allocations snap differently); no backend/data change; old configs default to the previous behavior (floor 20); reversible by reverting the patch. Effort to apply: minutes; the open questions above are the real cost.

## T4 — Batch run length: NOT admin-editable today → fix prepared

**Finding.** 120 days is **hardcoded twice** and both number inputs that display it are disabled:
- Admin Simulation Control: `batchSize = 120` (`simulation-control.component.ts:56`), input `[disabled]="true"` (`simulation-control.component.html:78`).
- Student Simulation Viewer: `batchDays = 120` (`simulation-viewer.component.ts:34`), input likewise disabled. (The runner's own fallback default is a third value, 100 — `simulation-runner.service.ts:175`.)
- No config, DB column, or Go setting carries it. But scheduling configs are stored by the Go API as an opaque JSON blob (`src/api/route_admin_scheduling.go`), so a new field needs **no Go or DB change** — the same pattern as `arrivalPerturbEarlyMinutes`.

**Prepared fix** — `diffs/T4-batch-days-admin-editable.patch`:
- `SchedulingConfig.batchDays?: number` (model + defaults of 120 in both default-config providers).
- New **"Batch Simulation → Batch Run Length (days)"** field in Admin → Scheduling Configuration (validated 1–1000), saved with the rest of the scheduling config.
- Admin Simulation Control initializes its batch size from the config and its input is **enabled** (per-run override; doesn't write back).
- Student viewer reads the admin-set value from the session's scheduling config (falls back to 120). Student input stays display-only.

**Verified.** Type-check clean, production build succeeds, tests 11/11.
**Risk & effort.** Risk: low — old games without the field behave exactly as today (120). Effort to apply: minutes.

## T5 — Fixed Cost: does not exist → fix prepared (one interpretation flagged)

**Finding.** `FinancialParameters` (`admin/models/simulation.models.ts:271–282`) has revenue ×2, per-day costs (MD, Nurse, Admin, Exam Room, AI Assistant), overtime/hour, and patient-delay/day — **no fixed cost**. Profit is computed in two places that must agree: the runner's post-run daily profit (`simulation-runner.service.ts:758–767`) and the student's live projection (`strategy.service.ts` `computeLocalSummary()`, lines 69–75). Storage is an opaque JSON blob on the Go side (`route_admin_strategy.go`) and validation is generic (any numeric field ≥ 0), so again **no Go/DB/validator change** is needed.

**Prepared fix** — `diffs/T5-fixed-cost-per-day.patch`:
- New field `costFixedPerDay` (default 0) in the model + shared defaults (old saved configs resolve to 0 via the existing `resolveFinancialParams` merge, so nothing breaks or NaNs).
- New **"Fixed Cost / Day ($)"** input in Strategy & Financial Configuration → Financial → Other Costs.
- Added once per day to daily costs in **both** profit calculations (runner + live projection), with cross-referencing comments.

**⚠️ Interpretation to confirm with Edi:** the note says "add a Fixed Cost"; the patch implements **fixed cost per day** because every other cost in the model is daily and profit is computed daily. If Edi meant a one-off/annual fixed cost, the field should instead be divided by `daysOpenPerYear` — a two-line change to the same patch. Also naming: "Fixed Cost" in the UI could alternatively live as its own labeled section rather than under Other Costs.
**Verified.** Type-check clean, production build succeeds, tests 11/11.
**Risk & effort.** Risk: low (default 0 = today's behavior). Effort to apply: minutes.

## T0 — Test suite: did not compile → fix prepared and verified green

**Baseline (pristine copy).** Go: `go build ./...` and `go vet ./...` both pass, zero errors (no Go test files exist). Angular suite **failed to compile**, exactly reproducing Brian's Oct 2026 finding, with two errors (also confirmed independently via `tsc -p tsconfig.spec.json`):
1. `src/app/auth/admin.guard.spec.ts(6,14): error TS2749: 'AdminGuard' refers to a value, but is being used as a type here. Did you mean 'typeof AdminGuard'?` — the spec injects the guard as a class; the guard is now a functional `CanActivateFn`.
2. `src/app/auth/login/login.component.spec.ts(1,10): error TS2305: Module '"@angular/core/testing"' has no exported member 'async'.` — the spec uses the removed `async` helper and `declarations` for a standalone component.

**Prepared fix** — `diffs/T0-test-suite-fix.patch` (spec files only; **no production code changed**):
- `admin.guard.spec.ts`: rewritten for the functional guard — invoked via `TestBed.runInInjectionContext` with stubbed AuthService/Router; now actually tests behavior (level > 1 allowed, level ≤ 1 redirected).
- `login.component.spec.ts`: modern standalone pattern (`imports`, `provideHttpClient`, `provideRouter`, MatDialog stub).
- Once the suite compiled, it **ran** and exposed 3 more stale specs of the same vintage, fixed with the same minimal patterns: `confirm`/`error-dialog` specs (`declarations` → `imports` + dialog-token stubs), `admin.component.spec.ts` (added router/http providers), and `app.spec.ts` (its "Hello, medi-hana" scaffold title assertion replaced with a router-outlet shell check — the root template is a bare `<router-outlet />`).

**Verification in `working/` (with T3+T4+T5 also applied):**
- `tsc -p tsconfig.spec.json --noEmit` — clean; `tsc -p tsconfig.app.json --noEmit` — clean.
- `npx ng test --watch=false` — **8/8 test files, 11/11 tests pass.**
- `npx ng build` — succeeds.
Full baseline detail: `BASELINE-REPORT.md`.

**Risk & effort.** Risk: very low (tests only). Effort to apply: minutes. Note: the suite is still tiny (11 smoke tests) — it will now at least run in CI/locally.

## T7 — Settings page redesign ⏭️ SKIPPED by Brian Nov 1, 2026 — do not apply

*Brian confirmed Nov 1, 2026 (~11:54 PM ET): skip the settings. This section is kept for reference only and is not part of the approval walk-through or the email to Edi.*

## T7 (reference only) — Settings page redesign as prepared

**Finding.** The admin "settings" surface is four pages; the overwhelming one is Strategy & Financial Configuration. Before the redesign, the active tab rendered **everything expanded at once**: Financial = 3 sections / 9 visible fields, Operational = 4 sections / 12 visible fields, AI = 6 fields (already gated by its enable toggle). Set-once tuning fields (patience distribution, NPS penalty, urgent mix) carried the same visual weight as everyday fields (prices, staff costs). Scheduling Configuration (6 sections / 15 fields) and Game Setup (5 sections) have the same all-expanded pattern; Simulation Control is a run console, not settings.

**Prepared fix** — `diffs/T7-settings-redesign.patch` (full spec: `SETTINGS-REDESIGN.md`):
- Progressive disclosure on strategy-config using `mat-expansion-panel` (already in the installed @angular/material; no new dependency): **Financial** keeps Revenue + Staff Costs visible and collapses *Other Costs* into a panel (visible fields 9 → 5); **Operational** keeps Capacity + Panel Size visible and collapses *Urgent Visit Mix* and *Appointment Abandonment* (12 → 6). Panel headers carry plain-language descriptions of what's inside. AI tab unchanged (its toggle already discloses).
- **Every field, binding, validator, and save path is preserved exactly** — panels only wrap existing markup, collapsed inputs stay in the form, and no class logic changed. Panels styled to match the existing section cards.
- Spec also maps the same pattern onto Scheduling Configuration and Game Setup as follow-ups (not implemented).

**Verification.** `tsc` app check clean, `ng build` succeeds, `ng test` 8/8 files, 11/11 tests pass.
**Risk & effort.** Risk: low — visual only; the one behavior note is that panels start collapsed on each visit/tab switch (no remembered state; adding persistence is a small follow-up if wanted). Effort to apply: minutes. Open choices (in the spec): remembered panel state? AI-tab panels for consistency? Edi pass on panel description wording.

---

### Verification summary (final `working/` tree, all prepared fixes applied)
- Go build + vet: pass (pristine baseline; no Go changes in any patch).
- Angular: spec + app type-checks clean; `ng build` succeeds; `ng test` 11/11 pass.
- Patches: `diffs/T0-test-suite-fix.patch`, `diffs/T3-clinic-controls-md-floor.patch`, `diffs/T4-batch-days-admin-editable.patch`, `diffs/T5-fixed-cost-per-day.patch` — each is a separate git commit on top of the pristine copy in `working/`, so they can be approved/applied independently.
- Not done, by design: nothing applied to the Mac project; no email drafted or sent (that's the next step after approvals).

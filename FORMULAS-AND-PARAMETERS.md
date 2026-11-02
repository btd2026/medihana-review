# MediHana — Formulas & Parameters (from the copied source)

*Extracted Nov 1, 2026 from the VM copy of `/Users/briandibassinga/Github/medihana`. No code was changed for this document. Line numbers refer to the copied files.*

## 1. Projected Staff Utilization (T1)

**Where:** `src/app/simulation/components/clinic-controls/clinic-controls.component.ts`, `ClinicControlsComponent.recomputeProjection()` (lines 257–348; core math 270–343). It is computed **live in the student Clinic Controls screen** every time a slider changes — it is *not* from the simulation engine and *not* from the backend. (The engine's "Resource Utilization" on the results dashboard is a different, post-run number based on actual busy time.)

**Formula, per staff type (Admin, Nurse, MD — exam rooms excluded):**

```
utilization%  =  ( expectedPatients × workPerPatient[staffType] )  ÷  ( staffCount × workdayMinutes )  ×  100
```

- `expectedPatients = appointmentsPerDay × (1 − noShowProbability)` — appointmentsPerDay from the student's choice, else the scheduling config default, else 20 (lines 280–286).
- `workPerPatient[staffType] = Σ over the 3 stages (pre-exam, exam, post-exam) of ( allocation%[stage, staffType] ÷ 100 × stageMean[stage] × stageMultiplier[stage] )` (lines 309–319).
- `stageMean[stage] = (minimum + mode + maximum) ÷ 3` of that stage's triangular distribution from the game's flow config (lines 290–295, 350–353).
- `stageMultiplier[stage] = inefficiencyParameter u` (flow config, default 1; templates seed 1.15) **if 2 or more staff rows have a non-zero allocation for that stage, else 1** (lines 297–306).
- `workdayMinutes` = scheduling config workday end − start, default 480 (8 h) if unset/unparseable (lines 356–378, `getWorkdayMinutes()`).
- Result is floored at 0 (`Math.max(0, util)`) and **can exceed 100%** — only the display bar is clamped to 100 (lines 342, 380–383).

Key code (lines 336–343):
```ts
const perPatient = workPerPatient.get(rt) ?? 0;
const totalDemand = expectedPatients * perPatient;
const capacity = count * workdayMinutes;
const util = capacity > 0 ? (totalDemand / capacity) * 100 : 0;
```

**Inputs trace:** student slider settings (appointments/day, staff counts, task allocations) + admin-configured scenario data (stage distributions, inefficiency `u`, scheduling config). So "where the number comes from" = a deterministic projection from current settings, not a simulation result.

## 2. Net Promoter Score (T2)

**Where:** `src/app/simulation/services/simulation-runner.service.ts`, `SimulationRunnerService.aggregateResults()` (private, line 663), NPS block at lines 776–788. Displayed rounded via `netPromoterScore: Math.round(nps)` (line 834) in the results dashboard and the admin simulation-control screen. It is a **synthetic score starting at 100 with penalties — it is not a survey-based promoter/detractor percentage.**

**Formula:**

```
NPS = clamp( 100 − delayPenalty − mdUnderPenalty − mdOverPenalty − abandonmentPenalty , −100, 100 )

delayPenalty        = min( avgInClinicDelay × 1.5 , 60 )
mdUnderPenalty      = max(0, 50 − mdBusyPercent) × 0.5
mdOverPenalty       = max(0, mdBusyPercent − 90) × 2
abandonmentPenalty  = avgAbandonedPerDay × abandonmentNpsPenalty
```

Key code (lines 777–788):
```ts
const delayPenalty = Math.min(avgInClinicDelay * 1.5, 60);
const mdUnderPenalty = Math.max(0, 50 - mdBusyPercent) * 0.5;
const mdOverPenalty = Math.max(0, mdBusyPercent - 90) * 2;
const abandonmentNpsPenalty = ops?.abandonmentNpsPenalty ?? DEFAULT_ABANDONMENT_NPS_PENALTY;
const abandonmentPenalty = (avgAbandonedPerDay ?? 0) * abandonmentNpsPenalty;
const nps = Math.max(
  -100,
  Math.min(
    100,
    100 - delayPenalty - mdUnderPenalty - mdOverPenalty - abandonmentPenalty
  )
);
```

**Inputs trace:**
- `avgInClinicDelay` = avg pre-exam delay + avg exam delay per patient, from the run's day results (lines 697–701). Post-exam delay is **not** counted.
- `mdBusyPercent` = total MD busy time ÷ total MD time × 100 across all simulated days (lines 707–721).
- `avgAbandonedPerDay` = abandoned requests ÷ window days, from the queue pre-simulation `preSimulateQueue()` (line 370), passed into `aggregateResults` by both interactive and batch runs.
- `abandonmentNpsPenalty` = admin-configured operational parameter (Strategy & Financial Configuration → Operational; seeded default **1.25**, also the code fallback `DEFAULT_ABANDONMENT_NPS_PENALTY = 1.25` at line 108; DB seed value 1.25 in `db/medi_hana_plain.sql`).

## 3. Task-allocation sliders (T3)

**Where the three-role allocation appears:**
- **Student editor (only interactive one):** `clinic-controls.component.ts` `onAllocationChange()` (lines 171–207) + sliders in `clinic-controls.component.html` (lines 56–66). Attributes = the 3 stages (Pre-Exam, Exam, Post-Exam); roles present per stage depend on the scenario template (Dr. Solo: MD only; With Admin: Admin+MD; With Nurse: Admin+Nurse+MD pre/post-exam, Nurse+MD exam).
- **Admin defaults & ceilings:** `resource-scenarios.component.ts` (edits default allocations, validates against ceilings) and `resource-scenario.service.ts` `getScenarioTemplateLimits()` (lines 327–356, per-template **maximum** % per stage/role: e.g. With Nurse pre-exam Admin 60 / Nurse 80 / MD 60).
- **Model:** `TaskResourceLimit` (`admin/models/simulation.models.ts:102`) has **only `maxAllocationPercent` — there is no minimum/floor field anywhere.** `maxAllocationFor()` logic is duplicated (not shared) in clinic-controls (line 166) and resource-scenarios (line 192).

**How it works today vs Brian's rule (MD floor ≈20 protected; Nurse+Admin share the rest and trade only against each other; MD gets the leftover):**
- ❌ **No MD floor exists.** Every student slider has `[min]="0"`, including MD; nothing stops MD going to 0.
- ❌ **Rebalancing trades against all other roles**, in saved-array order: when a slider pushes the stage total over 100, `onAllocationChange` cuts the *other rows in order* (which can cut MD when Nurse/Admin is raised, and vice-versa). When the total is under 100 it tops up the other rows in order up to their ceilings.
- ❌ **MD does not specifically get the leftover** — leftover goes to whichever rows come first in the array, capped by their ceilings.
- ✅ Stage totals are enforced to exactly 100 (`validate()`, lines 209–218) and per-role ceilings are respected.
- Prepared fix: see APPROVAL-BATCH.md item T3 (diff in `diffs/T3-clinic-controls-md-floor.patch`).

## 4. Batch run length (T4)

- **Today it is hardcoded to 120 days in two places and is NOT admin-editable:**
  - Admin: `admin/components/simulation-control/simulation-control.component.ts:56` — `batchSize = 120;`, passed as `batchDays: this.batchSize` when running (line 379). Its template already has a number input for it (`simulation-control.component.html:78`) but it is **`[disabled]="true"`**.
  - Student: `simulation/components/simulation-viewer/simulation-viewer.component.ts:34` — `batchDays = 120;`; its template input (`simulation-viewer.component.html:40`) is likewise `[disabled]="true"`.
  - The runner default if no value is passed is different again: `simulation-runner.service.ts:175` — `options.batchDays ?? 100`.
- **No config carries it:** not in SchedulingConfig, not in the DB, not in the Go API. Good news: scheduling configs are stored by the Go API as an opaque JSON blob (`src/api/route_admin_scheduling.go`), so adding a `batchDays` field to SchedulingConfig needs **no Go or DB change** — the exact pattern used by `arrivalPerturbEarlyMinutes` etc.
- Prepared fix: see APPROVAL-BATCH.md item T4 (diff in `diffs/T4-batch-days-admin-editable.patch`).

## 5. Fixed Cost (T5)

**Cost fields today** (`FinancialParameters`, `admin/models/simulation.models.ts:271–282`): revenuePerRoutinePatient, revenuePerUrgentPatient, costMdPerDay, costNursePerDay, costAdminPerDay, costExamRoomPerDay, costAiAssistantPerDay, costOvertimePerHour, costPatientDelayPerDay (the last one's UI input is currently hidden behind an `@if(false)` in strategy-config).

**There is NO fixed-cost field** — every cost is per-day-per-resource, per-hour, or per-patient. Profit is computed in two places that must stay in sync:
- `simulation-runner.service.ts` lines 758–767: `dailyCosts = MD×costMdPerDay + NURSE×costNursePerDay + ADMIN×costAdminPerDay + EXAM_ROOM×costExamRoomPerDay + (avgOvertimeMinutes/60)×costOvertimePerHour`; `avgDailyProfit = dailyRevenue − dailyCosts`.
- `simulation/services/strategy.service.ts` `computeLocalSummary()` lines 69–75 (the live student-facing projection; note it omits overtime and AI-assistant costs).

Storage is an opaque JSON blob on the Go side too (`src/api/route_admin_strategy.go`), and `validateFinancialParams` is generic (any numeric field must be ≥ 0), so a new field needs **no Go, DB, or validator change**. Defaults live in `shared/game-shared.service.ts` `getDefaultFinancialParams()` (lines 11–23).
- Prepared fix (interprets "Fixed Cost" as a fixed cost **per day**, matching the daily-profit model): see APPROVAL-BATCH.md item T5 (diff in `diffs/T5-fixed-cost-per-day.patch`).

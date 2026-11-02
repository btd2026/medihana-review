# MediHana — Review Package for Prof. Pinker

*Prepared by Brian Di Bassinga, Nov 1–2, 2026, from the MediHana meeting notes. This repository is a read-only review package: findings, the exact formulas behind two numbers, and proposed fixes prepared as patches. **Nothing here has been applied to the MediHana project** — each fix is awaiting approval.*

**Live site version:** https://btd2026.github.io/medihana-review/

## What's inside

| Document | What it answers |
|---|---|
| [APPROVAL-BATCH.md](APPROVAL-BATCH.md) | The master list — per item: finding, evidence (file + line), proposed action, risk. Formatted for yes/no decisions. |
| [FORMULAS-AND-PARAMETERS.md](FORMULAS-AND-PARAMETERS.md) | Exact formulas traced from the code: Projected Staff Utilization, Net Promoter Score, slider mechanics, batch length, cost fields. |
| [NPS-PROPOSAL.md](NPS-PROPOSAL.md) | A proposal to replace the current synthetic NPS with a real promoter/detractor calculation from per-patient data. |
| [BASELINE-REPORT.md](BASELINE-REPORT.md) | Build/test baseline of the copied source: Go builds clean; the Angular test suite did not compile (fix prepared). |
| `diffs/` | One patch per proposed fix, each independently applicable. |

## Headline findings

- **Projected Staff Utilization** is a live deterministic projection in Clinic Controls (expected patients × work per patient ÷ capacity) — not a simulation result.
- **Net Promoter Score** is synthetic: it starts at 100 and subtracts penalties (including for MD busy %, which patients never experience directly). A real per-patient NPS is proposed in NPS-PROPOSAL.md.
- **Task-allocation sliders** do not protect the MD: there is no MD floor anywhere, and rebalancing can trade against the MD. Fix prepared; per Brian's decision the floor is **admin-settable** (a Min % beside the existing Max % ceilings in Resource Scenarios, default 20, per stage) — the patch in `diffs/` is the revised admin-configurable version, verified 11/11 tests.
- **Batch run length (120 days)** is hardcoded and its inputs are disabled. Fix prepared: admin-editable in Scheduling Configuration, default 120.
- **Fixed Cost** does not exist in Financial Parameters. Fix prepared: a per-day fixed cost counted in both profit calculations.
- **Roles:** there are only two — student and full admin. There is no instructor role, so a teacher would currently need full admin (including user/game deletion). Open questions for Prof. Pinker are in APPROVAL-BATCH.md (item T6).
- **Test suite** did not compile (two stale specs, plus three more once it ran). Fix prepared — test files only, no production code; suite now passes 11/11.

## Verification

All prepared fixes were verified together on a copy of the source: TypeScript app + spec checks clean, production build succeeds, 11/11 tests pass, Go build + vet pass (no Go changes in any patch).

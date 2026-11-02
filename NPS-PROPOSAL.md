# MediHana — A real Net Promoter Score calculation (proposal)

*Brian's direction, Nov 1, 2026 (~11:55 PM ET): the current NPS is synthetic — "let's come up with an actual calculation." This proposal replaces the penalty fudge with the standard NPS definition, computed from the per-patient data the simulation already produces. Status: proposal for Brian's approval, then Edi's — NOT implemented, nothing patched.*

## What's wrong with the current one
`aggregateResults()` in `simulation-runner.service.ts` (lines 776–788) starts at 100 and subtracts invented penalties. Two structural problems:
1. **It isn't NPS.** Real NPS is %Promoters − %Detractors from individual respondents scored 0–10. This is a 0–100 index with NPS's name and range pasted on.
2. **Patients can't feel MD busy %.** The mdUnder/mdOver penalties punish/reward an operations metric patients never experience directly — its real effect on patients already shows up as delay, so it double-counts. The abandonment penalty weight (1.25) is an admin knob with no meaning a student can explain.

## The data already exists
`aggregateResults()` already loops over every simulated patient (`day.patients`) with per-patient fields: `delayPre`, `delayExam`, `delayPost`, `inClinicTime`, `systemTime`. Abandoned patients come from the queue pre-simulation (count via `avgAbandonedPerDay` × days, and `abandonmentRatePercent`). Everything below is computable in the same function, from the same run — no engine changes.

## Proposed calculation
**Step 1 — score each patient 0–10 from their own experience:**

```
delay_p  = delayPre + delayExam + delayPost        (all three stages — current formula drops post-exam)
score_p  = max(0, 10 − delay_p / 10)                (each 10 minutes of waiting costs 1 point; 100+ min → 0)
abandoned patients: score 0 (they sought care and never got it)
no-shows: excluded (they never experienced the clinic; tracked separately already)
```

**Step 2 — classify with the standard NPS bands:**

```
Promoter  : score ≥ 9      (≤ 10 min total delay)
Passive   : score 7–8      (20–30 min delay)
Detractor : score ≤ 6      (≥ 40 min delay, or abandoned)
```

**Step 3 — the actual NPS:**

```
NPS = 100 × (Promoters − Detractors) / (Promoters + Passives + Detractors)
    range −100…+100, exactly like real NPS
```

Population = everyone who sought care (seen + abandoned), so a clinic can't buy a good score by letting unhappy patients abandon.

## Worked sanity check
A run with 100 patients seeking care: 10 abandon (detractors), 90 seen. Of the seen: 45 waited ≤10 min (promoters), 25 waited 20–30 min (passives), 20 waited ≥40 min (detractors).
NPS = 100 × (45 − 30) / 100 = **+15** — a mediocre clinic with a real access problem, and a student can trace exactly which patients drove it. The current formula on the same run would return a number no one can decompose.

## What changes / what it costs
- The two MD-busy penalties and the `abandonmentNpsPenalty` knob come OUT of NPS (MD busy % stays on the dashboard as its own metric). The delay cap and the 1.5× weight go away too.
- One visible constant remains — 10 minutes of delay = 1 point — stated in one line, and it could be admin-configurable exactly like the old penalty was, if Edi wants a difficulty knob.
- Scores will generally read LOWER than the old formula (which started at 100). That's expected — worth warning Edi so the change isn't read as the sim getting harsher, but as the score becoming honest.

## Open questions for Edi
1. Appointment access delay (request→appointment days, already computed as `avgRequestToAppointmentDelayDays`) — should it also cost points (e.g. −1 per week of access delay, at the population level)? Real patients absolutely weigh it; v1 above is in-clinic only to stay strictly per-patient.
2. Keep the old score around as a separate "satisfaction index" for comparison during transition, or replace outright?
3. Should the 10-min-per-point slope be admin-configurable?

## If approved
Small patch, same function: compute per-patient scores inside the existing patient loop, count bands, replace lines 776–788. Verified the same way as the other batch items (tsc / build / 11 tests) and added to the approval batch as item T2b.

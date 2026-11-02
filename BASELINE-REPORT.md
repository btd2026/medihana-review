# MediHana — Baseline Report (Step 1)

*Run: Sun Nov 1, 2026 (late evening ET). Source copied read-only from Brian's Mac (`/Users/briandibassinga/Github/medihana`) to `~/workspace/medihana/` on this VM via the Mac device's file upload. **Nothing was changed on the Mac and nothing was written back.** The only local change after copying was `npm install` (creates `node_modules/` + `.angular/` locally) and removing the local `go build` output binary so the tree stays source-only.*

## TL;DR
- **Go backend builds clean:** `go build ./...` exit 0, `go vet ./...` exit 0, zero errors/warnings.
- **Frontend test suite does not compile — Brian's Oct 2026 finding reproduced exactly.** `npx ng test --watch=false` fails during bundle generation with **exactly 2 TypeScript errors, both in spec files** (no errors in application code). `tsc -p tsconfig.spec.json --noEmit` independently confirms the same 2 errors and no others.

## File inventory (copied from the Mac)
115 source files copied, preserving relative paths:

| Area | Files | Notes |
|---|---|---|
| Project root | 9 | package.json, package-lock.json, angular.json, tsconfig.json, tsconfig.app.json, tsconfig.spec.json, README.md, proxy.config.json, media-hana.code-workspace |
| src/ (top level) | 3 | index.html, main.ts, styles.scss |
| src/api/ (Go backend) | 13 | 11 .go files + go.mod + go.sum |
| src/app/ (Angular frontend) | 87 | admin 32, simulation 27, auth 11, shared 11, app root 6 |
| db/ | 2 | medi_hana.sql (35,852 B), medi_hana_plain.sql (56,082 B) |
| public/ | 1 | logo.svg |

**Skipped on the Mac (not copied):**
- `node_modules/` and `dist/` — skipped entirely (not enumerated/counted).
- `src/api/medihana` — compiled Go binary, 15,855,074 bytes.
- `src/api/medihana.log` — log file (0 bytes).
- Favicon binaries (names only): `src/favicon.ico`, `src/favicon-16x16.png`, `src/favicon-32x32.png`, `public/favicon.ico`, `public/favicon-16x16.png`, `public/favicon-32x32.png`.
- **Present, not copied (credential-bearing):** `LOCAL_TEST_LOGIN.txt` (root, 148 bytes) and `src/api/config.json` (195 bytes). Their contents were not read and are not reproduced anywhere in this report or the local tree.

Also present in the local folder but **not** part of the Mac copy: `edi-meeting-notes-2026-11-01.md` and `TODO-FROM-EDI-NOTES.md` (added by another agent working in the same local folder).

## Toolchain on this VM
- Go: **not pre-installed**; installed go1.27.3 linux/amd64 from go.dev into `~/workspace/tools/go` for this baseline (go.mod requires `go 1.27`).
- Node: **v24.22.1**; npm: **10.9.4**; npx available.
- `npm install --no-audit --no-fund` in the copied tree: **exit 0 in ~35 s**, no errors.

## Go backend — PASS
In `src/api` (module `medihana`; deps chi v5.3.2, google/uuid v1.6.0, jackc/pgx v5.11.0, golang.org/x/crypto v0.57.0):
- `go build ./...` → **exit 0**, no output.
- `go vet ./...` → **exit 0**, no output.

## Frontend test suite — DOES NOT COMPILE (reproduced)
Command: `npx ng test --watch=false` (builder `@angular/build:unit-test`, Vitest). Exit code **1**, failing at "Application bundle generation failed" before any test ran:

```
✘ [ERROR] TS2749: 'AdminGuard' refers to a value, but is being used as a type here. Did you mean 'typeof AdminGuard'? [plugin angular-compiler]

    src/app/auth/admin.guard.spec.ts:6:13:
      6 │   let guard: AdminGuard;
        ╵              ~~~~~~~~~~

✘ [ERROR] TS2305: Module '"@angular/core/testing"' has no exported member 'async'. [plugin angular-compiler]

    src/app/auth/login/login.component.spec.ts:1:9:
      1 │ import { async, ComponentFixture, TestBed } from '@angular/core/tes…
        ╵          ~~~~~
```

Independent cross-check, `npx tsc -p tsconfig.spec.json --noEmit` → exit 2, the same two errors and nothing else:
```
src/app/auth/admin.guard.spec.ts(6,14): error TS2749: 'AdminGuard' refers to a value, but is being used as a type here. Did you mean 'typeof AdminGuard'?
src/app/auth/login/login.component.spec.ts(1,10): error TS2305: Module '"@angular/core/testing"' has no exported member 'async'.
```

**What the two errors are (diagnosis only — nothing was fixed in this step):**
1. `admin.guard.spec.ts` was written for the old class-based guard pattern (`let guard: AdminGuard; TestBed.inject(AdminGuard)`), but `AdminGuard` in `src/app/auth/admin.guard.ts` is a **functional guard** (`export const AdminGuard = () => { … }`), so it can't be used as a type or injected that way. The spec needs rewriting for a functional guard (run it in an injection context).
2. `login.component.spec.ts` imports `async` from `@angular/core/testing` — that export was removed in modern Angular (replaced by using an `async` `beforeEach` directly / `waitForAsync`). It's a leftover from an old Angular version's generated spec; the same file also still uses `declarations: [LoginComponent]`, the pre-standalone pattern.

Also seen but **not an error**: a Sass deprecation warning — `src/styles.scss:29` uses `@import "tailwindcss"`, deprecated and slated for removal in Dart Sass 3.0.0.

## Spec/test files present (8)
- `src/app/app.spec.ts`
- `src/app/admin/admin.component.spec.ts`
- `src/app/auth/admin.guard.spec.ts` ← compile error #1
- `src/app/auth/player.guard.spec.ts`
- `src/app/auth/login/login.component.spec.ts` ← compile error #2
- `src/app/shared/confirm/confirm.component.spec.ts`
- `src/app/shared/error-dialog/error-dialog.component.spec.ts`
- `src/app/shared/utils.service.spec.ts`

No Go test files (`*_test.go`) exist in `src/api`.

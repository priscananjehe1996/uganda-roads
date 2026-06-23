# Source-Code Audit — Uganda National Roads Platform (UGROADS)

_Audit date: 2026-06-23 · Branch: `main` · Auditor: GIS/Platform engineering_

This is a point-in-time review of the platform source for security, build/deploy
integrity, architecture and data handling. It is advisory; items are ranked by
priority.

## 1. Summary

| Area | Status | Notes |
|------|--------|-------|
| Secrets / key handling | ✅ Good | Anon key only in client; no service_role in source or bundle. |
| Build & deploy | ✅ Fixed | Now runs on GitHub Actions (was failing on local-only steps). |
| Architecture | ✅ Sound | React + TS + Vite SPA; clear module boundaries. |
| Data integrity | 🟡 Watch | Committed `public/data` is the served source of truth; keep in sync. |
| Type safety | 🟡 Watch | A few `as unknown as` casts and `any` in data adapters. |
| Dependencies | 🟡 Watch | `npm install` (not `ci`) in CI tolerates lock drift — see §5. |

## 2. Security

- **Supabase keys.** `src/lib/supabase.ts` reads `VITE_SUPABASE_URL` /
  `VITE_SUPABASE_ANON_KEY` (project `vbidhkvzjigatfygnyc`). The anon key is
  publishable and safe for the browser bundle. **The `service_role` key does not
  appear in any source file, `.env`, or built output** — verified by grep. RLS
  must remain enabled on the Supabase project so the anon key is read-only for
  the public.
- **AssetBot / Fable LLM (`src/modules/AssetBot/fable.ts`).** Correctly designed
  for a public site: **no API key is bundled.** Chat is either proxied through a
  local server (key server-side) or uses an operator-pasted key kept only in
  that browser's `localStorage`. The `@anthropic-ai/sdk` import is dynamic
  (code-split) so it never loads unless the feature is used.
- **`.env` files** are git-ignored (`.env`, `.env.local`). Confirmed.
- **Recommendation.** Periodically confirm Supabase RLS policies are read-only
  for `anon`; never add a `service_role` key to any Vite env (it would ship to
  every visitor).

## 3. Build & deployment

- **Now CI-based.** `.github/workflows/deploy.yml` builds on GitHub's runners on
  every push to `main` and deploys `dist/` to `gh-pages`. This removed the
  dependence on local builds, which were unreliable (Google Drive cannot host
  `node_modules`; `C:\tmp` build dirs were auto-wiped).
- **Why it had been failing:** the build step ran `npm run build`, which chains
  `export-data` (Python, reads G: Drive) and `tsc -b` — neither valid on a CI
  runner. Fixed to `npx vite build` (data is committed under `public/data`).
- **`sync.yml`** is a weekly cron rebuild with the same corrected build.
- **Recommendation.** Treat `main` as the deploy trigger; do not rely on manual
  local builds. Keep `public/data` committed and current.

## 4. Architecture

- **Stack:** React 18 + TypeScript + Vite 6, Tailwind, Leaflet/react-leaflet,
  Recharts, ExcelJS, Supabase JS.
- **Module layout** (`src/modules/*`) is clean and domain-aligned: RMS, BMS, PMS
  (incl. ROMDAS readers), TIS/ATC, RoadCondition, RoadReserve, Budget, HDM4,
  Projects, Analytics, Admin, MindMap, Auth, GlobalCaseStudies, AssetBot.
- **Standalone apps** (NRMS/NBMS/NTIS/NPMS) share one codebase via per-app Vite
  configs + entry points and the `VITE_APP_ID` / `VITE_STANDALONE` flags.
- **Shared layer** (`src/shared`): `nowcast.ts` (live now-cast model),
  `dataDictionary.ts` (466 curated terms), `InfoTip.tsx`,
  `SortableFilterableTable.tsx` (auto-wires dictionary tooltips on columns).

## 5. Findings & recommendations (by priority)

1. **[Med] CI uses `npm install --legacy-peer-deps`, not `npm ci`.** Tolerant of
   lock drift but less reproducible. _Action:_ regenerate `package-lock.json`
   (`npm install --package-lock-only`) and consider restoring `npm ci` once the
   lock is verified in sync, so builds are deterministic.
2. **[Med] Declared dependencies vs. imports.** `@anthropic-ai/sdk` was imported
   but undeclared (only in a local `node_modules`), which broke the clean CI
   build. _Action (done):_ added to `package.json`. _Going forward:_ never rely
   on un-declared local packages; CI is the source of truth for resolvability.
3. **[Low] Type-safety casts.** A few `as unknown as <T>` and `any` in data
   adapters (e.g. budget series). _Action:_ tighten types where source shapes
   are known.
4. **[Low] Data sync.** `public/data/*` (52 files) is the served data and must be
   kept in step with the canonical G:/Supabase sources. _Action:_ keep the
   export step as the single path that refreshes `public/data` before commit.
5. **[Low] Forecast-year literals.** Hardcoded `2030`/`2035` are legitimate
   forecast targets and year-pills; current-state labels already use the dynamic
   year (`new Date().getFullYear()`). No action — verified during this audit.

## 6. Verified-clean during this audit

- No `service_role` key anywhere in `src/`, `.env*`, or build output.
- MoWT logo present in Header, Sidebar, LoginPage, AccessPending, RMSFieldShell
  and all three standalone entry points.
- "As of {year}" labels are dynamic; live now-cast ticks against the real clock.
- Traffic statistics are referenced to base year 2016 throughout the traffic
  modules and the dictionary.

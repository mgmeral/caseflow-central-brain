# MOBILE-001 — Mobile UI/UX Visual Modernization

## Objective
Give `caseflow-mobil` a real visual design system — color/typography/spacing tokens, iconography, color-coded status/priority badges, real empty states, and a shared component set — so it reads as a modern app visually consistent with `caseflow-fe`, instead of today's unstyled, single-flat-palette, icon-less screens built from raw `Text`/`View` + ad hoc `StyleSheet` numbers.

## Status
IN_PROGRESS

## Priority
P2 (product-quality/UX gap, not a functional defect or outage — but raised directly as a product concern, not a nice-to-have)

## User Request
"mobil uygulamamız çok atıl. bazı özellikler var ama yetersiz. kullanıcı deneyemi çok düşük. bir diğer sorun ise hiç bir görselliği yok. FE ile aynı görselleri ya da modern biz uygulamayı barındırmalı ama şu an mobil psuedo gibi. buraları güzelleştirmemiz lazım."

(Gloss: the mobile app feels stagnant; existing features are insufficient and UX is poor; on top of that it has no visual design at all — it should carry the same visuals as the FE, or otherwise look like a modern app, but right now it looks like a mobile pseudo/mockup; this needs to be made visually polished.)

This task scopes the **"no visual identity / looks like a pseudo app"** half of that complaint. The **"features are insufficient"** half is feature-completeness, not visual design, and is already tracked separately — see Context below.

## Context
Two things are already true and tracked elsewhere, confirmed by re-reading `caseflow-mobil/src` directly for this task:

- Feature-completeness gaps (read-only ticket workflow, no assign/transfer/notes, no tags/Jira/attachments, missing parity with `caseflow-fe`) are the subject of `tasks/active/ALIGN-001-MOBILE-FE-BE-ALIGNMENT.md` and `tasks/active/ALIGN-002-MOBILE-FULL-FEATURE-PARITY.md`. **Not duplicated here.**
- The **visual** gap this task addresses is not previously documented anywhere in Central Brain. Verified directly against `caseflow-mobil/src` today:
  - `src/shared/theme/` contains exactly one file, `colors.ts` — 9 flat hex values (`background`, `surface`, `surfaceMuted`, `text`, `muted`, `border`, `primary`, `onPrimary`, `danger`). No typography scale, no spacing scale, no radius/elevation tokens, no semantic status/priority/SLA-risk colors.
  - No icon library is installed (`package.json` has no `@expo/vector-icons`, `react-native-vector-icons`, or any SVG/icon dependency) — every screen is text-only, including tab bars and status/priority indicators.
  - `src/shared/components/` has exactly four components: `Screen`, `SectionCard`, `MetricCard`, `CenteredState`. There is no `Badge`/`Chip`, `Button` (variants), `Avatar`, or illustrated `EmptyState` — e.g. `HomeScreen.tsx` renders ticket `priority` and `status` as plain `Text`, not color-coded badges.
  - No animation/motion library (no `react-native-reanimated`, no skeleton/shimmer loading — `HomeScreen`'s loading state is the literal text "Loading dashboard").
  - No custom fonts (`expo-font` not used) — system default font only.
  - Only 16 `.tsx` files total across the whole app (8 screens + 4 shared components + a few others) — this is a small enough surface that a full visual pass is tractable in one task.
- `caseflow-fe` styles with Tailwind CSS (`caseflow-fe/tailwind.config.js` exists and is the actual source of FE's design tokens — colors/spacing/radii). **No shared, repo-agnostic design-token document exists in Central Brain today** — this task must either extract FE's Tailwind tokens into a shared reference or explicitly define a new mobile-first baseline that stays visually consistent with FE's existing palette, rather than assuming a shared source of truth already exists to copy from.

## Affected Repositories

| Repository | Agent | Responsibility | Status |
|---|---|---|---|
| caseflow-mobil | Claude | Build out `src/shared/theme/` into a full token set, add an icon library, build the missing shared components (Badge/Chip, Button, Avatar, illustrated EmptyState), and re-skin all 8 existing screens with them. | IN_PROGRESS |

**Not affected:**
- `caseflow-be` — no API/data change; this is a client-side visual layer only.
- `caseflow-fe` — read-only reference for its Tailwind tokens and component look; no FE code change in this task (extracting a literal shared-tokens package is a possible fast-follow, not required here — see Notes).
- `caseflow-ai-service` — not touched by mobile's UI layer.

## Dependencies

### Depends On
None.

### Blocks
None formally — see Notes for a scheduling recommendation (not a hard dependency edge) regarding `ALIGN-001-MOBILE-CORE`/`ALIGN-001-MOBILE-EXT` and `ALIGN-002`'s mobile sub-tasks, which add new UI to the same screens this task restyles.

## Contract Impact
- API: None.
- Event: None.
- Database: None.
- Authentication: None.
- None (pure client visual layer — no contract touched).

## Tasks

### Mobile
- [x] Extend `src/shared/theme/colors.ts` into a full `src/shared/theme/` module: typography scale (font sizes/weights/line-heights), spacing scale, radius tokens, elevation/shadow tokens, and semantic colors for ticket priority, status, and SLA-risk states (today rendered as unstyled text). Cross-check the palette/spacing against `caseflow-fe/tailwind.config.js` so the two clients are visually consistent, not coincidentally similar. — Done: `colors.ts` extended, plus new `spacing.ts`, `radii.ts`, `typography.ts`, `shadows.ts`, `status.ts` (priority/status/SLA → color config, mirroring `caseflow-fe`'s `Badge.tsx`/`PriorityBadge.tsx`/`TicketStatusBadge.tsx` variant-for-variant), `index.ts` barrel.
- [x] Add an icon library and replace text-only tab labels, status indicators, and action affordances with icon+label treatments matching how `caseflow-fe` uses icons for the same concepts. — Done, with a correction to this checklist's own assumption: `@expo/vector-icons` was not actually installed in this project (verified — zero icon dependency existed). Installed `lucide-react-native` + `react-native-svg` instead (via `npx expo install`, SDK-57-compatible versions), since `caseflow-fe` itself uses `lucide-react`/`lucide-react-native` gives icon-for-icon parity with FE rather than a different icon set that merely looks similar.
- [x] Build the missing shared components in `src/shared/components/`: `Badge`/`Chip`, `Button`, `Avatar`, illustrated `EmptyState`. — Done, plus two components beyond the original checklist: `PriorityBadge`/`StatusBadge`/`SlaBadge` (typed wrappers around `Badge` using the new status-config mapping) and `Skeleton`/`ListSkeleton` (needed once loading states were upgraded from spinner-only, see below).
- [x] Apply the new theme + components across all current screens (`HomeScreen`, `LoginScreen`, `CasesScreen`, `CaseDetailScreen`, `CustomersScreen`, `InboxScreen`, `NotificationsScreen`, `ProfileScreen`) — Done for all 8, including `RootNavigator.tsx` (tab bar icons/active-inactive tint, stack header styling) which the original checklist didn't call out separately.
- [x] Replace plain "Loading…" text states with skeleton/shimmer placeholders, and add pull-to-refresh on list screens. — Done: `CasesScreen`, `InboxScreen`, `CustomersScreen`, `NotificationsScreen` get `ListSkeleton` + `RefreshControl`; `HomeScreen` gets an inline metrics-grid skeleton + pull-to-refresh. `CaseDetailScreen`, `LoginScreen`, `ProfileScreen` intentionally keep the simple spinner (`CenteredState`) — brief, one-shot loads where a skeleton wasn't worth the extra surface; flagged here rather than silently scoped out.
- [ ] Check `app.json`'s `icon`/`splash` values are not left at Expo's placeholder defaults. — **Confirmed a real gap, not fixed**: `assets/icon.png` is literally Expo's stock template icon (visible construction guides/circles baked into the image, not a rendering artifact). Left as-is — generating real brand assets is a design decision outside this pass's scope, not a code change. Flagged for the user.
- [ ] Optional/evaluate-only: `react-native-reanimated` — **Not adopted.** `Skeleton`'s shimmer uses React Native's built-in `Animated` API instead; a full animation library wasn't needed for this pass's scope (press states via `Pressable`'s own `pressed` render-prop were enough). Revisit if a later task needs shared-element/gesture-driven transitions.

## Task Graph
```yaml
task_id: MOBILE-001
title: Mobile UI/UX Visual Modernization
status: PLANNED

tasks:
  - id: MOBILE-001-MOBILE
    repository: caseflow-mobil
    agent:
      provider: claude
    status: IN_PROGRESS
    depends_on: []
```

## Acceptance Criteria
- [ ] No hardcoded hex colors or magic-number spacing remain in any screen-level `StyleSheet.create` call — all visual values are pulled from `src/shared/theme/` (verifiable by grepping `src/` for hex literals outside `theme/`).
- [ ] Ticket priority, status, and SLA-risk are rendered as color-coded badges/chips everywhere they appear (`HomeScreen`, `CasesScreen`, `CaseDetailScreen`, `InboxScreen`), not raw text.
- [ ] Every list/detail screen has a real empty state (icon/illustration + message + action where applicable) instead of a plain text line.
- [ ] All 8 existing screens use the new shared component set (`Badge`, `Button`, `Avatar`, `EmptyState`) rather than one-off inline styling.
- [ ] A side-by-side screenshot walkthrough (mobile vs. the equivalent `caseflow-fe` view, for at least Home/dashboard, case list, and case detail) is attached to or linked from this task before it is marked DONE, so "visually consistent with FE" is verified rather than assumed.

## Validation
- [ ] Backend tests — N/A (no backend change)
- [ ] Frontend tests — N/A (no frontend code change; used only as a visual reference)
- [x] Mobile tests — `npm run typecheck` (`tsc --noEmit`) and `npm test` both pass clean, including the pre-existing `LoginScreen.test.tsx` (asserts on `"Sign In"` / the validation-error string, both preserved verbatim through the `Button` refactor). Jest needed a config fix unrelated to the visual work: `lucide-react-native` ships ESM-only by default and its `package.json` `"react-native"` field points at the `.mjs` build, which `jest-expo`'s default `transformIgnorePatterns`/transform regex doesn't handle — fixed via a `moduleNameMapper` to its CJS build plus an extended `transformIgnorePatterns` in `package.json`. **No new component-level test files were added** for `Badge`/`Button`/`Avatar`/`EmptyState` themselves — the Acceptance Criteria's test item is not yet satisfied, only the "don't break existing tests" bar.
- [ ] Integration validation — Metro successfully bundles the web target (`expo start --web`, HTTP 200, ~7.7MB bundle containing the new screens/components) and `curl` confirms real content is served, which rules out a build-breaking error. **Not done**: an actual on-screen visual check — this session had no connected browser-automation extension, so nothing was screenshotted. The Acceptance Criteria's side-by-side FE/mobile screenshot walkthrough is still outstanding.

## Agent Instructions

### Copilot
Read `docs/architecture/mobile.md`, `repos/mobile/README.md`, and this task's Context section first. For the visual reference, read (do not modify) `caseflow-fe/tailwind.config.js` and a handful of its key components (its card, badge/chip, and button components) to ground the token/component work in FE's actual current look rather than guessing. This is a **visual-layer-only** task — do not change any API-calling code, data-fetching logic, or business logic in `caseflow-mobil`; if a visual improvement is blocked by a missing data field (e.g., a status value with no obvious color mapping), flag it back to Central Brain rather than inventing a mapping. Do not modify `caseflow-be`, `caseflow-fe`, or `caseflow-ai-service`.

## Completion Requirements
A task is not COMPLETE until the shared theme/component work lands, all 8 screens are re-skinned, and the screenshot walkthrough in Acceptance Criteria is attached and reviewed.

## Notes

### Implementation pass (2026-09-19) — ownership deviation and current state
Implemented directly in this session (Claude, working in `caseflow-mobil` on request) rather than handed off to GitHub Copilot per the default in `agents/AGENT-OWNERSHIP.md` — an explicit, session-level override per that doc's "Provider is a property of the task graph, not a hardcoded assumption" rule, not a change to the repository's default ownership going forward.

What landed (uncommitted in `caseflow-mobil` as of this note — awaiting the user's go-ahead to commit):
- Full theme module (`colors.ts` extended + new `spacing.ts`/`radii.ts`/`typography.ts`/`shadows.ts`/`status.ts`/`index.ts`), cross-checked against `caseflow-fe`'s actual `Badge`/`PriorityBadge`/`TicketStatusBadge` components (not just its Tailwind config, which turned out to carry no custom color palette — only shadow/radius extensions — so the FE component source was the real reference for status/priority colors).
- New shared components: `Badge`, `PriorityBadge`, `StatusBadge`, `SlaBadge`, `Button`, `Avatar`, `EmptyState`, `Skeleton`, `ListSkeleton`.
- All 8 screens + `RootNavigator.tsx` re-skinned to use them.
- `lucide-react-native` + `react-native-svg` added as the icon library (see Tasks checklist above for why this was chosen over `@expo/vector-icons`).

Known gaps, left for a follow-up rather than silently closed:
1. **No visual verification happened.** No connected browser-automation extension was available in this session; verification was limited to `tsc --noEmit`, `npm test`, and confirming Metro's web bundle builds and serves (200 OK, correct bundle contents) — a build-success check, not a "does it actually look right" check. The Acceptance Criteria's screenshot walkthrough is still required before this task can move to DONE.
2. **No component-level tests** were added for the new shared components — existing tests were kept green, but the Validation checklist's "component/snapshot tests" item is unmet.
3. **App icon/splash are still Expo's stock placeholder assets** (confirmed by visual inspection — the icon has construction-guide circles baked into the PNG). Needs real brand assets from the user/design, not something to generate blindly.
4. Not yet committed — the working tree in `caseflow-mobil` has the changes but no commit, pending user confirmation.

- **Scheduling recommendation (not a hard dependency edge):** `ALIGN-001-MOBILE-CORE`/`ALIGN-001-MOBILE-EXT` and `ALIGN-002`'s mobile sub-tasks are already `READY`/in-flight and add substantial new UI (status change, assign/transfer/notes, tags, Jira, attachments, notification-channel admin, mail templates, dashboard/reports, customer detail) to the same screens this task restyles — most notably `CaseDetailScreen`. Building that new UI with today's plain-text styling and then restyling it again here is likely rework. Recommend either sequencing this task's shared-component work first, or close coordination between whoever implements each task concurrently, so new feature UI is built directly against the new `Badge`/`Button`/`EmptyState` components rather than twice. This is a human scheduling call, not something this task can force by itself.
- **Out of scope for v1:** dark mode, a full custom-font pipeline (`expo-font`) unless trivial, and a literal shared design-token package published for both `caseflow-fe` and `caseflow-mobil` to import (worth considering as a fast-follow once mobile's own tokens stabilize, but not required to close this task).
- If FE's own Tailwind tokens turn out to be inconsistent or undocumented in a way that blocks extracting a clean reference palette, note that finding here and in `repos/frontend/README.md` rather than guessing a palette.

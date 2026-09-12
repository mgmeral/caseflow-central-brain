# CaseFlow Mobile (`caseflow-mobil`)

React Native + TypeScript mobile client foundation for the CaseFlow platform.

## Current scope

Phase 1 foundation:

- Expo-based React Native app with a single shared codebase
- React Navigation app shell
- TanStack Query setup for server state
- Zustand session store backed by `expo-secure-store`
- JWT login + refresh-ready API client for the existing `caseflow-be` contract
- Domain-oriented source structure
- Initial mobile screens for Home, Cases, Inbox, Customers, Notifications, and Profile
- Read-only case detail and conversation foundation
- Jest + React Native Testing Library setup

## Repository structure

```text
src/
├── app/
├── auth/
├── cases/
├── conversations/
├── core/
├── customers/
├── inbox/
├── notifications/
├── settings/
├── shared/
└── types/
assets/
App.tsx
app.json
package.json
tsconfig.json
```

## Environment

```bash
cp .env.example .env
```

```env
EXPO_PUBLIC_API_BASE_URL=http://localhost:8080/api
EXPO_PUBLIC_ENABLE_AI=true
EXPO_PUBLIC_ENABLE_PUSH=false
```

## Scripts

```bash
npm start
npm run android
npm run ios
npm run web
npm run typecheck
npm test
```

## Auth model

Follows the **existing backend auth contract** (see [../backend/frontend-contract.md](../backend/frontend-contract.md)):

- `POST /api/auth/login`
- `POST /api/auth/refresh`
- `POST /api/auth/logout`
- `GET /api/auth/me`

Does **not** implement OIDC/PKCE — the backend does not currently expose that flow.

## Notes

- Inbox tab is shown only when the signed-in user has `ADMIN_POOL_VIEW`.
- Conversation rendering currently adapts the backend's email-thread contract (`GET /tickets/{id}/email/thread`).
- Push notifications and reply attachments remain backend-dependent follow-up work.

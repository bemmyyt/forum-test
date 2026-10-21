# Forum API migration status

This branch deliberately leaves the existing production forum behavior unchanged unless the URL contains `?forumApiPreview=1`.

## Preview
After deploying the **matching backend commit** on Webdock, open `https://forum.adamisbored.xyz/?forumApiPreview=1` on a staging deployment of this branch. This reads the first 50 posts plus a bounded reply collection through `GET /v1/forum/feed`. If the API is unreachable, it falls back to existing Firebase reads. Posting, replying and post deletion are disabled in preview. Other account/admin/DM functionality is not migrated and MUST NOT be considered read-only.

**Do not merge as a completed migration.** This is a reviewable, staged implementation. Do not turn on `ENABLE_WRITES` yet.

## Blocking work before production
1. Inspect the deployed Firebase Security Rules (not included in this repository). They must deny direct writes to posts and replies and protect all user/admin/DM documents; verify that the current account handoff, registration, profiles, bans, and DM flows still work after their API migration.
2. Move every privileged mutation from browser Firebase SDK to server-side authenticated endpoints, including admin moderation and ban operations. Do not trust a username as proof of admin status; use Firebase Auth UID/claims.
3. Add server-side content moderation and enforce approved/pending visibility for **both** API and fallback reads. The API currently records pending writes if enabled; the legacy Firestore reader would not hide them.
4. Fix the current unbounded Firebase reads, and implement proper cursor pagination for posts/replies and user lookups. The preview feed currently returns only 50 posts and at most 500 replies.
5. Keep DMs private: public fallback must never expose the directMessages collection; DM endpoints must authorize conversation membership.
6. Test authenticated posting, editing/deleting, banned accounts, admin roles, CORS, rate limits, token expiry, API outage, and rollback on staging.
7. Only after those checks, switch the default frontend to the API and enforce read-only fallback for all mutations.

## VPS update (when ready for staging)
The running Webdock API was installed from an **older branch** and does not have `/v1/forum/feed` until the backend is deployed. Do not deploy the new backend or merge the frontend without a staging test. API runs via `forum-api.service`; the existing service should remain active during review.

## DM API progress (2026-10-21)
- Added `GET /v1/forum/dms` and `GET /v1/forum/dms?partner=UID` requiring a verified Firebase ID token. Both query sender/recipient by the authenticated UID and never expose unrelated messages.
- Frontend DM reads now call the authenticated API; unrestricted Firestore DM collection reads were removed. Direct DM **writes still use Firestore** and must be migrated before API-only rules.
- Both queries are capped at 500 documents each; implement pagination before considering this a complete DM history solution.
- Deploy the matching backend and frontend **together on staging**; do not merge this draft yet. Test existing conversations, sending, banned users, offline behavior, and token expiration.
- Existing public `users` reads and admin writes still require migration; the currently published emergency Firestore rules may disable admin functionality. Never loosen DM rules to work around a broken client.

## DM write migration progress
- Frontend DM send action now calls authenticated `POST /v1/forum/dms`; direct Firestore DM write was removed from that action.
- The endpoint currently fails closed with HTTP 503 until server-side moderation is implemented. This is intentional: DO NOT deploy this frontend to production expecting DM sends to work.
- API writes remain disabled by default via `ENABLE_WRITES=false`. Do not enable writes until server moderation, security rules, and integration tests are complete.
